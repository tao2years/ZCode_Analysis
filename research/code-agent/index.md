# ZCode 源码研究索引

本批目标：通过源码学习 ZCode 的运行思想、技术方案和设计取舍，并按用户确认的顺序研究 A“会话/Agent Loop/工具结果”和 B“消息/上下文/压缩/记忆/大结果”专题。A、B 已做静态源码覆盖，现正按用户新确认的“可据此实现同等关键行为”标准回补，修订尚待用户复核；其他模块尚未全仓研究。运行目标应用或真实 Provider 不是本次研究的完成条件。

- 源码基线：`872ad960de7ec172591f7e1952f7849229f94521`，分支 `main`。
- 研究日期：2026-09-21。
- 开始时工作区：只有未跟踪的 `skill-drafts/`；没有已跟踪源码改动。本批只新增研究文档和更新 Skill 验证记录。
- 研究方法：静态源码追踪；本阶段没有运行目标 Agent 或调用模型。
- 读者深度标准：2026-09-22 按用户反馈提升为“能据报告复现关键行为”；详见[研究读者与深度约定](research/reader-contract.md)。
- 使用入口：系统研究。按已确认流程，初步全景后确认一次深挖重点。

## 阅读路径

1. [整体运行模型](overview.md)：从用户输入到模型、工具、持久化与界面反馈。
2. A 阶段：[会话、输入准入与 Agent Loop](topics/a-session-admission-agent-loop.md) → [工具调度与结果闭合](topics/a-tool-scheduling-and-results.md)。
3. B 阶段：[消息与 Provider 输入](topics/b-message-context-and-provider-input.md) → [上下文管理全貌](topics/b-context-management.md) → [上下文压缩](topics/b-compaction.md) → [项目记忆](topics/b-memory.md) → [大工具结果](topics/b-large-tool-results.md)。
4. [系统收束与工程借鉴](synthesis.md)：专题证据修正后的系统性质、迁移条件与限制。
5. [读者与深度约定](research/reader-contract.md)、[证据台账](research/evidence.md)、[进度与未解问题](research/progress.md)、[Skill 试用记录](research/skill-trial.md)。

## 覆盖与状态

下表已阅读项均对应上述基线；“静态深挖完成”只表示源码覆盖，不代表已通过用户新粒度验收。未研究项不宣称有有效结论。

| 机制                                    | 深度                   | 状态         | 关键缺口                                 | 文档                                                 |
| --------------------------------------- | ---------------------- | ------------ | ---------------------------------------- | ---------------------------------------------------- |
| Desktop/UI → Host → CLI → Provider → UI | 主链路概览             | 初步全景完成 | owner/lease 与跨 Host 路由尚待深入       | [Overview](overview.md)                              |
| Command admission、会话与 Agent Loop    | 源码专题               | 静态深挖完成 | 重试、取消和恢复的跨层设计仍可扩展       | [A1](topics/a-session-admission-agent-loop.md)       |
| 工具调度、权限与结果闭合                | 源码专题               | 静态深挖完成 | Provider adapter 契约仍可扩展            | [A2](topics/a-tool-scheduling-and-results.md)        |
| 消息重建、Context 与 Provider 投影      | 源码专题               | 静态深挖完成 | 最终 wire 精确形态不在本批证据内         | [B1](topics/b-message-context-and-provider-input.md) |
| 上下文预算、跨 Session 按需读取、缓存与媒体投影 | 源码专题 | 已补主链路，待读者复核 | Reactive 此前埋在表格；当前已单独展开。Provider 精确 token 与 cache 命中未知；`SessionMemory` trigger 未见核心发起点 | [B5](topics/b-context-management.md) |
| 完整 compact、microcompact、reactive    | 源码专题               | 静态追踪完成，整体呈现待复核 | B2 有 Reactive 细节；B5 的发现路径此前不足，当前已补图。Provider 外部行为未验证 | [B2](topics/b-compaction.md)                         |
| Project Memory 与跨会话冲突             | 源码专题               | 静态深挖完成 | 无法由源码保证所有竞争时序的最终结果     | [B3](topics/b-memory.md)                             |
| 大结果可恢复性与 artifact 生命周期      | 源码专题               | 静态深挖完成 | 仓库外清理、具体远端路径可达性未知       | [B4](topics/b-large-tool-results.md)                 |
| Desktop/mobile 交付语义                 | 关键契约概览           | 待深入       | 完整 owner/lease、跨 Host 路由和故障注入 | [Overview](overview.md)                              |
| Skills 与项目指令                       | 关联机制概览           | 待专题       | 完整 Skill 加载、预算与冲突未深挖        | [B1](topics/b-message-context-and-provider-input.md) |
| Hooks、子 Agent、工作流、权限           | 识别关联入口           | 未专题研究   | 完整跨模块行为                           | [进度](research/progress.md)                         |
| 可观测性与测试                          | 工具环境与部分文件核查 | 未专题研究   | 诊断设计与测试覆盖思想                   | [进度](research/progress.md)                         |

## 下一批推荐

本批已补上下文预算、按需加载与缓存、Memory 并发、artifact 生命周期、reactive compact 和冷恢复异常。若继续扩展，优先研究 owner/lease 与远程重放、子 Agent 和扩展机制；它们目前不是本批完成范围。
