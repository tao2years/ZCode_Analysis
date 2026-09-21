# 生成与验证记录

日期：2026-09-21。

- 设计：用户已确认。质量优先、重点深入、两个入口、中文总分总、增量 index、可移植；不加入费用预算流程。
- 内容与文件：已创建 `code-agent-source-research/`，包含 SKILL.md、6 个按需参考文件和 Codex UI 元数据。
- 安装：未安装；当前仅为可审阅和可分发草稿。
- 仓库基线检查：`node scripts/check-workspace-freshness.mjs` 通过。
- Skill 结构检查：`python -X utf8 C:/Users/taos/.codex/skills/.system/skill-creator/scripts/quick_validate.py skill-drafts/code-agent-source-research` 通过。首次未开启 UTF-8 的执行因系统 GBK 解码失败，不是内容校验通过。
- 内部引用与脚手架检查：读取全部 7 个 Markdown 文件，内部链接均可解析，无初始化 TODO 占位。
- 仓库依赖：使用匹配 `mise.toml` 的 Node 24.14.0 和 corepack pnpm 10.33.2 完成 `pnpm install --frozen-lockfile`。可选原生模块 `cpu-features`/`ssh2` 因缺少本机 C++ 编译环境未构建，安装仍退出 0。
- 仓库检查：`corepack pnpm typecheck` 退出 0；`corepack pnpm lint` 退出 0，70 warnings、0 errors。`corepack pnpm fmt:check` 退出 1，报告 2803 个全仓文件格式问题，包含大量既有文件，未记为通过。
- 实际研究试跑：已由用户明确选择当前 ZCode 仓库，完成系统全景和 A/B 串行静态深挖。产物位于 `research/code-agent/`；七篇专题覆盖会话/Agent Loop、工具调度、消息投影、上下文管理、压缩、Memory 与大工具结果。用户追加的六项已在同一基线上增量研究并回写 Overview。
- 测试支持：`node --import tsx --test packages/services/test/nonCliAcpRetirement.test.ts` 退出 0，5/5 通过，只支持服务层 Memory 文件读取及当前会话投影恢复。未发现直接覆盖本批 CLI 核心机制的测试入口。
- 试跑验收：全景、专题深度、表示分离、正确性分支、静态证据、增量文档和未知处理通过。用户随后澄清目标是学习源码中的设计思想，真实应用、故障注入和 Provider wire 不属于完成条件；Skill 已据此改为源码研究优先。详情见 `research/code-agent/research/skill-trial.md`。

## 下一步

若继续研究，可选择 owner/lease、Provider adapter 或子 Agent 的独立专题；是否安装 Skill 另行决定。
