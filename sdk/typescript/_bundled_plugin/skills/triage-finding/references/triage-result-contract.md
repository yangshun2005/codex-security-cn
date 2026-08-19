# 分类结果契约

此契约用于 `$triage-finding` 的首次输出。该技能对提供的发现进行静态、内联分类。它不使用队列或深度分类模式。

## Schema 适配

`../../../schemas/findings.schema.json` 并非分类输入的标准化形态。

该 Schema 适用于完整的 Codex Security 扫描产物。它要求包含扫描元数据、生成的发现 ID、出现 ID、指纹、严重性、修复建议、来源信息和位置。分类输入通常以来自 SARIF 片段、CVE、安全公告、扫描器工单、漏洞赏金报告或粘贴文本的不完整声明形式出现。标准化步骤应保留这些声明，而不要虚构出完整扫描的字段。

如果用户提供了有效的 `codex-security.findings` 产物，请将其作为输入源，并将其字段映射到此分类契约中。保留 `findingId`、`occurrenceId` 和 `ruleId` 作为来源标识符；不要要求新的分类输入符合完整扫描的 Schema。

## 顶层 JSON

```json
{
  "schema_version": "triage-finding/v0",
  "repository": {
    "path": "/path/to/repo",
    "revision": "optional git sha"
  },
  "findings": []
}
```

## 单个发现 JSON

`findings` 中的每个条目必须使用以下结构：

```json
{
  "triage_item_id": "triage-001",
  "input_id": "scanner-or-user-id",
  "source_type": "sarif",
  "title": "finding title",
  "normalized_input": {
    "vulnerable_component": "package, file, API, route, function, service, or unknown",
    "claimed_source": "attacker-controlled input or unknown",
    "claimed_sink": "sink or broken control or unknown",
    "claimed_control": "missing or bypassed guard, sanitizer, auth check, or unknown",
    "affected_version_or_path": "affected version, path, config, or unknown",
    "preconditions": ["required condition"],
    "impact": "claimed impact or unknown",
    "references": ["source-provided reference"]
  },
  "verdict": "confirmed",
  "confidence": "high",
  "affected_locations": [
    {
      "label": "entrypoint",
      "path": "relative/path",
      "lines": "12-18",
      "detail": "why this location matters"
    }
  ],
  "reachable_path": ["step 1", "step 2"],
  "boundary_assessment": {
    "product_surface": "hosted service, CLI, library API, local developer UI, MCP/tooling, example/demo, test/fixture, docs, generated, vendored, or unknown",
    "source_trust": "untrusted, trusted_operator, trusted_developer_config, local_only, or unknown",
    "boundary_crossed": true,
    "policy_basis": "SECURITY.md, package/deploy evidence, product docs, code comments, or unknown"
  },
  "exploitability_stack_rank": {
    "rank_queue": "confirmed",
    "rank": 1,
    "rationale": "why this finding is more or less exploitable than other findings with the same verdict",
    "drivers": ["attacker reachability", "privilege required", "preconditions", "source-to-sink control", "guard strength"]
  },
  "evidence": ["static evidence observed"],
  "counterevidence": ["static evidence that weakens or defeats the claim"],
  "proof_gaps": ["missing evidence or reason for human review"],
  "recommended_next_step": "fix-finding",
  "fix_finding_handoff": "prompt-ready summary for confirmed findings"
}
```

必需的来源类型值：

- `sarif`
- `cve`
- `advisory`
- `scanner_ticket`
- `bug_bounty`
- `codex_security_finding`
- `freeform`
- `unknown`

允许的判定结果：

- `confirmed`
- `not_actionable`
- `needs_review`

允许的置信度值：

- `high`
- `medium`
- `low`

`boundary_assessment` 记录来源是否跨越受支持的产品安全边界，而不仅仅是数据流是否到达汇聚点。当表面或策略不明确时，使用 `unknown` 字符串，并将 `boundary_crossed` 设为 `null`；对于边界情况不明确的，优先使用 `needs_review`。

`exploitability_stack_rank` 记录结果集中的优先级。为 `confirmed` 和 `needs_review` 使用单独的队列。在每个队列内，从 `1` 开始连续分配唯一的正整数排名。排名 `1` 是其队列中最容易被利用的发现；使用 `rank_queue` 而非仅用数字来标识队列。对于 `not_actionable`，将 `rank_queue` 和 `rank` 设为 `null`，使用空的 `drivers` 数组，并将 `rationale` 设置为简短说明，例如 `not actionable`。

对于不可用的可选证据列表，使用空数组。除非判定结果为 `confirmed`，否则将 `fix_finding_handoff` 设为 `null`。