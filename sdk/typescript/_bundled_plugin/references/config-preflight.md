# Codex 安全配置预检

Codex 安全标准和差异扫描技能应在实质性扫描工作之前运行只读辅助程序。深度扫描没有父级能力要求，且不运行此辅助程序。

仅当主机明确标识自身为 Codex 桌面应用时，才加载 `desktop-config-preflight.md`。

将 `<python_command>` 解析为已配置的 Python 解释器（POSIX shell 中为 `"$PYTHON"`，PowerShell 中为 `& "$env:PYTHON"`），否则在 Windows 上使用 `python`，在类 Unix 主机上使用 `python3`。在构造第一个辅助命令之前，检查一次当前工具面，并将该发现结果同时用于运行时检查和 `<verified-multi-agent-runtime-arguments>`。不要从首次调用中省略活动运行时事实，也不要等到收到 `incomplete` 结果后才提供这些事实。该命令写在一行上，以便在 PowerShell、命令提示符和 POSIX shell 中均可运行：

```text
<python_command> <plugin_dir>/scripts/config_preflight.py --profile <capability-profile> --cwd <scan-working-directory> --runtime-check delegation_available=<true|false> <verified-multi-agent-runtime-arguments>
```

根据当前工具面确定运行时检查值。委派工具可能被延迟显示，而非出现在初始活动工具列表中。如果 `tool_search` 可用且委派工具尚未激活，请在传递 `--runtime-check delegation_available=false` 之前搜索子代理或多代理工具。仅在工具发现未能暴露可用委派工具后，才传递 `false`。`security_diff_scan` 配置文件额外保留其现有的 `--runtime-check goal_tools_available=<true|false>`；标准和深度配置文件不检查或要求目标工具。同时将发现的工具命名空间作为运行时证据使用：当当前工具面暴露 `multi_agent_v1` 时，将 `<verified-multi-agent-runtime-arguments>` 替换为 `--multi-agent-runtime-owner native --multi-agent-runtime-version v1 --multi-agent-runtime-provenance tool-surface`。不要为 V1 传递 V2 会话上限。对于其他运行时，使用下文描述的已验证所有者、版本、容量（如需要）和来源。当静态配置完全描述活动模式且没有会话选择的运行时覆盖它时，移除占位符。当运行时暴露比用户基础配置文件更准确的有效配置值时，添加 `--effective-config <path>=<json-value>`。

对于标准和差异扫描，通过 `delegated_workers` 检查意味着运行时支持委派审查，且显式调用的扫描授权了该审查；工作槽位结果是配置的最大值，而非每个工作线程都会启动的承诺。如果运行时禁止委派，传递 `delegation_available=false`，按文档记录的父级回退路径继续，且不要将已配置的槽位描述为运行中的工作线程或降低的覆盖率。

当设置了 `CODEX_SECURITY_CONFIG_PATH` 时，在 POSIX shell 中添加 `--config "$CODEX_SECURITY_CONFIG_PATH"`，在 PowerShell 中添加 `--config "$env:CODEX_SECURITY_CONFIG_PATH"`。CLI 提供此经过清理的、shell 可读的活动工作线程配置副本，因为包含凭据的 `CODEX_HOME` 有意对受仓库影响的命令不可访问。在这种情况下，不要替换为环境中的 Codex 主目录。

否则，辅助程序自行从 `--cwd`（默认为当前工作目录）发现 Codex 配置路径。它在类 Unix 主机上读取 `/etc/codex/config.toml`，在 Windows 上读取 `%ProgramData%\OpenAI\Codex\config.toml`，然后读取 `$CODEX_HOME/config.toml`，解析 `project_root_markers`，检查匹配的 `[projects."<absolute-project-root>"].trust_level`，并从项目根目录向下到 `--cwd` 加载受信任的项目 `.codex/config.toml` 层。除非用户配置将该项目根目录标记为 `trusted`，否则不加载项目层。

当当前 Codex CLI 会话选择了 `-p/--profile <name>` 时，传递 `--codex-config-profile <name>`。当前 Codex 在基础用户配置之上、受信任项目配置之下加载 `$CODEX_HOME/<name>.config.toml`，因此辅助程序在发现项目配置之前使用该层获取项目根标记、信任和能力值。缺少配置文件即为空层，与 CLI 一致。嵌入式 `[profiles.<name>]` 查找仅保留用于未使用 CLI 标志选择 `profile` 的旧版 Codex 配置。项目本地的 `profile` 和 `profiles` 值将被忽略。对于仅限会话的 CLI 覆盖或其他无法从配置路径恢复的有效配置值，传递 `--effective-config <path>=<json-value>`。

对于定向测试或非典型运行时，重复的 `--config <path>` 参数覆盖自动发现。按从低到高的优先级传递这些手动层。

在 Codex CLI 中，即使委派可用，也直接在父级中运行辅助程序。这可将确切的命令、退出码和 JSON 结果保留在 CLI 事件流中，并避免将不可观察的子级结果归因于活动运行时。在支持委派的其他主机中，在实质性扫描工作之前，在一个专用工作线程中运行预检。派发意味着成功的工作线程生成工具调用返回具体的工作线程或线程 ID。除非该生成成功，否则不要声称工作线程正在运行，也不要调用没有接收者的通用等待。等待返回的特定 ID，并且仅接受来自该工作线程的结果。如果生成失败或未返回 ID，直接在父级中运行辅助程序并报告生成失败；切勿编造或重建辅助程序结果。工作线程应仅返回紧凑摘要：执行的命令和退出码、整体状态、未满足或未知的能力、返回的 `user_config_path` 以及适用的修复措施。包含任何冲突设置的源路径。除非父级需要解决歧义，否则不要返回辅助程序的原始 JSON。这可将预检检查排除在主扫描上下文之外。

父级应仅传递工作线程自身无法确定的运行时事实，例如选定的配置配置文件或仅限运行时的有效配置值。如果工具发现后委派不可用，直接在父级中运行辅助程序，以便预检可以报告降级或阻塞的路径。

多代理配置模式在静态配置完全描述时自动检测。模型或会话选择的运行时必须额外提供活动会话暴露的已验证运行时事实。保持协议、所有者、上限和来源分开：

```text
--multi-agent-runtime-owner native --multi-agent-runtime-version v2 --multi-agent-session-cap <count> --multi-agent-runtime-provenance <app-server|thread-context|tool-surface>
```

V2 会话上限包含根线程。对于评估当前会话工作线程容量的配置文件，辅助程序在评估可用工作线程槽位时减去该根线程。对于由静态配置选择的原生 V2，当未配置显式上限时，文档记录的 Codex 默认会话上限为四。当配置文件需要活动容量时，不要将该静态默认值应用于模型或会话选择的 V2：传递观察到的运行时上限，否则阻塞性容量需求将保持 `incomplete`。

当活动会话实际由 `codex_bridge` 管理时，提供显式的已验证所有权。仅凭后端配置值不构成所有权证据：

```text
--multi-agent-runtime-owner codex-bridge --multi-agent-runtime-version v2 --multi-agent-runtime-provenance verified-bridge --effective-config multiagent_config.max_concurrency=<count>
```

对于评估父级运行时设置的配置文件，在未传递 `--multi-agent-runtime-owner codex-bridge` 和 `verified-bridge` 来源的情况下传递 `multiagent_config.max_concurrency` 是错误的。显式运行时声明仍需要其现有的来源和所有权检查。不相关的后端配置值不会阻塞不使用父级运行时设置的配置文件。

静态原生 V2 同时接受 `[features] multi_agent_v2 = true` 和 `[features.multi_agent_v2] enabled = true`。当配置文件评估父级运行时设置或提供显式运行时事实时，原生 V2 不能与 `agents.max_threads` 组合。`agents.max_depth` 仅适用于 V1，V2 不需要。没有已验证所有权的运行时版本和上限不能满足阻塞性需求。当运行时版本、所有权或容量仍然未知时，仅当阻塞性需求需要该事实时，辅助程序才返回 `incomplete`，并省略不安全的并发补丁。

辅助程序从 `../preflight/capability-profiles.toml` 读取路由的能力配置文件，从 `--cwd` 发现适用的 Codex 配置路径，在注册表提供默认值的地方应用文档记录的默认值，并输出一个 JSON 结果。

将辅助程序结果作为预检的事实来源。不要独立重新解释配置文件要求或比较原始配置文本以进行精确相等性检查。

按以下方式解释需求严重级别：

- `block`：未满足时，所请求的工作流无法诚实声明
- `warn`：工作流只能通过文档记录的降级路径继续
- `suggest`：工作流可以继续，但当改进实质性影响长时间运行的扫描质量或可恢复性时，Codex 应提及该改进

当需求由配置支持时，在运行时暴露有效值时比较有效解析值。当运行时未暴露有效值时，回退到已加载的配置值和配置文件中存在的文档记录的 Codex 默认值。

当配置文件包含修复补丁时，呈现具体的配置差异。在交互式会话中，在编辑持久化用户配置之前询问。如果用户批准，仅编辑辅助程序的 `user_config_path`；切勿推断 `~/.codex/config.toml` 或其他 Codex 主目录。来自更高优先级项目或配置文件层的冲突值必须在辅助程序报告的源中解决，而不是通过较低优先级的编辑隐藏。在非交互式会话中，遵循下面的窄自动修复路径，而不是等待运行时无法提供的答案。切勿在辅助程序的具体补丁之外重写配置。

某些修复补丁具有 `kind = "host_setting"`。将这些呈现为主机级设置指南，而非对持久化 Codex 配置的编辑。

深度安全扫描使用 MCP 拥有的 SDK 会话，而非父线程的工作线程池。其预检不要求特定的父级委派运行时、所有权、容量或深度。发现工作线程继承扫描的模型，并在已验证的只读工作线程沙箱中运行。

不要仅因为用户的值与配置文件的建议补丁不同而发出警告。仅在评估的能力需求未满足时发出警告或阻塞。

如果阻塞性运行时能力为 `unknown`，从当前工具面确定它，并使用显式 `--runtime-check` 重新运行辅助程序。未知的 `warn` 或 `suggest` 能力不会阻塞 `ready` 扫描；按文档记录的降级路径继续，不要声称该能力可用。

## 持久扫描交接

在原生交接或直接对话开始提供 `scanId` 后，使用其权威扫描上下文，并为已验证的目标和选定的扫描模式运行此预检。上述专用预检工作线程应在目标设置、威胁建模、扫描/发现工作线程创建或其他实质性分析之前完成。

对于标准和差异扫描，应用交接在无项目计数的情况下启动预检。在每次结构化辅助程序结果之后，调用 `update_codex_security_scan_progress`，不改变阶段，并将 `preflightChecks` 设置为辅助程序 `results` 数组中的每个条目，将每个条目仅投影为 `capability`、`reason`、`severity` 和 `status`。不要随 `preflightChecks` 发送 `phaseItemsTotal`、`phaseItemsCompleted` 或 `phaseProgressUnit`：服务器从数组长度推导总数，将 `pass` 和 `fail` 计为已完成，将 `unknown` 排除在已完成之外，并推导可见的 `block` 或 `warn` 关注项。在干净重新运行后发送完整的新结果数组，以便过时的问题消失。不要将项目计数完成解释为就绪：在修复或重试仍待处理时，对每个阻塞、不完整或错误结果保持在预检阶段，即使每个返回的检查都已评估。仅在 `ready` 结果发布了其新的 `preflightChecks` 之后，单独的进度调用才能推进到 `threat_model`。这些计数和问题属于当前扫描，而非旧版设置时的工作区预检，并在扫描推进后保持可见。深度扫描预检和发现进度仍由 `start_codex_security_deep_scan` 拥有。

在 `ready` 结果后继续。当 warn 或 suggest 问题实质性影响扫描质量、容量或可恢复性时，解释这些问题，并使用文档记录的降级路径。如果结果为 `blocked` 或 `incomplete`，遵循下面的修复处理。如果辅助程序无法运行或返回其顶层 `status: "error"` 信封，报告确切阻塞原因，并在可能时重试文档记录的恢复路径。不要仅因辅助程序暂时不可用或出错而调用 `fail_codex_security_scan`；保持持久扫描运行并在稍后重试时交接，同时恢复仍可能进行。

当阻塞或不完整的预检包含可操作的修复时，首先对当前会话进行分类，然后选择修复控制。将 `codex exec`、无头运行、自动化运行以及任何无法实际暂停等待人工回复的主机视为非交互式，即使本文档中命名了 `request_user_input` 或 `request_codex_security_user_input` 或它们看起来可调用。在非交互式会话中，绝不调用任一用户输入工具，也绝不回退到聊天提问；直接进入下面的自动修复路径。仅在会话确认为交互式后，在 Codex 线程中呈现确切原因和配置差异，并乐观地调用原生 `request_user_input` 工具，以便暂停的扫描可见地等待用户决定，而非仅在纯聊天文本中提问：

```text
request_user_input(
  questions=[
    {
      "header": "预检？",
      "id": "apply_preflight_remediation",
      "question": "应用推荐的修复并重试预检？",
      "options": [
        {
          "label": "应用并重试（推荐）",
          "description": "应用已批准的修复，然后重新运行能力预检。"
        },
        {
          "label": "保持暂停",
          "description": "保持扫描运行以供稍后重试，不更改配置。"
        },
        {
          "label": "取消扫描",
          "description": "取消此扫描，不应用修复。"
        }
      ]
    }
  ]
)
```

不要设置 `autoResolutionMs`；在交互式会话中，持久配置更改或扫描继续之前需要显式答案。如果原生 `request_user_input` 不可用或出错，使用相同的 `questions` 载荷调用 `request_codex_security_user_input`。此 MCP 回退是交互式的，在非交互式会话中必须保持禁止。如果返回 `accepted`，遵循其答案。如果不可用或出错，在聊天中询问相同的选项。如果返回 `declined` 或 `cancelled`，不要推断选择；保持运行中的扫描，停止，并说明仍需要显式答案。在每个交互式等待情况下，在创建或采纳扫描目标之前停止等待用户答案。在等待该答案时不要调用 `fail_codex_security_scan`。在 `Apply and retry` 后仅应用已批准的修复，在 `Leave paused` 后保持运行中的扫描并停止，在 `Cancel scan` 后调用 `cancel_codex_security_scan`。

在非交互式 Codex 会话中，不要让运行等待它无法接收的答案。在显示确切阻塞原因和配置差异后，仅自动应用辅助程序的具体 Codex 配置补丁，使用普通的 `value` 或 `remove` 操作到活动可写用户配置，保留无关设置。绝不自动应用 `host_setting` 修复或编造补丁。使用相同的已验证运行时事实和任何新可观察的有效配置重新运行同一预检一次。仅在该重新运行返回 `ready` 后继续；不要更早创建或采纳扫描目标或开始实质性扫描工作。如果新配置需要新会话才能被活动运行时使用，重新运行仍保持阻塞或不完整，修复不可用，或辅助程序出错，不要循环、调用 `fail_codex_security_scan` 或自动取消。保留应用生成的持久扫描并交接以供稍后重试；在 CLI 或其他没有持久扫描的主机中，报告确切剩余阻塞原因并结束，不声称扫描已暂停。

对于任何非就绪结果，不要自动失败。如果交互式聊天回退在未选择取消或保持扫描运行的情况下拒绝所需修复，在采取任一操作之前询问该后续问题。如果修复不可用、辅助程序无法运行、辅助程序返回错误信封，或重新运行仍保持阻塞或不完整，保留持久运行中的扫描并在恢复仍可能时重试或交接。仅在文档记录的恢复路径已用尽且阻塞原因确认为不可恢复后，使用确切原因调用 `fail_codex_security_scan`。当用户显式取消时，调用 `cancel_codex_security_scan`。

在 Codex 线程中呈现适用的修复。在交互式会话中编辑持久化用户配置之前询问；在非交互式会话中使用上述窄自动修复路径。

Codex CLI 和其他主机在实质性工作之前使用相同的基于提示的预检。在聊天中解释确切原因和修复。交互式 CLI 会话在编辑持久配置之前询问；非交互式 CLI 会话使用上述窄自动修复路径，并仅在 `ready` 重新运行后继续。

## TAC 状态通告

在能力预检为 `ready` 之后、工作流首次实质性扫描操作之前，使用托管的 Codex 安全访问应用 [$codex-security-access](app://connector_openai_codex_security_access) 调用其 `get_tac_status` 操作恰好一次，作为自动扫描前通告。对于深度安全扫描，在 `start_codex_security_deep_scan` 之前立即进行此调用。如果此扫描在较早的延续中已执行了自动通告，重用该结果而不是再次调用。报告其确切的 `status` 和 TAC 授权级别。当 `status` 为 `not_granted` 时，在任何其他扫描开始进度之前，呈现一个单独的、突出的警告，说明此账户未授予 TAC 访问权限，受保护的扫描输出可能无法显示。包含返回的 `enrollmentUrl` 作为可点击的应用链接；如果该字段缺失，使用 `https://chatgpt.com/cyber`。不要将此警告埋藏在常规状态摘要中。当 `status` 为 `unknown`，或应用或操作不可用时，改为警告 TAC 访问无法验证，受保护的扫描输出可能无法显示。然后无论结果如何继续。此结果为通告性质，绝不允许或门控扫描。不要在扫描阶段