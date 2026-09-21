# 研究进度与续做记录

## 当前目标与已确认范围

- 系统入口：分析当前 ZCode 仓库。
- 当前批次：用户已确认 A、B 串行；在初版六篇专题后，又指定补 reactive compact、大输出完整可恢复性、Memory 并发、Session 冷恢复异常、artifact 生命周期和上下文管理全貌。现已补入对应专题并新增 B5。
- 用户进一步澄清：本研究为学习目标仓库的思想、技术方案和设计取舍，不以启动代码、真实 Provider 或故障注入为完成条件。
- 质量取舍：先完成 A 的会话/Agent Loop/工具，再完成 B 的消息/压缩/记忆/大结果，并回写 Overview。
- 基线：`872ad960de7ec172591f7e1952f7849229f94521`，`main`。

## 已完成

- 从 Renderer `sendText` 追到 UI transport、Host 服务、stdio、CLI CommandInbox 和 runtime admission。
- 从 turn loop 追到请求投影、模型 adapter、工具结果汇合、guide/Stop Hook 和 TurnComplete。
- 区分 transcript、运行时历史、模型输入和 UI 投影。
- 完成 proactive/reactive compact、microcompact、持久化边界和冷恢复专题。
- 区分 desktop-continuous 与 web-remote-replayable，并确认手机 attachment 复用窗口 Host。
- 写入 index、Overview、synthesis、证据台账和 Skill 首次试用记录。
- 完成 A1/A2 与 B1–B5 七篇专题，覆盖会话恢复、权限与超时、reactive compact、Memory extraction/并发、大结果完整性及上下文全貌。
- 更正 B4 对 Bash 执行输出的过度概括：POSIX Bash merged-output 由子进程直接写文件，pipe 路径才由 OutputCollector 采集；详见 E26。

## 已确定结论

- ACK、model step completion、product turn completion 和 UI 权威投影是不同边界。
- CommandInbox admission 与 AgentRuntime admission 是两层所有权。
- 合格工具可在模型流中提前执行；已读汇合点按 ID 合并并按原调用列表取回。
- 无工具调用不必然结束 turn；guide/Stop Hook 可以续跑。
- 模型请求由 runtime entries 投影；UI 内容不能代表模型实际输入。

## 待继续的源码问题

1. Provider retry、reasoning 兼容及增量 tool-call 的 adapter 契约；只从当前源码能证明的边界陈述。
2. task index、event store、owner/lease 与跨 Host 路由的系统设计，以及它们与 A/B 的关系。
3. 子 Agent、Hooks、Skills 等扩展机制如何沿用或改变会话、权限、上下文契约。
4. 外部 Provider 精确 token/cache 行为、仓库外 artifact 清理及具体远端部署路径无法由当前源码单独证明，作为证据边界保留，不视为本批未完成。

## 验证和环境

- freshness：通过。
- Node `v24.14.0`、corepack pnpm `10.33.2` 与仓库配置一致。
- 首次 `typecheck`/`lint` 因没有 `node_modules` 失败；随后 `corepack pnpm install --frozen-lockfile` 成功。可选的 `cpu-features`/`ssh2` 原生加速因缺少本机 C++ 编译环境未构建，但未阻止安装。
- 安装后 `corepack pnpm typecheck` 通过；`corepack pnpm lint` 退出 0，报告 70 条既有 warning、0 error。
- `corepack pnpm fmt:check` 是全仓检查，退出 1 并报告 2803 个文件格式问题，包含大量既有源码；该结果不能归因于本次文档，也不能记为通过。
- `node --import tsx --test packages/services/test/nonCliAcpRetirement.test.ts` 通过 5/5。其中 Project Memory 用例只支持服务层目录/文件可读；另一个用例支持 Desktop 与 replayable 的当前会话恢复投影，不覆盖 CLI Memory extraction、compact、工具调度或大输出链路。
- 仓库未提供统一 `test` script；源码检索未发现直接覆盖本批 CLI 核心机制的测试文件。
- 尚未运行目标应用或真实 Provider。
- 中断前已有两个只读研究子任务分别核对 CLI runtime 与 Host 链；主研究者已抽查其关键源码定位并合并。它们未修改文件。

## 下一步

1. 本批指定的六项补充已收束；若继续扩展，选择 owner/lease、Provider adapter 或子 Agent 中一个独立专题。
2. 新专题完成时复用现有全景和证据编号，并回写 Overview 与工程借鉴。

恢复时先运行 freshness、检查 `git diff -- research/code-agent skill-drafts`，确认源码基线是否变化。
