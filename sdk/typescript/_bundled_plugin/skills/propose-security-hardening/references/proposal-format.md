# 安全加固提案格式

基于漏洞披露、提供的发现、事件或评估文档、源代码证据、已完成的 Codex 安全扫描，或以上内容的组合，使用此格式进行衍生加固分析。该分析是一项设计产物，既不属于其源证据的一部分，也不证明任何发现已被修复。

## 目录

1. [产物集](#产物集)
2. [写作语气与叙事](#写作语气与叙事)
3. [结构化分析](#结构化分析)
4. [组合文档格式](#组合文档格式)
5. [提案格式](#提案格式)
6. [图表规则](#图表规则)
7. [权衡规则](#权衡规则)
8. [实施交接](#实施交接)

## 产物集

在一个分析目录下编写以下内容：

```text
<analysis_dir>/
├── context.md
├── hardening.json
├── hardening.md
├── proposals/
│   └── <opportunity-id>.md
├── diagrams/
│   ├── <opportunity-id>-before.mmd
│   └── <opportunity-id>-<option-id>-after.mmd
└── implementation/
    └── <option-id>.md
```

`context.md` 是本地工作上下文，可包含本地源码根路径。其他产物必须可分发，且仅使用仓库相对源码路径和分析相对产物链接。

仅在用户选择某个选项或明确要求实施规划后，才创建 `implementation/`。

## 写作语气与叙事

面向技术功底扎实但可能不熟悉该子系统或原始扫描的安全工程师和软件工程师撰写。文档应让读者感觉是一位资深安全工程师平静地引导同行思考设计问题：专业而温暖、精确、对不确定性坦诚、乐于接受意见。不要听起来机械、危言耸听、官僚化或过于随意。

使用第一人称作为设计评审语气的重要组成部分：

- 在实质性引导讨论中通篇使用第一人称复数来引导共同推理：“我们可以看到当前所有权边界为何发生漂移”、“如果我们保留快速路径”或“我们仅在旧代际排空期间承担此内存成本”；
- 真实且克制地使用第一人称单数来陈述作者实际执行的工作和所提出的建议：“我检查了这些调用方”、“我进行了测量”、“我无法验证设备暴露面”或“在当前约束下我推荐选项 2”；
- 当依据是源码审查、提供的证据、类比或假设时，绝不暗示代码已运行、性能已被测量或行为已被观察。用平实的语言说明所依据的基础。

这不是代词配额。不要用孤立的“我们”或“我”语句装饰机械化的行文。第一人称应展现推理过程，邀请读者进入设计选择，并明确作者的证据基础。仅以第一人称写一个象征性的开头和结尾的提案仍不符合此标准。

让专业判断自然流露。解释某个选项的吸引力所在、什么让作者犹豫、哪种权衡看起来是相称的，以及哪些不确定性阻碍了更坚定的结论。诸如“让我犹豫的是……”、“此选项的吸引人之处在于……”、“我们应该坦诚面对……”或“如果……我会对此感到满意”等措辞体现了这种语气，但并非固定脚本。使用符合实际设计的语言，避免在组合文档中重复套话。

构建连贯的技术论证，而不是填充模板。耐心地将相关参与者和边界、观察到的故障、允许其发生的结构性条件、期望的不变量以及可用的设计选择联系起来。保留必需的表格，并在精确差异、覆盖映射或跨选项比较受益于紧凑视图时充分使用它们。将它们视为供快速浏览和参考的第二层，而非替代向读者解释比较为何重要的行文。引入图表和源码引用，然后用文字解释重要的边。

清晰冷静地讨论选项。为每个严肃的备选方案给出其最强有力的合理理由、成本、残余风险以及应被选中的条件。提出推荐时不要带有推销腔调或选项表演。倾向于使用“在当前约束下我推荐选项 1”和“如果……选项 2 更可取”等措辞，而非绝对性声明。当局部修复是相称的，就直接说明，无需制造架构项目。

组合文档应简洁，但提案不应读起来像简略的分诊记录或拼凑的要点列表。为组合文档提供足够的行文，解释为什么这些机会构成一个连贯的决策集。为每个提案提供更充分的讨论，使工程师能够质疑诊断、比较选项并开始实施，而无需重新构建论证。使用段落进行推理，使用列表呈现真正适合列表形式的材料。保持行文的自然节奏；在不损害清晰度、链接、表格、代码引用或技术语言的前提下，欢迎终端友好的换行。

在接受提案之前，确保叙事本身（不依赖表格）完成了以下所有事项：

- 将观察到的证据与推断的结构性条件联系起来，并解释该推断为何合理；
- 为每个严肃的选项给出其最强有力的理由，包括它保留了什么、改变了什么、控制机制如何运作以及风险残留何处；
- 解释实质性安全、性能、内存、可靠性、运营和迁移影响背后的机制；
- 使作者的深思熟虑的观点可见，包括每个选项的吸引之处、主要关切以及何种证据能够解决这些关切；
- 介绍每个图表和表格，然后解释读者应从中获取的与决策相关的边或比较；
- 提供有条件的推荐，并指明使另一个选项更可取的事实、约束或优先级。

拒绝并重写以下行文：非个人化的、机械地镜像标题结构的、或将选项压缩为图表、差异表和一短段落的行文。深度应跟随决策的复杂性；不要为了达到人为的长度目标而填充简单的观点。

对于复杂的架构备选方案，围绕图表和表格的一段引言和一段结语通常不够。用连贯的行文展开选项，使其能够独立成立：首先给出其最强有力的理由并解释机制；然后推理安全性和残余风险；再花实际精力讨论可能改变决策的资源、可靠性和迁移影响。为该选项解释可信的引入和回滚姿态，而不仅仅是为最终推荐。压缩真正中性或简单的要点，而不是制造等长的章节。

## 结构化分析

将 `hardening.json` 编写为 UTF-8 JSON，结构如下。当附加字段具有有意义的语义时允许添加，但不要将 `extensions` 用作本应属于提案的行文的倾倒场。

第一个示例基于扫描。对于普通文档，使用紧随其后描述的 `sourceEvidence` 替代方案。

```json
{
  "documentType": "codex-security.hardening-analysis",
  "schemaVersion": "1.0",
  "analysisId": "hardening_20260619_example",
  "sourceScan": {
    "scanId": "scan_example_001",
    "manifestSha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "targetRevision": "deadbeef",
    "snapshotDigest": "codex-security-snapshot/v1:sha256:...",
    "sourceDrift": "none"
  },
  "assessment": {
    "outcome": "opportunities_identified",
    "summary": "该扫描支持一个跨领域遏制机会。"
  },
  "constraints": {
    "profile": "balanced",
    "changeHorizons": ["incremental", "medium_term", "foundational"],
    "nonNegotiables": [],
    "assumptions": [
      "未提供测量的延迟或内存预算。"
    ]
  },
  "opportunities": [
    {
      "opportunityId": "centralize-archive-containment",
      "title": "集中归档目标遏制",
      "summary": "将目标派生和遏制移到一个专属的提取边界之后。",
      "diagnosis": "多个提取路径可以独立构造文件系统目标。",
      "evidence": [
        {
          "claimType": "observed",
          "sourceKind": "finding",
          "findingId": "csf_852f90d6e1177502ff113d4a",
          "path": "src/extract.py",
          "claim": "归档条目路径在未经验证遏制的情况下到达文件系统写入。"
        },
        {
          "claimType": "inferred",
          "sourceKind": "source",
          "path": "src/extract.py",
          "claim": "目标策略由调用方而非写入边界所有。"
        }
      ],
      "desiredInvariants": [
        "每次提取写入都使用被证明保持在调用方输出根目录之下的目标。"
      ],
      "proposalPath": "proposals/centralize-archive-containment.md",
      "options": [
        {
          "optionId": "local-guards",
          "title": "加强局部防护",
          "kind": "baseline",
          "summary": "修补每个现有提取路径并添加共享回归测试用例。",
          "diagramPaths": {
            "before": "diagrams/centralize-archive-containment-before.mmd",
            "after": "diagrams/centralize-archive-containment-local-guards-after.mmd"
          },
          "findingCoverage": [
            {
              "findingId": "csf_852f90d6e1177502ff113d4a",
              "effect": "addresses",
              "tacticalFixRequired": true,
              "rationale": "局部遏制检查本身就是战术修复。"
            }
          ],
          "tradeoffs": [
            {
              "dimension": "security",
              "direction": "improves",
              "confidence": "high",
              "basis": "source-derived",
              "assessment": "观察到的写入路径拒绝逃逸条目，但未来调用方仍可能省略防护。",
              "validationPlan": "运行原始遍历 PoC 并搜索每个提取写入路径。"
            },
            {
              "dimension": "performance",
              "direction": "neutral",
              "confidence": "medium",
              "basis": "source-derived",
              "assessment": "局部词法遏制检查不增加 I/O 或进程边界。",
              "validationPlan": "在防护前后对代表性归档提取进行基准测试。"
            },
            {
              "dimension": "memory",
              "direction": "neutral",
              "confidence": "medium",
              "basis": "source-derived",
              "assessment": "防护仅需要有界的临时路径值。",
              "validationPlan": "在提取包含许多条目的归档时比较峰值 RSS。"
            },
            {
              "dimension": "reliability",
              "direction": "improves",
              "confidence": "medium",
              "basis": "source-derived",
              "assessment": "逃逸条目在文件系统副作用之前失败。",
              "validationPlan": "混合有效和无效条目进行测试，并验证确定性失败行为。"
            },
            {
              "dimension": "operability",
              "direction": "neutral",
              "confidence": "low",
              "basis": "hypothetical",
              "assessment": "不引入新服务，但拒绝遥测可能有用。",
              "validationPlan": "确认现有提取错误在生产环境中是否可观测。"
            },
            {
              "dimension": "migration",
              "direction": "neutral",
              "confidence": "high",
              "basis": "source-derived",
              "assessment": "该更改保留当前提取 API。",
              "validationPlan": "对有效的相对归档条目运行兼容性覆盖测试。"
            }
          ],
          "residualRisks": [
            "遏制策略可能在调用点之间漂移。"
          ],
          "implementationReadiness": {
            "affectedComponents": ["src/extract.py"],
            "workPackages": ["添加遏制强制和回归覆盖。"],
            "acceptanceCriteria": ["原始遍历 PoC 无法在输出根目录之外写入。"],
            "migrationNotes": [],
            "rollback": "回滚聚焦的防护和测试更改。"
          }
        }
      ],
      "recommendedOptionId": "local-guards",
      "recommendation": "仅当交付时间主导复发风险时使用基线方案。"
    }
  ],
  "openQuestions": []
}
```

对于披露、提供的发现或其他非扫描集合，将 `sourceScan` 替换为具有完整性记录的证据标识：

```json
{
  "sourceEvidence": {
    "kind": "document_collection",
    "label": "内核漏洞披露文档",
    "collectionSha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "artifactCount": 12,
    "sourceDrift": "unknown"
  }
}
```

当选项映射到披露文档而非规范扫描发现时，使用 `evidenceCoverage` 替代 `findingCoverage`：

```json
{
  "evidenceCoverage": [
    {
      "evidenceId": "evidence-001",
      "effect": "mitigates",
      "tacticalFixRequired": true,
      "rationale": "结构性边界缩小了复发风险，而受影响路径仍需要其直接的生存期修复。"
    }
  ]
}
```

### 必需语义

- 至少记录 `sourceScan` 或 `sourceEvidence` 之一。当扫描由外部证据补充时，两者可以同时存在。
- 对于已完成扫描的分析，当摘要可用且已验证时，`sourceScan.manifestSha256` 将分析绑定到密封输入。当加固在正常扫描完成之前运行时，不要要求或虚构它。当源扫描提供时，至少记录一个不可变目标标识 `targetRevision` 或 `snapshotDigest`。
- 对于非扫描分析，`sourceEvidence.collectionSha256` 将分析绑定到已清点的输入集合。记录其 `kind`、面向读者的 `label` 和正的 `artifactCount`。目标修订或快照摘要为可选，因为普通披露可能不标识这些。
- `sourceDrift` 为 `none`、`present` 或 `unknown` 之一。
- `assessment.outcome` 为 `opportunities_identified` 或 `local_remediation_preferred`。
- `assessment.summary` 是扫描报告索引使用的简洁面向读者结论。不得声称提议的工作已实施。
- `opportunities_identified` 要求至少一个完整的机会。`local_remediation_preferred` 要求空的机会列表和一个解释为何战术修复是相称的组合文档。
- `claimType` 为 `observed` 或 `inferred`。提议的行为属于选项文本，不属于证据数组。
- `sourceKind` 为 `finding`、`disclosure`、`document`、`source`、`coverage`、`threat_model`、`poc` 或 `experiment`。对披露、文档、PoC 或实验证据使用 `evidenceId`，对规范发现使用 `findingId`。
- `kind` 为 `baseline`、`incremental`、`structural`、`isolation` 或 `foundational`。
- 每个选项必须包含至少一个 `findingCoverage` 或 `evidenceCoverage` 映射。其 `effect` 为 `addresses`、`mitigates`、`unaffected` 或 `unknown`。
- `direction` 为 `improves`、`regresses`、`neutral` 或 `unknown`。
- `confidence` 为 `high`、`medium` 或 `low`。
- `basis` 为 `measured`、`source-derived`、`analogous` 或 `hypothetical`。
- 当约束不支持明确推荐时，`recommendedOptionId` 可为 `null`。否则必须命名同一机会中的某个选项。
- 每个机会和选项 ID 必须在分析内唯一，并使用小写字母、数字、点、下划线或连字符。
- 每个选项必须评估 `security`、`performance`、`memory`、`reliability`、`operability` 和 `migration`。使用诚实的 `neutral` 或 `unknown` 条目，而不是省略不方便的维度。

## 组合文档格式

按顺序使用以下标题编写 `hardening.md`。对于普通或混合集合使用 `Evidence Basis`；对于仅从 Codex 安全扫描派生的分析，`Source Scan` 仍然可接受。

```markdown
# 安全加固审查：<目标>

## 证据基础
## 约束
## 机会组合
## 推荐摘要
## 后续决策
```

在 `Opportunity Portfolio` 下，使用紧凑表格：

| 机会 | 证据 | 选项 | 推荐 | 提案 |
| --- | --- | --- | --- | --- |

使用确切的 `proposalPath` 链接每个提案。使推荐以记录的约束为条件。保持此文档易于快速浏览；将完整的技术论证放在提案文件中。以足够的行文开头，为未参与扫描的读者提供背景，并使用推荐摘要以温暖的设计评审语气解释推理，而不仅仅是重复表格。

`Evidence` 单元格必须在不开 `context.md` 的情况下具有意义。使用简短的发现或文档标题（可选后跟其 ID），或链接到提案的清晰面向读者组标签。不要写诸如 `E021, E022, E031` 之类的裸列表或不透明的规范发现哈希。例如，优先使用 `Netlink 长度和暂存失败（E021、E031）` 或将紧凑标签（如 `6 个解码边界发现`）链接到定义全部六个发现的提案。

对于 `local_remediation_preferred` 评估，保留所有必需的组合文档标题。在 `Opportunity Portfolio` 下，说明没有结构性机会符合条件；在 `Recommendation Summary` 下，解释局部修复结论。不要为虚构选项创建提案或图表文件。

## 提案格式

将每个提案命名为 `proposals/<opportunity-id>.md`，并按顺序使用以下标题：

```markdown
# 安全加固提案：<标题>

## 决策
## 执行摘要推荐
## 证据
## 当前设计与故障模式
## 期望不变量
## 约束与非目标
## 先前架构
## 选项
### 选项 1：<基线，当有用时>
### 选项 2：<第一个备选方案>
## 比较
## 推荐
## 证据覆盖与残余风险
## 迁移与推出
## 验证计划
## 实施工作包
## 开放问题
```

要求：

- 面向读者的选项编号从 1 开始，包括当选项 1 为基线时；绝不要将基于零的实现索引暴露为“选项 0”；
- 保持结构化 `optionId` 值具有语义且独立于显示顺序，以便选项可以在不重命名机器面向标识的情况下重新排序；
- 在 `Executive Recommendation` 中使用编号和简短描述性标题介绍每个选项，然后再仅按编号引用该选项；
- 在推荐子集之前使完整选项集可见，并避免可能被误认为编号选项的编号步骤列表；
- 在 `Evidence` 中明确标识观察到的和推断的声明；
- 在提案中使用每个不透明发现或证据 ID 时定义它；将其与简洁标题和一行说明其确立内容的陈述配对，当多个条目有贡献时使用紧凑证据