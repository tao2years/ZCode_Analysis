# B1：消息重建、上下文组成与 Provider 实际输入

## 研究问题与边界

本专题解释一条消息从持久记录进入 runtime history、再投影为 Provider 请求的过程，并说明 system prompt、项目指令、Skills、Memory index、动态提醒、工具契约和 reasoning 分别位于哪一层。基线为 `872ad960de7ec172591f7e1952f7849229f94521`，仅做静态源码验证。

## 四层表示

| 层                        | 典型内容                                                               | 主要用途                          |
| ------------------------- | ---------------------------------------------------------------------- | --------------------------------- |
| 持久 Session message/part | user/assistant、tool part、compact summary、语义与可见性元数据         | 恢复、分叉、UI 时间线             |
| Runtime entry             | `ModelInputMessage` 或带 source 的 attachment entry                    | turn 内追加、替换、压缩与因果排序 |
| Provider-neutral 请求     | system/user/assistant/tool、toolCallId、reasoning block、cache control | 统一模型协议边界                  |
| AI SDK / Provider wire    | Provider 特定 system、tool-result、reasoning metadata、媒体投影        | 实际网络请求                      |

同一内容在四层可能具有不同角色和可见性。例如 compact summary 持久化为隐藏的 user message，但 Provider 可见；UI tool error 和模型读取的 `modelContent` 也可能不同。

## 1. Context 初始化与顺序

runtime 首次需要上下文时加载项目 Memory root 与 `MEMORY.md` index、发现 Skills，并由 `ContextBuilder` 生成 system messages 和 meta-user attachments。[context.ts:31-76](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts)

Builder 的主要顺序是：CLI/product prefix；自定义或默认身份；动态行为、session guidance、Memory 使用说明、环境、输出风格和 context management；Skills listing；最后是项目/用户指令、Memory index 与日期。工具说明不再复制进 system prompt，真实工具契约通过 model request 的 `tools` 字段发送。[builder.ts:39-220](../../../apps/zcode-cli/packages/core/src/context/builder.ts)

**此前遗漏的 Harness 是身份提示的一部分。**默认交互身份的 `# Harness` 明确告诉模型：普通文字会作为 Markdown 展示；工具受用户选择的权限模式约束，拒绝后应调整而非原样重试；运行中可能收到 system 管理的规则/提醒，Hook 可拦截工具且其输出应视作用户反馈；优先使用专用文件/搜索工具，独立工具调用可并行；源码引用用 `file_path:line_number`。这是**提示给模型的运行环境契约**，不是 `Harness` 类统一执行这些行为。动态工作流子代理身份复用同一块，但另有“结果交给脚本而非直接对用户说话”的契约。若配置 `customSystemPrompt`，Builder 用自定义正文替换默认身份，也跳过动态 system 段；因此不能断言每个模型请求都含 `# Harness`。[身份与 Harness](../../../apps/zcode-cli/packages/core/src/context/sections/identity.ts) · [工作流身份](../../../apps/zcode-cli/packages/core/src/context/sections/workflow-actor.ts) · [Builder 分支](../../../apps/zcode-cli/packages/core/src/context/builder.ts)

**Memory 提示也要拆成两层。**`# Memory` system 段告诉模型事实文件格式、类型、链接、写后更新 `MEMORY.md`、写前查重和冲突修正；它只在 `memoryRoot` 存在且没有被自定义 system prompt 跳过时加入。`MEMORY.md` 索引作为 meta-user request context 另行注入；具体事实正文不因这段提示自动进入模型。去重与语义冲突是提示词要求，不能冒充硬性事务保证。[Memory 提示](../../../apps/zcode-cli/packages/core/src/context/sections/memory.ts) · [索引入口](../../../apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts) · [Builder 条件](../../../apps/zcode-cli/packages/core/src/context/builder.ts)

### Harness 逐条对照：模型指令与代码边界

默认身份中的 Harness 被标记为 `stable` system 段；Builder 将 CLI prefix、稳定身份、动态 system 分成最多三条 system message，并分别标 cache marker。它与 Memory 索引（meta-user attachment）不是同一角色；进入 Provider 前仍需经历 runtime entry 投影。[身份 section](../../../apps/zcode-cli/packages/core/src/context/sections/identity.ts) · [Builder 装配](../../../apps/zcode-cli/packages/core/src/context/builder.ts)

| `# Harness` 告诉模型什么 | 对应实际边界 | 不能据提示词推断什么 |
| --- | --- | --- |
| 普通文字作为 Markdown 展示 | 这是输出格式指令，文字由产品 UI/终端消费。 | 不能用它证明所有产品表面都是终端。 |
| 工具受权限模式约束；拒绝后调整做法 | 工具执行前由权限流程判定，拒绝也可能来自配置/策略。 | 不能把每个 denied 都写成用户现场点击拒绝。[权限流程](../../../apps/zcode-cli/packages/core/src/tool/executor/permission-flow.ts) |
| 运行中可能有 system 提醒；Hook 输出当作反馈 | Hook `additionalContexts` 转为 `hook_context` runtime attachment，再由投影形成模型可见消息；它不是用户亲自发的原始消息。 | 不能把 Hook 内容等同持久真实 user query。[Hook 注入](../../../apps/zcode-cli/packages/core/src/runtime/methods/hooks.ts) |
| 独立工具调用可在一次响应里并行 | 模型可以提出多次调用；实际调度、依赖和权限由工具执行层决定。 | 不能把提示语当成“任何工具必并行”的实现保证。[调度入口](../../../apps/zcode-cli/packages/core/src/tool/executor/batch-runner.ts) |
| 引用源码使用 `file_path:line_number` | 指导模型输出可定位引用。 | 不能据此证明 UI 点击行为或每条引用都有效。 |

`# Harness` 只是默认交互身份和工作流身份复用的**一小段**。模型最终看到的约束还受动态行为、Session guidance、Memory、用户/项目指令、运行时提醒和工具 schema 影响；只分析 `identity.ts` 会漏掉这些来源。[Builder 决策](../../../apps/zcode-cli/packages/core/src/context/builder.ts) · [动态段](../../../apps/zcode-cli/packages/core/src/context/dynamic-sections.ts) · [每步提醒](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts)

组装后，stable/dynamic system sections 成为 system messages；Skills 和其他请求上下文成为带 source 的 meta-user attachments。[builder.ts:231-306](../../../apps/zcode-cli/packages/core/src/context/builder.ts) `buildContextHistoryEntries` 再把两者转换为 runtime entries。[context-history-entries.ts](../../../apps/zcode-cli/packages/core/src/runtime/methods/context-history-entries.ts)

因此“上下文”不等于单一 system string：部分指令在 system role，部分以经过标记的 user-role system reminder 进入请求，工具 schema 则独立存在。

## 2. 每个模型步骤重新组装

turn loop 每个步骤先处理待注入 runtime commands，再执行 microcompact 和完整 compact 检查，随后初始化当前模型可用工具，并按运行模式追加 plan、mode、todo、output style 等提醒。output style 是本 turn 的 Provider-visible runtime attachment，但不持久化到 Session。[turn-loop.ts:35-167](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts)

接着复制 `turnRequestState.entries` 并调用 Provider projection。projection 后才设置 latest non-system cache-control，避免 synthetic user entry 抢占缓存锚点；另行生成过滤了一次性 output-continuation 内容的 recordable messages，用于持久 ModelRequest 事件。[turn-loop.ts:168-213](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts)

真正调用模型前还按当前 context usage 计算本步最大输出 token。`messages` 和 `tools` 分开传入，流式回调分别接收 reasoning、text 和 tool call。[turn-model-step.ts:150-262](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

## 3. Runtime entry 到 Provider-neutral message

`buildProviderRequestMessages` 先保留每个投影消息的来源映射，再调整 meta attachment 与真实 user/assistant/tool 的因果位置，处理 mid-conversation system，移除 runtime metadata，最后设置 cache marker。当前 provider-neutral 层不合并相邻 user 消息；注释说明相邻 user 合并应由 Anthropic 序列化层承担。[provider-request-messages.ts:27-96](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts)

**投影的可执行顺序：**(1) 记录原 entry 与投影消息的映射；(2) 从尾向头扫描，把符合条件的 meta attachment 移到最近真实输入/assistant/tool 边界的因果位置，`goal_state_change` 和 history-continuity 来源不走普通移动；(3) 按模型是否支持中途 system 处理相应 entry；(4) attachment 包成带来源对应 wrapper 的 user 消息，**剥离给模型看的消息上的内部 metadata**；(5) 默认不合并相邻 user；(6) 仅实际请求需要时清除旧非 system cache marker，再给最后一条非 system 消息标 `ephemeral`。输出同时有 `messages`、仅供内部对应的 `sourceEntries` 和最新真实 user 的索引；后两者不发给模型。Microcompact/Auto 的判定投影设置 `applyCacheControl=false`，因此只是候选消息，后续提醒与预算仍可能改变最终请求。[投影实现](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts) · [Microcompact 投影](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [每步最终重投影](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts)

attachment entry 必须使用已知 system-reminder source，并在投影时包装为 user role 内容；`goal_state_change` 等会保持原因果位置，普通 meta attachment 可以向真实输入附近移动。[provider-request-messages.ts:98-203](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts)

`sourceEntries` 不进入模型消息，只用于 token usage 归属和诊断。这使系统可以在不污染 Provider 内容的前提下，将 Provider usage 对应回 runtime entry。

## 4. Provider 适配与 reasoning

AI SDK adapter 将 Provider-neutral message 转成 SDK 结构：system/user/assistant/tool 分别投影，tool result 强制携带 toolCallId 与 toolName；结构化媒体根据 Provider 能力变为 tool result part、后置 user media part 或文本占位。[transform.ts:38-151](../../../apps/zcode-cli/packages/adapters/src/model/transform.ts)

assistant reasoning block 会携带 Provider 特定 metadata。两类特殊清理发生在 wire 边界：空文本且无 metadata 的流式 reasoning 壳被删除；OpenAI Responses 在无 `previousResponseId`/conversation 的无状态重放中，会去掉带 stored item reference 的 reasoning，避免不受支持的引用导致请求失败。[transform.ts:178-268](../../../apps/zcode-cli/packages/adapters/src/model/transform.ts)

这说明 reasoning 不是普通 UI 文本的别名。它在 runtime history 中保留结构和 Provider metadata，适配器再依据本次 API 形态决定是否可重放。

## 5. 冷恢复如何重建

resume 先读取 Session 与当前 branch/rewind 元数据，重置 runtime-local Context 和 read state，再加载当前 `MEMORY.md`、恢复 compact timeline、从持久 messages hydration runtime history，最后恢复 mode、权限、todo 和 goal。[resume.ts:38-184](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts)

hydrator 按 active branch 与最后一个 compact boundary 选择消息。工具完成结果优先使用持久 metadata 中的 `modelContent`；未完成工具被转换为中断结果。随后新的 Provider 请求仍会经过正常 projection，而不是直接复用 UI transcript。[session-history-hydrator.ts:45-195](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts)

### 假设示例：UI 与模型输入不同

假设工具失败后 UI part 展示“读取失败（可重试）”，而 `metadata.modelContent` 保存 Provider 当时看到的结构化错误。重启后 hydrator 使用后者创建 tool message；接着 adapter 生成 `error-text` tool-result。UI 字符串没有被直接塞进请求。该示例是源码语义推演，不是运行验证。

## 状态变化与缓存边界

- Context prefix 可以在模型变化或动态源变化时重建；canonical history 保留源信息，Provider 请求是每步投影产物。
- cache-control 只留在最新合适的非 system message；一次性 continuation 要从持久 ModelRequest 轨迹过滤。
- Provider media budget 在 provider-clean messages 上应用；UI 可见附件不保证以原形进入模型。
- Context token 既可能使用本地估算，也可能以最近已提交 assistant 的 Provider usage 为基线加增量，避免每次完全依赖粗略估算。[compact.ts:310-350](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts)

**媒体预算也有确定算法，不是“旧媒体随机裁剪”：**先按模型 input format 做能力投影，再按 data URL 的 UTF-8 字节给当前请求全部 image/video/无 text 的 file 计量，默认总上限 40 MiB。最新**真实** user 消息中的媒体优先保护；它们独自超限就报附件过大，不静默移除。其余历史媒体按消息索引、块索引从新到旧遍历，能完整放进剩余额度才保留，放不下就跳过并把该块换成文字说明；不会切一半 data URL。假设保护媒体 20 MiB，历史最近 10 MiB、再早 25 MiB：40 MiB 内保留 20+10，跳过 25；不是保留 25 的前 10 MiB，也不回头重排已选结果。原 runtime entry、持久附件和 UI 不因本次请求投影被删除。[媒体预算](../../../apps/zcode-cli/packages/core/src/runtime/helpers/media-budget.ts)

## 设计取舍与迁移条件

以下为研究者推断：多层表示增加了投影和恢复代码，但让 UI、持久性和 Provider 兼容性可以独立演进。迁移时应把“模型实际输入”设为可观测的一等边界，并保留 source/visibility 元数据；否则压缩、缓存、reasoning 重放和工具错误恢复会互相污染。若只存最终字符串，冷恢复很难精确重建 tool/reasoning 协议。

## 验证与未知

- 已静态追踪 ContextBuilder、turn request assembly、Provider-neutral projection、AI SDK transform 和 cold hydration。
- 未捕获真实 Provider HTTP payload，故不能宣称所有 Provider 最终 wire 形态已运行验证。
- Provider SDK 内部对相邻 user、cache-control 和 reasoning 的最终编码仍是本批有边界未知。
