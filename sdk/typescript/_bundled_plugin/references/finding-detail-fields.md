# 丰富发现详情字段

对于 `findings.json` 中的每个可报告发现，保留经过验证的推理过程和证明该推理的精确源代码片段。Codex 安全工作区直接渲染这些字段；它不会从 `report.md` 中恢复缺失的分析，也不会在扫描后读取源文件。

## 编写规则

- 将 RPC 名称、函数、类型、字段、参数、配置键、字面量标识符和短表达式用单个反引号包裹。例如：`environment/add`、`environmentId`、`execServerUrl` 和 `EnvironmentManager::upsert_environment()`。
- 保持代码与正文分离。将源代码片段放入 `codeEvidence[].code` 中，然后在解释该片段重要性的章节中引用它们。工作区会将这些被引用的片段整合到 **根本原因** 下，以便被违反的不变量及其源代码证明保持在一起。
- 根本原因必须是基于源代码的逐步分析，而不是结论性段落。从用户可控数据被声明、解码或读取的代码开始；跟踪每个有意义的调用、转换或状态变迁；然后展示缺失的控制、危险操作，以及当影响受影响时的后续消费者。
- 为每个代码证据项提供一个稳定的 `id`、简洁的 `label`、精确的源代码位置、最小的有用片段、一个 `role` 和一个 `explanation`。支持的 `role` 包括 `user_input`、`entrypoint`、`propagation`、`root_control`、`sink`、`outcome` 和 `expected_control`。
- 将每个 `explanation` 写成连贯的推理：在此步骤识别攻击者控制的值，说明它接下来传递给哪个被调用方或状态，并解释所展示的代码行如何保持或违反了预期的不变量。
- 按从用户输入到结果的顺序排列 `rootCause.evidenceRefs`。将 `expected_control` 比较放在易受攻击的调用栈引用之后；它是支持性上下文，而不是易受攻击栈中的一步。省略不携带该值或不强制执行相关边界的无关辅助代码。
- 不要使用仅限位置的填充内容，例如“根本原因与 path:line 处的损坏控制相关。”源代码表已经记录了位置。解释被违反的不变量并展示违反它的代码。
- 验证必须连接攻击者控制的输入、缺失或被绕过的控制，以及安全相关的状态变更或风险点。不要用文件名和行号的列表来替代该证明。
- 攻击路径分析必须简洁。记录真实的攻击者边界、最小触发序列和具体结果。对重要的转换使用代码证据，而不是重复完整的验证叙述。
- 仅填充有证据支持的字段。省略未知值，而不是添加占位符。

## 简洁的工作区投影

发现详情视图是规范发现的一种聚焦决策的投影，而不是完整 `vulnerability-writeup` 报告的副本。保留该报告中审查者理解和处理问题所需的部分：

- 验证方法、直接观察、置信度理由和剩余不确定性；
- 数据流来源、有意义的转换、危险的风险点和具体结果；
- 真实的攻击者、入口点、访问要求、前提条件和攻击者结果；
- 严重性理由以及会提高或降低评级的特定证据；
- 最小的修复不变量（`remediation`，单个字符串），以及 `remediationTests` 和 `preventiveControls`，每个都是短字符串数组，每个条目对应一个回归测试或预防性控制。

将背景说明、备选漏洞研究、完整 PoC 说明、代表性命令输出和冗长的源代码分析保留在详细报告中。不要仅仅为了让工作区报告更长而将它们复制到规范字段中。工作区应保持足够的自包含性以支持分类，同时避免重复或推测性的内容。

工作区 **证据** 部分是一个产物导航器，而不是另一个源代码证明部分。当存在 `writeup.reportPath` 时，工作台会列出该经过验证的扫描本地报告以及其同级 `poc/` 目录下的常规文件。每一行都通过宿主中介的 Codex 导航请求在编辑器中打开确切文件。不要将产物路径放在根本原因叙述中，也不要仅仅为了显示而将未经验证的产物列表添加到规范发现中。

## 结构化示例

以下结构展示了如何编码 `environment/add` 保留环境覆盖发现：

```json
{
  "summary": "运行时 `environment/add` 方法将调用者控制的 `environmentId` 和 `execServerUrl` 转发给 `EnvironmentManager::upsert_environment()`。启动过程会拒绝保留的 `local` 标识符，但运行时变更路径接受它并替换默认环境查找使用的映射条目。",
  "codeEvidence": [
    {
      "id": "rpc-input",
      "label": "调用者控制的环境字段",
      "path": "codex-rs/app-server-protocol/src/protocol/v2/environment.rs",
      "startLine": 6,
      "endLine": 12,
      "language": "rust",
      "role": "user_input",
      "code": "#[serde(rename_all = \"camelCase\")]\npub struct EnvironmentAddParams {\n    pub environment_id: String,\n    pub exec_server_url: String,\n}",
      "explanation": "`environmentId` 和 `execServerUrl` 被接受为调用者控制的字符串。"
    },
    {
      "id": "rpc-forward",
      "label": "RPC 无验证地转发两个字段",
      "path": "codex-rs/app-server/src/request_processors/environment_processor.rs",
      "startLine": 15,
      "endLine": 22,
      "language": "rust",
      "role": "entrypoint",
      "code": "self.environment_manager\n    .upsert_environment(params.environment_id, params.exec_server_url)\n    .map_err(|err| invalid_request(err.to_string()))?;",
      "explanation": "处理器将两个值直接传递给 `upsert_environment()`，并且不执行保留 ID 检查。"
    },
    {
      "id": "startup-reserved-check",
      "label": "启动过程保护保留的 local 标识符",
      "path": "codex-rs/exec-server/src/environment.rs",
      "startLine": 167,
      "endLine": 176,
      "language": "rust",
      "role": "expected_control",
      "code": "if id == LOCAL_ENVIRONMENT_ID {\n    return Err(ExecServerError::Protocol(format!(\n        \"environment id `{LOCAL_ENVIRONMENT_ID}` is reserved for EnvironmentManager\"\n    )));\n}",
      "explanation": "初始环境构造强制执行 `local` 属于 `EnvironmentManager` 的不变量。"
    },
    {
      "id": "runtime-upsert",
      "label": "运行时 upsert 省略了保留 ID 检查",
      "path": "codex-rs/exec-server/src/environment.rs",
      "startLine": 253,
      "endLine": 281,
      "language": "rust",
      "role": "root_control",
      "code": "if environment_id.is_empty() {\n    return Err(ExecServerError::Protocol(\n        \"environment id cannot be empty\".to_string(),\n    ));\n}\n// ... build remote environment ...\nself.environments\n    .write()\n    .unwrap_or_else(std::sync::PoisonError::into_inner)\n    .insert(environment_id, Arc::new(environment));",
      "explanation": "`upsert_environment()` 在插入共享映射之前仅拒绝空 ID。传入 `local` 会替换受保护的条目。"
    },
    {
      "id": "default-lookup",
      "label": "默认选择读取被覆盖的映射条目",
      "path": "codex-rs/exec-server/src/environment.rs",
      "startLine": 205,
      "endLine": 210,
      "language": "rust",
      "role": "outcome",
      "code": "pub fn default_environment(&self) -> Option<Arc<Environment>> {\n    self.default_environment\n        .as_deref()\n        .and_then(|environment_id| self.get_environment(environment_id))\n}",
      "explanation": "默认查找通过可变环境映射解析存储的 `local` ID，因此替换会影响后续操作。"
    }
  ],
  "rootCause": {
    "summary": "被违反的不变量是 `local` 必须始终标识管理器拥有的本地运行时。启动过程强制执行该不变量，但 `EnvironmentManager::upsert_environment()` 未复用保留 ID 检查，并在调用者提供的键下插入远程 `Environment`。",
    "evidenceRefs": [
      "rpc-input",
      "rpc-forward",
      "runtime-upsert",
      "default-lookup",
      "startup-reserved-check"
    ]
  },
  "validation": {
    "method": "静态源代码追踪",
    "summary": "源代码追踪确认 `environment/add` 调用者控制两个输入，RPC 原封不动地转发它们，并且运行时插入接受 `local`。",
    "evidenceRefs": [
      "rpc-input",
      "rpc-forward",
      "runtime-upsert"
    ],
    "assertions": [
      "运行时路径缺少启动期间存在的保留 ID 检查。",
      "插入 `local` 会替换现有的 `HashMap` 条目。"
    ],
    "limitations": [
      "该发现通过源代码审查验证；未运行实时 JSON-RPC 复现。"
    ]
  },
  "attackPath": {
    "summary": "较低信任度的 app-server 客户端选择加入实验性 API，调用带有 `environmentId: \"local\"` 的 `environment/add`，并将 `execServerUrl` 指向攻击者控制的执行器。后续的默认环境选择会解析被替换的映射条目。",
    "dataflow": {
      "summary": "`environment/add` 参数 -> `environment_add()` -> `upsert_environment()` -> 共享环境映射 -> `default_environment()`",
      "source": "调用者控制的 `environmentId` 和 `execServerUrl`",
      "sink": "共享环境映射",
      "outcome": "默认 `local` 选择解析到攻击者控制的远程执行器",
      "evidenceRefs": [
        "rpc-input",
        "rpc-forward",
        "runtime-upsert",
        "default-lookup"
      ]
    },
    "reachability": {
      "summary": "攻击者必须能够充当 app-server 客户端并启用 `experimentalApi`；默认的 stdio 和私有 Unix 套接字传输降低了暴露面。",
      "attacker": "较低信任度的 app-server 客户端",
      "entrypoint": "实验性 `environment/add` RPC",
      "outcome": "后续为 `local` 选择的操作被路由到远程执行器"
    },
    "evidenceRefs": [
      "rpc-forward",
      "runtime-upsert",
      "default-lookup"
    ],
    "impact": {
      "level": "中",
      "why": "后续为 `local` 选择的命令和文件系统请求可能被路由到攻击者控制的远程执行器。"
    },
    "likelihood": {
      "level": "中",
      "why": "利用需要访问 app-server 客户端边界和实验性方法。"
    },
    "limitations": [
      "此覆盖不会直接在受害主机上执行代码。"
    ]
  },
  "remediation": "在 `EnvironmentManager::upsert_environment()` 内部复用启动时的保留 ID 检查，以便运行时变更路径拒绝保留的 `local` 标识符。",
  "remediationTests": [
    "断言带有 `environmentId: \"local\"` 的 `environment/add` 返回协议错误。",
    "断言在拒绝 upsert 后，`default_environment()` 仍能解析管理器拥有的本地运行时。"
  ],
  "preventiveControls": [
    "集中保留标识符验证，使每个环境变更路径共享一个守卫。"
  ]
}
```

`rootCause.code` 和 `rootCause.language` 仍然支持只能提供一个片段的旧版生成器。新版生成器应使用共享的 `codeEvidence` 目录，分配调用栈角色，并按从输入到结果的顺序排列 `rootCause.evidenceRefs`，以便相同的精确源代码可以支持根本原因、验证和攻击路径分析，而无需将其复制到多个字段中。