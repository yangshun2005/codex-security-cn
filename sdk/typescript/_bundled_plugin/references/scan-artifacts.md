# 扫描产物路径

除非用户明确提供不同的输入或输出路径，否则 Codex Security 扫描工作流应使用以下共享路径约定。

## 基础路径

- `plugin_dir=<codex-security 插件根目录>`
- `repo_name=<repo_root 的基名>`
- `target_id=<来自 references/scan-contract.md 的稳定扫描目标标识>`
- `system_temp_dir=<平台临时目录>`
- `security_scans_dir=<system_temp_dir>/codex-security-scans/<repo_name>`
- `scan_id=<commit>_<扫描时间戳>`
- `scan_dir=<security_scans_dir>/<scan_id>`
- `target_paths_file=<CODEX_SECURITY_TARGET_PATHS_FILE>` 用于 SDK 限定路径扫描；在 POSIX shell 中引用为 `"$CODEX_SECURITY_TARGET_PATHS_FILE"`，在 PowerShell 中引用为 `"$env:CODEX_SECURITY_TARGET_PATHS_FILE"`。此只读范围输入位于隔离的 Codex home 目录中，在模型可写的扫描目录之外。在最终确定之前，直接将其传递给 `make-repo-scope-input --scopes-file` 和 `bind-repo-scopes --scopes-file`，并且不要打印、评估、修改或将其内容视为 shell 语法。
- `artifacts_dir=<scan_dir>/artifacts`
- `context_dir=<artifacts_dir>/01_context`
- `discovery_dir=<artifacts_dir>/02_discovery`
- `coverage_dir=<artifacts_dir>/03_coverage`
- `reconciliation_dir=<artifacts_dir>/04_reconciliation`
- `findings_dir=<artifacts_dir>/05_findings`

插件会自动解析平台临时目录。对于手动工作流，请使用当前进程的临时目录（例如，Windows 上的 `%TEMP%` 或 Unix 类主机上配置的 `$TMPDIR`），而不是硬编码 `/tmp`。

将 `<python_command>` 解析为配置的 Python 解释器（POSIX shell 中的 `"$PYTHON"` 或 PowerShell 中的 `& "$env:PYTHON"`），否则在 Windows 上使用 `python`，在 Unix 类主机上使用 `python3`。

## 威胁模型（阶段 1）路径

- 已解析的 SECURITY.md 指南：`<context_dir>/security_guidance.md`
- 仓库范围的威胁模型：`<security_scans_dir>/threat_model.md`
- 每次扫描的威胁模型副本：`<context_dir>/threat_model.md`
- 后续扫描阶段应将 `<context_dir>/threat_model.md` 视为唯一事实来源。
- 当仓库范围的威胁模型已存在时，将其原样复制到 `<context_dir>/threat_model.md`，以确保可审计性。

每个仓库范围的威胁模型应以以下两行结尾：

- `Repository: <target_id>`
- `Version: <不可变 Git 树的修订版本；否则为快照摘要>`

## 发现（阶段 2）路径

### 紧凑型深度及工作台支持的差异发现

工作台拥有的标准扫描通过 `record_codex_security_scan_draft` 提交发现和覆盖率；SDK 拥有的标准扫描直接写入未密封的规范文件。深度扫描运行完整的标准扫描工作器，每个工作器通过其绑定的 `record_codex_security_scan_draft` 工具提交其已验证的发现、覆盖率、威胁模型和可选范围。协调器在语义上缩减这些完整结果，并写入父扫描的未密封 `scan-manifest.json`、`findings.json` 和 `coverage.json`。父扫描不列出候选、不重新运行验证或攻击路径阶段，也不提交另一个草稿。工作台支持的差异扫描保留下述紧凑型产物。

- 工作台支持的差异扫描通过 `record_codex_security_discovery_candidates({ scanId, candidates })` 一次性记录所有候选，并通过 `list_codex_security_candidates({ scanId, cursor?, limit? })` 读取规范候选。
  - 写入方根据分配的源路径验证候选，合并具有相同 CWE ID、位置和可选实例的行，保留其文本，并分配确定性的 `candidate_id` 值。
  - 规范化后，紧凑验证为每一行添加恰好一个 `validation` 对象，包含 `disposition`（`reportable`、`suppressed`、`not_applicable` 或 `deferred`）、`method`、`confidence`（`high`、`medium` 或 `low`）、`confidence_rationale`、简洁的 `rubric` 和 `evidence`、`counterevidence_or_proof_gap`、`remaining_uncertainty` 以及可选的 `artifact_paths`。仅当 `source`、`control`、`sink` 或 `preconditions` 能澄清或与发现字段不同时才添加。
  - 紧凑攻击路径分析为每个标记为 `reportable` 或 `deferred` 的验证行添加恰好一个 `attack_path` 对象，包含 `decision`（`reportable`、`ignore` 或 `deferred`）、`dataflow`、`reachability`、`counterevidence`、`impact` 和 `likelihood`（`high`、`medium`、`low`、`ignore` 或 `unknown`）、`severity`（`critical`、`high`、`medium`、`low`、`ignore` 或 `unknown`）、`severity_rationale`、`change_conditions`，以及延迟时的 `proof_gap`。`reportable` 决策要求严重级别为 `critical`、`high`、`medium` 或 `low`；`ignore` 要求严重级别为 `ignore`；`deferred` 使用暂定的可报告严重级别或 `unknown`。
  - 通过 `record_codex_security_candidate_validations` 记录所有验证，并通过 `record_candidate_attack_paths` 记录所有符合条件的攻击路径决策。这些工具原子地保留所有发现字段和候选顺序。
- 可选的紧凑验证证据：`<discovery_dir>/validation_artifacts/<candidate_id>/`
  - 仅为实际的 PoC、精心构造的输入或日志创建此目录，并从行的 `validation` 对象中引用这些路径。不要创建占位性的逐候选目录或叙述性报告。

以下工作清单、逐发现回执和阶段报告路径仅适用于独立或遗留差异工作流。紧凑型工作台差异扫描使用一个共享的 `<discovery_dir>/candidate_ledger.jsonl`，由 `record_codex_security_discovery_candidates` 写入，并由绑定的批处理工具 `record_codex_security_candidate_validations` 和 `record_candidate_attack_paths` 更新；它们不创建逐发现账本、报告或回执。标准扫描和深度扫描直接组装已验证的发现，无需持久化的源清单或候选账本。

### 差异发现和覆盖率

- 咨询种子研究：`<context_dir>/seed_research.md`
- 变更源输入：`<discovery_dir>/rank_input.jsonl`
- 限定的深度审查输入：`<discovery_dir>/deep_review_input.jsonl`（如适用）
- 发现报告：`<discovery_dir>/finding_discovery_report.md`

### 深度审查

- 限定的工作账本：`<discovery_dir>/work_ledger.jsonl`（如适用）
- 限定的原始候选：`<discovery_dir>/raw_candidates.jsonl`（如适用）

### 候选对账

- 紧凑差异候选账本：`<discovery_dir>/candidate_ledger.jsonl`
- 独立或遗留差异候选发现目录：`<findings_dir>/`
- 独立或遗留差异逐发现目录：`<findings_dir>/<candidate_id>/`
- 独立或遗留差异逐发现候选账本：`<findings_dir>/<candidate_id>/candidate_ledger.jsonl`
- 限定的去重报告：`<reconciliation_dir>/dedupe_report.md`（如适用）
- 限定的去重候选：`<reconciliation_dir>/deduped_candidates.jsonl`（如适用）

### 覆盖率

- 仓库范围覆盖率账本：`<coverage_dir>/repository_coverage_ledger.md`
  - 这是覆盖率产物，而非发现列表：应包含已检查的表面，并带有 `not_applicable`、`suppressed`、`deferred` 或 `reportable` 处置。
- 已审查表面摘要：`<coverage_dir>/reviewed_surfaces.md`（如适用）

## 验证（阶段 3）路径

标准扫描和深度标准扫描工作器在其最终发现语义中直接包含验证。紧凑型工作台支持的差异扫描通过其绑定的批处理工具记录验证，并可使用上述可选的紧凑证据路径。独立或遗留差异工作流可使用以下路径：

- 扫描级验证摘要：`<findings_dir>/validation_summary.md`（如适用）
- 逐发现验证报告：`<findings_dir>/<candidate_id>/validation_report.md`
- 逐发现验证产物：`<findings_dir>/<candidate_id>/validation_artifacts/`

## 攻击路径分析（阶段 4）路径

标准扫描和深度标准扫描工作器在其最终发现语义中直接包含攻击路径分析。紧凑型工作台支持的差异扫描通过其绑定的批处理工具在每个候选的嵌套 `attack_path` 记录中记录攻击路径决策。独立或遗留差异工作流可使用以下路径：

- 扫描级攻击路径分析报告：`<findings_dir>/attack_path_analysis_report.md`（如适用）
- 逐发现攻击路径分析报告：`<findings_dir>/<candidate_id>/attack_path_analysis_report.md`

## 最终报告路径

- 工作台拥有的标准或工作台支持的差异草稿：`record_codex_security_scan_draft({ scanId, handoffClaimToken?, scope?, threatModel?, findings, coverage })`
- 绑定的深度标准工作器结果：`record_codex_security_scan_draft({ scanId, scope?, threatModel?, findings, coverage })`；深度协调器写入聚合的父草稿
- SDK 拥有的标准草稿：SDK 提供的扫描目录下的未密封 `scan-manifest.json`、`findings.json` 和 `coverage.json`
- 深度、工作台支持的差异或明确请求的标准完成结果：`get_codex_security_completed_scan({ scanId, handoffClaimToken? })`
- 最终扫描报告：`<scan_dir>/report.md`
- 详细漏洞说明：`<scan_dir>/findings/<slug>/<slug>.md`
- 逐发现 PoC 和支持文件：`<scan_dir>/findings/<slug>/poc/...`
- 结构化加固组合：`<scan_dir>/hardening/hardening.md`
- 加固分析、提案和图表：`<scan_dir>/hardening/...`
- 最终报告验证说明（当验证失败时）：`<scan_dir>/report_validation.md`

## 修复发现路径

- 修复报告（使用现有扫描产物目录时）：`<artifacts_dir>/fix_report.md`

## 放置规则

- 将扫描阶段输出和支持证据放在上述编号的产物子目录下。
- 将修复发现输出保留在编号的扫描阶段之外，因为修复发现可以独立运行或针对现有扫描运行。
- 不要直接编写最终的 `report.md`。将完整的扫描级报告语义放入规范 JSON 文件中。`findings/<slug>/<slug>.md` 中的详细逐发现说明和 `hardening/` 下的派生设计指南对于每种扫描模式都是可选的。最终确定过程确定性地写入未密封的 `report.md` 投影，并链接任何记录的说明和加固组合。不要将这些派生文档添加到密封产物列表中。
- 将完整的扫描包保留在 `scan_dir` 下。