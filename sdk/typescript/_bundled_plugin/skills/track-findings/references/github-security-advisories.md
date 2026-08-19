# GitHub 草稿安全公告

仅针对 `github-advisory` 目标阅读此参考文档。

## 契约

- 为每个已验证发现创建一个维护者拥有的私有草稿。不要批量处理公告。
- 在每个 API 请求中使用经过身份验证的 `gh api --hostname github.com`，并为本次运行固定所选身份和仓库。切勿依赖环境中的 `GH_HOST` 或推断的仓库上下文。
- 需要密封的 `git_revision` 目标、其经过验证的公共规范非 Fork 源仓库、默认分支以及 `ADMIN` 查看者权限。验证确切的修订版本和每个选定的发现路径。不要使用外部跟踪器。

以 `GH_HOST=github.com gh repo view github.com/{owner}/{repo}` 方式运行仓库元数据检查。将 `owner` 和 `repo` 作为单独验证的路径段，并在每个 API 端点中使用其确切值。

在每个请求中使用以下请求头：

```text
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2026-03-10
```

仅使用：

- `GET /repos/{owner}/{repo}/security-advisories?state={triage|draft|published|closed}`
- `POST /repos/{owner}/{repo}/security-advisories`
- `GET /repos/{owner}/{repo}/security-advisories/{ghsa_id}`

不要使用 `PATCH`、`/cve`、`/forks`、`/reports`、协作者或致谢工作流、临时 Fork、发布或关闭转换，或对现有公告的更新。

## 载荷

创建载荷需要 `summary`、`description` 和 `vulnerabilities`。每个漏洞都需要经过验证的生态系统、规范的包名称以及有证据支持的受影响版本范围。不要从扫描的提交中推断受影响的发布版本。仅当该发布版本存在时才包含 `patched_versions`。

仅提供经过验证的 `cvss_vector_string` 或 GitHub `severity` 之一。不要从分数或描述性文本中推导向量，也不要映射信息性严重级别。仅包含高置信度的 CWE 根因映射。保持 `cve_id`、`credits` 和 `start_private_fork` 未设置。

将描述视为最终公开内容。包括影响、受影响版本、前提条件、安全的技术和验证细节、修复或变通方法、经过验证的源上下文、角色感知位置，以及带标签的发现 ID 和指纹绑定。警告如果公告发布，这些绑定将公开。排除凭据、签名 URL、仅限内部使用的证据以及不必要的漏洞利用载荷。

`git_worktree`、`directory_snapshot` 和 `git_diff` 目标无法满足此公告源契约。应阻止使用普通位置、替换基础或头部修订版本，或声称这些修订版本代表扫描字节。

## 重复项

API 没有幂等键或全文公告搜索。分页遍历 `triage`、`draft`、`published` 和 `closed` 状态，而不打印无关的公告正文。首先匹配确切的发现 ID 和指纹绑定，然后有针对性地审查同包候选。

在每个重复请求中传递 `--hostname github.com`。从环境继承主机名的请求不是有效的重复检查。

复用恰好一个匹配的 `draft` 或 `published`。在 `triage` 或 `closed` 匹配、多个匹配或语义歧义时阻止。切勿更新现有公告。

## 创建与验证

预览完整的结构化载荷、CLI 身份、仓库、源上下文、包元数据、严重级别或 CVSS 选择、重复结果和警告。获得明确批准。

在创建之前立即重新运行源验证，并重新检查认证、仓库身份和权限、源上下文、包元数据和重复项。如有任何变化，再次预览。

将批准的 JSON 写入仓库和扫描目录之外权限为 `0600` 的临时文件，检查其中是否包含机密，并使用 `gh api --hostname github.com --input` 发送一次。切勿盲目重试不确定的创建；搜索精确绑定并停止。

通过允许的 `GET` 端点使用 `gh api --hostname github.com` 读取返回的 `ghsa_id`。将规范化后的结构化回读与批准的载荷进行比较，并要求 `state: draft` 后再报告成功。