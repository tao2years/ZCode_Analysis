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
- 根据用户追问，B5 增补 Microcompact 的逐层持久边界、model-io JSONL 与权威 transcript 的区别，以及 `ReadSessionContext` 跨会话按需读取；证据见 E28、E29。同步更新 Overview、synthesis、index 和 Skill 的问题库/模板/证据标准。
- 再按用户反馈将 B5 的 Microcompact 与 Auto compact 展开为同页编号子流程，补投影定义、估算公式与来源、双触发、工具批次筛选、保留/撤销条件、摘要分组实例及失败去向；证据见 E30。Skill 增加“能按消息序列和参数复算决策”的验收标准。

## 已确定结论

- ACK、model step completion、product turn completion 和 UI 权威投影是不同边界。
- CommandInbox admission 与 AgentRuntime admission 是两层所有权。
- 合格工具可在模型流中提前执行；已读汇合点按 ID 合并并按原调用列表取回。
- 无工具调用不必然结束 turn；guide/Stop Hook 可以续跑。
- 模型请求由 runtime entries 投影；UI 内容不能代表模型实际输入。
- Microcompact 清理 canonical 运行时历史与当轮 request entries，不回写持久 Session tool part；默认边界 event 不作为冷恢复重放依据。JSONL model-io 有自己的缩减与轮转规则，不能充当权威 Session 备份。
- `#sess_*` 只触发提示，`ReadSessionContext` 才按需读取其他 Session；Project Memory 是独立的 workspace 文件记忆；`CompactTrigger.SessionMemory` 在核心实现中只看到映射分支，未见生产发起点。

## 待继续的源码问题

1. Provider retry、reasoning 兼容及增量 tool-call 的 adapter 契约；只从当前源码能证明的边界陈述。
2. task index、event store、owner/lease 与跨 Host 路由的系统设计，以及它们与 A/B 的关系。
3. 子 Agent、Hooks、Skills 等扩展机制如何沿用或改变会话、权限、上下文契约；包括 `ReadSessionContext` 的跨会话权限边界与 subagent persistent memory 的相互关系。
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
- 本轮文档/Skill 修订验证：`quick_validate.py` 在 `PYTHONUTF8=1` 下通过；更新文档的本地 Markdown 链接检查通过；`git diff --check` 通过；使用仓库固定版本 `corepack pnpm typecheck` 通过、`corepack pnpm lint` 退出 0（70 条既有 warning、0 error）。直接 `pnpm` 命中了宿主 pnpm 11 并因依赖目录版本不一致拒绝无 TTY 自动清理，随后改用项目锁定的 pnpm 10.33.2 成功；没有执行依赖清理。
- 本轮 Mermaid 直接 Node API 解析受 DOMPurify 的 Node DOM 初始化错误阻挡；所改主图只调整节点文案，不改箭头与语法结构。本轮不将它写成已重新渲染验证。
- 中断前已有两个只读研究子任务分别核对 CLI runtime 与 Host 链；主研究者已抽查其关键源码定位并合并。它们未修改文件。

## 下一步

1. 本批指定的六项补充已收束；若继续扩展，选择 owner/lease、Provider adapter 或子 Agent 中一个独立专题。
2. 新专题完成时复用现有全景和证据编号，并回写 Overview 与工程借鉴。

## 跨设备 handoff（2026-09-21）

- 本轮用户追问的两点：Microcompact 是否修改持久 transcript/JSONL；上下文全貌是否漏掉 Session memory。答案已落在 [B5](../topics/b-context-management.md) 的 00–01、02；E28/E29 记录了证据边界。此轮只做静态研究和文档/Skill 修订，未启动目标 Agent 或 Provider。
- 当前源码研究基线仍是 `872ad960de7ec172591f7e1952f7849229f94521`。研究文件与 Skill 版本以本次推送的 Git commit 为准；后续设备先 fetch/pull，再运行 `node scripts/check-workspace-freshness.mjs`，检查 `git status --short` 和基线差异。
- 继续提问时从 [index](../index.md) 进入，先读相关专题和 E28/E29，再按问题追上下游。若涉及新事实，更新同一专题、证据、Overview/index 和 Skill 中确实需要改变的规则；避免重新做完整研究访谈。
- 仍待验证：自定义 eventStore 的具体持久行为、真实 Provider token/cache 表现、外部调用者是否使用 `CompactTrigger.SessionMemory`。当前结论只限已追踪核心生产路径。

恢复时先运行 freshness、检查 `git diff -- research/code-agent skill-drafts`，确认源码基线是否变化。

## 跨设备 handoff 增补（2026-09-22）

- 用户本轮校准了研究粒度：复杂子模块应有短而可读的独立小流程图，同页讲清每一步；不能只写“估算/裁剪”，必须给公式、参数来源、分组实例、保留与回滚门槛，同时压缩重复散文。已落实于 [B5 的 02–03](../topics/b-context-management.md)、[E30](evidence.md) 和 Skill 的 `report-templates.md` / `evidence-and-depth.md`。
- 特别核对：Microcompact 用本地投影估算；Auto 优先用 Provider usage 基线加增量，因此 Microcompact 清掉已落在旧 usage 基线内的内容时，Auto 的触发计数未必同步下降。该点是源码推论，未作真实 Provider 验证。
- 本轮没有改目标仓库代码或运行目标 Agent。后续问答应先在对应专题内补可计算实例和源码边界，积累到适当时机再回写 Overview 与 Skill；不要把假设实例写成目标仓库运行事实。
- 本轮验证：freshness 通过；B5 的 3 个 Mermaid 图在本机 Chrome 中用仓库安装的 Mermaid 解析成功（未做截图级视觉检查）；Skill `quick_validate.py` 通过；本地 Markdown 链接与 `git diff --check` 通过；`corepack pnpm typecheck` 通过；`corepack pnpm lint` 退出 0，仍为 70 条既有 warning、0 error。
- 用户继续追问“开启后是否每轮都裁剪、哪些工具如何处理、Prompt Cache 是否受影响”。B5 的 02 新增连续两个 model step 的完整例子，明确每步检查、满足触发后扫描、收益够才写入的三层；并写出缓存前缀可能失配、marker 重设与实际命中未知的边界。证据见 E31；Skill 的深度验收同步加入连续步骤与缓存权衡要求。
- 用户指出仍不足以据描述复现实现。B5 的 02 进一步明确算法单位是**旧批次中合格 tool result 的整个 content**，无 preview/头尾保留/中间裁剪；新增保持源码判定顺序的伪代码，以及能算出 `867` token 节省的假设输入。E32 记录参数和回写证据；Skill 验收改为“熟悉 Agent 开发的读者可据此实现同样行为”。

## 主动深度复核与跨设备 handoff（2026-09-22）

- 用户要求把上述粒度**主动扩展到其他相关专题**，不再等待逐段提醒，并将偏好持久保存。已写入[读者与深度约定](reader-contract.md)、Skill 核心流程和 continuity；后续设备先读此约定。
- 已复核并修订 A1/A2/B1/B2/B3/B4：A1 补 busy/guide/queue/reject 的判定顺序；A2 补拓扑 level、并发组和按 ID 汇合；B1 补 Provider 投影步骤与 40 MiB 媒体选择；B2 修正 Auto/Reactive 摘要超窗和 manual 丢旧组的差异；B3 补 memory extraction 的 pending 合并/cursor 场景；B4 补通用 UTF-8 字节裁剪、artifact preview、POSIX Bash 与 pipe 双路径的小流程。证据见 E33/E34，Overview 与 synthesis 只回写受影响的系统结论。
- 本轮文档例子仍是源码规则的假设性推演，没有启动目标 Agent/Provider，也未实测远端缓存或 artifact 生命周期。后续细化优先从用户新问题所在机制下手，并自动扫描相邻专题同类缺口；尚未以该新标准全量复核 owner/lease、子 Agent、Hook/Skill 全链路等未专题研究范围。
- 验证结果：本轮 11 张研究图在本机 Chrome 中用仓库 Mermaid 完成解析及 SVG 渲染；Skill `quick_validate.py`、21 个 Markdown 文件的本地链接、`git diff --check`、`corepack pnpm typecheck` 均通过；`corepack pnpm lint` 退出 0，仍是 70 条既有 warning、0 error。研究仍未运行目标 Agent/Provider，修订质量尚待用户继续校准。研究目录的 [Skill 试用记录](skill-trial.md) 已把原“深入实现通过”更正为“初版不足、修订待复核”。

## Microcompact 图与冷恢复反弹校准（2026-09-22）

**最新阅读偏好修正：**上一版把公式、变量和配置名塞进图，用户明确反馈难读。现改为白话节点，保留 60 分钟、5 组、256 token 等关键数字；门槛计算紧接图下用中文名称和数字例子说明，再提供源码变量。不要把“可复现”误解为“流程图代码化”。本条修正优先于下方历史记录中的“图内写变量/公式”。Skill 与 reader-contract 已同步修正。

- 用户进一步明确：**图自身**至少写触发式及变量、旧结果扫描方向、合格对象、完整替换文本和收益提交条件；持久化差异放在图下解释。已重画 [B5 的 Microcompact 子流程](../topics/b-context-management.md)；E35 保留恢复路径证据，Skill 的模板/证据标准和[读者约定](reader-contract.md)同步固化这一粒度。
- 冷恢复反弹结论：仅做 Microcompact 时，SQLite 原 tool part 未改；下一次冷恢复会重建它们，下一次模型步再依次尝试 Microcompact、Auto compact。Auto 优先使用最近 Provider usage，若它反映清理后输入而恢复带回更早全文，可能低估；output preflight 不硬阻断超窗，Provider 报错后 Reactive compact 仍可失败。此为静态推论，**尚未实测冷恢复或 Provider**。
- 下次继续时先检查文档及 Skill 的 diff，核对当前分支与推送状态；如果用户继续追问恢复风险，可用假设时间轴和 token 数再推一次，并明确哪些状态已经完整 compact。相邻 B2/B1 的估算与恢复表述也应保持与 B5 一致。
- 本次校验：11 张 Mermaid 图在本机 Chrome 中用仓库 Mermaid 解析并渲染 SVG 成功；变更 Markdown 的本地链接检查、Skill `quick_validate.py`、`git diff --check`、`corepack pnpm typecheck` 通过；`corepack pnpm lint` 退出 0，仍有 70 条已有 warning。未启动目标 Agent、未运行真实冷恢复或 Provider 超窗。

## 完整压缩后内容续接的具体例子（2026-09-22）

- 用户要求对 Plan、Skill、Memory、Plugin 展示真实的提示格式与注入逻辑，并问 Skill 是否有执行轨迹分析。已在 [B5](../topics/b-context-management.md) 增加四类内容的假设任务例子，逐项区分 Context 前缀、通用摘要、最近保留组和显式后置提醒；[B2](../topics/b-compaction.md) 添加入口，证据见 E36。
- 核心结论：压缩提交时专门生成的后置提醒是已批准 Plan 文件和已读文件状态；Skill catalog、Memory 索引随 Context 前缀持续可见。旧 Skill 正文及执行过程只可能通过最近保留组原文或通用摘要延续；未在生产压缩路径发现 Skill 专属 trajectory 摘要或 checkpoint。Plugin 旧引用由摘要/保留选择决定，新的用户引用才重新解析当前能力。
- 下次继续时，若用户需要设计迁移建议，可比较“通用摘要续接”与“结构化 Skill 执行状态”的收益、持久化成本及可靠性，明确后者属于我们的设计提案，不是 ZCode 已有机制。本次仍为静态源码研究，未运行目标 Agent/Provider。
- 验证：本轮为文档与 Skill 草案修改；Skill `quick_validate.py`、变更文件本地链接、`git diff --check`、`corepack pnpm typecheck` 通过；`corepack pnpm lint` 退出 0，保留 70 条既有 warning。新增的是文字例子，没有新增 Mermaid 图或运行目标 Agent。
