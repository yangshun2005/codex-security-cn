

> **[中文化] codex-security**
>
> 此项目是 [openai/codex-security](https://github.com/openai/codex-security) 的中文翻译版本。
> - 原项目 Stars: 0
> - 主语言: 
> - 许可证: 
> - 翻译日期: 2026-08-19
> - 原始 README: [README_en.md](README_en.md)
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


---

# Codex 安全

`@openai/codex-security` 是一个用于查找、验证和修复代码中安全漏洞的 CLI 和 TypeScript SDK。

**更多详情请参阅 [Codex 安全文档](https://learn.chatgpt.com/docs/security/cli)**。

某些网络安全请求和受保护的发现结果需要通过“网络可信访问”进行审批。如需申请或检查您的访问权限，请访问 [chatgpt.com/cyber](https://chatgpt.com/cyber)。

## 快速开始

需要 Node.js 22.13.0 或更高版本（22.x 版本线）、Node.js 24.x 或 Node.js 26.x；Python 3.10 或更高版本；以及 Codex 安全的访问权限。

```bash
npm install @openai/codex-security
npx @openai/codex-security login
npx @openai/codex-security scan .
npx @openai/codex-security scan . --patch
npx @openai/codex-security scan . --patch --patch-severity high --json
npx @openai/codex-security scan . --patch --patch-severity high --create-pr
npx @openai/codex-security scan . --model gpt-5.6-terra --effort high
npx @openai/codex-security scan . --scan-prompt-file scan.md --post-scan-prompt-file follow-up.md
npx @openai/codex-security scan . --mode deep --workers 2 --subagents 0 --stop-after-no-new 3 --max-discovery-runs 10 --max-time-hours 1.5
```

对于 CI 环境，请设置 `OPENAI_API_KEY` 或 `CODEX_API_KEY`，而无需登录。环境 API 密钥会直接传递给当前扫描，且永远不会存储在 Codex 的凭据主目录或系统密钥环中。

显示发现结果摘要后，带有发现结果的交互式扫描会询问是否打开发现结果浏览器，您可以在其中查看完整详情、选择严重性阈值、选择单个发现结果，并为每个发现结果添加补丁指令。每个选定的发现结果都会在其独立的已保存 Codex 桌面任务中运行。使用 `--patch --patch-severity high` 修复高严重性和严重发现结果。添加 `--create-pr`，或在审查期间启用 Pull Request 选项，以提交已验证的文件并打开 GitHub Pull Request。普通扫描不会更改仓库文件。

深度扫描的发现过程默认在 96 小时后停止。设置 `--max-time-hours` 为任意正数小时（包括小数小时），最大为 96。达到限制时，已完成的发现结果会被保留并返回。

要使用其他推理提供商，请设置其 API 密钥并选择模型：

```bash
export OPENROUTER_API_KEY="<your-openrouter-api-key>"
npx @openai/codex-security scan . --provider openrouter --model anthropic/claude-sonnet-4.5

export FIREWORKS_API_KEY="<your-fireworks-api-key>"
npx @openai/codex-security scan . --provider fireworks --model accounts/fireworks/models/qwen3-235b-a22b

export AWS_BEARER_TOKEN_BEDROCK="<your-bedrock-api-key>"
export AWS_REGION="us-east-2"
npx @openai/codex-security scan . --provider amazon-bedrock --model openai.gpt-5.6-luna
```

Amazon Bedrock 还支持标准 AWS 访问密钥、配置文件、Web 身份、容器凭据以及默认的 AWS 凭据链。

本地登录遵循 Codex 配置的凭据后端，包括托管设备所需的系统密钥环。Codex 安全将登录和扫描凭据保存在同一个私有的持久状态目录中。

如果同时存在 ChatGPT 登录和 API 密钥，交互式扫描会询问使用哪个凭据。CI 和其他非交互式扫描保持现有的 API 密钥优先级。需要时请明确选择凭据：

```bash
npx @openai/codex-security scan . --auth chatgpt
npx @openai/codex-security scan . --auth api-key
```

要使您的 ChatGPT 登录成为自动默认选项，请取消设置任何已配置的 API 密钥：

```bash
unset OPENAI_API_KEY CODEX_API_KEY
```

扫描历史记录存储在 Codex 安全工作台状态目录中。如果该目录无法写入，请将 `CODEX_SECURITY_STATE_DIR` 设置为仓库之外的可写目录。

`findings list [repository]` 显示仓库各次扫描中的未关闭发现结果，并识别在最新扫描中未确认的发现结果。

使用 `patch OCCURRENCE_ID` 修复单个已保存的发现结果，或使用 `patch --scan SCAN_ID --severity high` 修复已保存扫描中的选定发现结果。添加 `--json` 获取结构化结果，或添加 `--create-pr` 在验证后打开 GitHub Pull Request。如果发布失败，请使用打印的 `patch --resume-pr BRANCH` 命令重试，而无需再次运行 Codex。

使用 `patch --linear-issue SEC-123` 导入并修复 Linear Issue，或使用 `patch --linear-project "Security backlog" --linear-filter '{"labels":{"name":{"eq":"security"}}}'` 修复项目中的匹配未关闭 Issue。设置 `CODEX_SECURITY_LINEAR_API_KEY` 以授权只读 Linear 访问。

`scans compare BEFORE_SCAN_ID AFTER_SCAN_ID` 自动按根本原因匹配发现结果，复用已保存的匹配项，并识别新增、持续存在、重新打开、已解决或未知的发现结果。当覆盖不完整或原始位置未被审查时，缺失的发现结果保持未知状态。

## 发布扫描发现结果

将已完成扫描中的每个发现结果发布到 Linear 团队：

```bash
npx @openai/codex-security publish scan /path/to/scan \
  --to linear \
  --linear-team TEAM_ID
```

添加 `--linear-project PROJECT_ID` 将 Issue 放入 Linear 项目中，或省略该参数以直接在团队中创建 Issue。现有的 `--project` 标志仍作为别名。省略扫描目录以交互方式选择已完成的扫描。您也可以设置 `CODEX_SECURITY_LINEAR_TEAM` 和可选的 `CODEX_SECURITY_LINEAR_PROJECT`，而不传递目标标志。添加 `--dry-run` 预览 Issue，或添加 `--json` 返回机器可读的结果。

默认情况下，发布使用您现有的 Codex 登录和已连接的 Linear 应用，无需单独的 Linear 令牌。如需直接通过 Linear API 发布，请将 `CODEX_SECURITY_LINEAR_API_KEY` 设置为 Linear 个人 API 密钥。直接发布默认将 Issue 保持未分配状态；传递 `--linear-assignee EMAIL_OR_USER_ID` 以选择 Linear 用户：

```bash
export CODEX_SECURITY_LINEAR_API_KEY=YOUR_LINEAR_PERSONAL_API_KEY
npx @openai/codex-security publish scan /path/to/scan \
  --to linear \
  --linear-team TEAM_ID \
  --linear-project PROJECT_ID \
  --linear-assignee teammate@example.com
```

使用 `--linear-assignee USER_ID` 选择 Linear 用户 ID 而非电子邮件地址，或省略该标志以保持 Issue 未分配。

`--linear-api-key KEY` 也选择直接发布，并优先于环境变量。建议使用环境变量，以避免 API 密钥出现在 shell 历史和进程列表中。每个发现结果都会创建一个新 Issue，其中包含扫描 ID、受影响的代码位置、源代码片段和修复建议。请选择有权接收仓库源代码和漏洞详情的发布目标。

## 详细诊断信息

添加 `--verbose` 将扫描诊断信息打印到 stderr：

```bash
npx @openai/codex-security scan . --verbose
```

`CODEX_SECURITY_LOG_LEVEL=debug` 也可启用诊断信息；`LOG_LEVEL=debug` 是其回退选项。JSON 结果仍输出到 stdout。

详细诊断信息可能包含敏感数据。分享前请审查本地日志。已保存的失败摘要、批量扫描回执和常规活动源会省略包含可识别凭据的消息。

使用 `npx @openai/codex-security scans logs SCAN_ID` 检查扫描及其工作进程的已保存会话事件。扫描期间按 `d` 检查未脱敏的详细信息；`a`、`m` 和 `1`–`9` 分别选择全部、主会话或工作进程会话。这些事件可能包含凭据。

## TypeScript SDK

```ts
import { CodexSecurity } from "@openai/codex-security";

const security = new CodexSecurity();
const result = await security.run(".");
await security.run(".", {
  mode: "deep",
  workers: 2,
  subagents: 0,
  stopAfterNoNew: 3,
  maxDiscoveryRuns: 10,
  maxTimeHours: 1.5,
});

console.log(result.reportPath);
await security.close();
```

## 容器化批量扫描

使用官方镜像和附带的 Docker Compose 配置，对固定到不可变 Git 修订版的仓库进行非交互式、可恢复的扫描。有关身份验证、私有结果存储和可选的 Ubuntu AppArmor 加固，请参阅 [容器快速开始](sdk/typescript/README.md#containerized-bulk-scans)。

传递 `--knowledge-base PATH` 与每个仓库共享安全文档；对多个文件或目录重复该选项。

使用 `--scan-prompt-file PATH` 添加共享扫描指令，并添加 `prompt` CSV 列以提供仓库特定的指令。使用 `--post-scan-prompt-file PATH` 在每次扫描后运行后续操作，包括不完整或失败的扫描。

有关完整的命令帮助、运行时默认值、原生多智能体工作进程限制、环境变量、深度扫描配置和 SDK 选项，请参阅 [包 README](sdk/typescript/README.md) 和 [官方 CLI 参考](https://learn.chatgpt.com/docs/security/cli/reference)。