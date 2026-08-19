# Jira Issue

仅当目标为 `jira` 时，才使用此参考。

## 契约

- 跟踪一个已验证的发现或明确选择的最多 25 个批次。每个发现使用一个 Jira Cloud issue。
- 仅使用原生 [$atlassian](app://connector_692de805e3ec8191834719067174a384) 应用。复用需要读取权限但不需要写入权限。创建和更新需要两者。如果应用不可用、已断开连接、无法读取目标或无法执行已批准的变更，则停止。
- 从重复检查到回读，固定一个已认证的 Atlassian 身份、站点和 `cloudId`、项目密钥以及 issue 类型。批次中的每个项目使用相同的目标和 issue 类型。需要其他站点、项目或 issue 类型的工作，请启动单独的运行。
- 要求用户明确确认项目受众已获批准查看发现详情。一次确认可涵盖一个经过审查的精确批次。Jira 创建权限并不能证明谁可以读取这些 issue。

请勿使用旧版 Jira 连接器、Jira Data Center、Jira Service Management 请求工作流、CLI 工具、直接 REST、浏览器自动化或 Computer Use。

## 目标与字段

按以下顺序调用 Rovo 工具：

1. 使用 `getAccessibleAtlassianResources` 解析精确站点，并使用 `atlassianUserInfo` 获取当前身份。
2. 使用 `getVisibleJiraProjects` 确认项目允许预期操作：创建操作为 `action: create`，更新为 `edit`，复用为 `browse`。
3. 使用 `getJiraProjectIssueTypesMetadata` 解析选定的 issue 类型。
4. 使用 `getJiraIssueTypeMetaWithFields` 获取其当前字段。

从当前请求中的明确选择或一个明确的活动结果中选择站点、项目和 issue 类型。遇到歧义时停止。结果分页时，获取每一页。对于批次，确认其提议的 `create`、`update` 或 `reuse` 结果所需的每项操作。保持目标固定。

构建每个创建载荷，包含：

- `cloudId`、`projectKey`、`issueTypeName`、`summary` 和 Markdown `description`
- 可选的顶层 `additional_fields`、`assignee_account_id` 和 `parent`

将优先级、组件、标签和自定义字段放入 `additional_fields`，切勿放在顶层。包含活动元数据要求的每个字段。仅在验证其键或 ID 和值有效并获得用户批准后，才使用可选字段。

切勿：

- 猜测自定义字段 ID
- 将发现严重性映射到 Jira 优先级
- 推断经办人
- 发明标签作为幂等键

在描述中包含规范的发现 ID 和主要指纹作为带标签的文本。添加已批准的发现详情、修复建议以及主技能要求的源块或基于角色的纯文本位置。仅包含已确认为项目受众批准的内容。

## 重复项

对于每个选定的发现，使用 `searchJiraIssuesUsingJql`。对发现 ID 和指纹使用项目范围的 JQL，但分别搜索每个值。不要在一次查询中组合多个发现的绑定。将扫描派生的值转义为 JQL 数据，使用 `nextPageToken` 分页，并搜索所有状态。不要打印无关的 issue 描述。

JQL 分词并不能证明精确匹配。使用 `getJiraIssue` 读取每个合理的候选。比较其带标签的绑定、受影响区域、根本原因和源上下文。在精确绑定搜索之后，仅在已确认受众安全的情况下，使用狭窄的语义术语。

- `create`：两个搜索均已完成，且没有审查过的候选是同一发现。
- `reuse`：一个 issue 同时携带两个精确绑定，且其已批准的内容已是最新。
- `update`：一个 issue 明确是同一发现，且已预览精确的提议字段更改。
- `blocked`：候选无法读取，绑定指向不同或多个 issue，或仍存在语义歧义。

更新只能更改已批准的字段。保留非自有字段。不要转换 issue 或添加评论作为跟踪的一部分。

## 预览、写入与验证

在任何变更之前，预览：

- 已认证的身份
- 站点 URL 和 `cloudId`
- 项目密钥和 issue 类型
- 受众确认和重复项结果
- 精确的 summary 和 Markdown description
- 每个附加字段

对于批次，按执行顺序显示每个项目，并获得一次明确批准，涵盖该精确列表。

在每次创建、更新或复用之前，立即使用该发现的精确 ID 重新运行源验证。然后重新检查身份、站点、预期操作访问权限、项目、issue 类型元数据、受众确认和重复项结果。如有任何变化，再次预览。

按已批准的顺序串行处理批次。对于每个 `create`，仅调用 `createJiraIssue` 一次。对于每个 `update`，仅使用已批准的字段调用 `editJiraIssue` 一次。当变更可能已成功时，切勿重试。如果创建未返回一个 issue 密钥，搜索精确绑定并作为不确定情况停止。

在继续下一项之前，使用 `getJiraIssue` 读取结果密钥。验证站点、项目、issue 类型、summary、description、两个绑定、源上下文和已批准的元数据。Jira 可能将 description 返回为渲染内容或文档；语义上比较其文本、结构和链接，而不是要求字节级相同的 Markdown。在第一个失败或不确定的结果处停止批次。仅在回读通过后，报告成功并构建规范的站点 URL。

如果批次提前停止，从精确的提供者回读和绑定搜索中重建已完成的项目。重新运行源和重复项检查。然后在恢复之前再次预览剩余项目。

## 非目标

不要添加评论、转换 issue、记录工时、附加文件或链接 issue。不要管理关注者、创建项目或用户、更改项目设置或执行 Jira Service Management 请求操作。不要将一个已批准的项目用作第二次变更或未预览发现的许可。