# 系统结论与工程借鉴

基线：`872ad960de7ec172591f7e1952f7849229f94521`。这里的工程借鉴是基于实现的分析推断，除非另有引用，不表示仓库作者明确表达的设计动机。

## 当前系统级认识

ZCode 的核心并不是一个孤立的“模型—工具 while loop”，而是一组相互扣合的边界：客户端命令身份与恢复、Host attachment 与 workspace 路由、协议接纳、会话执行接纳、多步模型循环、持久化以及可恢复的 UI 投影。局部代码只有放回这些边界，才能解释任务在刷新、排队、远控和压缩之后为何仍能保持一致。

当前看到的三个系统性质最值得保留：

1. **提交与观察解耦。** ACK 只承诺命令接纳结果，权威展示由投影通道给出。收益是传输重试和 UI 恢复不必伪造 session 事实；代价是必须维护 commandId、对账和断档恢复。
2. **多级 admission 各管一层。** CommandInbox 管协议幂等与顺序，AgentRuntime 管 busy 时的 start/steer/queue/reject。收益是协议一致性和产品调度可以分开演进；代价是取消、重启和错误必须明确跨层传播，否则容易出现“已接纳但未执行”的灰区。
3. **模型上下文是投影。** transcript、运行时 entries、Provider messages 和 UI rows 有意不同。收益是可以插入动态指令、压缩历史、适配 Provider 并控制展示；代价是恢复必须保存足够元数据，测试也必须跨转换层验证。

工具流中启动和多种交付 profile 进一步说明，系统在优化首响应和远程恢复成本。但这些优化把时序复杂度推向了结果汇合、迟到事件和客户端水位，因此不能只测试成功主路径。

## 已完成的 A/B 专题如何修正全景

### A：输入接纳到完整 Agent Loop

整体流程先讲清：`sendText → CommandInbox → runtime admission → model step → tool feedback → guide/Stop Hook → TurnComplete → UI projection`。

专题确认了三层不同的协作边界：CommandInbox 负责可重试命令身份和 per-session 顺序；runtime admission 决定 start/guide/queue/reject；foreground execution identity 约束 stop 不得取消新任务。工具侧也不是简单 `Promise.all`：调度器按依赖和副作用分组，结果按 ID 汇合、按原调用顺序回灌，每个 tool use 必须得到结果，普通失败与 turn stop 分开表达。详见 [A1](topics/a-session-admission-agent-loop.md) 和 [A2](topics/a-tool-scheduling-and-results.md)。

### B：消息重建、上下文与压缩恢复

整体流程先讲清：`持久记录 → 冷/热加载 → runtime entries → 动态注入 → Provider projection → 模型请求`，再单独展开压缩。

专题确认了 Context 是每步投影产物：持久 messages、runtime entries、Provider-neutral messages 与 wire 各有变换。完整 compact 以 summary + boundary 表达 active history，保留最近轮并支持冷恢复；reactive compact 在一次被拒绝的 model step 上替换请求状态后重试。Memory 只自动加载 index，事实按需读取；后台提取是受限 Agent，文件 revision 检查提供乐观防护而非跨 Runtime 事务。工具大输出既有 POSIX Bash 直接写文件路径，也有 pipe collector，再经过通用 result budget；只有仍被保存且可读的字节才能恢复。详见 [B1](topics/b-message-context-and-provider-input.md)、[B5](topics/b-context-management.md)、[B2](topics/b-compaction.md)、[B3](topics/b-memory.md)、[B4](topics/b-large-tool-results.md)。

## 后续专题顺序

在 A/B 之后，建议按发现选择：

- 子 Agent、workflow 与父会话的上下文/取消/结果汇合；
- Project Memory 在其他存储实现或部署中的跨会话冲突与清理策略；
- Desktop continuous 与 web replayable 的故障恢复、owner/lease 和跨 Host 路由；
- Provider retry、空响应、限流、reasoning 兼容和请求观测；
- Hooks、权限、MCP/Plugin/Skills 的统一扩展与安全边界。

这些目前只有入口识别，不能当作已经理解。

## 可迁移判断及条件

如果自研 Agent 需要桌面刷新恢复、手机查看、busy 输入和长任务，值得借鉴“命令信封 + 权威投影 + 水位恢复”的组合。前提是服务端/运行时有唯一事实 owner，并能为每条命令和投影建立稳定关联；仅在前端加 optimistic list 不足以复现这一性质。

如果要做工具流中提前执行，必须同时具备稳定 toolCallId、明确的 early-execution eligibility、结果去重、声明顺序汇合，以及取消后的迟到结果策略。ZCode 的静态实现已确认这些锚点，但尚未用真实 Provider 流验证增量 tool call 在所有 adapter 上稳定。

如果要做上下文压缩，应把摘要记录、运行时替换和 Provider 请求重建设计成显式边界，并为恢复保存模型实际可见的工具错误等元数据。最近原文保留区也需要持久 identity，不能只保留内存数组切片。

如果要做文件式记忆，建议保留先读后写与 revision 检查，同时明确接受“Agent 负责语义一致性”的代价，或补充真正跨进程的 CAS/锁与冲突合并。索引/body 分离适合可审计和按需加载，但不会自动解决陈旧、跨会话覆盖和 TTL。

如果要做大输出落盘，API 应分别暴露原始观测字节、模型返回字节、磁盘完整性、path/URI 与实际清理策略。ZCode 的 Node artifact store 接收 `retention` 意图却没有可见的删除/过期接口；一个布尔 `truncated` 或一个 `retention: session` 字段都不能单独证明内容可恢复或会按期清理。

## 当前不能下的结论

- 不能称所有工具都并行或都能在 stream 中执行。
- 不能称 ACK 意味着任务已经开始，更不能称任务已经完成。
- 不能称 UI rows 是模型完整历史。
- 不能称 compact 删除了原始数据库历史。
- 不能称手机拥有独立 Agent runtime。
- 不能称核心链路已通过测试或真实 Provider 验证。
