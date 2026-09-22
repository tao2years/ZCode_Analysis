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
