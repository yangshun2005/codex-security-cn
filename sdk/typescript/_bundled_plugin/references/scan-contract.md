# 已完成扫描契约

本契约定义了已完成扫描的规范化机器可读文档及其可读的 Markdown 报告投影。

## 规范化文档

一个完整的语义化包在 `<scan_dir>` 下包含以下文件：

- `scan-manifest.json`：最终确定后不可变的已完成扫描回执
- `findings.json`：已完成扫描的语义化发现记录
- `coverage.json`：带有详细回执引用的结构化覆盖摘要

规范化 UTF-8 文档大小由生产者和 SDK 一致地限制：`scan-manifest.json` 限制为 16 MiB，`findings.json` 限制为 128 MiB，`coverage.json` 限制为 32 MiB。最终确定过程会在密封或更改扫描输出之前拒绝超出大小限制的输入或生成的文档。将详细证据保留在扫描本地产物中，并从规范化摘要中引用它们。

供丰富消费者使用的可选结构化发现详情记录在 `finding-detail-fields.md` 中。它们仍然是每个语义化发现记录的一部分，而不是从可读报告中解析出的投影。

现有的 `report.md` 输出仍然是一个可读投影。生成的导出（如 SARIF）也是下游投影，不属于规范化语义化真相来源的一部分。

深度扫描 SARIF 结果和 CSV 行保留其现有的实例级展示，仅添加用于分组子报告的规范化候选 ID。标准扫描 CSV 展示保持不变。

此包记录不可变的扫描观察结果。它不是工作流状态数据库。消费者必须单独存储可变注解、生命周期决策、外部链接、保留策略和同步状态。

保留是消费者的明确决策。生成已完成扫描包时不得静默地将其复制到归档中。

## 清单语义

密封的清单记录了完成时间戳以及该包中包含的规范化文档和不可变证据回执的哈希值。可读报告和生成的导出是投影，不包含在规范化密封中。后续适配器可以读取密封包以创建投影，但不得修改密封的清单或规范化文档。请将投影单独存储。每个密封清单都恰好包含每个规范化 JSON 文档的一个产物记录，且产物路径不得重复。

## 目标快照

根据被审查的内容（而非扫描调用方式）选择目标类型：`git_worktree` 用于已检出的 Git 工作区，`directory_snapshot` 用于非 Git 目录，`git_diff` 用于基于 Git 的变更集，`git_revision` 用于精确的不可变 Git 树。

对于基于工作台的扫描，请使用记录的目标契约，而不是从检出状态推断类型。
干净的 Git 检出具有 `allowedKinds: ["git_revision"]`：使用其记录的修订版本并省略 `snapshotDigest`。
脏检出具有 `allowedKinds: ["git_worktree"]`：精确复制 `requiredSnapshotDigest`。

| 类型 | 必需的快照字段 |
| --- | --- |
| `git_revision` | `revision` |
| `git_worktree` | 可用时的 `revision` 和 `snapshotDigest` |
| `git_diff` | `snapshotDigest`；可用时包含 `baseRevision` 和 `headRevision` |
| `directory_snapshot` | `snapshotDigest` |

`targetId` 标识稳定的仓库或工作区。当存在规范的绝对远程 URL 时，优先使用其清理后的摘要。否则，使用稳定本地工作区标识的摘要。切勿持久化远程 URL 凭据、查询参数、片段或令牌。

对于脏工作树和工作树差异，根据被审查内容的确定性表示计算 `snapshotDigest`，包括暂存更改和适用的已审查未跟踪文件。对于已提交或修订范围差异，根据确切的权威差异类型和不可变的基线/头部修订版本推导。对于目录快照，对审查范围的排序相对路径和文件哈希清单进行哈希处理。将结果编码为 `codex-security-snapshot/v1:sha256:<64 位小写十六进制字符>`。

## 发现标识

每个编写的发现包括：

- 稳定的 `ruleId`：漏洞类别或生成的规则族
- 稳定的 `identity.anchor`：语义根控制锚点
- 可选的 `identity.instance`：可独立攻击的兄弟实例

所有三个值均使用小写连字符格式。在邻近行移动和文件重命名时保持其稳定。最终确定过程推导出：

- 来自目标 ID、规则 ID、锚点和实例的 `fingerprints.primary`
- 来自指纹的 `findingId`
- 来自扫描 ID 和指纹的 `occurrenceId`

不要将行号放入 `identity.anchor` 中。当两个兄弟漏洞共享规则和语义锚点时，为它们赋予不同的稳定 `identity.instance` 值。

指纹匹配是一种对账信号，而非两个发现等价的证明。将模糊匹配视为未解决。

当发现具有多个受影响位置时，如果已知，将易受攻击的控制位置标记为 `root_control`。适配器将第一个 `root_control` 位置放在首位，否则回退到第一个受影响位置，同时保留每个不同的入口点、包装器、汇点、具体实现和代码证据出现作为可匹配位置。

## 规则 ID 策略

`ruleId` 标识稳定的漏洞族，而非每次扫描的发现。
优先使用：

`<主类别>.<稳定控制族>`

示例：

- `path-traversal.archive-extraction`
- `authorization-bypass.object-update`
- `sql-injection.query-builder`

请单独使用 CWE 分类法。不要在 `ruleId` 中包含文件名、行号、扫描 ID 或展示编号。

## 覆盖

`coverage.json` 防止下游消费者将 `未观察到` 与 `未扫描` 混淆。

记录：

- 扫描模式和清单策略
- 包含和排除的路径
- 已审查的表面
- 详细回执引用
- 显式排除
- 延迟工作
- 完整性

`mode` 记录请求的扫描工作流：

| 模式 | 含义 |
| --- | --- |
| `repository` | 仓库范围扫描 |
| `scoped_path` | 仅限于显式请求路径的扫描 |
| `diff` | 当没有更具体的模式适用时，基于 Git 的变更集扫描 |
| `commit` | 提交与其解析的基线进行比较 |
| `branch_diff` | 分支或 Pull Request 变更集与其基线进行比较 |
| `working_tree` | 已暂存或未暂存的本地更改 |
| `deep_repository` | 详尽的重复仓库范围扫描 |

`inventoryStrategy` 记录生产者如何枚举已审查的内容，独立于请求的扫描工作流：

对于整个仓库的深度扫描，将 `inventoryStrategy` 保持为 `repository`；重复发现是工作流元数据，而非不同的清单策略。

| 清单策略 | 含义 |
| --- | --- |
| `repository` | 仓库范围的受跟踪类源文件清单 |
| `scoped_path` | 受限于请求路径的仓库清单 |
| `diff` | 从已审查的 Git 变更集中选择的文件 |
| `directory` | 确定性的非 Git 目录清单 |
| `custom` | 由详细回执描述的生产者定义清单 |

当请求的范围被完全审查时使用 `complete`，当范围内的工作被延迟时使用 `partial`，当生产者无法建立足够的覆盖来做出区分时使用 `unknown`。

按以下顺序将详细台账闭包映射到已完成的表面摘要中：

| 已完成表面条件 | 处置 |
| --- | --- |
| 至少一个 `reportable` 行 | `reported` |
| 否则，至少一个 `deferred` 行 | `needs_follow_up` |
| 否则，至少一个 `suppressed` 行 | `rejected` |
| 否则，适用的表面已被检查且没有候选者存活 | `no_issue_found` |
| 否则，表面不适用 | `not_applicable` |

使用 `pattern` 和 `reason` 记录每个显式排除。使用稳定的 `id`、`reason` 以及可选的 `paths` 或 `surfaceIds` 记录每个延迟单元。

详细台账保留在编号的扫描产物目录下。
回执引用必须指向 `artifacts/` 下的常规非符号链接文件。
`coverage.json` 是供适配器和比较使用的结构化摘要。

## 规范化报告语义

三个规范化 JSON 文件也是最终报告生成的唯一语义输入。生产者不得编写 `report.md`；最终确定过程在验证规范化密封后确定性地投影它。报告仍然是一个未密封的下游投影，可以在不更改规范化 JSON 或证据产物的情况下重新生成。

记录报告特定的语义，而不重复已在其他位置表示的数据：

- `scan.scope`：可选的叙述性 `summary`、已审查的产物名称、运行时/测试状态、验证模式、扫描上下文和限制。包含/排除路径仍然是权威的范围边界。
- `scan.threatModel`：简明摘要以及资产、信任边界、攻击者能力、安全目标和假设。
- 发现 `validation`：验证方法、直接证据、反证以及报告使用的结论。
- 发现 `codeEvidence`：带有标签、位置、语言和解释的稳定、精确的源代码片段；`rootCause`、`validation` 和 `attackPath` 通过 `evidenceRefs` 选择它们需要的片段。
- 发现 `rootCause`：被违反的不变量以及破坏它的代码。不要用路径/行重述替代解释。
- 发现 `attackPath.dataflow`：来源、转换、汇点、结果以及简洁的源到汇叙述。
- 发现 `attackPath.reachability`：攻击者、入口点、前提条件、结果以及简洁的可达性叙述。
- 发现 `severity`：分配的级别以及理由和会提高或降低级别的具体证据。
- 发现修复：现有的最小修复字符串以及可选的测试和预防性控制。
- 覆盖表面：已审查表面表使用的可选 `riskArea` 和 `notes`。
- 覆盖 `openQuestions`：具体的未解决问题和可选的可复制后续提示。

这些字段对于与现有 v1 规范化产物的兼容性是可选的。省略时，最终确定过程仅从剩余的规范化 JSON 中发出显式、确定性的回退文本。它永远不会读取现有报告来恢复缺失的语义。

## Schema

- `schemas/scan-manifest.schema.json`
- `schemas/findings.schema.json`
- `schemas/coverage.schema.json`

V1 消费者忽略未知属性以实现前向兼容。生产者仍必须验证已记录的字段，并且不应随意发出未记录的属性。

代表性的有效契约示例位于 `examples/completed-scan/` 下。