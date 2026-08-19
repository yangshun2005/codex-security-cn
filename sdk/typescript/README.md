# `@openai/codex-security`

用于运行 Codex 安全扫描的开源 TypeScript SDK 和 CLI。该仅支持 ESM 的包包含 TypeScript 声明、`codex-security` 可执行文件以及匹配的 Codex 运行时。

> [!NOTE]
> 此包遵循语义化版本控制。在 `1.0.0` 之前，其公共 API 可能在次要版本之间发生变化。

## 安装

```bash
npm install @openai/codex-security
npx @openai/codex-security --version
```

该包支持 macOS、Linux 和 Windows，需要 Node.js 22.13.0 或更高版本（22.x 发布线）、Node.js 24.x 或 Node.js 26.x。扫描、批量扫描、导出、扫描历史和已保存的发现结果还需要 Python 3.10 或更高版本。Python 3.10 还需要 `tomli`。在 `scan`、`bulk-scan` 或 `export` 命令中使用 `--python`；在 SDK 中使用 `pythonPath`。设置 `PYTHON` 可为任何基于 Python 的命令选择解释器。

当有新版本可用时，CLI 会显示适合您安装方式的更新命令。设置 `CODEX_SECURITY_NO_UPDATE_NOTICE=1` 可隐藏该通知。在 CI 环境以及 stderr 不是终端时，通知也会被禁用。

## 从 TypeScript 运行扫描

使用 `npx @openai/codex-security login` 登录，或设置 `OPENAI_API_KEY` 或 `CODEX_API_KEY`。然后创建客户端并扫描您拥有或有权限评估的仓库：

```ts
import { CodexSecurity } from "@openai/codex-security";

const security = new CodexSecurity();

try {
  const result = await security.run("/path/to/repository", {
    outputDir: "/path/outside/repository/results",
  });

  console.log(result.reportPath);
  console.log(result.findings.findings.length);
} finally {
  await security.close();
}
```

SDK 支持仓库、路径、已提交差异和工作树目标。使用 `security.preflight()` 验证本地输入，使用 `onWorkerStatus` 和 `onReconnect` 观察长时间运行的扫描，并使用 `AbortSignal` 取消扫描。

成功的结果在可用时包含 `repositoryFindings` 中的开放仓库发现结果；`findings` 始终是当前扫描的结果。匹配早期发现结果可能产生一次额外的模型调用，包括在扫描成本限制内。

结果可能包含源代码摘录、漏洞详情和复现步骤。请将结果目录和已保存的报告放在仓库之外，并限制只有授权的审查人员才能访问。

### SDK 配置和扫描选项

将运行时配置传递给 `CodexSecurity` 构造函数：

| 选项             | 描述                                                                 |
| ---------------- | -------------------------------------------------------------------- |
| `pluginPath`     | 使用 Codex Security 插件目录或 ZIP 文件，而非捆绑的插件。            |
| `pythonPath`     | 在查询 `PYTHON` 之前选择 Python 解释器。                             |
| `codexOverrides` | 将支持的设置深度合并到隔离的 Codex 配置中。                          |

将扫描配置传递给 `security.run(repository, options)` 或 `security.preflight(repository, options)`：

| 选项                      | 描述                                                                                       |
| ------------------------- | ------------------------------------------------------------------------------------------ |
| `auth`                    | 选择 `"auto"`、`"chatgpt"` 或 `"api-key"`。                                                |
| `target`                  | 选择仓库、仓库相对路径、已提交差异或工作树差异。                                           |
| `mode`                    | 选择 `"standard"` 或 `"deep"`；深度模式支持仓库和路径目标。                                |
| `knowledgeBasePaths`      | 添加架构文档、安全策略、威胁模型或目录。                                                   |
| `outputDir`               | 选择外层 Git 工作树之外的产物目录。                                                        |
| `archiveExisting`         | 在开始扫描之前，将 `outputDir` 中已有的结果归档。                                          |
| `maxCostUsd`              | 当预估模型成本超过正数美元金额时停止。                                                     |
| `maxTimeHours`            | 将深度扫描的发现阶段限制为正数小时数，最多 96 小时。                                       |
| `failureSeverity`         | 在保存的扫描配方中记录发现严重性策略。                                                     |
| `parentScanId`            | 将重运行链接到现有的父扫描。                                                               |
| `expectedPluginVersion`   | 重放扫描时要求使用原始插件版本。                                                           |
| `signal`                  | 使用 `AbortSignal` 取消扫描。                                                              |

进度和生命周期回调包括 `onAuthentication`、`onCost`、`onOutputArchived`、`onOutputDirReady`、`onScanStarted`、`onTrustedAccessStatus`、`onReconnect`、`onSessionEvent`、`onWorkerStatus`、`onWarning` 和 `onObserverError`。`onSessionEvent` 接收保存的扫描和工作线程事件及其线程 ID 和工作线程编号。预检不会启动运行时、进行身份验证、解析 Python、检查插件或运行这些扫描生命周期回调。

## 身份验证

本地使用，请使用 ChatGPT 登录：

```bash
npx @openai/codex-security login
npx @openai/codex-security scan .
```

在远程或无头机器上，使用设备身份验证：

```bash
npx @openai/codex-security login --device-auth
```

对于 CI，设置 `OPENAI_API_KEY` 或 `CODEX_API_KEY`。要改为存储 API 密钥，请通过标准输入传递：

```bash
printenv OPENAI_API_KEY | npx @openai/codex-security login --with-api-key
```

环境 API 密钥直接提供给当前扫描，绝不会保存到 Codex 凭据主目录或系统钥匙串中。只有显式的 `login --with-api-key` 才会存储 API 密钥。

要显式传递 Codex 访问令牌，请使用 `login --with-access-token` 并通过标准输入提供令牌。访问令牌环境变量不会自动用作扫描 API 密钥。

要使用其他推理提供商，请设置其 API 密钥并选择其提供商：

```bash
export OPENROUTER_API_KEY="<your-openrouter-api-key>"
npx @openai/codex-security scan . --provider openrouter --model anthropic/claude-sonnet-4.5

export FIREWORKS_API_KEY="<your-fireworks-api-key>"
npx @openai/codex-security scan . --provider fireworks --model accounts/fireworks/models/qwen3-235b-a22b

export AWS_BEARER_TOKEN_BEDROCK="<your-bedrock-api-key>"
export AWS_REGION="us-east-2"
npx @openai/codex-security scan . --provider amazon-bedrock --model openai.gpt-5.6-luna
```

在 Windows 上，在 PowerShell 中设置 API 密钥：

```powershell
$env:OPENAI_API_KEY = "<your-api-key>"
npx @openai/codex-security scan C:\code\repository
```

使用 `npx @openai/codex-security login status` 和 `npx @openai/codex-security logout` 检查或移除已存储的登录信息。Codex Security 将其登录信息保存在 `$CODEX_SECURITY_STATE_DIR/codex-home` 下的私有、稳定的 Codex 主目录中；如果未配置状态目录，则保存在 `$CODEX_HOME/state/plugins/codex-security/codex-home`。在受管理的 Windows 设备上，会保留 `SYSTEM` 和本地 `Administrators` 的继承访问权限，同时保护主目录免受其父目录未来更改的影响。其他用户和广泛的组会被拒绝，并且支持 PowerShell 受限语言模式。登录、状态、注销和扫描使用相同的主目录。Codex 使用其配置的文件或系统钥匙串后端管理凭据，并遵循受管理设备策略。仅当专用主目录中尚未包含已存储的凭据时，才会导入现有的基于文件的 Codex 登录信息。注销会阻止后续扫描自动重新导入该环境登录信息，直到您显式重新登录。

默认情况下，环境 API 密钥优先于已存储的登录信息。当同时存在已存储的 ChatGPT 登录信息和环境 API 密钥时，交互式扫描会询问使用哪个凭据。JSON 输出、试运行、CI 和其他非交互式扫描从不提示，并保持自动 API 密钥优先。使用 `--auth` 显式选择凭据来源：

```bash
npx @openai/codex-security scan . --auth chatgpt
npx @openai/codex-security scan . --auth api-key
```

`--auth chatgpt` 使用已存储的登录信息并忽略 `OPENAI_API_KEY` 和 `CODEX_API_KEY`。`--auth api-key` 需要这些环境变量之一。省略 `--auth` 或传递 `--auth auto` 可为现有 CI 和无人值守扫描保留自动 API 密钥优先。SDK 接受相同的选择，如 `security.run(repository, { auth: "chatgpt" })` 和 `security.preflight(repository, { auth: "chatgpt" })`。

要改为将已存储的 ChatGPT 登录信息设为自动默认值，请取消设置任何已配置的 API 密钥变量：

```bash
unset OPENAI_API_KEY CODEX_API_KEY
```

交互式选择仅适用于当前扫描，不会持久化。

当配置了环境密钥时，ChatGPT 登录和 `codex-security login status` 会识别有效的扫描凭据来源，而不会打印其值，包括在不存在已存储登录信息时。

某些网络安全请求和受保护的发现结果需要通过 Cyber 可信访问获得批准。要申请或检查您的访问权限，请访问 [chatgpt.com/cyber](https://chatgpt.com/cyber)。

## CLI

```bash
npx @openai/codex-security scan
npx @openai/codex-security scan /path/to/repository
npx @openai/codex-security scan /path/to/repository --headless
npx @openai/codex-security scan /path/to/repository --patch
npx @openai/codex-security scan /path/to/repository --patch --patch-severity high --json
npx @openai/codex-security scan /path/to/repository --patch --patch-severity high --create-pr
npx @openai/codex-security scan /path/to/repository --model gpt-5.6-terra
npx @openai/codex-security scan /path/to/repository --model gpt-5.6-terra --effort high
npx @openai/codex-security scan /path/to/repository --path src --path tests
npx @openai/codex-security scan /path/to/repository --knowledge-base /path/to/threat-models --knowledge-base /path/to/architecture.pdf
npx @openai/codex-security scan /path/to/repository --scan-prompt-file scan.md --post-scan-prompt-file follow-up.md
npx @openai/codex-security scan /path/to/repository --diff origin/main --json
npx @openai/codex-security scan /path/to/repository --output-dir /path/outside/repository/results
npx @openai/codex-security scan /path/to/repository --output-dir /path/outside/repository/results --archive-existing
npx @openai/codex-security scan /path/to/repository --verbose
npx @openai/codex-security scan /path/to/repository --dry-run
npx @openai/codex-security scan /path/to/repository --fail-on-severity high
npx @openai/codex-security scan /path/to/repository --max-cost 5
npx @openai/codex-security scan /path/to/repository --mode deep --workers 2 --subagents 0 --stop-after-no-new 3 --max-discovery-runs 10 --max-time-hours 1.5
npx @openai/codex-security install-hook
npx @openai/codex-security bulk-scan
npx @openai/codex-security bulk-scan --model gpt-5.6-terra --effort high
npx @openai/codex-security bulk-scan --workers 4 --mode deep --max-attempts 3 --max-cost 25
npx @openai/codex-security bulk-scan repositories.csv --output-dir /path/outside/repositories/security-scans --workers 4 --knowledge-base /path/to/threat-models --knowledge-base /path/to/architecture.pdf
npx @openai/codex-security bulk-scan repositories.csv --output-dir /path/outside/repositories/security-scans --scan-prompt-file scan.md --post-scan-prompt-file follow-up.md
npx @openai/codex-security scans list /path/to/repository
npx @openai/codex-security scans list --scan-root /path/outside/repository/results
npx @openai/codex-security scans show
npx @openai/codex-security scans show SCAN_ID
npx @openai/codex-security scans logs
npx @openai/codex-security scans logs SCAN_ID
npx @openai/codex-security scans rerun
npx @openai/codex-security scans rerun SCAN_ID
npx @openai/codex-security scans match PREVIOUS_SCAN_ID CURRENT_SCAN_ID
npx @openai/codex-security scans match --all
npx @openai/codex-security scans compare
npx @openai/codex-security scans compare PREVIOUS_SCAN_ID
npx @openai/codex-security scans compare PREVIOUS_SCAN_ID CURRENT_SCAN_ID
npx @openai/codex-security findings
npx @openai/codex-security findings list /path/to/repository
npx @openai/codex-security findings false-positive OCCURRENCE_ID --reason "The route already checks permissions"
npx @openai/codex-security export
npx @openai/codex-security export /path/outside/repository/results --export-format sarif --output /path/outside/repository/results.sarif
npx @openai/codex-security export /path/outside/repository/results --export-format csv --output /path/outside/repository/findings.csv
npx @openai/codex-security export /path/outside/repository/results --export-format json --output /path/outside/repository/findings.json
npx @openai/codex-security publish scan /path/outside/repository/results --to linear --linear-team TEAM_ID
npx @openai/codex-security publish scan --to linear --linear-team TEAM_ID
npx @openai/codex-security validate /path/outside/repository/findings.json "Possible SQL injection in src/query.ts:42"
npx @openai/codex-security validate "Possible SQL injection" --effort high
npx @openai/codex-security patch /path/outside/repository/findings.json "Missing authorization check in src/routes.ts:18"
npx @openai/codex-security patch "Missing authorization check" --effort high
npx @openai/codex-security patch OCCURRENCE_ID
npx @openai/codex-security patch --scan SCAN_ID --severity high --json
npx @openai/codex-security patch --scan SCAN_ID --severity high --create-pr
npx @openai/codex-security patch --resume-pr codex-security/patch-SCAN_ID
npx @openai/codex-security patch --scan latest --severity medium
npx @openai/codex-security patch --linear-issue SEC-123 --linear-issue SEC-124
npx @openai/codex-security patch --linear-project "Security backlog" --linear-filter '{"labels":{"name":{"eq":"security"}}}'
```

运行 `npx @openai/codex-security --version` 查看已安装的 CLI 版本，或运行 `npx @openai/codex-security info --json` 查看包、捆绑插件、Codex 运行时、默认模型、推理强度和首次扫描命令。使用 `--dry-run` 的扫描还会报告其有效模型和推理强度，包括 `--codex` 覆盖项，而无需启动 Codex 或联系网络。

`install-hook` 在每次提交前扫描暂存和未暂存的更改。它遵循 `core.hooksPath`，不会替换现有钩子，并会阻止高严重性发现结果或失败的扫描。设置 `--fail-on-severity` 可更改阈值。

`--path` 将扫描范围限定为一个或多个路径，`--diff` 扫描已提交的更改，`--working-tree` 扫描暂存和未暂存的更改。深度扫描支持仓库和路径目标。输出目录必须在被扫描目录及其任何外层 Git 工作树之外。生成 SARIF 时，会写入 `<scan-dir>/exports/results.sarif`。

工作树快照包括来自未跟踪的嵌套 Git 仓库的文件。已初始化的子模块必须干净，并检出到父仓库记录的提交。

重复 `--knowledge-base PATH` 可添加多个文件或目录；`bulk-scan` 会与每个仓库共享它们。目录会递归搜索 Markdown、文本、PDF 和 Word（`.docx`）文件。

### 配置深度扫描

对于 `scan --mode deep`，`--workers` 限制并发的发现工作线程，`--subagents` 控制每个工作线程的子代理，`--stop-after-no-new` 在这么多轮未发现新问题后停止，`--max-discovery-runs` 限制总运行次数，`--max-time-hours` 限制发现时长。这些选项也可用于 SDK 扫描：

```ts
await security.run("/path/to/repository", {
  mode: "deep",
  workers: 2,
  subagents: 0,
  stopAfterNoNew: 3,
  maxDiscoveryRuns: 10,
  maxTimeHours: 1.5,
});
```

在 `~/.codex/codex-security/config.toml` 中设置默认值，或在配置了 `$CODEX_HOME` 时在其下设置。显式的 CLI 和 SDK 设置会覆盖这些默认值：

```toml
[deep_scan]
workers = 2
subagents =
```