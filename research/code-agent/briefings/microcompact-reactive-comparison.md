# 上下文压缩策略的机制比较：Microcompact 与 Reactive Compact

## 摘要

本报告比较 ZCode 与自研 Code Agent 在两个时点的上下文治理：模型请求前的 **Microcompact**，以及 Provider 报告超窗后的 **Reactive Compact**。两者的主要差异不是是否压缩，而是**压缩对象、信息保留粒度和升级到摘要的时机**。ZCode 一侧依据源码基线 `872ad960de7ec172591f7e1952f7849229f94521`；自研一侧按确定的方案规则展开。收益与成本属于机制分析，不等同于性能实测。

![图 1：请求前 Microcompact 机制比较](assets/01-microcompact-comparison.png)

![图 2：超窗后 Reactive Compact 路径比较](assets/02-reactive-comparison.png)

两幅图的[可编辑 HTML 源文件](compact-comparison-slides.html)采用 16:9 画布，可直接用于汇报或重新导出。

## 1. 研究问题与比较框架

长任务中，工具结果不断进入模型历史，使上下文占用升高；本地 token 估算与 Provider 的真实窗口判断还可能不一致。因此需要区分两类决策：**请求前能否主动释放空间**，以及**请求已被拒绝后如何使当前任务继续**。Microcompact 处理前者，Reactive Compact 处理后者。比较维度固定为触发条件、操作对象、信息损失、恢复步骤和失败边界，避免仅凭“压缩率”判断优劣。

## 2. 请求前：Microcompact

| 维度 | ZCode | 自研方案 |
| --- | --- | --- |
| 触发条件 | 每个 model step 前检查；启用后，距上次 assistant 完成严格超过 **60 分钟**，或投影消息估算达到微压缩门槛。默认门槛为完整 Auto Compact 门槛的 90% 与“该门槛减 2,000 token”中的较小值。 | 压力 `≥40%` 且本周期未执行 Soft 时执行一次；压力 `≥60%` 且本周期未执行 Hard 时执行一次。首次观察已达 60% 时，两档可在同一请求中先后执行。 |
| 对象 | 仅合格工具结果；从旧到新按 assistant 的工具调用批次分组，保留最近 **5 组合格批次**的原文。默认跳过错误、已清理及含媒体的结果。 | 只处理**已完成 turn** 的旧只读/搜索工具结果。Soft 保护最近 **12 个 user turns**，Hard 保护最近 **8 个**；Read、Grep、Glob 和明确只读的 Bash 可裁，Edit/Write、Agent、权限及状态变更工具排除。 |
| 操作 | 选中一条结果时，将其**整个 content** 替换为 `[Old tool result content cleared]`，不保留前后片段或预览；工具调用 ID、参数和顺序不变。 | Soft：单结果 `>4,000` 字符时保留头尾各 **1,500**；Hard：单结果 `>1,000` 字符时仅保留头部 **1,000**。两档均加入说明省略字符数的提示，不删 tool result block。 |
| 状态与提交 | 替换后重新估算；总节省不足默认 **256 token** 时撤销此次所有替换；只改活跃 Runtime 历史。 | `softApplied` / `hardApplied` 为 runtime-only 周期标志。每档本周期最多执行一次；成功的 auto/manual/reactive/session-memory compact 或会话 reset 重置，失败 compact 不重置。原始 transcript 保留。 |

ZCode 的门槛**不是 context window 的 90%**。若窗口为 200,000 token、输出预留取 21,000、缓冲为 13,000，完整 Auto Compact 门槛是 166,000，Microcompact 门槛为 `min(149,400, 164,000)=149,400` token；60 分钟时间条件仍可使较低占用的会话进入候选扫描。该清理在常规运行时配置中默认关闭，需要显式启用。其顺序为 **Microcompact → Auto Compact → 普通模型请求**。[默认值与候选算法](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) · [运行时开关与回写](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts) · [步骤顺序](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) · [门槛计算](../../../apps/zcode-cli/packages/core/src/compact/policy.ts)

这一差异带来不同的信息损失函数：ZCode 对选中结果是**离散的全文删除**，单次释放空间明确；自研方案采用**分级的部分保留**，把释放量与结果可读性一起调节。按设计推论，后者有利于后续模型继续利用工具证据；前者的空间回收更容易预测。哪一方带来更高任务正确率，需要在同一任务集上测量，不能从机制直接推出。

ZCode 只修改 Runtime 的 canonical history 与本轮 request entries，不回写持久 Session 的 tool part。因此若仅执行 Microcompact，冷恢复可能把旧工具全文重新带回有效历史；随后下一次模型步骤才重新检查 Microcompact/Auto Compact。此处是 ZCode 实现的状态边界，不能把当前内存的 token 节省理解为跨重启持久压缩。[运行时提交](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts) · [冷恢复](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts)

### 2.1 ZCode 的逐步算法

1. **构造判定输入。**每个 model step 前，把当前 `turnRequestState.entries` 投影为 Provider-neutral 消息，投影时不设置 cache marker；这一步只在内存构建候选请求，尚未调用 Provider。估算器对每条消息计算 `ceil((正文可估字符数 + 工具名称及 JSON 入参字符数) / 3)` 再求和，reasoning 文本也纳入相应投影。[入口](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts) · [投影与回写](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [估算器](../../../apps/zcode-cli/packages/core/src/compact/manual.ts)
2. **判断是否扫描。**设模型窗口为 `W`，输出预留为 `R=min(模型最大输出, 21,000)`，缓冲为 `B=13,000`（默认），则完整 Auto 门槛 `T=max(0,W−R−B)`；Micro 门槛 `M=max(0,min(floor(0.9T),T−2,000))`，配置可以直接覆盖 `M`。距上次 assistant 完成超过 60 分钟时优先以时间触发；否则需本地估算 `E≥M`。没有上次完成时间时无法走时间分支。[门槛](../../../apps/zcode-cli/packages/core/src/compact/policy.ts) · [Micro 决策](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts)
3. **选择候选。**按消息顺序扫描；每遇一条带 tool calls 的 assistant 消息开始新批次。只收集具备 `toolCallId`、非错误、未被先前清理、结果不含 image/video/file，且工具名属于 **Read、Bash、Grep、Glob、WebFetch、WebSearch、Edit、Write、ApplyPatch** 的 tool message。一个批次只有合格结果才计为合格组；孤立但合格的工具结果各自成组。保留最后五个合格组，对更旧组合格结果逐条处理；同组不合格结果保持原样。这也说明 ZCode 的白名单包含 Edit/Write，而自研方案明确排除 mutation result。[候选与分组](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts)
4. **替换与收益判定。**每个选中结果的 `content` 整体变为 33 字符常量 `[Old tool result content cleared]`，再以同一估算器计算 `E_after`。若 `max(0,E_before−E_after)<256`，返回原消息，整次不提交；否则按 `toolCallId` copy-on-write 回写 Runtime entries，再同时替换 canonical `MessageHistory` 与本轮请求 entries，发出边界事件。持久 Session 原 tool part 不随之改写。[替换及回滚](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts) · [按 ID 回写](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts) · [提交](../../../apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts)

**可复算例（构造数据，并非运行记录）：**已满足 61 分钟时间条件；投影估算总量 120,000 token。共有 G1 至 G7 七组合格工具调用，G1 两条、G2 一条旧结果各为 900 个纯文本字符，G3–G7 是近期保留组。三条旧结果各从 `ceil(900/3)=300` 变为 `ceil(33/3)=11` token，总节省 `3×(300−11)=867` token，超过 256，故提交；G1/G2 中不合格的结果不会被顺带替换。若同批结果合计仅省 200 token，则三条替换全部撤销。这个例子显示 ZCode 的“门槛触发扫描”与“节省足够才实际改状态”是两个不同判定。[完整算法](../../../apps/zcode-cli/packages/core/src/compact/microcompact.ts)

### 2.2 自研方案的周期状态、候选与精确动作

自研方案维护 Runtime 内的 `HistoryPruneCycleState {softApplied, hardApplied}`。在同一 compaction 周期，压力 `p≥0.40` 且 `softApplied=false` 才执行 Soft，压力 `p≥0.60` 且 `hardApplied=false` 才执行 Hard；执行后立即标记对应档位。两档互不互斥：首次观察到 `p≥0.60` 可以同一步先后执行，Soft 已执行过后压力才升到 60% 也仍可执行 Hard。某档位已经触发，后续请求不会反复裁同一档。成功的 auto、manual、reactive、session-memory compact，以及 session reset/新会话，重置两个标志；失败的 compact 不重置。这个周期状态与 ZCode “每个 model step 检查，满足触发后再看候选和收益”的模型不同。

候选必须同时满足两类限制。**语义限制**：只允许 Read、Grep、Glob，以及命令被明确判为只读/搜索型的 Bash（例如 `ls`、`find`、`rg`、`grep`、`cat`、`Get-Content`、`Select-String`）。Edit、Write、MultiEdit、Agent、权限工具、状态变更工具、语义不明确的 Bash 和 Snip 排除。**时间限制**：只处理已经到达 assistant 回复或稳定请求边界的 completed turns，Soft 保护最近 12 个 user turns，Hard 保护最近 8 个；正在执行的 tool loop 不得被裁。user message、assistant 最终文本、tool use/result 配对结构、mutation result 与原始 transcript 保持不变。

| 档位 | 单条结果的判断与变换 | 结果中的解释性标记 |
| --- | --- | --- |
| Soft | 仅当旧结果文本 `>4,000` 字符，保留前 1,500 与后 1,500 字符，省略中间。`≤4,000` 原样保留。 | 明确说明“旧工具结果已裁剪/截断”、中间省略的字符数、保留头尾各 1,500 字符，以及原文仍在 transcript；标记不冒充工具原始输出。 |
| Hard | 仅当旧结果文本 `>1,000` 字符，保留前 1,000 字符并省略后续。`≤1,000` 原样保留。 | 明确说明 Hard prune、后续省略的字符数、只保留头部 1,000 字符，以及原文仍在 transcript。至少保留头部内容，不退化为纯占位符。 |

标记模板为：

```text
[旧工具结果已裁剪/截断：中间 N 个字符已省略，保留头部 1500 字符和尾部 1500 字符；原始内容仍保存在 transcript 中]
[旧工具结果已硬裁剪/截断：仅保留头部 1000 字符，后续 N 个字符已从本轮模型上下文中省略；原始内容仍保存在 transcript 中]
```

按已确定的规则，决策顺序可写成如下伪代码；`selectCompletedOldResults(k)` 只返回保护最近 `k` 个 user turns 之外、属于只读/搜索工具的文本结果：

```text
读取本轮压力 p 与当前周期状态 (softApplied, hardApplied)
if p >= 0.40 and not softApplied:
    for result in selectCompletedOldResults(12):
        if chars(result.text) > 4000:
            result.content = head(result.text, 1500) + SoftNotice(N) + tail(result.text, 1500)
    softApplied = true
if p >= 0.60 and not hardApplied:
    for result in selectCompletedOldResults(8):
        if chars(result.text) > 1000:
            result.content = head(result.text, 1000) + HardNotice(N)
    hardApplied = true
成功完成任一完整 compact 或 reset/new session: 两个 flag 均清零
完整 compact 失败: 两个 flag 不变
```

伪代码刻意把**档位触发**与**单结果长度判断**分开：压力达到 40% 不代表每条结果都会修改；长度不够、处于保护窗口或尚未完成的 turn 保持原样。若一次请求的初始压力已达 60%，两个 `if` 都能进入；Hard 看到的可能是 Soft 之后的当前内容。上述规则只替换 result content，不删 user/assistant 消息或 tool-use/result 配对，也不修改原始 transcript。

**可复算例（构造结果）：**一条 5,200 字符的 Read 输出位于最近 12 个 user turns 之外，且其他候选条件满足。Soft 后保留 1,500 字符头部与 1,500 字符尾部，正文省略 `5,200−3,000=2,200` 字符，再加入带该数字的裁剪标记。之后压力上升到 60%，它也位于最近 8 turns 之外；Hard 对当前模型可见结果保留开头 1,000 字符并追加 Hard 标记。原始 5,200 字符仍可从 transcript 读取，Tool ID 与结果 block 不变。这个例子体现两档**可先后作用于同一旧结果**，但每档在一个周期只执行一次。

**一个需要在实现与指标中单独处理的算术边界：**Hard 对“刚超过 1,000 字符”的结果追加提示，输出可能反而比输入更长。例如 1,001 字符结果变成“头部 1,000 字符 + 提示”，若提示超过 1 字符就没有净节省。这不否定分级策略，但说明压力触发、文本裁剪和实际 token 回收是三个不同量；测量中应记录提示成本与净节省。Soft 若作用于已裁剪结果、Hard 再接续，也应让“省略 N 字符”的口径指向清楚的原文长度，而不是把上一档提示误当工具输出。

## 3. 超窗后：Reactive Compact

| 阶段 | ZCode | 自研方案 |
| --- | --- | --- |
| 入口 | 普通请求出现可识别的 context-exceeded 错误，或满足条件的超窗 finish reason。 | Provider 报告当前请求超出上下文窗口。 |
| 第一层恢复 | 直接从**当前失败请求**的 entries 选择旧轮生成摘要，最近至少一组保留原文；同一 model step 最多一次 Reactive 完整压缩尝试。 | 根据“估算输入 + 最大输出”超过压缩阈值的量，裁剪历史工具结果；每轮裁剪后**重新请求 Provider**，最多三轮。 |
| 升级策略 | 摘要请求自身超窗时，可调整“摘要旧组 / 保留近期组”的边界；没有 Reactive 内的三轮工具裁剪。 | 三轮后仍超窗，回到前一轮正常消息，对旧历史压缩，再拼回本轮增量消息。 |
| 续跑 | 持久化摘要与 CompactBoundary，替换 Runtime 和本轮请求，重建 turn machine 后重试**同一**模型步骤。再次超窗不会无限重复 Reactive。 | 以“旧历史摘要 + 本轮增量”构成新的当前请求，继续当前任务。 |

两种兜底在**功能目标**上相同：缩短旧历史，同时不丢失当前任务所需的近期内容；但其**状态构造**不同。ZCode 没有显式回退到“上一轮成功的 Provider 请求”再拼接增量。它以**当前失败请求**为输入，按 assistant-started rounds 分组，摘要较旧组、原样保留近期组，之后提交并重试。[失败请求的恢复入口](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) · [Reactive 调用](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact.ts) · [分组与选择](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts) · [历史替换](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

ZCode 的“一次”指**对普通 model step 的一次 Reactive 完整压缩**，不等于摘要阶段只调用一次模型：摘要请求内部可因 prompt-too-long 或媒体过大调整输入并重试。若压缩被禁用、可摘要历史不足、快速回填保护生效、摘要失败，或摘要后的普通请求再次超窗，恢复仍可失败。[摘要重选和失败路径](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts) · [一次性保护](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

### 3.1 ZCode 的 Reactive 状态转换

以失败请求的有效历史 `P + R1 + R2 + R3 + R4` 为例：`P` 是 Context 前缀，`R1…R4` 是按**新 assistant 消息开始**划分的四组。基础选择把 `P+R1+R2+R3` 交给摘要请求，把 `R4` 原样保留；`P` 进入摘要请求提供背景，但不会作为要摘要的正文。若最初的超窗错误提供可解析 token 差值，选择器可能在摘要前把更多近期组（如 R3）移到原文保留区。摘要请求自身若超窗，也沿同一方向移动近期组，缩小**摘要请求**；这可能使最终活动上下文反而更大，不能误写成“每重试一次必定更短”。Reactive 不使用手动 compact 可用的“删除最旧摘要组”兜底。[分组与初始选择](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts) · [内部重选](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

摘要模型接收“选中的旧历史 + 通用 compact prompt”，prompt 要求整理用户意图、技术细节、文件与错误等，并禁止工具调用；它是**独立模型请求**，不是把某条旧工具结果直接截短。成功后构成 `P + 隐藏 summary 消息 + 保留组原文 + 后置提醒`；后置提醒可包含已批准 Plan 文件引用及已读文件状态。提交时持久化 summary 与 CompactBoundary，替换 Runtime 历史及当前请求 entries，随后同一个 model step 再次构造普通请求。[摘要 Prompt](../../../apps/zcode-cli/packages/core/src/compact/prompt.ts) · [摘要请求、提醒与提交](../../../apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts)

因此，ZCode 的“恢复点”不是一份先前成功请求的字节级副本，而是**失败请求上的新分段视图**。它保留最近组，摘要较旧组；若重试普通请求仍超窗，当前 model step 的 Reactive 标志阻止再次进入同一完整压缩循环。[恢复门](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts)

### 3.2 自研方案的分层恢复序列

自研路径先在失败请求上处理历史工具结果。令 `H_ok` 表示前一轮可正常工作的消息基线，`Δ_now` 表示本轮增加的消息。若估算输入为 `I`、当前最大输出预算为 `O`、压缩阈值为 `T`，则可以用 `D=max(0,I+O−T)` 表达**本轮需要释放的超额量**；裁剪历史工具结果的规模随 `D` 调整，而非固定每轮删相同长度。每轮裁剪后都会重新请求 Provider，成功即结束恢复。三轮仍被拒绝时，兜底构造可写成：

```text
收到 Provider 超窗
  → 计算超额量，裁剪历史工具结果 → 重新请求 Provider（第 1 轮）
  → 仍超窗：按新超额量继续裁剪 → 再请求 Provider（第 2 轮）
  → 仍超窗：再裁剪并重新请求 Provider（第 3 轮）
  → 仍超窗：Summary(H_ok) + Δ_now
  → 用重建后的消息继续当前任务
```

这一路径与 ZCode 的主要差异不在最后是否存在 summary，而在**摘要之前最多经历三次“裁剪 → Provider 反馈”闭环**，以及摘要输入与本轮增量被显式分开。实现这一设计时，`H_ok` 与 `Δ_now` 必须构成无重叠且无遗漏的分区；尤其要维护 assistant tool call 与 tool result 的 ID 配对，并说明失败请求产生的 assistant 错误记录归属哪一侧。这是从恢复序列推导出的必要状态约束，不是 ZCode 的现成实现。

### 3.3 一个会改变结论的选组例子

设 ZCode 当前失败请求正文分为 R1–R4 四组，本地估算分别为 10,000、12,000、1,000、1,200 token；Provider 错误给出超额差值 2,000。基础策略先保留 R4，旧组 R1–R3 进入摘要请求。初始超额差值减去已保留的 R4 后还差 800；选择器从近向旧累计，R3 的 1,000 已覆盖差值，于是进一步保留 R3，最终**摘要 R1–R2，原样保留 R3–R4**。此时摘要请求从 23,000 token 的旧组缩成约 22,000 token 的旧组，但恢复后的活动上下文要同时容纳 summary、R3 和 R4；因此“摘要请求变短”不等于“最终普通请求必定入窗”。数字是按选择器规则构造的演算，不是 Provider 实测。[初始超窗差值选组](../../../apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts)

自研方案在同一类压力下先改变历史工具结果的内容，而不立即改变 R1–R4 的轮次结构；只有三轮局部治理仍不足时才把 `H_ok` 压缩为 summary，再附加 `Δ_now`。这使“哪些事实被直接裁剪”和“哪些旧轮被模型摘要”成为两个相互独立的决策层。两种设计的质量比较，应观察摘要前是否已损失关键工具证据，以及摘要后是否还能正确完成当前工具调用链。

## 4. 状态边界与失败语义

| 表示或状态 | ZCode Microcompact | ZCode Reactive Compact | 自研路径的必要约束 |
| --- | --- | --- | --- |
| 当前模型请求 | 旧合格工具结果的正文被固定标记替代；节省不足则保持原请求。 | 用当前失败请求的旧轮 summary、近期原文及后置提醒重建，再发起同一步骤。 | 三轮裁剪与兜底拼接都必须维护工具调用 ID、结果顺序和本轮输入的单次出现。 |
| Runtime 历史 | 成功时与本轮 request entries 一起替换。 | 成功后替换 canonical history，并重建 turn machine。 | Soft/Hard 只改旧工具结果正文；`softApplied` / `hardApplied` 记录本周期已执行的档位。Reactive 三轮重新请求 Provider，随后才进入摘要兜底。 |
| 持久 Session | 原 tool part 不随 Microcompact 清理；只出现边界事件。 | 隐藏 summary、后置提醒与 CompactBoundary 写入持久记录；冷恢复据边界选有效历史。 | 原始 transcript 不因工具裁剪而改写；“前一轮正常消息”和“本轮增量”须形成无重叠且无遗漏的边界。 |
| 失败结果 | 门槛未达、无候选、可清组不足或节省不足时，维持原状态并进入后续 Auto 判断。 | 功能关闭、历史不足、快速回填保护、摘要失败或重试再次超窗均可使原错误继续暴露。 | 三轮 Provider 重试仍超窗时升级摘要；compact 失败不重置 Soft/Hard 周期标志，成功 compact 重置。 |

Reactive 的保护还分两层：**同一 model step 的一次性标志**阻止成功摘要后无限重入；**快速回填保护**会在完整 compact 后不足 3 个已完成工具批次、且连续快速再触发达到 3 次时拒绝再次压缩。初始普通请求的超窗可能来自抛错，也可能来自无 tool call、非 output-token continuation 的超窗 finish reason；媒体附件自身过大走媒体错误路径，不能混作 Reactive context-exceeded。[步骤恢复门](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts) · [快速回填跟踪](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-loop-state.ts) · [模型错误分类](../../../apps/zcode-cli/packages/core/src/runtime/helpers/model-errors.ts)

上述状态表也解释了为什么不能把两种方案的兜底都画成“回到上一个可用点”。ZCode 的可用点由**当前 entries 上的一次新摘要提交**形成；自研路径显式区分 `H_ok` 与 `Δ_now`。比较时应验证这两种构造是否保存同样的任务约束、已读文件状态及未闭合工具调用，而不仅看最终 token 数。

**Prompt Cache 与恢复后的再次裁剪。**任一方案只要改变先前工具结果的模型可见内容，下一次请求从首个被改结果起就不再是相同消息前缀；它之前未变的前缀仍可能复用，实际 cache-read token 必须由 Provider usage 观察。ZCode 的 Microcompact 只影响活跃 Runtime，冷恢复从原 tool part 重建时可能重新膨胀。自研方案也保留原始 transcript，而档位标志只在 Runtime 内；若新 Runtime 从原文重建，系统须在新周期重新计算压力与候选，不能把上个 Runtime 的 40%/60% 动作当成已经永久应用。相反，若已发生成功完整 compact，两侧都应以各自持久摘要边界来定义有效历史。这里比较的是状态推导与实现约束，不是缓存命中率的实测结果。[ZCode 请求投影与 cache marker](../../../apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts) · [冷恢复](../../../apps/zcode-cli/packages/core/src/runtime/methods/resume.ts)

## 5. 设计洞察与汇报结论

1. **信息保留与空间确定性。**ZCode 的整条替换迅速移除旧结果正文；自研方案用 12/8-turn 保护窗口、Soft 头尾各 1,500 与 Hard 头部 1,000 字符，把结果保留量作为可控变量。前者的回收量更可预测，后者保留更多原始证据，但提示文本会占用一部分预算。
2. **摘要调用时机与请求成本。**ZCode 在 Reactive 入口直接进入完整摘要；自研方案先给工具结果最多三轮“裁剪 → Provider 重试”机会。局部治理成功可避免摘要模型调用；若三轮均失败，则额外 Provider 请求与状态管理成本必须计入比较。
3. **当前任务连续性。**ZCode 从失败请求直接分组，保留最近轮；自研方案明确区分前一轮正常消息与本轮增量。两种表达都必须保证工具调用与结果配对、当前用户输入及必要提醒在重建后仍可解释。
4. **可比较的实证指标。**建议同一任务回放时记录：每阶段释放的 token、模型可见工具内容比例、超窗恢复成功率、额外模型调用数和恢复耗时、后续任务正确率、Prompt Cache read token，以及冷恢复后的上下文变化。机制图只回答“如何工作”，实验才能回答“哪种更好”。
5. **裁剪提示本身的预算。**自研 Soft/Hard 的解释性标记有可读性价值，但也是模型输入。特别是 Hard 处理刚超过 1,000 字符的结果时，提示可能使文本净长度增加；这类结果应在报告指标中单独统计，而不只报告“触发了 Hard 的条数”。

汇报时可按“问题与两级时机 → 图 1 的信息保留策略 → 图 2 的超窗恢复路径 → 四项设计洞察”展开。两张图只放决策链；本报告保留算法边界和源码依据，供讲者回答实现细节。
