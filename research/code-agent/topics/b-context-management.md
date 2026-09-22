# B5：上下文管理的完整模型

## 从用户输入读到完整 Runtime 循环：源码入口

这条主线**能在源码里连续追踪，但不是由一个 `SessionManager` 函数包办**。按下面顺序阅读；前五处负责「这一条输入怎样开始」，最后两处负责「一个 turn 怎样反复请求模型、执行工具并结束」。这里以桌面/Web 使用的 V4 `sendText` 为入口；CLI/TUI 等其他入口会在 `app.sendInput` 或 Core Runtime 汇合，不能把 V4 协议入口当成所有输入的唯一入口。

| 顺序 | 源码入口 | 它实际负责什么 |
| --- | --- | --- |
| 1 | [V4 `sendText`](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/commands/handlers/session-flow.ts)（184 行） | 校验正文/附件，处理 held queue 与立即发送，再调用 `startPromptTurn`。 |
| 2 | [`startPromptTurn`](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/commands/prompt-turn.ts)（57 行） | 检查恢复告警与模型、设定本轮限制，调用 `record.app.sendInput`；接纳回执先返回，后台等待本轮 completion 并清理归因。 |
| 3 | [App `sendInput`](../../../apps/zcode-cli/packages/bootstrap/src/app/input-facade.ts)（193 行） | 准备输入与附件，再交给 `runtime.admitPrompt`。 |
| 4 | [Core `admitPrompt`](../../../apps/zcode-cli/packages/core/src/runtime/methods/prompt-admission.ts)（20 行） | 在同一 Runtime 判断忙碌、转为 steer/queue 或为新 turn 建 reservation 并入命令队列。排队的输入此时**尚未执行**下面的 turn 主链。 |
| 5 | [`runRuntimeCommand`](../../../apps/zcode-cli/packages/core/src/runtime/methods/runtime-command-queue.ts)（187 行） | FIFO 消费 prompt 命令，调用 `executeTurnCommand`。 |
| 6 | [`executeTurnCommand`](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts)（93 行） | **单轮总控**：冻结模型选择、初始化 Context、运行 hooks、记录输入到运行时历史与持久 Session、建立本轮状态，调用 `runRegularTurnLoop`，最后结算并发出终态。手动 `/compact` 和 rewind 在这里走专门分支。 |
| 7 | [`runRegularTurnLoop`](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts)（43 行） | **每个 model step 的循环总控**：drain 待处理输入 → microcompact → auto compact → 准备工具/提醒 → 投影 Provider 消息 → `runModelBackedTurnStep`；该步骤要求继续时再次循环。 |
| 8 | [`runModelBackedTurnStep`](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)（88 行）、[`executeToolCallsForModelStep`](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts)（43 行） | 发起/消费模型响应、处理上下文超窗的 reactive compact、输出续写与工具调用；工具结果入本轮历史后由第 7 步开始下一次 model step。 |

Session 的**冷恢复**是新输入前可能发生的另一条入口：协议层重新激活 session record，Core 的 [`resumeFromStore`](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts)（59 行）从持久记录重建有效历史。它不替代上面的单轮执行器。若只想抓核心控制流，先读第 **6 → 7 → 8** 步，再向上补第 **1 → 5** 步。

## 能力是否真正接入：静态核查清单

“源码里有”不等于“这个产品入口默认运行”。下表区分 Builder/Runtime 实现、产品配置与已确认的调用；没有实际启动目标 Agent，不能据此宣称某台设备的实时设置。

| 能力 | 当前源码可确认的接入状态 | 对阅读主流程的影响 |
| --- | --- | --- |
| 默认身份中的 `# Harness`、Context management、Memory 使用说明 | 默认交互身份注入 Harness；Memory 说明还要求有效 memory root。`customSystemPrompt` 会替换默认身份并跳过动态 system 段；工作流子代理另有身份契约但复用 Harness。 | 不能把提示词当运行时强制逻辑，也不能说它们每轮无条件存在。[Builder](../../../apps/zcode-cli/packages/core/src/context/builder.ts) · [Harness](../../../apps/zcode-cli/packages/core/src/context/sections/identity.ts) |
| Project Memory | CLI 基础配置 `features.memory=true`、`memory.use=true`；桌面/Web 的产品 `memoryEnabled` 设置默认 `false`，Session 创建/恢复时关闭值会覆盖 CLI 默认。真正加载仍需有效存储根、合格 task type；后台提取还要求本地 workspace、store/file port 和未关闭 extraction。 | 同一仓库有实现，不代表当前桌面会话已启用；即便启用，也只自动加载索引，正文按需读。[CLI 默认](../../../apps/zcode-cli/packages/contracts/src/config/index.ts) · [产品默认](../../../packages/shared/src/validationAppSettings.ts) · [覆盖](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server-operations.ts) · [运行条件](../../../apps/zcode-cli/packages/core/src/runtime/helpers/project-memory.ts) |
| Microcompact | 每次 model step 都调用检查函数，但它要求 `compact.microcompact.enabled === true`；当前已追的常规 runtime 配置组装未设置此字段，故**默认不执行清理**，可由显式内部 runtime 配置开启。 | 不能把上文 02 的裁剪算法写成默认实际发生。[循环调用](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) · [启用门](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts) · [配置组装](../../../apps/zcode-cli/packages/bootstrap/src/app/runtime-config.ts) |
| Auto / Reactive compact | 完整 compact 在 CLI 基础配置默认允许；Runtime 循环有 auto 判断，Provider 超窗有 reactive 分支，均受 `compact.enabled` 和各自前置条件约束。 | 与默认关闭的 microcompact 不同；“允许”仍不代表某次输入一定压缩。[默认](../../../apps/zcode-cli/packages/contracts/src/config/index.ts) · [自动入口](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) · [Reactive](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) |
| `SessionMemory` compact trigger | 契约枚举与 phase/reason 映射存在；在已检索的 CLI、Core、共享及 UI 调用路径中，未找到把它传给压缩执行器的生产调用点。 | 标为**未确认接入**，不能推断有独立 Session-memory 自动压缩。[枚举](../../../apps/zcode-cli/packages/contracts/src/compact/index.ts) · [映射](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) |
| `legacy` 上下文预算策略 | 协议仍接受 `legacy`，但 Runtime 构造时归一为共享默认 `preflight-v1`；输出上限计算的 legacy 参数只保留兼容。 | 不应把两个字符串描述为当前可切换的两套算法。[协议](../../../packages/shared/src/zcode-protocol/index.ts) · [Runtime 归一](../../../apps/zcode-cli/packages/core/src/runtime/agent-runtime.ts) · [预算](../../../apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts) |
| 提示词/兼容接口中尚未接入的分支 | `ContextBuilder.setToolRegistry()` 目前是 no-op；Agent/AskUserQuestion 的 Session guidance 文本在源码中被注释；`buildCompactSummaryMessage` 可接受 transcript 路径、最近消息保留、REPL 清空标记，但当前 compact 调用只传 `suppressFollowup`。 | 这些只能列为兼容/预留形态，不能写成当前模型必见的提示。[Builder](../../../apps/zcode-cli/packages/core/src/context/builder.ts) · [Guidance](../../../apps/zcode-cli/packages/core/src/context/dynamic-sections.ts) · [摘要构造与调用](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) |

这份清单只核查本批上下文主链与产品配置，不代表全仓所有灰度功能已审完。对其他能力仍按“声明 → 配置默认与覆盖 → 运行时门 → 生产调用点 → Provider 实际输入”追踪，缺任一环就保留状态边界。

## 一条消息经过上下文系统的路径

**阅读方向：从上往下，按 00 → 08。** `02` 到 `08` 是一个 **model step**；工具结果、输出续写或 Reactive compact 成功会回到 `02`，同一用户输入可能经过多次。手动 `/compact` 是独立命令入口，使用后文的完整摘要链路，不经过本图的自动阈值判断。基线：`872ad960de7ec172591f7e1952f7849229f94521`。

```mermaid
flowchart TD
    S00["00 新消息进入并完成准入"] --> S01["01 恢复有效历史；装配 Context 与本轮输入"]
    S01 --> S02["02 可选清理旧工具结果：microcompact"]
    S02 --> S03["03 判断是否完整压缩：auto compact"]
    S03 --> S04["04 加入提醒和工具定义；构造请求消息"]
    S04 --> S05["05 估算输入 token；确定本步输出上限"]
    S05 --> S06["06 处理媒体附件；调用 Provider"]
    S06 --> S07{"07 请求结果是什么？"}
    S07 -- 正常结束 --> DONE["本轮完成"]
    S07 -- 工具结果或输出续写 --> S02
    S07 -- Provider 判定上下文超窗 --> S08["08 尝试响应式完整压缩"]
    S08 -- 摘要成功，重建本步请求 --> S02
    S08 -- 无法压缩或再次超窗 --> ERROR["结束并报告错误"]
```

**分支读法：**先按编号往下；遇到下表的条件，执行该行「动作」，再按「下一站」继续。`T` 指 auto compact 的输入 token 阈值，不是所有资源共用的阈值。

| 步骤 | 判断标准 | 动作与下一站 |
| --- | --- | --- |
| 01 | Runtime 是否需要冷恢复；本轮输入是否引用 `#sess_*` | 若需恢复，先从 Session 消息与 compact boundary 重建有效历史；随后加入新输入。Skills metadata 与 Memory 索引属于运行时上下文初始化。引用其他 Session 只附加提示，不自动导入其 transcript；若模型需要，会在 07b 调用 `ReadSessionContext`。→ 02 |
| 02 | `compact.microcompact.enabled === true`，且空闲超过默认 60 分钟或估算 token 达到 microcompact 阈值。默认阈值为 `max(0, min(floor(T × 0.9), T - 2K))`；另须有合格旧工具结果、保留最近默认 5 组、预计至少节省 256 token | 替换符合条件的旧工具结果；否则原样继续。→ 03 |
| 03 | auto compact 启用、有可摘要历史、连续失败未达默认 3 次，且 `tokenCount >= T`。默认 `T = max(0, contextWindow - min(maxOutputTokens, 21K) - 13K)`；窗口未知按 200K | 达标时选历史、摘要、持久化 boundary 并替换当前历史。成功或非取消失败都继续 → 04；rapid-refill breaker 触发则报错，取消则结束。 |
| 05 | `contextWindow - estimatedInput - 1K > 0` | 输出上限取模型上限与该剩余值的较小值；否则保留模型上限，交由 Provider/后续恢复路径判定。→ 06 |
| 06 | 媒体编码字节总量是否超过默认 40 MiB | 未超限 → Provider；超限但本轮真实用户媒体可保留 → 旧媒体按最近优先保留，其余变占位文本；仅本轮媒体已超限 → 附件错误，不进入 reactive compact。 |
| 08 | Provider 抛出 context-exceeded；或在无 tool call、无输出续写的分支返回相应 finish reason | 同一 model step 最多尝试一次 reactive compact；成功后回到 **02** 重建请求；失败、跳过则保留原错误，取消则结束。详细流程见下文 **08**。 |
| 07b | Provider 返回工具调用 | 各工具先处理原始输出，通用结果序列化再按各工具预算截断/落盘；结果提交到请求历史后回到 **02**。已追踪的批量汇合代码没有统一的「并发总结果 token 闸门」。 |
| 07c | Provider 达到输出 token 上限且允许续写 | 追加一次性续写输入，回到 **02**；若无法继续则返回输出上限错误。 |
| 07d | 没有待执行工具、续写或恢复分支 | 正常结束本轮。 |

[turn.ts:477-614](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts) [turn-loop.ts:47-213](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) [runtime microcompact.ts:24-115](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts) [compact/microcompact.ts:77-195](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) [policy.ts:67-155](../../../apps/zcode-cli/packages/core/src/compact/policy.ts) [model-token-limits.ts:17-40](../../../apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts) [media-budget.ts:75-150](../../../apps/zcode-cli/packages/core/src/runtime/helpers/media-budget.ts) [turn-model-step.ts:647-664](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-tools.ts:235-269](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts)

这里把分支标准放在编号旁，后文再分别展开内部流程。特别注意：auto compact 在 **03**、普通请求输出预算在 **05**、媒体字节预算在 **06**、Provider 拒绝后才可能发生的 reactive compact 在 **08**，四者的触发来源和失败策略不同。完整摘要的共享实现与手动入口见 [B2](b-compaction.md)，工具结果细节见 [B4](b-large-tool-results.md)。阅读时还须区分持久 Session 消息、运行时历史、当前 request entries、最终 Provider 请求和 UI 展示。

## 00–01. 一条新输入从哪里取得历史：冷恢复与 Context 初始化

这里的**冷恢复**有明确的运行时含义：协议层的 resident registry 里已没有这个 Session 的可执行 runtime，但 Session Store 仍有持久化记录。再次访问该 session 时，`activateSessionForResume` 先等待可能正在进行的去激活；若 registry 已有 record，直接复用；否则读取持久 Session，创建 record，调用 `app.resume()` / `resumeFromStore`。显式 `session/resume` 会走这条路径，V4 冷订阅也能先激活 runtime，再做自己的持久投影。进程重启后再次打开持久会话也属于同一类。它**不是** Provider prompt cache 过期，也不是每次新消息都重新读全库。[server-operations.ts:1402-1522](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server-operations.ts)

为什么 runtime 会不常驻？常驻池默认目标 8、最高水位 16，候选连续空闲默认 10 分钟可因 idle timeout 去激活；超过高水位时可按 LRU 向目标数回收。只有已持久化、没有运行中工作/排队命令/交互/订阅/operation lease 的 Session 才可回收。去激活删除 resident record、关闭 app 并清理内存 event store，**不删除**持久 Session。故“过了 10 分钟就必定冷恢复”也不准确：必须先满足回收资格且确实发生去激活。[session-resident-pool.ts:7-11](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/session-resident-pool.ts) [session-resident-pool.ts:147-215](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/session-resident-pool.ts) [session-resident-pool.ts:226-260](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/session-resident-pool.ts) [session-residency.ts:45-67](../../../apps/zcode-cli/packages/bootstrap/src/zcode-protocol/session-residency.ts)

恢复不是把数据库消息原样拼成一个字符串。`resumeFromStore` 读取 Session 与消息，恢复工作目录、任务类型、workspace identity、环境和 shell 选择；重置旧 Runtime `MessageHistory` 与 ContextBuilder，先从持久记录 hydrate read-file state，再重新解析上下文来源、发现 Skills、加载当前 `MEMORY.md` 索引。接着修复未完成的 compact timeline，再根据分支/rewind 和最后有效 compact boundary 选择模型有效历史并 hydrate。assistant/tool parts 会还原成模型消息；未完成的工具调用变为 *interrupted* 工具结果，不会擅自重放工具。started/retrying 的 compact timeline 若已有 boundary 则收敛为 completed，否则收敛为 interrupted。最后恢复 mode/执行状态、权限、Todo、目标状态，并运行 `SessionStart(source="resume")` hook；目标状态另作为运行时提醒注入。[resume.ts:91-179](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) [resume.ts:191-262](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) [session-history-hydrator.ts:54-199](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts) [compact-persistence.ts:200-287](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-persistence.ts)

Context 初始化从 `ContextSourcePort` 得到指令/环境快照，并加载 Skill catalog、Memory root 和索引；随后 ContextBuilder 生成 system 与 meta-user 前缀。Memory 正文没有在每条消息开始时做语义检索：自动进入前缀的是索引（最多 200 行、25,000 字符），详情靠模型后续 Read/Grep/Glob。已运行的同一 runtime 复用 `memoryIndexContent` 快照；重新初始化/冷恢复才自然重新读取磁盘索引。每个新 turn 可从已有快照重建模型相关前缀，但不是重新扫描 Memory 文件。[context.ts:35-73](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts) [context.ts:168-195](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts) [request-user-context.ts:47-76](../../../apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts) [index-content.ts:13-37](../../../apps/zcode-cli/packages/core/src/memory/index-content.ts) [context-refresh.ts:7-54](../../../apps/zcode-cli/packages/core/src/runtime/methods/context-refresh.ts)

**Context 前缀也有条件分支。**默认身份、动态行为、环境和 Context 管理说明进入 system 侧；配置了输出风格或 Memory 后，对应说明也进入 system 侧；可用 Skill 列表、仓库/用户指令与 Memory 索引进入 meta-user 侧。显式 `customSystemPrompt` 会替换默认 system 体系并跳过上述动态 system 段，不能据“默认会注入”推断每种任务都看到它们；Workflow 子代理又有另一套身份与指导选择。Skill 列表只有 `Skill` 工具在当前运行时可用时才加入。模型/输出风格等配置变化可从已有快照重建前缀，不能误写成每次重读 Memory 磁盘文件。[ContextBuilder 分支](../../../apps/zcode-cli/packages/core/src/context/builder.ts) [前缀重建](../../../apps/zcode-cli/packages/core/src/runtime/methods/context-refresh.ts) [Skills 生成](../../../apps/zcode-cli/packages/core/src/context/sections/skills.ts)

**这里有两种容易混称的“Session memory”。**其一是当前 Session 的持久消息：它供冷恢复和完整 compact 的有效历史选择使用，不是每轮自动向模型灌入全部旧 transcript。其二是**跨 Session 按需读取**：真实用户输入出现 `#sess_*` 时，只附加一个提醒，明确旧会话不会自动展开；模型确需背景时，调用 `ReadSessionContext(sessionId, query, strategy)`。该工具读取另一会话的持久 Session 消息、先按有效分支/compact boundary 选取可读内容，再按 query 与字符预算选片段；有模型时可用单独的轻量提取请求，失败则回退本地选段。工具结果作为当前会话的一次 tool result 进入后续上下文，不等于本会话 `/resume`，也不是自动语义检索全部历史。[references.ts:5-27](../../../apps/zcode-cli/packages/core/src/session-context/references.ts) [turn.ts:863-872](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts) [read-session-context.ts:38-143](../../../apps/zcode-cli/packages/core/src/tool/handlers/read-session-context.ts) [session-context/read-session-context.ts:54-100](../../../apps/zcode-cli/packages/core/src/session-context/read-session-context.ts)

第三种是**Project Memory**：启用时跨会话共享同一 workspace identity 下的事实文件，初始化自动加载 `MEMORY.md` 索引，详情按需 Read；成功 turn 后可从 durable Session 快照后台提取更新。Subagent profile 还有独立 persistent memory 作用域。它们都不同于 `CompactTrigger.SessionMemory` 这个契约枚举：当前核心运行时检索到它用于 compact phase/reason 的分支映射，未找到发起该 trigger 的调用，故不能据枚举断言存在已启用的 Session-memory compact 流程。[project-memory.ts:5-28](../../../apps/zcode-cli/packages/core/src/runtime/helpers/project-memory.ts) [project-memory-extraction.ts:32-80](../../../apps/zcode-cli/packages/core/src/runtime/helpers/project-memory-extraction.ts) [runtime compact.ts:40-63](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) [subagent/persistent-memory.ts:70-100](../../../apps/zcode-cli/packages/core/src/subagent/persistent-memory.ts)；项目记忆详见 [B3](b-memory.md)。

新用户输入随后被写入运行时历史与 Session；当前 turn 从 `MessageHistory` 复制 `request entries`。这解释了为何后面的压缩要同时处理 canonical runtime history 和本轮请求副本。持久消息、运行时 entries、Provider 投影、UI 行仍是不同表示。[turn.ts:477-529](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts) [turn.ts:595-614](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts)

## 02. Microcompact：只清旧工具结果，不生成摘要

**可复现的核心规则：不截取单条结果的前/后片段，也不按 Token 预算裁中间。选中一条 tool result，就把它的整个 `content` 替换为 33 字符的 `[Old tool result content cleared]`；未选中的结果保持原样。Token 数只控制是否尝试以及总替换是否值得提交。**

这 9 个工具共享**同一条结果级规则**，没有 `Read` 留文件头、`Bash` 留日志尾之类的专用算法。被替换的是模型对话中那条旧结果的 content，不是被读文件、工作区文件、命令输出落盘文件、工具调用参数或持久 Session 原记录。若一次批次内仅部分结果符合条件，只替换符合条件的那些结果。[候选规则](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) · [回写边界](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts)

**每次准备调用模型前，都经过下面的检查。** 从上往下读；图中数字是默认配置。清理门槛怎么算，紧接在图下解释。

```mermaid
flowchart TD
    M0{"① 已开启微压缩<br/>且总压缩开关未关闭？"} -- 否 --> MS["保持原消息"]
    M0 -- 是 --> M1["② 整理下一次要发给模型的消息<br/>估算它们占多少 token"]
    M1 --> M2{"③ 距上次助手完成回复超过 60 分钟<br/>或者消息已达到清理门槛？"}
    M2 -- 否 --> MS
    M2 -- 是 --> M3["④ 从旧到新找可清理的工具结果<br/>同一次助手调用的一批工具算一组<br/>错误、已清理结果和图片等媒体跳过"]
    M3 --> M4{"⑤ 除了保留最新 5 组合格结果<br/>还有更旧的合格组吗？"}
    M4 -- 否 --> MS
    M4 -- 是 --> M5["⑥ 将更旧组的合格结果全文<br/>暂换为一行“旧工具结果已清理”<br/>不留开头、结尾或预览"]
    M5 --> M6{"⑦ 重新估算<br/>这次至少省下 256 token 吗？"}
    M6 -- 否，撤销本次替换 --> MS
    M6 -- 是 --> MC["采用清理后的消息"]
    MS --> NEXT["继续检查是否需要完整摘要压缩"]
    MC --> NEXT
```

**③ 清理门槛怎么算？** 先从模型的上下文容量中扣出回复预留空间（最多 21,000 token），再扣 13,000 token 缓冲，得到“完整摘要压缩门槛”。微压缩会更早介入：取该门槛的 **90%** 和“该门槛减 2,000”中较小的值。例如上下文容量为 200,000、模型允许回复 32,000 token 时：完整压缩门槛为 `200,000 − 21,000 − 13,000 = 166,000`；微压缩门槛为 `149,400`。这是按默认规则举例，模型属性或配置改变后须重新计算。

**④ 哪些工具可清？** 默认只选 Read、Bash、Grep、Glob、WebFetch、WebSearch、Edit、Write、ApplyPatch 的结果。组内有不合格结果时，仅跳过那些结果；并不连带跳过整组。“保留 5 组”只数含合格结果的组，不是最近 5 条消息。

**⑥ 替换成什么？** 图中用中文表达含义，源码实际写入的固定文本是 `[Old tool result content cleared]`。工具调用参数、调用 ID 和结果顺序保留，工具结果正文整条替换。下面再给精确参数和源码对应关系，便于实现时核对。

| 决策点 | 实际输入与计算 | 容易误解的边界 |
| --- | --- | --- |
| ② 整理待发送消息 | `buildProviderRequestMessages(entries, applyCacheControl: false)` 把 runtime attachment 包成模型消息，处理其顺序及会话中途 system 的表示，剥离内部 metadata；**尚未调用 Provider**。Microcompact 以这份消息判断“模型会看到什么”，再凭 `toolCallId` 回写对应 runtime entry。 | 不是读取 SQLite/UI 文本，也不是最终 wire payload；工具 schema 和之后的媒体预算不在本次估算中。 |
| ②–③ 估算与触发 | 对每条投影消息：`ceil((内容文本长度 + tool-call 名称及 JSON 入参长度) / 3)`，逐条求和；reasoning 文本计入，image/video 等按文字占位估，**不用 Provider usage**。`T = max(0, W − min(R, 21,000) − B)`；默认 `W=200,000`、`R=模型声明的最大输出或 32,000`、`B=13,000`。`M` 默认 `max(0, min(floor(0.9×T), T−2,000))`，可由 `microcompact.thresholdTokens` 覆盖。 | 时间阈值是距**上次 assistant 完成**严格大于默认 60 分钟；没有完成时间就不能走时间分支。时间条件先判断，同时满足时标记为 time-based。配置可覆盖 idle 分钟数。 |
| ④–⑦ 选择、替换与收益 | 默认候选工具：Read、Bash、Grep、Glob、WebFetch、WebSearch、Edit、Write、ApplyPatch；默认排除错误、已清过、含 image/video/file 的结果。每遇到一次 assistant tool-call 批次建立一组，同批多个合格结果同组；仅统计**合格组**。保留最新默认 5 组（配置可覆盖，但最少 1 组），清更旧组中全部合格结果。用替换前后同一估算器算差额；小于默认 256 token 则撤销整个清理。 | 不是删最早 5 条消息，也不是按字节截一段；单个不合格结果不使同批其他合格结果失去资格。 |

**实现级伪代码（保持源码判定顺序）：**

```text
每次 model step 前：
  若 compact.enabled=false：退出
  messages = project(runtimeEntries, applyCacheControl=false)
  before = sum_per_message(ceil((contentTextChars + toolCallNameAndJsonChars)/3))
  T = max(0, (W - min(R, 21000)) - B)
  M = 配置的 thresholdTokens，否则 max(0, min(floor(T*0.9), T-2000))
  若 micro 未显式开启：退出
  若 now-lastAssistantCompletedAt <= idleMinutes*60000 且 before < M：退出
  顺序扫描 messages：每遇 assistant(toolCalls 非空) 开新批次；
    收集后续 role=tool、工具名在白名单、带 toolCallId、非错误、
    未带清理标记、content 无 image/video/file 的结果；空批次不计数
  groups = 所有非空合格批次（孤立的合格 tool result 各自成组）
  k = max(1, 配置的 keepRecentToolResults，否则 5)
  若 groups.length <= k：退出
  对 groups[0 : groups.length-k] 内每条合格结果：整个 content 换为固定标记
  after = 用同一估算器对替换后 messages 求和
  若 max(0,before-after) < 配置的 minTokenSavings（否则 256）：撤销并退出
  按 toolCallId 把替换写入 runtime MessageHistory 与本轮 request entries；发边界事件
```

**可复算例子（假设输入，非运行记录）：**`W=200,000`、`R=32,000`、`B=13,000`，故 `T=166,000`、`M=149,400`。投影总估算 `120,000`，但距上次 assistant 完成已过 61 分钟，因时间条件仍进入扫描。合格批次 G1…G7 中，G1 有 Read、Bash 两条结果，G2 有 Grep 一条；它们各为 900 字符纯文本，其余消息合计估算 `119,100`。保留 G3…G7，清 G1/G2 三条：每条从 `ceil(900/3)=300` 变为 `ceil(33/3)=11` token，合计节省 `3×(300−11)=867`，结果估算 `119,133`；`867≥256`，提交。若三条合计只省 200 token，则整次撤销，没有“先清一点”的退让。下一步已清的 G1/G2 不再入候选；若出现 G8 且仍触发，剩余合格组 G3…G8 中才轮到 G3。[投影](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [估算器](../../../apps/zcode-cli/packages/core/src/compact/manual.ts) · [候选与替换](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) · [阈值构造](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts)

**Prompt Cache 的影响：**Microcompact 判断用的投影指定 `applyCacheControl: false`，它本身既不发模型请求，也不操作 Provider 缓存；若跳过或因收益不足撤销，缓存输入不因它改变。若真正清掉 G1/G2，下一次发给模型的消息在首条被清结果处就与旧请求不同，因此跨过该位置的**完全相同前缀缓存不能原样复用**；在它之前未变的前缀仍可能复用。最终请求会在重新投影后设置新的 ephemeral cache marker，适配器再映射到 Provider；源码不能保证某个 Provider 实际命中多少、缓存何时过期或是否按此粒度计费。因此代价是一次可能的缓存命中下降，收益是旧工具正文更少占用模型上下文；这里没有看到“先比较缓存成本再决定是否 microcompact”的门槛。[Micro 投影](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [最终请求及 marker 时机](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) · [marker 位置](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts) · [Provider 映射](../../../apps/zcode-cli/packages/adapters/src/model/transform.ts)

**采用清理结果时，只改两份活跃表示：**运行时 canonical `MessageHistory` 与本轮 `request entries`。被选结果的 content 变为 `[Old tool result content cleared]`；tool-call ID、顺序和配对保留。默认 `MicrocompactBoundary` 进入内存 eventStore；原 Session tool part 不回写，UI transcript 不缩短。故同一 Runtime 的后续模型步骤继续看到占位符，但冷恢复从持久 tool part 重建时不会回放这次清理。[runtime microcompact.ts:73-102](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts) [turn-tools.ts:286-345](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts) [events.ts:263-280](../../../apps/zcode-cli/packages/core/src/runtime/methods/events.ts)

| 冷恢复时的来源 | 原工具结果还在吗？ | 会不会直接回到模型？ |
| --- | --- | --- |
| SQLite Session tool part | Microcompact 不改原 `output`/持久媒体。 | hydrator 可重建；但若后来有**完整 compact boundary**，仍要先按 boundary 选择有效历史。 |
| `model-io-<session>.jsonl` 诊断 | 可能记过清理前或清理后的请求；默认会缩减、限额、轮转、脱敏。显式全量保留模式另有规则。 | **不会作为冷恢复源**，不能凭 JSONL 曾含原文保证可恢复。 |

若之后的完整 compact 摘要基于已清理的运行时历史生成，Microcompact 还可能**间接影响持久化摘要**，尽管它没有修改原 tool part。[冷恢复](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) · [tool part hydration](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts) · [SQLite store](../../../apps/zcode-cli/packages/adapters/src/storage/session-store.ts) · [诊断 JSONL](../../../apps/zcode-cli/packages/adapters/src/model/runner-debug.ts)

**退出再恢复，会不会把旧全文又塞进模型？** 会有“恢复后重新膨胀”的可能，但恢复动作本身还没有向 Provider 发请求。按实际顺序看：

1. 退出前仅发生 Microcompact、没有后续完整 compact boundary 时，SQLite 保留旧 tool part 原文；`resumeFromStore` 按有效历史重新 hydrate，旧结果回到新的 `MessageHistory`。它也从最近有效 assistant 的持久完成时间恢复 `lastAssistantCompletedAtMs`；并不重放旧的 MicrocompactBoundary。若已经成功完整 compact，冷恢复会先按该 boundary 选摘要和保留段，旧消息虽然仍在库中，却不全回到有效历史。[resume.ts:152-189](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) [hydrator.ts:54-199](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts) [compact-session.ts:4-46](../../../apps/zcode-cli/packages/core/src/agent/compact-session.ts)
2. 下一条用户输入进入第一个 model step，**先**用重建的完整 entries 再跑图中 ①–⑦，**后**跑 Auto compact。如果 Microcompact 已启用，距上次 assistant 完成超过 60 分钟，或重新投影估算 `E >= M`，它会再次扫描并清理；未启用、两个触发均未满足、合格旧组不够或节省不足 256 时，旧结果保持原样。故不能说“恢复时自动沿用上次 Microcompact 的节省”。[turn-loop.ts:67-103](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) [microcompact.ts:24-115](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts)
3. Auto compact 接着按 `T` 判断是否另发摘要请求。它优先取**最近已提交 assistant 的 Provider usage 基线 + 后续消息本地估算**，而非无条件重算恢复后全文。特别是上次 Microcompact 清理后又发过模型请求时，该 usage 可反映“已清理的上下文”，冷恢复却重新引入更早 tool part 全文；若旧内容位于 usage 基线之前，Auto 的增量计算不会补回它们，**存在低估、未提前压缩的条件性风险**。这是源码推论，尚未用真实 Provider 验证。[compact.ts:317-350](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) [usage.ts:54-65](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step-usage.ts)
4. 普通请求前的 output preflight 用同一 `estimateCurrentModelInputTokens` 估算输入，并把输出上限裁到 `min(R, floor(W-estimatedInput-1000))`；若算得非正，反而保留原输出上限，交给后续错误恢复，**不是阻断超窗输入的硬闸门**。若 Provider 报 context-exceeded，或返回相应 finish reason，Reactive compact 在同一 model step 最多尝试一次完整摘要并重试；摘要失败、历史不足、功能关闭或保护条件阻断时，原超窗错误仍可能抛出。因此“退出再进来一定不会撑爆”并无源码保证。[model-token-limits.ts:17-40](../../../apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts) [turn-model-step.ts:239-252](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-model-step.ts:375-439](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-model-step.ts:499-524](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

这里的“撑爆上下文”是下一次 Provider 输入超窗；冷恢复重建旧消息也有本地内存成本，但源码路径没有据此给出可量化内存上限。Microcompact 不保证清理后低于 Auto compact 阈值；清理后下一站仍是 03。

## 03. Auto compact：判定、摘要、替换和冷重建

**阅读方向：A1 → A6；跳过或失败回到总流程 04。**与微压缩不同，Auto compact 的触发值优先使用最近一次已提交的 Provider usage。

```mermaid
flowchart TD
    A1["A1 将当前 entries 投影成模型消息"] --> A2["A2 取最近 Provider usage 基线；估算新增消息"]
    A2 --> A3{"A3 启用、历史足够、未触发失败熔断，且 token 达到 T？"}
    A3 -- 否 --> AS["跳过；原历史进入 04"]
    A3 -- 是 --> A4["A4 分开 Context 前缀；旧组待摘要，最新组保留原文"]
    A4 --> A5["A5 独立模型请求：旧组 + 压缩 Prompt"]
    A5 --> A6{"A6 摘要有效？"}
    A6 -- 是 --> AC["持久化 summary/boundary；替换运行时历史；进入 04"]
    A6 -- 否 --> AF["按错误类型重选/重试；最终失败保留原历史"]
```

| 步骤 | 具体规则 | 输出或边界 |
| --- | --- | --- |
| A1–A3：判断 | 投影方式与微压缩的步骤 ② 类似。找到最近已提交 assistant 的有效 usage：若它已覆盖 assistant 输出，用 `contextUsageTokens + 后续消息本地估算`；否则用 `inputTokens + 从该 assistant 起的本地估算`。没有有效 usage 才对全部投影消息按微压缩的字符估算。`T = max(0, W − min(R,21K) − B)`，默认 `W=200K`、`R=模型最大输出或32K`、`B=13K`；`tokenCount >= T` 才可压缩。 | `compact.enabled=false`、正文分组少于 2 组**或**没有 assistant、连续失败达到默认 3 次、未到 T 均跳过。达到 T 后还检查 rapid-refill breaker：若距上次 compact 不足 3 个已完成工具批次且连续快速再触发达到 3 次，就报错。Microcompact 的“空闲 60 分钟”不参与 A3。 |
| A4：选择 | 从 active entries 复制快照，单独取出 system/Context 前缀；剩余历史按**新 assistant 消息开始**划组。Auto 默认保留最新 1 组逐条原文，其余组用于摘要；至少还要有 1 组可摘要。 | 组不是固定“一条 user + 一条 assistant”；tool result、后续 user/attachment 可能同属一组。前缀只供摘要模型理解，不算作被摘要的会话正文。 |
| A5：生成 | 将压缩 Prompt 作为最后一条 user 消息接在选中历史后，随后做媒体能力与字节预算投影。Prompt 要求按时间保留用户要求、技术细节、错误、反馈及待办，以 `<analysis>`/`<summary>` 返回，禁止工具调用；模型输出最多 20K，工具数超过 100 时请求不传工具。Auto 无自定义指令，手动 `/compact` 可追加。 | 这是**另一笔模型调用**，不是本轮普通请求就地改写。模型真的返回 tool call 或摘要无效会失败。完整 Prompt 见 [`compact/prompt.ts`](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts)。 |
| A6：提交 | 去掉 `<analysis>`，整理 `<summary>`，生成一条对 UI 隐藏、对模型可见的 synthetic user summary；持久化 summary 和 compact boundary，再把运行时历史换成 `当前 Context 前缀 + summary + 最后保留组原文 + 后置提醒`。 | 旧 Session 消息不物理删除，但后续模型不自动再看到被摘要组全文；冷恢复按 boundary 复原有效历史。 |

**分组实例（假设序列）：**`前缀, U1, A1(tool), T1, U2, A2(tool), T2, U3, A3` 会切成 `G1=[U1]`、`G2=[A1,T1,U2]`、`G3=[A2,T2,U3]`、`G4=[A3]`。默认摘要输入是 `前缀+G1+G2+G3+压缩 Prompt`，G4 原样进入新历史；不是按 user 消息切轮。[实际分组函数](../../../apps/zcode-cli/packages/core/src/compact/rounds.ts) · [runtime 分组调用](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts)

**微压缩估算与 Auto 的 A2 不能混为一笔账：**Microcompact 的本地估算会直接反映已替换的旧工具结果；Auto compact 若找到了较早的 Provider usage 基线，只估算该 assistant 之后的消息。若清掉的结果已包含在基线内，A2 不会从旧 usage 中反向扣除节省值，因此微压缩成功也不能保证 A3 低于阈值。这是由两个计数路径推得的条件性结论，不是运行验证。[Microcompact 本地估算](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) · [Auto usage 基线](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts)

**异常不是一个笼统“重试”。**摘要请求媒体过大时，剥离媒体重试；摘要请求超窗时，把更多最近组从摘要输入移到“保留原文”，使摘要请求变小，但压缩后的上下文可能因此更大。Auto/Reactive 不用“丢最旧组”兜底；Auto 整个可重试操作最多 3 次，Reactive 一次。Auto 最终失败保留原历史继续普通请求；取消直接传播。已触发 rapid-refill breaker 则在调用摘要前报错。这里的“重选组”和“整个操作重试”是两层不同的尝试。[触发和 usage](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) · [阈值](../../../apps/zcode-cli/packages/core/src/compact/policy.ts) · [分组](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts) · [摘要请求与提交](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

**A6 还有两个易漏细节：**保留组的 assistant usage 会失效，防止旧 usage 被误当作新请求 token 基线；summary message 没有传 `transcriptPath`，因此旧 Session 事实虽在，模型不会自动得到全文读取路径。摘要有损，只有保留组逐条留原文。[prompt.ts:119-165](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) [runtime compact.ts:72-87](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) [runtime compact.ts:157-166](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) [compact-active.ts:517-548](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

**哪些东西单独保留或重新注入？**下表把“模型能否继续看到”与“底层事实是否还在”分开：

| 对象 | compact 当下 | 之后冷恢复 / 限制 |
| --- | --- | --- |
| Session transcript | 旧消息不物理删除；写 summary message、`CompactBoundary`、后置提醒，随后替换 runtime history | `activeSessionMessages` 从最后有效 boundary 选历史，并按 `preservedSegment` 的 head/tail ID 插回最近保留消息；旧全文仍在 store，不等于自动回到模型上下文。[compact-active.ts:549-625](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) [compact-session.ts:4-46](../../../apps/zcode-cli/packages/core/src/agent/compact-session.ts) |
| 已读文件内容 | 从 `readFileState` 选最近最多 5 个合格 Read；单文件约 5K token 内可重新注入带行号内容，过大或总预算不足则只注入路径和“可再 Read”提示；保留段已含该 Read 时跳过 | 后置提醒随 summary 一起持久化；提交后 runtime `readFileState.clear()`。文件本身仍在工作区，但不是无损重灌所有已读文件。[compact-post-reminders.ts:8-66](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-post-reminders.ts) [compact-active.ts:484-501](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) [compact-active.ts:615-625](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) |
| 已批准 Plan | 若 Session 对应 plan 文件存在，读出并生成 `plan_file_reference` 后置提醒 | 提醒持久化，可在冷恢复中还原；文件读取有字节上限，缺失则不注入。不是所有规划内容都自动保存成 plan 文件。[plan-file-continuity.ts:59-101](../../../apps/zcode-cli/packages/core/src/runtime/helpers/plan-file-continuity.ts) |
| Todo 与长期目标 | Todo 不靠摘要作唯一备份：`TodoWrite` 更新 Session Store；目标也有独立 Session 存储。普通 step 仅在 Todo reminder 条件满足时读取并注入当前列表 | cold resume 分别读取 Todos/target；目标状态明确注入运行时历史。Todo reminder 是条件性提醒（默认距上次写入和上次提醒各至少 10 个 assistant turns），不是每轮必有。[todo.ts:47-71](../../../apps/zcode-cli/packages/core/src/tool/handlers/todo.ts) [turn-loop.ts:138-155](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) [runtime-reminders.ts:81-84](../../../apps/zcode-cli/packages/core/src/runtime/helpers/runtime-reminders.ts) [runtime-reminders.ts:163-179](../../../apps/zcode-cli/packages/core/src/runtime/helpers/runtime-reminders.ts) [resume.ts:233-262](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) |
| Skills 与 Memory | Skills catalog 和 Memory 索引属于 Context 前缀，随前缀保留；已调用 `Skill` 得到的正文是工具结果，进入摘要区时由模型概括，落在最近保留组才逐字保留 | cold resume 重新发现 Skills、重读 Memory 索引；`Skill` 正文可再次调用工具从磁盘加载，Memory 详情可 Read。未发现“当前执行 Skill”的独立运行时栈或无损重注入步骤，不能把 catalog 可用误说成旧正文仍在上下文。[context.ts:35-73](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts) [skill.ts:16-70](../../../apps/zcode-cli/packages/core/src/tool/handlers/skill.ts) [request-user-context.ts:47-76](../../../apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts) |
| Plugin 引用与 Hook 追加内容 | `plugin_reference` reminder 由真实用户输入解析，匹配当时 live catalog 后写为 model-only synthetic Session 消息；旧引用是否进 summary 或最近原文由位置决定。Hook additional context 是 runtime attachment，也受同样的摘要/保留选择 | plugin reminder 若仍属有效历史可 hydration；新 turn 仅对新的真实引用重新解析。`SessionStart` hook 在新 runtime 的 resume 路径运行；同一 runtime 有 once 标志，auto compact 不会凭空重新运行所有 hook。`UserPromptSubmit` 旧输出没有独立“无损恢复”为当前提醒的证据。[plugin-reference.ts:80-149](../../../apps/zcode-cli/packages/core/src/runtime/methods/plugin-reference.ts) [hooks.ts:20-47](../../../apps/zcode-cli/packages/core/src/runtime/methods/hooks.ts) [hooks.ts:106-117](../../../apps/zcode-cli/packages/core/src/runtime/methods/hooks.ts) [resume.ts:254-262](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts) |

### 压缩后模型实际会看到什么：四个具体例子

下列路径和包装文字来自源码；`review`、文件路径及摘要内容是**假设性示例**，并非运行记录。先看压缩时的共同处理：系统把 Context 前缀留在新历史，另发一次摘要请求，要求模型整理用户意图、以前做过的事、文件改动、错误、反馈和待办；再用返回的摘要生成一条隐藏于 UI、对模型可见的续接消息。摘要请求**没有 Skill 专属的“执行轨迹分析”指令或结构化字段**；能否记录“Skill 已做到第几步”取决于旧消息是否进入摘要区、模型是否把它写进通用摘要。最近保留组则原文续接。[压缩 Prompt](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) · [新历史拼装](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [压缩提交](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

例如同一假设任务在压缩后的**模型可见顺序**如下。方括号是本文为帮助阅读加的层名；英文提示开头对应源码，省略号处取决于文件内容和模型生成的摘要：

```text
[Context 前缀] The following skills are available for use with the Skill tool:
               - review: 检查变更 ... (file: .../SKILL.md)
               Contents of .../MEMORY.md (user's auto-memory, persists across conversations):
               - 部署约束 → deploy.md — 发版前先核对环境
[压缩摘要]     This session is being continued from a previous conversation...
               Summary: ... 已按 review Skill 改完协议；测试待补 ...
[最近保留组]   近期未摘要的消息和工具结果，按原顺序
[Plan 提醒]   A plan file exists from plan mode at: ...
               Plan contents: ... 先改协议，再补测试 ...
```

这不是一个实际 wire payload：最终请求还会经过 Provider 投影与其他运行时提醒；如果对应 Plan 文件不存在，就没有最后一段。如果摘要模型没有写出“测试待补”，摘要段也不会自动生成该事实。[Context Skill 文本](../../../apps/zcode-cli/packages/core/src/context/sections/skills.ts) [Memory 索引文本](../../../apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts) [续接消息模板](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) [Plan 提醒模板](../../../apps/zcode-cli/packages/core/src/runtime/helpers/plan-file-continuity.ts)

| 内容 | 压缩后的具体例子 | 来源和保真程度 |
| --- | --- | --- |
| 已批准 Plan | 假设 `.zcode/plans/plan-当前会话.md` 内写着“先改协议，再补测试”。压缩会读该文件，追加一个类似 `A plan file exists from plan mode at: ...`、`Plan contents:`、原 Plan 正文、`If this plan is relevant ... continue working on it.` 的提醒。 | 这是实际的 `plan_file_reference` 提示格式，不靠摘要模型猜 Plan；文件不存在或空白时没有该提醒，其他读取错误会向压缩流程抛出。成功添加的提醒会随摘要持久化，冷恢复仍可还原。仅限**已写成批准 Plan 文件**的内容。[plan-file-continuity.ts:59-101](../../../apps/zcode-cli/packages/core/src/runtime/helpers/plan-file-continuity.ts) [compact-active.ts:484-501](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) |
| 已调用 Skill | Context 前缀重新列可用 Skill，例如 `- review: 检查变更 ... (file: .../SKILL.md)`；元数据过预算时退化为名称、别名和路径。此前 `Skill(review)` 工具若返回过，模型当时实际看到的是 `<skill_content name="review">` 包裹的完整指令（最多读取 100 KB，并可标记截断）。压缩后若这条工具结果位于摘要区，模型只看到**通用摘要选出的相关事实**；若位于最近保留组，仍看到原工具结果和后续操作。 | **目录继续可见 ≠ 正文自动重注入 ≠ 执行进度被独立恢复。**摘要 Prompt 会要求记“以前做过的事、文件改动、错误、待办”，可能概括 Skill 执行轨迹，但没有逐步轨迹生成器、Skill checkpoint 或“运行中的 Skill”对象；需要原指令时可以再次调用 `Skill` 加载。[skills.ts:30-70](../../../apps/zcode-cli/packages/core/src/context/sections/skills.ts) [skill.ts:18-70](../../../apps/zcode-cli/packages/core/src/tool/handlers/skill.ts) [prompt.ts:14-67](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) |
| Project Memory | Context 前缀仍有 Memory 使用说明与 `Contents of .../MEMORY.md (user's auto-memory, persists across conversations):`，后面是索引内容。假设索引有 `- 部署约束 → deploy.md — 发版前先核对环境`，模型能看到这条指针；`deploy.md` 正文只有在后续显式读取或它的旧 Read 结果仍在保留区/摘要里时才可见。 | 压缩不自动生成新 Memory，也不把所有 Memory 文件正文塞回请求。热会话使用 Context 初始化时的索引快照；冷恢复重新初始化才读取磁盘索引。已读文件还可能经独立的 post-compact Read 提醒恢复：小文件附带带行号内容，过大则只给路径和“再用 Read”的提示。[memory.ts:8-50](../../../apps/zcode-cli/packages/core/src/context/sections/memory.ts) [request-user-context.ts:47-86](../../../apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts) [compact-post-reminders.ts:8-75](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-post-reminders.ts) |
| Plugin 引用 | 假设用户曾引用某 Plugin，运行时解析成功后提示体形如 `<plugin_reference>`、`id`、`skills: [...]`、`mcp_servers: [...]`、`subagents: [...]`，并注明这是能力元数据。若该旧提示在摘要区，压缩后仅可能留下摘要对它的概括；在最近保留组则原文继续可见。之后**新的用户输入**再次引用 Plugin，才会重新与当前 live 能力求交并生成新提示。 | Plugin 提示不是压缩后统一重注入的内容。旧提示是持久的 model-only 消息，但完整压缩的 boundary 仍决定它是否在有效历史中；Plugin 能力此时是否仍可用还要看当前 live catalog。[reminder.ts:158-180](../../../apps/zcode-cli/packages/core/src/plugin-reference/reminder.ts) [plugin-reference.ts:80-149](../../../apps/zcode-cli/packages/core/src/runtime/methods/plugin-reference.ts) |

例如一个假设任务：用户要求按 `review` Skill 检查协议，Agent 已调用 Skill、改完协议，但尚未补测试。若 Skill 调用和改动落在待摘要旧组，下一次模型请求可能看到“可用 Skill 列表 + 摘要中‘协议已改、测试待补’ + Plan 文件提醒”；**不会因目录里仍有 review 就自动恢复已加载的全部 Skill 指令，也不会有系统生成的逐步骤轨迹**。如果摘要漏掉“测试待补”，但批准 Plan 文件明确写了它，Plan 提醒仍能补回；若只是模型临时想过、未写入 Plan/Todo/文件，也没有独立备份。[摘要指令](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) [Plan 提醒](../../../apps/zcode-cli/packages/core/src/runtime/helpers/plan-file-continuity.ts) [Skill 工具](../../../apps/zcode-cli/packages/core/src/tool/handlers/skill.ts)

**持久提交与冷重建顺序。**先持久化 summary text/compaction part 和后置提醒；再写 boundary event、completed timeline；最后替换 runtime `MessageHistory` 并清 read-file state。持久化提醒任一步失败会 best-effort 回滚本次消息。冷恢复选择最后有效 boundary，插回 boundary 记载的最近保留段；若进程在 compact started/retrying 时退出，有 boundary 则修复为 completed，没有则 interrupted。这里的“恢复上下文”是按持久结构**重建新的模型有效视图**，不是把压缩前所有信息无损解压回来。[compact-active.ts:575-630](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) [compact-persistence.ts:289-381](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-persistence.ts) [resume.ts:147-179](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts)

**Memory 与 compact 不是同一条备份链。**项目 Memory 文件是独立的持久事实；成功 turn 结束后可另行调度后台提取，从 durable conversation 快照提炼并更新 Memory。`compactActiveConversation` 本身不把 summary 自动当成新的 Memory 文件；冷恢复只是重新读取现有索引，详情仍按需访问。摘要 Prompt 中“保留约束、错误与用户反馈”的要求属于模型生成目标，不能当作字节级保真承诺。[turn.ts:688-703](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn.ts) [project-memory-extraction.ts:32-80](../../../apps/zcode-cli/packages/core/src/runtime/helpers/project-memory-extraction.ts) [context.ts:60-72](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts) [prompt.ts:14-43](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts)

**“正在执行 Skill”还要区分两种状态。**如果 Skill 工具已经返回，正文只是历史中的一条 tool result：它可能进入摘要、最近保留组，或因 microcompact 被清理。如果进程在工具仍处于 pending/running 时消失，冷恢复会给该 tool call 放入 interrupted 结果，不会从调用中间继续执行；后续是否重新调用 Skill 由新的模型步骤决定。源码中没有把 Skill 当成可 checkpoint/restore 的独立解释器或指令栈。[skill.ts:18-70](../../../apps/zcode-cli/packages/core/src/tool/handlers/skill.ts) [session-history-hydrator.ts:149-190](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts)

## 04. 动态请求装配：哪些内容此刻真正可见

前面的 03 结束后，loop 才初始化 MCP、按当前执行状态筛选工具，并视条件追加 plan/mode/Todo/output-style 等提醒；然后从本轮 entries 构造 Provider-neutral 消息，调整 attachment 顺序和 cache marker。工具定义在请求 `tools` 字段，不能把其 token 成本当成一段普通 system 文本。模型可见消息与用于记录的 `ModelRequest` 投影还可能不同：一次性 output-token continuation 会进入真实请求，却从记录投影过滤。[turn-loop.ts:105-213](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) [provider-request-messages.ts:286-334](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts)

Memory 的常驻部分是索引而非自动检索出来的正文；`Skill` 目录信息是 catalog 而非全部 SKILL.md；Plugin 引用提醒只在用户确实提交引用且能与 live 能力交集匹配时出现。一次消息最终看到的内容取决于这几条路径和前面的 compact 结果，不宜用“系统把所有资料拼接后裁剪”概括。[context.ts:60-72](../../../apps/zcode-cli/packages/core/src/runtime/methods/context.ts) [skill.ts:36-70](../../../apps/zcode-cli/packages/core/src/tool/handlers/skill.ts) [plugin-reference.ts:80-131](../../../apps/zcode-cli/packages/core/src/runtime/methods/plugin-reference.ts)

## 05. 为什么压缩后还要估算 token

这里是**两次不同决策**，不是 03 没有估算。03 在当前历史上估算输入压力，问题是“要不要花一个独立模型请求生成摘要”；05 在 **03 可能已经替换过的当前请求** 上再次估算，问题是“本次普通模型请求最多允许输出多少 token”。此外 04 又可能追加提醒与工具可见能力，所以 03 的旧数字不能直接作为 05 的新请求预算。两次都优先利用已提交 assistant 的 Provider usage 基线加本地增量，否则退回本地估算；估算不等于最终 Provider tokenizer。[compact.ts:201-214](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) [compact.ts:313-349](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) [turn-loop.ts:105-213](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) [turn-model-step.ts:230-250](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

05 的公式是：模型声明的输出上限（未声明时 32K）与 `contextWindow − estimatedCurrentInput − 1K` 取较小正值；若估算剩余非正数，保留模型 baseline 上限，让 Provider 错误触发后续恢复，而不是把启发式估算当作本地硬拒绝。03 的阈值则预留至多 21K 输出和默认 13K buffer，目的不同。[model-token-limits.ts:5-40](../../../apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts) [policy.ts:67-87](../../../apps/zcode-cli/packages/core/src/compact/policy.ts)

**数字例子（假设模型窗口 200K、输出上限 32K）：**03 的默认 auto 阈值为 `200K − 21K − 13K = 166K`。旧历史估算 170K 且可摘要，先 compact；假设新历史变 60K，05 再算可用输出为 `min(32K, 200K − 60K − 1K) = 32K`。若 compact 因历史不足而未运行，输入估算 190K，则 05 上限约 9K。若估算已超过窗口，代码不会因该估算直接停止，而由 Provider 的真实超窗反馈决定是否走 reactive 分支。以上数字仅说明代码公式，未运行 Provider。

## 06–07. 媒体、工具结果与异常如何回到下一步

06 在 `runModelTextRequest` 中依次解析附件路径、按模型输入能力投影，再对本次请求所有媒体按**编码后字节**施加默认 40 MiB 聚合预算。若超限，保护最新真实用户消息的媒体，历史媒体按最近优先保留，丢弃的旧块变文字占位；若最新用户媒体自己已超限，直接抛当前附件过大错误。这是请求投影，不改 Session transcript，也不能靠摘要旧消息解决单个新附件过大。[model.ts:36-77](../../../apps/zcode-cli/packages/core/src/runtime/methods/model.ts) [media-budget.ts:75-150](../../../apps/zcode-cli/packages/core/src/runtime/helpers/media-budget.ts)

07 收到工具调用时，工具调度可并发，但返回结果按 tool-call ID 汇合并按原调用顺序提交。每个工具可能先在自己的适配器层限制原始输出；之后通用 `resultBudget` 默认按 UTF-8 字节给模型可见内容 100,000 bytes 上限并截断，显式启用 artifact 策略的工具才会将那一层的完整可见内容外置。以 Bash 为例，它在更早的进程采集层给 inline 内容 30,000 bytes 上限、持久文件最多 5 GiB：前台超 inline 才保留文件，后台始终持久化。因此“有 artifact path”并不等于原始 stdout 永远完整。批量汇合处没有统一的“所有并发工具结果总 token”闸门；这些结果提交到运行时历史与当前请求后回到 02，由下一次 micro/auto 判断处理。旧结果通过 microcompact 清掉后，只有先前提供了仍可访问的文件路径且采集层确实保留了完整内容，模型才能用 Read 恢复读取；普通截断结果不能凭清除标记逆推出原文。[turn-tools.ts:235-269](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts) [turn-tools.ts:279-366](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts) [result-serialization.ts:33-86](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts) [result-serialization.ts:161-217](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts) [bash.ts:70-71](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash.ts) [bash.ts:420-427](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash.ts)

Provider 抛 context-exceeded，或在无 tool call/无输出续写时返回相应 finish reason，才走 reactive compact；同一个 model step 最多尝试一次，成功后替换 entries、重建 turn machine，再回到 02 重新组装。若没有足够历史、compact 禁用或恢复失败，就保留原错误；rapid-refill breaker 另有显式错误。summary 请求自身的 prompt-too-long、媒体太大按 03 的内部重试处理，不能与普通请求的 reactive 分支混为一谈。输出 token 到上限则走独立的 continuation 分支。[turn-model-step.ts:429-439](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-model-step.ts:500-524](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-model-step.ts:647-705](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [turn-model-step.ts:742-802](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

这套设计的可迁移要点是把**摘要历史、保留原文、独立持久状态、外置文件和请求时投影**分开设计。完整 compact 不是无损存档：Session 旧消息仍在，但当前模型不自动看见；文件与 Todo 仍可读取，但未必自动注入；Skill/Hook 旧内容若落入摘要区，只能依赖摘要质量或显式重新加载。对当前仓库的判断是源码静态追踪，未将假设例子写作实际运行验证。

## 08. Reactive compact：Provider 拒绝之后如何重试同一步

这条路径与 **03 的主动压缩**共用摘要和持久化实现，但**不要求本地估算先达到主动阈值**。例如本地估算低于门槛，Provider 仍判定请求超窗（真实 tokenizer、工具 schema 等可能造成差异），模型步骤就会进入这里；下述例子是条件演示，不是一次已观察到的 Provider 调用。[普通请求错误入口](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [Reactive 入口](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts)

```mermaid
flowchart TD
    R1["① Provider 报上下文超窗<br/>抛错，或返回超窗结束原因"] --> R2{"② 本次模型步骤已经尝试过<br/>或者连续快速压缩保护已触发？"}
    R2 -- 是 --> RF["停止恢复；报告错误"]
    R2 -- 否 --> R3{"③ 完整压缩启用<br/>且仍有足够旧历史可摘要？"}
    R3 -- 否 --> RF
    R3 -- 是 --> R4["④ 从当前请求选旧消息生成摘要<br/>最近消息保留原文"]
    R4 --> R5{"⑤ 摘要请求成功？"}
    R5 -- 否 --> RF
    R5 -- 是 --> R6["⑥ 持久化摘要与边界<br/>替换本轮消息，重建模型步骤"]
    R6 --> R7["⑦ 回到 02，重新装配并请求 Provider"]
    R7 -- 再次超窗 --> RF
```

**① 哪些失败真正触发？**普通请求抛出的错误需被识别为 `ModelContextExceeded`，或结果的标准/原始 finish reason 明确标记为超窗；后一条路径还要求本地终止未处理、没有工具调用、没有输出上限续写。当前媒体附件过大走媒体错误路径，不会靠这条 Reactive compact 修复。[错误识别](../../../apps/zcode-cli/packages/core/src/runtime/helpers/model-errors.ts) [抛错路径](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [finish reason 路径](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

**②–③ 为什么会跳过？**一次模型步骤只允许一次 Reactive 尝试；快速回填保护在距上次完整压缩不足 3 个已完成工具批次且连续快速再触发达到 3 次时阻断。之后才从**当前请求 entries** 重新投影；总 compact 被禁用、历史不足两组或没有可摘要 assistant 时返回 skipped。这里不因 `tokenCount < T` 跳过，也不使用 UI 消息或整份 SQLite transcript 重新摘要。[步骤保护](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) [快速回填计数](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop-state.ts) [Reactive 条件](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts)

**④–⑤ 摘要模型具体收到什么？**先选旧组作摘要输入、最近至少一组原文保留，再附通用压缩 Prompt，单独发一笔模型请求。若 Provider 的初始错误给出可解析的 token 超额差值，且历史足够，可在第一次摘要请求前多保留最近组；摘要请求自身超窗时继续把最近待摘要组移到保留区，从而缩短摘要请求。Reactive **不会**启用手动压缩可用的“丢掉最旧摘要组”降级；扩大原文保留区反而可能让最终上下文更大。摘要请求媒体太大时先剥离媒体重试；取消直接向外传播。[初始重选](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts) [摘要请求与异常处理](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) [压缩 Prompt](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts)

**⑥–⑦ 成功与失败怎样落地？**成功时写隐藏 summary、后置提醒和 CompactBoundary，替换 canonical 运行时历史及本轮 request entries，清连续压缩失败计数，重建 turn machine，随后回到 02 重组**同一个**模型步骤。若摘要 skipped/failed，原超窗错误继续抛出；非取消失败增加连续失败计数。若重试的普通请求再次超窗，一次尝试标记仍在，不能无限循环。Provider 抛错的分支还会先持久化失败 assistant 记录；不能把它和摘要后的成功模型响应合并成一次普通成功。[提交和替换](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) [Reactive 结果](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) [同一步重试](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

本专题的上下文入口还包括**手动 `/compact`**：它直接调用完整摘要链路，可带自定义摘要指令，不经 03 的自动 token 阈值；手动摘要请求超窗时允许进一步丢最旧组，语义与 Auto/Reactive 不同。具体选组和失败规则见 [B2](b-compaction.md)。在当前报告中，Provider 最终 wire 的逐字段预算、真实 tokenizer 偏差和缓存命中率仍没有运行证据；这些不能由本地图和静态摘要冒充已验证行为。
