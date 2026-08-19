# SARIF 适配器

SARIF 是一种确定性导出，并非 Codex Security 的权威数据源。

该适配器：

- 读取密封语义包，且不修改其清单
- 将 SARIF 与规范密封分开存储
- 生成 SARIF 2.1.0 格式
- 使用稳定的 `ruleId` 值
- 保持规则描述符在多次扫描间稳定
- 生成仓库相对路径的 POSIX 路径
- 在可用时，将根控制位置置于 GitHub 注释的首位，并在 `locations` 中输出每个受影响或代码证据位置，以便漏洞汇聚点保持可匹配性
- 在 `codexSecurity/v1` 下保留语义指纹
- 在能够安全地对可用源根目录内有界、常规且非符号链接的源文件进行哈希时，生成 GitHub 的源代码行 `primaryLocationLineHash`
- 将分类严重性映射到 SARIF `level`
- 在深度扫描的每个子结果属性下保留规范的 `candidateId`，以便消费者在不改变原始 SARIF 结果展示的情况下对结果进行分组

生命周期、丰富的验证证据、攻击路径上下文和覆盖率在 SARIF 中会有损失或被省略。请在语义 JSON 中保留这些信息。

最终确定期间的自动 SARIF 导出是尽力而为的，因此投影错误不会使规范密封失效。当消费者需要 SARIF 并应暴露导出错误时，请使用严格的适配器入口点。

参考：

- [GitHub 代码扫描的 SARIF 支持](https://docs.github.com/en/code-security/reference/code-scanning/sarif-files/sarif-support-for-code-scanning)
- [OASIS SARIF 2.1.0 JSON Schema](https://docs.oasis-open.org/sarif/sarif/v2.1.0/os/schemas/sarif-schema-2.1.0.json)