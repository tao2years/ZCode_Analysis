# 源码研究证据台账

基线：`872ad960de7ec172591f7e1952f7849229f94521`。除 E15 外，本页均为源码确认；已执行一组外围服务层测试，但未运行应用或真实 Provider。行号只对该基线有效。

## E01：Renderer 命令身份、恢复线索与权威投影分离

- `packages/ui/src/v4/commandFactory.ts:14-69`：clientId 在客户端稳定保存，commandId 使用 UUID v7；CAS 和 row targeting 命令要求 revision/epoch。
- `packages/ui/src/v4/SessionPane.tsx:1395-1510`：发送前记录 pending command 和 optimistic command；accepted/duplicate 后等待会话投影出现该 commandId。
- `packages/ui/src/v4/pendingCommandRegistry.ts:1-10,85-95`：账本仅保存恢复线索，不参与 conversation projection，也不自动重放。
- `packages/ui/src/v4/conversationProjectionStore.ts:1264-1295`：权威 queue/userInput 出现 sourceCommandId 后清理 overlay。

支持：ACK 与 UI 权威显示是不同边界。限制：未在真实刷新/丢 ACK 场景运行。

## E02：UI 服务代理与传输

- `packages/ui/src/v4/V4ConversationContext.tsx:120-169,278-304`：从作用域服务建立 conversation transport，并向 pane 暴露 sendCommand。
- `packages/ui/src/v4/agentConversationTransport.ts:350-370`：握手与能力检查后调用 `sendConversationCommandV4`。
- `packages/client/src/messageport.ts:25-37`：Desktop MessagePort 依次构造 protocol、ChannelClient 与 RemoteServiceAccess。
- `packages/desktop/src/renderer/src/main.tsx:305-307`：Renderer 注册 MessagePort 服务代理。
- `packages/web/src/main.tsx:445`：Web 使用 WebSocket 取得服务。

支持：`packages/client` 是 UI 到 Host 的服务访问层，而非模型 SDK。

## E03：Renderer 投影的一致性规则

- `packages/ui/src/v4/sessionDataLayer.ts:1-9`：per-session 数据层按 topic 持有投影 store，生命周期由租约引用计数控制。
- `packages/ui/src/v4/conversationProjectionStore.ts:653-720`：snapshot 整体替换；重复/迟到 frame 丢弃；seq 断档触发恢复；连续 delta 才应用并推进水位。
- 同文件 `:1213-1245`：accepted input 的 watchdog 只请求恢复，不重放 command。

支持：Renderer 保存只读投影和有限 overlay，不拥有 CLI 会话事实。

## E04：Host 作用域、进程与 stdio 转发

- `packages/desktop/src/host/index.ts:1964-2024`：Host 在每个 attachment 上建立 connection-scoped 服务并经 MessagePort 暴露。
- `packages/services/src/zcode-agent/zcodeAgentService.ts:5026-5089`：V4 命令入口补充可信 clientMode、可选浏览器上下文，然后发 `V4_METHODS.command`。
- `packages/services/src/zcode-agent/zcodeAgentProcessManager.ts:835-875`：按 workspaceKey 复用客户端，并去重在途启动。
- 同文件 `:1019-1037,1083`：启动 CLI 子进程、连接 stdio transport、创建协议 client。
- `packages/services/src/zcode-agent/zcodeAgentService.ts:2050-2099`：返回通知经校验分发到 conversation 等 emitter。
- `packages/desktop/src/main/desktopWindowLifecycle.ts:136-165`：Renderer reload 复用现有窗口 Host，重新 attachment。

支持：Main/Host 管生命周期和路由，不是 AgentRuntime 会话状态 owner。

## E05：workspace 身份与手机 attachment

- `packages/ui/src/hooks/useWorkspaceServices.tsx:1-110`：UI 按本地/远程 workspace 元数据选择作用域服务；断连远程目标返回拒绝命令的代理。
- `packages/desktop/src/main/desktopRemoteSessions.ts:832-886`：手机远控 attachment 校验 session、窗口、workspacePath、workspaceIdentity 和 workspaceKey，再取已有窗口 Host。
- `packages/services/src/zcode-agent/zcodeAgentService.ts:5055-5063`：远程身份存在时移除仅本地可用的 TTFT 字段。

支持：workspaceIdentity 是身份隔离的一部分；手机路径在该 attachment 函数中没有另起 Host/CLI。

## E06：协议 CommandInbox 的接纳职责

- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts:566`：V4 command 转给 gateway。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/command-inbox.ts:82-211`：固定 key gate → per-session gate，重新查询已有事实、分配 admissionSeq、pin 后返回 execute outcome；settle 释放 session gate。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/v4-gateway.ts:2360-2470`：处理命令、持久化输入接纳事实并调用 host executor。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/commands/prompt-turn.ts:103-160`：调用 app.sendInput，区分 rejected/queued，并异步观察 completion。

支持：这层负责幂等和权威接纳顺序，不等于持锁等待长任务完成。去重表和各异常 settle 尚待专题核验。

## E07：运行时接纳与回合所有者

- `apps/zcode-cli/packages/bootstrap/src/app/input-facade.ts:191-220`：sendInput 进入 runtime.admitPrompt。
- `apps/zcode-cli/packages/core/src/runtime/methods/prompt-admission.ts:20-128`：busy 时选择 reject、steer、deferred queue；启动输入先 reserve，再加入可取消 command queue。
- `apps/zcode-cli/packages/core/src/runtime/methods/runtime-command-queue.ts:185-210`：runtime command 调用 executeTurnCommand 并携带 foreground abort signal。
- `apps/zcode-cli/packages/core/src/runtime/agent-runtime.ts:132-335`：runtime 持有 session、history、tools、permissions、stores、command queue 和 active turn 等状态。

支持：协议 admission 与会话执行 admission 是两层。

## E08：持久化记录和运行时历史

- `apps/zcode-cli/packages/bootstrap/src/app/create-app.ts:726-736`：AgentRuntime 同时注入默认内存 eventStore 与 sessionStore。
- `apps/zcode-cli/packages/core/src/runtime/methods/message-persistence.ts:29-166,270-363`：用户输入、promotion、assistant message/part 分别持久化。
- `apps/zcode-cli/packages/core/src/agent/message-history.ts:26-74,135-220`：运行时历史由 model message entry 和 runtime attachment entry 等组成。

支持：持久 transcript、运行时历史和事件流不是一个对象。限制：冷恢复选择规则尚未深入。

## E09：每步请求重建与模型边界

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:43-90`：每次循环先检查取消，再做 microcompact/auto compact。
- 同文件 `:168-213`：从 request entries 生成 Provider projection，记录模型请求状态并执行一个 model-backed step。
- `apps/zcode-cli/packages/core/src/runtime/helpers/runtime-provider-request-messages.ts:9-23`：是否保留中途 system message取决于 runtime 配置和模型能力。
- `apps/zcode-cli/packages/core/src/runtime/methods/model.ts:119-235`：构造请求并进入 generateText/streamText。
- `apps/zcode-cli/packages/adapters/src/model/model.ts:72-90` 与 `runner-runtime.ts:1-58`：adapter prepare 后调用 AI SDK runtime。

支持：模型输入是运行时历史的投影。限制：尚未核验各 Provider 的最终 wire payload。

## E10：工具的流中执行、结果汇合和表示差异

- `apps/zcode-cli/packages/core/src/runtime/methods/streaming-tool-coordinator.ts:73-100,339-390`：只有满足条件的调用才在流中启动。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts:628-750`：模型流结束后 drain 流中结果并执行/汇合工具调用。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts:223-260`：流中与其余结果按 toolCallId 合并，再按原 `coreToolCalls` 顺序取回。
- 同文件 `:327-366`：UI/log error 与 modelContent 分开保存；模型历史写入 modelContent。

支持：完成顺序、回灌顺序和 UI 展示内容不能混同。限制：并发资格、权限、取消、大输出未完成专题研究。

## E11：压缩闭环的初步位置

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:67-90`：请求前主动压缩入口。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts:753-790`：context exceeded 后的响应式恢复入口。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact.ts:258-286`：执行 active compact 并替换本轮 request entries。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts:484-626`：获得摘要、持久化摘要和边界事件、替换 MessageHistory、返回 post-compact entries。

支持：压缩跨摘要持久化、事件、运行时历史和当前请求。限制：尚未核验历史选择、失败恢复、重启和 Provider reasoning。

## E12：模型单步与 product turn 结束不同

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-stop.ts:180-236`：无工具时可消费 inline guide，Stop Hook 也可让同一 turn 继续；最终才 break。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn.ts:614-655`：循环退出后先结算和持久化稳定边界，再发布 TurnComplete。

支持：不能用“模型没调用工具”直接判定整个用户任务结束。

## E13：连续与可恢复交付 profile

- `packages/shared/src/zcode-protocol-v4/core.ts:22-59`：continuous/replayable 定义不同 flush、stream paths、output cap 和 toolProgress。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/conversation-topic-publisher.ts:703-780`：订阅选择 profile；按 logEpoch/seq 判断 snapshot 或 resume，并对重放 delta 做 profile filter/coalesce。
- `packages/ui/src/v4/conversationProjectionStore.ts:695-720`：客户端依据连续水位应用或恢复。

支持：两种客户端消费同一权威执行事实，但交付语义和成本不同；replayable 的 output stream cap 为 0 不等于没有最终工具结果。

## E14：系统指令、Skills 与记忆的初步注入

- `apps/zcode-cli/packages/core/src/context/builder.ts:82-216`：按稳定 system、动态 system、Skills、项目指令/记忆索引等构造 sections，最终分别组装 system messages 与 meta-user attachments。
- `apps/zcode-cli/packages/core/src/context/sections/skills.ts:17-56`：Skills 元数据受预算限制并作为 meta_user 注入。
- `apps/zcode-cli/packages/core/src/context/sections/request-user-context.ts:15-70`：项目指令与 memory index 汇合成 meta_user context。
- `apps/zcode-cli/packages/core/src/context/sections/memory.ts:8-50`：文件型记忆使用规范进入 system。

支持：指令并非单一字符串，Skills 元数据和完整 Skill 内容也不是同一加载阶段。限制：发现、显式加载、写入和召回链尚未深入。

## E15：本轮命令验证结果

- `node scripts/check-workspace-freshness.mjs`：通过，main 与 origin/main 同步。
- `corepack pnpm --version` → `10.33.2`；`node --version` → `v24.14.0`，与 `mise.toml` 一致。
- 初次 `corepack pnpm typecheck`/`lint` 因缺少 `node_modules` 退出 1。随后 `corepack pnpm install --frozen-lockfile` 成功；可选的 `cpu-features`/`ssh2` 原生模块因本机缺少 C++ 编译环境未构建。
- 安装后 `corepack pnpm typecheck`：退出 0。
- 安装后 `corepack pnpm lint`：退出 0，70 warnings、0 errors；这些 warning 位于既有源码，本批未修改产品源码。
- `corepack pnpm fmt:check`：退出 1，报告 2803 个全仓文件格式问题，包含大量既有源码和本批 Markdown。该全仓结果只证明当前 checkout 不满足格式检查，不能单独归因于本批。
- `node --import tsx --test packages/services/test/nonCliAcpRetirement.test.ts`：退出 0，5/5 通过。其 Project Memory 用例验证服务层能列出和读取当前 memory 文件；会话用例验证 Desktop 与 replayable 投影恢复。它们不覆盖 CLI Memory extraction、compact、工具调度或大输出链路。
- 根包与 `apps/zcode-cli` 均无统一 `test` script；测试文件检索未找到直接覆盖本批 CLI 核心机制的测试。

这些命令支持仓库静态构建一致性和两个外围契约。它们不是目标应用、真实 Provider 或本批核心异常路径的运行验证。

## E16：会话生命周期与命令幂等

- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/commands/handlers/session-mgmt.ts:23-172`：新会话先为 deferred；首输入走普通 prompt-turn；delete 语义为 close/cleanup，并非物理删除 transcript。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/command-inbox.ts:107-300`：in-flight/live/settled 三类事实、双重 exact lookup、per-session gate、admissionSeq 与 live input pin。
- 同文件 `:284-451`：exact lookup 跨内存和持久事实，row 命令检查 epoch/revision。
- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/cold-session-resume.ts:34-110`：冷恢复 single-flight，区分 not-found 与 resume-failed。

支持：[A1](../topics/a-session-admission-agent-loop.md) 的会话与协议准入结论。

## E17：Runtime 准入与 foreground 取消

- `apps/zcode-cli/packages/core/src/runtime/methods/prompt-admission.ts:26-130`：按 active/steerable/attachment 和 queueDelivery 选择 start、guide、queue、reject。
- `apps/zcode-cli/packages/core/src/runtime/methods/runtime-command-queue.ts:415-475`：foreground execution ID、expected ID stop 与内部抢占时保留 auto-drain。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn.ts:624-704`：assistant 持久后 TurnComplete；成功后调度 Memory extraction。

支持：协议 admission、runtime admission 和 foreground execution 是不同边界。

## E18：工具调度、权限、超时和结果闭合

- `apps/zcode-cli/packages/core/src/tool/scheduler.ts:48-105,233-262`：依赖拓扑、默认并发 10、元数据并发资格。
- `apps/zcode-cli/packages/core/src/tool/executor/batch-runner.ts:24-142`：逐组执行、组内并发；仅 stopTurnAfterResult 跳过后组，并为其余调用生成 ToolCancelled。
- `apps/zcode-cli/packages/core/src/runtime/methods/streaming-tool-coordinator.ts:339-361`：流中提前执行的完整资格条件。
- `apps/zcode-cli/packages/core/src/tool/executor/permission-flow.ts:52-430`：规则、hook/client race、修改输入重验、project/session updates。
- `apps/zcode-cli/packages/core/src/tool/executor/timeout.ts:16-227`：可暂停 deadline、父 abort 与调用级 override。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts:149-408`：结果按 ID 合并、原序取回，UI part 与 modelContent 分离，取消在 sibling 闭合后生效。

支持：[A2](../topics/a-tool-scheduling-and-results.md)。限制：未运行流式 Provider 和取消时序。

## E19：消息与 Provider 输入投影

- `apps/zcode-cli/packages/core/src/context/builder.ts:82-306`：system sections、Skills 与 meta-user attachments 的顺序；工具 schema 不复制到 prompt。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:100-213`：每步动态提醒、工具过滤、Provider projection 与 recordable projection。
- `apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts:27-203`：source tracking、attachment 因果重排、MCS、runtime metadata 剥离与 cache control。
- `apps/zcode-cli/packages/adapters/src/model/transform.ts:38-268`：AI SDK role/tool/media 转换及 Provider-specific reasoning 处理。
- `apps/zcode-cli/packages/core/src/runtime/methods/resume.ts:38-184` 与 `agent/session-history-hydrator.ts:45-255`：branch/compact 选择、Context 重载、tool result/reasoning history 恢复。

支持：[B1](../topics/b-message-context-and-provider-input.md)。限制：未捕获最终 HTTP payload。

## E20：完整 compact 与 microcompact

- `apps/zcode-cli/packages/core/src/compact/policy.ts:6-151`：有效窗口、输出 reserve、buffer、失败熔断与 token source。
- `apps/zcode-cli/packages/core/src/runtime/helpers/compact-selection.ts:30-205,211-407`：按 round 选择摘要/保留区，prompt-too-long 的重选与截断降级。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts:170-630`：summary 请求、media/context retry、summary/boundary 持久化和 runtime replace。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-persistence.ts:200-441`：中断 timeline 恢复、summary/reminder 原子式 best-effort rollback。
- `apps/zcode-cli/packages/core/src/agent/compact-session.ts:1-88`：冷恢复按 boundary 重插 preserved segment。
- `apps/zcode-cli/packages/core/src/compact/microcompact.ts:77-256` 与 `runtime/methods/microcompact.ts:24-115`：旧工具结果的局部清理、保护条件、runtime replace 和 boundary event。

支持：[B2](../topics/b-compaction.md)。限制：未运行长上下文和故障注入。

## E21：项目 Memory

- `apps/zcode-cli/packages/core/src/memory/project-root.ts:10-23` 与 `runtime/helpers/project-memory.ts:4-20`：workspace identity/path 作用域与启用条件。
- `apps/zcode-cli/packages/core/src/context/sections/memory.ts:8-50`、`request-user-context.ts:15-86`、`memory/index-content.ts:3-37`：system 使用规则、meta-user index 和索引上限。
- `apps/zcode-cli/packages/core/src/runtime/helpers/project-memory-extraction.ts:28-174`：成功 turn 后 active durable snapshot、后台受限 agent、最多 5 turns。
- `apps/zcode-cli/packages/core/src/memory/extraction.ts:42-303`：cursor/coalescing、直接写跳过、用户 prose 门槛与提取 prompt。
- `apps/zcode-cli/packages/core/src/memory/recall/manifest.ts:10-136`：最多 200 项、mtime 排序、frontmatter preview。
- `apps/zcode-cli/packages/core/src/tool/executor/memory-file-permission.ts:9-80` 与 `memory/origin-session.ts:8-52`：memory Markdown 权限与 origin session stamping。

支持：[B3](../topics/b-memory.md)。限制：未发现程序化 TTL 或跨 runtime 冲突仲裁；未运行提取。

## E22：大工具结果

- `apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts:33-217,294-348`：默认 100KB、truncate/artifact、写失败回退、2K preview。
- `apps/zcode-cli/packages/core/src/tool/result-persistence-format.ts:1-57`：persisted-output envelope。
- `apps/zcode-cli/packages/adapters/src/storage/index.ts:41-169,476-496`：session-scoped artifact path/URI 与文本/字节读回。
- `apps/zcode-cli/packages/adapters/src/exec/output-collector.ts:1-176`：inline head、tail、on-truncate/always 落盘、单流与 aggregate 上限。
- `apps/zcode-cli/packages/core/src/tool/handlers/bash-model-content.ts:25-179`：Bash preview、background path 与 Read 提示。
- `apps/zcode-cli/packages/core/src/tool/handlers/read.ts:52-68`、`read-text.ts:25-57,170-245`：绝对 path 与分段读取。

支持：[B4](../topics/b-large-tool-results.md) 的第一轮结果链路；E26 对 POSIX Bash 与生命周期做了更精确的补充。限制：未启动目标 Agent 或验证磁盘失败及具体远端路径场景。

## E23：Reactive compact 的原步骤恢复

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-model-step.ts:375-439,499-524`：throw 与 finishReason 两种超窗入口；失败 assistant step 记录后进入恢复，空响应被一般错误包装前先检查超窗。
- 同文件 `:742-806`：同一步只尝试一次，rapid-refill breaker，成功后重建 TurnMachine 并返回 loop `continue`。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact.ts:352-467`：从当前 request entries 投影，检查禁用/历史不足，compact 成功替换 entries，失败计数与取消分支。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts:267-480,630-685`：summary 内部 prompt/media 调整，与 Auto-only 最多三次外层操作重试。

支持：[B2](../topics/b-compaction.md) 对重试单位、失败分支和状态重建的解释。限制：Provider 错误分类的外部表现未被本次运行验证。

## E24：Session 冷恢复的异常收敛

- `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/cold-session-resume.ts:42-110`：同 session single-flight，区分 notFound 与 resumeFailed。
- `apps/zcode-cli/packages/core/src/runtime/methods/resume.ts:59-190,190-323`：先恢复会话/branch/read-state，再初始化 Context、收敛 compact timeline、hydrate history、恢复会话连续性状态与 SessionResumed。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-persistence.ts:200-287`：started/retrying 的 operation 有 boundary 则 completed，否则 interrupted；修复后要求重载首帧数据。
- `apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts:111-194`：完成结果保留媒体或旧 output，错误优先 modelContent，未完成工具补中断结果。

支持：[A1](../topics/a-session-admission-agent-loop.md) 和 [B2](../topics/b-compaction.md) 的冷恢复结论。限制：不代表自动重放执行中的外部工具副作用。

## E25：Project Memory 并发保护的强度

- `apps/zcode-cli/packages/core/src/memory/extraction.ts:85-180`：调度器状态与 cursor 仅在单个 scheduler 实例内，串行并合并 latest pending。
- `apps/zcode-cli/packages/core/src/runtime/helpers/project-memory-extraction.ts:32-80`：每个 runtime 懒建自己的 scheduler，snapshot 读取 active durable branch。
- `apps/zcode-cli/packages/core/src/tool/handlers/write.ts:80-145,280-335` 与 `edit.ts:420-525`：完整 Read、陈旧检查、expectedRevision 和 atomic write。
- `apps/zcode-cli/packages/adapters/src/fs/index.ts:292-320,473-485,700-735`：Node FS 在写前 stat 校验 revision，再独立执行 atomic rename；检查与 rename 之间没有此端口内的跨进程锁。

支持：[B3](../topics/b-memory.md) 的乐观防护与跨 Runtime 边界。竞争时序是源码推断，不能宣称实际发生过覆盖。

## E26：大输出完整性与 artifact 生命周期

- `apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-run.ts:69-100`：POSIX Bash merged-output 用 BashFileOutput，其他 pipe 流用 OutputCollector。
- `apps/zcode-cli/packages/core/src/tool/handlers/bash.ts:70-71,420-429,479-495`：inline 30,000 bytes、persisted 5 GiB、前台 on_truncate/后台 always，以及通用 artifact result budget。
- `apps/zcode-cli/packages/adapters/src/exec/bash-file-output.ts:14-66,130-185` 与 `node-execution-adapter-results.ts:94-160`：直接文件写入、读 preview、按模式保留/删除和可选截断。
- `apps/zcode-cli/packages/contracts/src/interfaces/tool-artifact-store.port.ts:7-35,92-130`：retention 枚举和 read/write/stat port，但无删除/过期方法。
- `apps/zcode-cli/packages/bootstrap/src/app/create-app.ts:348-357,395-403` 与 `adapters/src/storage/index.ts:41-169`：默认 artifact/exec 两个根，Node artifact store 写入不使用 retention 字段。

支持：[B4](../topics/b-large-tool-results.md) 的逐层可恢复性与默认存储生命周期边界。不能将“未在该 port/store 发现清理”扩大成仓库外绝无清理。

## E27：上下文来源、预算、媒体与缓存

- `apps/zcode-cli/packages/core/src/runtime/methods/context.ts:40-90`、`context/builder.ts:130-306`：Context 初始化快照与 system/meta-user 分区。
- `apps/zcode-cli/packages/core/src/context/sections/skills.ts:8-65`、`tool/handlers/skill.ts:17-62`、`memory/index-content.ts:1-37`：Skills metadata、按需加载正文和 Memory index 上限。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:105-213`：每步工具筛选、动态 reminder、Provider/recordable projection。
- `apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts:5-76`、`compact/policy.ts:6-151`、`runtime/methods/compact.ts:184-340`：输出 preflight 与 auto compact 两种预算。
- `apps/zcode-cli/packages/core/src/runtime/helpers/media-budget.ts:28-141,188-260`：40 MiB 媒体预算、当前用户媒体保护与历史媒体文字占位。
- `apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts:286-334`、`adapters/src/model/transform.ts:499-513`：ephemeral marker 与 Anthropic provider options 映射。
- `apps/zcode-cli/packages/core/src/runtime/methods/context-usage.ts:119-254`：usage breakdown 将 schema 单独统计，并明确估算置信度。

支持：[B5](../topics/b-context-management.md)。限制：不证明最终 Provider tokenizer 计数或实际 prompt cache 命中。

## E28：Microcompact 的持久边界与诊断 JSONL

- `apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts:73-102`：替换 canonical MessageHistory 和当前 request entries，发 MicrocompactBoundary event；没有保存修改后的 Session tool part。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts:286-345`：工具完成时持久化原 tool part output/媒体元数据。
- `apps/zcode-cli/packages/core/src/runtime/methods/events.ts:81-108,263-280,420-423`：事件先进入 eventStore；durable Session 分支不包含 MicrocompactBoundary。默认 eventStore 创建见 `apps/zcode-cli/packages/bootstrap/src/app/create-app.ts:727-731`。
- `apps/zcode-cli/packages/core/src/runtime/methods/resume.ts:91-174` 与 `agent/session-history-hydrator.ts:149-178`：冷恢复从 SessionStore 消息 hydrate tool result，没有回放 microcompact event；完整 compact boundary 仍会限制有效历史。
- `apps/zcode-cli/packages/adapters/src/storage/session-store.ts:1-6`、`model/runner-debug.ts:34-46,470-535,627-755`：权威 SessionStore 使用 SQLite；model-io JSONL 是另一路有界、可轮转/缩减的请求诊断记录。

支持：[B5](../topics/b-context-management.md) 的逐层可见性。限制：这是当前默认组合和已追踪生产路径的静态结论；自定义注入的 eventStore 可有不同持久性，诊断日志是否仍含某次原文取决于记录时机与保留策略。

## E29：三种 Session 相关上下文入口

- `apps/zcode-cli/packages/core/src/session-context/references.ts:5-27` 与 `runtime/methods/turn.ts:863-872`：真实用户输入出现 `#sess_*` 时附加读取提醒，但不自动展开旧 Session。
- `apps/zcode-cli/packages/core/src/tool/handlers/read-session-context.ts:38-143,146-179,215-320`：按 sessionId 读取另一 Session 的持久消息，按 query/strategy/预算选取；可调用独立轻量模型提取，失败回退本地内容。
- `apps/zcode-cli/packages/core/src/session-context/read-session-context.ts:54-100,241-330`：先按 active branch/compact boundary 选历史，再评分、分块和限额。
- `apps/zcode-cli/packages/core/src/runtime/helpers/project-memory.ts:5-28`、`runtime/helpers/project-memory-extraction.ts:32-80`：Project Memory 是 workspace 作用域的文件式长期事实与后台提取，独立于跨 Session transcript 读取。
- `apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts:40-63`：`CompactTrigger.SessionMemory` 仅见 phase/reason 映射；对 core 生产调用点的检索未发现该 trigger 的发起者，不能据枚举宣称已启用会话记忆压缩。

支持：[B5](../topics/b-context-management.md) 对当前会话恢复、跨 Session 读取和 Project Memory 的区分。限制：缺少外部插件或未来调用者的证明，未运行目标 Agent/Provider。

## E30：Microcompact 与 Auto compact 的可计算判断和消息分组

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:67-103`：每个 model step 先试 Microcompact，再试 Auto compact。
- `apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts:90-150`、`runtime/helpers/provider-request-messages.ts:45-104`：runtime entries 投影为模型请求消息，Microcompact 回写对应 runtime entry。
- `apps/zcode-cli/packages/core/src/compact/manual.ts:102-140`、`packages/shared/src/usage-stats.ts:9`：字符估算的组成和除数。
- `apps/zcode-cli/packages/core/src/compact/microcompact.ts:77-168,171-256`、`runtime/methods/microcompact.ts:24-115`：空闲/Token 双触发、默认参数、工具批次分组、保留最近组、最低收益以及 runtime 历史替换。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact.ts:184-245,313-349`、`compact/policy.ts:1-155`：Auto 阈值、Provider usage 基线和失败门槛。旧工具结果已包含在 usage 基线时，Microcompact 的节省不会被 Auto 的增量估算反向扣除；这是两条计数路径的条件性静态推论。
- `apps/zcode-cli/packages/core/src/compact/rounds.ts:1-33`、`runtime/helpers/compact-selection.ts:30-60,350-362`：Auto 按新 assistant 消息切组；开头 user 消息可能单独成组，最近组原文保留。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact-active.ts:180-238,267-305,308-480,517-625,635-690`、`compact/prompt.ts:1-165`：摘要输入、Prompt、两层重试、持久化和历史替换。

支持：[B5](../topics/b-context-management.md) 的 M1–M6/A1–A6 小流程、阈值演算及分组实例。限制：例子使用假设消息和数字，没有运行目标 Agent 或 Provider；Provider 实际 tokenizer/cache 行为仍未知。

## E31：Microcompact 的逐步动作与 Prompt Cache 边界

- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:67-103,168-213`：每个 model step 前检查 Microcompact；Auto compact 与动态提醒后才构造实际 Provider 请求。
- `apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts:90-150`：Microcompact 以 `applyCacheControl:false` 投影内存 entries；成功后只把选中 tool result 的 content 按 ID 回写 runtime entries。
- `apps/zcode-cli/packages/core/src/compact/microcompact.ts:77-168,171-256`：先估算、触发、分候选组、保留最近组、替换、重估节省；低于最低收益恢复原消息。已清标记被排除，不会在下一步重复清理。
- `apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts:73-115`：成功才替换 canonical history 与当前 request entries；跳过时只记录 debug。
- `apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts:45-99,299-334`、`apps/zcode-cli/packages/adapters/src/model/transform.ts:499-513`：最终请求重投影后设置 ephemeral cache marker，再映射到 Provider 选项。

支持：[B5](../topics/b-context-management.md) 的连续两步例子和缓存影响。缓存失配从首个被替换结果开始是请求内容对比的静态推论；具体 Provider 命中率和计费需运行数据，当前未实测。

## E32：Microcompact 的实现级裁剪算法

- `apps/zcode-cli/packages/core/src/compact/microcompact.ts:8-40,77-168,171-256`：默认阈值参数、触发优先级、候选工具白名单、结果筛选、按 assistant tool-call 消息划组、保留最近合格组、整条 content 替换为 33 字符常量，以及节省不足时全部回滚。没有单条结果的 preview、头尾保留或中段裁剪。
- `apps/zcode-cli/packages/core/src/compact/manual.ts:102-140`、`packages/shared/src/usage-stats.ts:9`、`apps/zcode-cli/packages/contracts/src/model/index.ts:420-445`：每条消息按内容文本与 tool-call 入参长度除以 3 估算；reasoning 和媒体内容的文字表示有独立规则。
- `apps/zcode-cli/packages/core/src/compact/policy.ts:6-99`、`runtime/methods/microcompact.ts:24-115`：Auto 阈值参数如何派生 Micro 阈值、配置覆盖、成功时写入哪两份运行时状态。
- `apps/zcode-cli/packages/core/src/runtime/helpers/compact.ts:90-150`：按 toolCallId 将投影上已清结果回写 runtime tool entry；其他消息、工具调用参数不改。

支持：[B5](../topics/b-context-management.md) 的实现级伪代码与 900 字符结果的可复算例子。例子是基于源码规则构造的假设数据，未运行目标 Agent 或 Provider。

## E33：大工具结果的两层可复现预算

- `apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts:33-217,294-348`：formatter 后按 UTF-8 bytes 与可选 JS 字符阈值判定；默认 100,000 bytes/head/truncate；artifact 成功存完整 handler 后文本，失败通常走有界截断，但显式字符阈值有保留原文的例外。
- `apps/zcode-cli/packages/core/src/tool/executor/result-content-projection.ts:181-235`、`core/src/tool/result-persistence-format.ts:1-70`：截断后缀优先占预算，余下按 code point 保留最长 head/tail；artifact preview 默认前 2,000 JS 字符、可在后半段换行处收束。
- `apps/zcode-cli/packages/core/src/tool/handlers/bash.ts:420-429`、`adapters/src/exec/output-collector.ts:43-175`、`adapters/src/exec/bash-file-output.ts:145-185`、`adapters/src/exec/node-execution-adapter-results.ts:94-160`：Bash 30,000 bytes inline 与 5 GiB 持久限额；pipe 和 POSIX 直写采用不同采集路径；`truncated` 与 `artifactTruncated` 指向不同层。

支持：[B4](../topics/b-large-tool-results.md) 的 R1–R4 与 B1–B6 子流程和可复算裁剪。例子为静态推演，未执行目标命令。

## E34：相邻专题的决策顺序与消费者

- `apps/zcode-cli/packages/core/src/compact/rounds.ts:1-33`、`runtime/helpers/compact-selection.ts:30-139,167-205,240-330`、`runtime/methods/compact-active.ts:267-305,688-715`：完整 compact 的 assistant-started 分组、最近组保留、Auto/Reactive 摘要超窗重选，与 manual 等入口的丢最旧组兜底相互区分。支持 [B2](../topics/b-compaction.md)。
- `apps/zcode-cli/packages/core/src/runtime/helpers/provider-request-messages.ts:45-203,299-334`、`runtime/helpers/media-budget.ts:52-141,188-260`：runtime 到 Provider-neutral 投影顺序、来源映射、cache marker、40 MiB 媒体预算的保护与新近优先算法。支持 [B1](../topics/b-message-context-and-provider-input.md)。
- `apps/zcode-cli/packages/core/src/tool/scheduler.ts:48-185`、`tool/executor/batch-runner.ts:24-142`、`runtime/methods/turn-tools.ts:149-363`：工具依赖 level、并发安全分组、stop 与普通失败分离、按原调用 ID 顺序汇合。支持 [A2](../topics/a-tool-scheduling-and-results.md)。
- `apps/zcode-cli/packages/core/src/runtime/methods/prompt-admission.ts:26-130`：busy、guide、queue、reject、started 的条件顺序。支持 [A1](../topics/a-session-admission-agent-loop.md)。
- `apps/zcode-cli/packages/core/src/memory/extraction.ts:67-180,220-303`：运行中只保留最新 pending snapshot、cursor 推进条件、直接写入跳过与三词用户文本门槛。支持 [B3](../topics/b-memory.md)。

以上例子均标为假设性源码推演，不表示运行目标 Agent、Provider 或命令后的观察。

## E35：Microcompact 冷恢复反弹与超窗兜底边界

- `apps/zcode-cli/packages/core/src/runtime/methods/microcompact.ts:24-115`、`compact/microcompact.ts:77-256`：每个 model step 用本轮 entries 重新投影、计算 M 并筛选旧工具结果；成功仅回写运行时历史及本轮 entries。
- `apps/zcode-cli/packages/core/src/runtime/methods/resume.ts:130-190`、`agent/session-history-hydrator.ts:54-199`、`agent/compact-session.ts:4-46`：冷恢复从持久 Session tool part 重建有效历史及上次 assistant 完成时间；完整 compact boundary 会限制有效历史，Microcompact 的运行时替换不作为恢复后的内容来源。
- `apps/zcode-cli/packages/core/src/runtime/methods/turn-loop.ts:67-103`：下一次模型步按 Microcompact → Auto compact 顺序执行，恢复动作本身不执行普通 Provider 请求。
- `apps/zcode-cli/packages/core/src/runtime/methods/compact.ts:317-350`、`turn-model-step-usage.ts:54-65`：Auto 与 preflight 可用最近 assistant 的 Provider usage 加后续本地估算；若 usage 代表 Microcompact 后的较小输入，而冷恢复又带回更早原文，存在静态可推导的低估条件。
- `apps/zcode-cli/packages/core/src/runtime/methods/model-token-limits.ts:17-40`、`turn-model-step.ts:239-252,375-439,499-524,742-806`：preflight 只调 output cap，非正剩余时仍用 baseline；Provider context-exceeded 后尝试 reactive compact，但它可失败、跳过或受保护条件阻断。

支持：[B5](../topics/b-context-management.md) 的 M0–M8 决策图及恢复反弹说明。**尚未运行真实冷恢复或 Provider 超窗实验**；低估及最终报错是源码条件性推论，不宣称必然发生。
