# B4：工具大结果的截断、落盘与恢复读取

## 研究问题与边界

本专题区分两条容易混淆的链路：通用 ToolExecutor 在 handler 返回后应用 `resultBudget`；Bash/进程适配器在读取 stdout/stderr 时就执行 inline、tail、文件和进程输出上限。基线为 `872ad960de7ec172591f7e1952f7849229f94521`。

## 1. 通用结果预算

工具先通过自己的 formatter 生成完整 `modelContent`，再按 UTF-8 字节计算预算。默认 `maxInlineBytes` 和 `maxModelBytes` 都是 100,000，默认 strategy 为 truncate；实际模型上限取两者较小值。工具还可声明 JS 字符上限。[result-serialization.ts:33-86](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts)

在预算内时保留原始结构化 `modelContent`。超限后有两种策略：

- `truncate`：按 head/tail 方向裁剪，并加入 originalBytes、maxModelBytes、strategy 和可用 artifact path 的后缀。
- `artifact`：若启用 artifact 且 store 写入成功，把完整 Provider-visible 序列化内容落盘，模型只看到 `<persisted-output>`、路径和前 2,000 字符 preview。

[result-serialization.ts:161-217](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts) [result-persistence-format.ts](../../../apps/zcode-cli/packages/core/src/tool/result-persistence-format.ts)

artifact 写入携带 sessionId、turnId、toolCallId、toolName、contentType 和 retention。写入失败不会把成功工具改成失败，而是回退到有界截断；对显式字符阈值的 Provider 文本有一条保留原文的兼容分支。[result-serialization.ts:294-348](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts)

官方 CUA structured frame 是例外：image/image_ref 原子对受独立保护，文本仍受预算，returnedBytes 还会计入真实媒体载荷，不能按序列化占位符低估。[result-serialization.ts:101-157](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts)

## 2. Artifact 的物理表示与可读性

Node store 在按 session 隔离的目录创建文件，文件名包含 toolCallId 与随机 artifactId，同时返回 `zcode-artifact://<session>/<artifactId>` 和本地绝对 path。文本按 UTF-8 写入；读取时文本返回 UTF-8，非文本返回 base64，另有可选的原始字节 API。[storage/index.ts:41-169](../../../apps/zcode-cli/packages/adapters/src/storage/index.ts) [tool-artifact-store.port.ts](../../../apps/zcode-cli/packages/contracts/src/interfaces/tool-artifact-store.port.ts)

serializer 优先把 store 返回的 path 放进模型 preview，其次才是 URI。因此 Node 装配下模型可以用普通 Read 按绝对路径分段读取；Read 明确要求绝对 `file_path` 并支持 offset/limit。[read.ts:52-68](../../../apps/zcode-cli/packages/core/src/tool/handlers/read.ts) [read-text.ts:25-57](../../../apps/zcode-cli/packages/core/src/tool/handlers/read-text.ts)

若未来 store 只返回 URI 而没有 path，当前通用 preview 会显示 URI；普通 Read 不接受该 URI。系统确有 artifact API 供附件、V4 和 UI 读取，但本批未找到一个通用模型工具直接接受 `zcode-artifact://`。因此“模型总能自行恢复读取”只对可见本地 path 成立。

## 3. Bash 的流式采集先发生

这里必须修正此前过度概括：**POSIX Bash merged-output 路径**使用 `BashFileOutput`，子进程直接写到输出文件，Node 读取文件前段形成 inline preview；`OutputCollector` 用于其余 pipe stdout/stderr 路径。后者在进程运行时累积 total bytes、有限 inline head 和独立 tail；`persistOutput=always` 立即落盘，`on_truncate` 在超过 inline limit 时创建文件并补写 head，受单流和 aggregate budget 约束。[node-execution-adapter-run.ts:69-100](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-run.ts) [bash-file-output.ts:14-66](../../../apps/zcode-cli/packages/adapters/src/exec/bash-file-output.ts) [output-collector.ts](../../../apps/zcode-cli/packages/adapters/src/exec/output-collector.ts)

Bash handler 实际给执行适配器的 inline 上限为 30,000 bytes、持久输出上限为 5 GiB；后台模式 `always`，前台 `on_truncate`。[bash.ts:70-71](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash.ts) [bash.ts:420-429](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash.ts) POSIX 文件路径的 `readBashOutput` 返回文件总大小、读取字节数和 `artifactTruncated=false`；但前台结算会按模式决定保留或 best-effort 删除文件，若设置更低的 artifact 上限还可把文件截短并标记 `artifactTruncated=true`。[bash-file-output.ts:153-185](../../../apps/zcode-cli/packages/adapters/src/exec/bash-file-output.ts) [node-execution-adapter-results.ts:94-160](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-results.ts)

Bash foreground 的 model formatter 若看到 `persistedOutputPath`/`rawOutputPath`，生成同样的 `<persisted-output>` envelope；original size 使用截断前 observed bytes。background 命令则告诉模型输出仍在写哪个文件，并明确提示用 Read 检查中间结果。[bash-model-content.ts:25-96](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash-model-content.ts) [bash-model-content.ts:108-151](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash-model-content.ts)

进程适配器还可在持久输出达到硬上限时终止 process tree，并返回 output-limit error；这和“模型上下文只看 preview”是不同层面的保护。[node-execution-adapter-run.ts:39-61](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-run.ts) [node-execution-adapter-results.ts:165-231](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-results.ts)

## 4. 两层可能叠加

Bash handler 首先把有限 inline/落盘元数据格式化为 Provider content；随后通用 serializer 仍会应用工具的 `resultBudget`。若通用 artifact strategy 再次落盘，它保存的是完整的 Bash provider-visible content，而不是已经丢失的原始 stdout。注释专门避免 formatter 二次进入后把 stderr 或 provider error 完整追加回模型。[bash-model-content.ts:25-50](../../../apps/zcode-cli/packages/core/src/tool/handlers/bash-model-content.ts)

所以要判断“完整输出在哪里”，必须问：

1. 进程适配器是否从第一字节开始持久化，文件是否又因 persisted limit 截断？
2. handler 返回给 serializer 的 `modelContent` 是否已经是 preview envelope？
3. 通用 artifact 保存的是 handler 后内容还是原始执行流？

对 Bash 来说，原始流的主要恢复来源是 adapter 给出的 output path；通用 artifact 不是恢复被 adapter 丢弃字节的后门。

## 5. 持久化、UI 与冷恢复

tool result part 会保存展示状态、original/returned bytes、truncated 和 artifactPath 等元数据；下一模型请求使用 `modelContent`。string modelContent 还会写进 part metadata，冷恢复时优先使用它，所以重启后模型继续看到当时的 preview/路径，而非重新把大文件全文塞回上下文。[turn-tools.ts:260-363](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts) [session-history-hydrator.ts:146-190](../../../apps/zcode-cli/packages/core/src/agent/session-history-hydrator.ts)

这是一种“保留引用，不自动再注入全文”的恢复策略。path 是否仍可读，取决于对应的 Node artifact 文件或 Bash 执行文件是否保留；下文分别追踪这两套生命周期，不能只凭 `retention` 字段推断文件必然存在。

### 逐层可恢复性

| 层                         | 保存内容                                      | 完整性与读取边界                                                                                                   |
| -------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| POSIX Bash 执行文件        | 子进程原始合并输出                            | 前台超 inline 才按 `on_truncate` 保留；硬上限可终止执行，结算可按 artifact 上限截文件；模型收到本地 path 后可 Read |
| pipe `OutputCollector`     | 有界 inline、tail 和可选落盘文件              | `persistOutput` 模式与单流/aggregate 上限共同决定文件是否完整；`artifactTruncated` 是该层信号                      |
| Bash formatter             | preview、大小、path、错误说明                 | 仅是模型可见摘要，不是原始流；不能从它逆推出全部 stdout/stderr                                                     |
| 通用 result serializer     | handler 的 Provider-visible 内容或其 artifact | 只保存上一层输出；若上一层已省略原始字节，这里再落盘也无法补回                                                     |
| Session tool part 与冷恢复 | UI 展示字段、统计、`modelContent` 和 path     | 重启时重建当时的模型 preview，不自动读取大文件全文，也不验证文件仍存在                                             |

[node-execution-adapter-results.ts:28-160](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-results.ts) [result-serialization.ts:161-348](../../../apps/zcode-cli/packages/core/src/tool/executor/result-serialization.ts) [turn-tools.ts:260-363](../../../apps/zcode-cli/packages/core/src/runtime/methods/turn-tools.ts)

**假设示例：**POSIX Bash 前台产生 300 KB 合并输出。30,000 bytes 的 preview 从直接写入的执行文件读取；因为超 inline，该文件按 `on_truncate` 模式保留，模型内容给出 preview 与 path。若文件后来被删除，冷恢复仍可恢复 path 文本，却不能凭该文本恢复 300 KB。这个例子是源码时序推演，不是实际运行记录。

## 6. Artifact 生命周期和远端边界

默认 Node Tool Artifact Store 位于 `<storageRoot>/cli/artifacts/<sessionId>/`，每次写入返回随机 ID、URI、path、字节数和时间；URI 读回时在 session 目录中按 ID 找文件。Port 定义了 `session | project | temporary` retention 字段，但当前 Node store 写入只使用 sessionId/contentType/content，没有把 retention 写入索引，也没有删除/过期方法。因此 **retention 是调用契约中的意图，不能据此宣称默认 store 已按该策略自动清理**。[create-app.ts:348-357](../../../apps/zcode-cli/packages/bootstrap/src/app/create-app.ts) [tool-artifact-store.port.ts:7-35](../../../apps/zcode-cli/packages/contracts/src/interfaces/tool-artifact-store.port.ts) [tool-artifact-store.port.ts:92-130](../../../apps/zcode-cli/packages/contracts/src/interfaces/tool-artifact-store.port.ts) [storage/index.ts:41-169](../../../apps/zcode-cli/packages/adapters/src/storage/index.ts)

Bash 原始执行文件属于另一目录 `<storageRoot>/cli/exec/<sessionId>/<toolCallId>-stdout.log`；前台未达到保留条件的文件在结算时 best-effort 删除，准备阶段失败时只删除本次创建的文件。不能把它与 `zcode-artifact://` store 的 retention 混成同一生命周期。[create-app.ts:395-403](../../../apps/zcode-cli/packages/bootstrap/src/app/create-app.ts) [node-execution-adapter-base.ts:80-115](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-base.ts) [node-execution-adapter-results.ts:94-132](../../../apps/zcode-cli/packages/adapters/src/exec/node-execution-adapter-results.ts) [bash-file-output.ts:45-66](../../../apps/zcode-cli/packages/adapters/src/exec/bash-file-output.ts)

URI 的读取发生在 CLI app 的 artifact store；普通文件 path 的读取经当前 runtime 的 `FileSystemPort`，附件服务注释也明确要求沿该端口才能在 SSH/WSL/Docker 命中正确文件系统。故远端是否能让**模型工具**读到执行文件，取决于实际注入的文件系统端口与 path 所在主机是否相同；本批未找到可对所有远端部署一概而论的契约。[create-app.ts:1033-1050](../../../apps/zcode-cli/packages/bootstrap/src/app/create-app.ts) [read.ts:52-68](../../../apps/zcode-cli/packages/core/src/tool/handlers/read.ts)

## 设计取舍与迁移条件

以下为研究者推断：分层预算把进程内存、磁盘、模型上下文和媒体协议分别保护起来，代价是“truncated”必须带层级解释。迁移时至少保留 original bytes、returned bytes、artifact completeness、path/URI 和 retention；只存一个 `truncated=true` 无法判断能否恢复。模型后续读取必须有可调用的读取工具，落盘路径本身不是完整闭环。

## 验证与未知

- 已静态追踪通用 serializer、Node artifact store、POSIX 直接输出文件、pipe collector、前台清理、Read 的 range 能力和 cold hydration。
- 默认 Node store 的 `retention` 未落实为可见的删除/过期接口；仓库外清理或其他 store 实现不在此结论内。
- 未启动目标程序；磁盘满、远端文件系统注入和具体部署下的 path 可达性仍是有边界未知，不影响上述源码契约的解释。
