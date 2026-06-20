# OKF / LLM-Wiki 详细手册

日期：2026-06-20

适用对象：

- 想系统理解 LLM-Wiki、OKF、RAG、Agentic Search 差异的技术负责人。
- 想把企业知识库从“文档检索”升级为“Agent 可用知识系统”的工程团队。
- 想在 CANNBot Skills 这类大规模 skill 仓库中建立知识治理、知识编译、知识验证体系的维护者。

## 1. 执行摘要

这里的 OKF 建议按 **Open Knowledge Framework，开放知识框架** 理解：它不是某个已经被业界统一定义的软件包，而是一套面向 LLM/Agent 的知识组织工程范式。核心目标是把散落在文档、代码、API、测试、Issue、PR、经验笔记里的知识，整理成可读、可搜索、可链接、可验证、可演化的开放知识资产。

LLM-Wiki 是 OKF 的一种关键实现形态：它把原始资料编译成结构化 Wiki 页面，并通过搜索、读页、沿链接跳转、错误账本修复等方式，让 agent 像研究员一样逐步获取证据，而不是一次性从向量库里拿 top-k chunk。

最重要的判断：

1. LLM-Wiki 不是 Vector RAG 的“升级版 embedding 索引”，而是检索界面的改变：从 `lookup` 变成 `reasoning substrate`。
2. OKF 不是只管“知识内容”，还要管来源、权限、版本、证据、测试、错误修复、贡献流程和退化监控。
3. Wiki 编译本身会引入风险：事实丢失、摘要误写、页面过长、链接退化、过期知识、误导性引用。
4. 因此可行的工程路线是混合系统：Wiki 负责结构化导航，BM25/向量负责召回，原文负责最终证据，评测负责防退化。

## 2. 术语表

| 术语 | 含义 | 在 OKF 中的位置 |
|---|---|---|
| Raw Corpus | 原始资料集合，例如 Markdown、PDF、代码、日志、API 文档、Issue、PR | 真源或候选知识源 |
| Knowledge Unit | 最小知识单元，例如一个规则、一个 API 约束、一个错误模式、一个操作步骤 | 编译输入的中间表示 |
| Wiki Page | 面向人和 agent 的结构化页面 | 知识呈现和检索单元 |
| Source of Truth | 真源，某个事实或规则的唯一上游维护位置 | 防止知识漂移 |
| Citation | 事实到原始来源的引用 | 可验证性 |
| Error Book | 错误账本，记录断链、事实丢失、引用错配、页面退化等问题 | 自修复和治理 |
| Compiler | 从原始资料生成 Wiki/索引/元数据的程序或流程 | 知识构建 |
| Evaluator | 对 Wiki 和回答质量进行诊断的测试系统 | 防退化 |
| Refiner | 根据评测失败和 Error Book 修复 Wiki 的流程 | 演化 |
| Agentic Retrieval | 由 agent 通过工具多步检索，而非一次性 top-k 检索 | 使用方式 |

## 3. OKF 的核心定义

一个可落地的 OKF 至少包含八层。

### 3.1 知识源层

知识源层回答：“什么资料可以被纳入知识系统？”

典型来源：

- 产品文档：README、quickstart、用户手册、API 文档。
- 工程规范：开发规范、测试规范、贡献规范、架构说明。
- 代码：脚本、schema、测试、模板、示例工程。
- 工作流：AGENTS、Teams、插件配置、CI 配置。
- 运营知识：Issue、PR、评审结论、故障复盘。
- 外部可信来源：论文、官方文档、标准、SDK 文档。

准入要求：

- 来源可定位：每条关键结论能回到具体文件、URL、commit、版本。
- 权限可判断：私有资料、许可证、敏感信息不应进入公开 Wiki。
- 版本可记录：至少记录来源 commit 或抓取日期。
- 责任可归属：知道谁维护该知识域。

### 3.2 规范层

规范层回答：“知识应该长什么样？”

需要定义：

- 页面类型：skill、task、concept、api、error、workflow、decision、source、test。
- 页面 frontmatter：id、title、kind、source、owners、tags、updated_from_commit、stability。
- 正文结构：摘要、适用场景、前置条件、步骤、证据、相关页面、风险、验证方式。
- 链接规则：相对路径、跨域引用、反向链接、废弃页面跳转。
- 引用规则：Wiki 摘要不能替代原始来源；技术硬结论必须指向真源。

### 3.3 编译层

编译层回答：“如何从原始资料生成可用 Wiki？”

编译不是简单摘要。一个合格编译器要做：

- 解析 frontmatter、标题、Markdown 链接、代码块、表格。
- 识别页面主题、触发条件、依赖关系、任务步骤。
- 抽取“事实/规则/约束/命令/风险/验证”。
- 建立正向链接和反向链接。
- 标记真源和下游消费关系。
- 保留源路径、行号、commit 或抓取时间。
- 生成错误账本，而不是吞掉异常。

编译方式可以分阶段：

1. 规则编译：只做解析、索引、链接和元数据。
2. LLM 辅助编译：让模型生成摘要和关系建议，但必须保留来源。
3. 评测驱动编译：用 probe 问题检查事实是否丢失，再迭代修复。

### 3.4 索引层

索引层回答：“如何找到页面？”

推荐混合索引：

- 标题索引：适合命中明确 skill、API、错误码。
- 标签索引：适合按领域、任务、平台过滤。
- BM25/全文索引：适合关键词、错误信息、命令片段。
- 向量索引：适合语义相近但措辞不同的问题。
- 图索引：适合跨 skill 依赖、真源关系、任务链路。
- 时间索引：适合最近变更、过期风险、版本差异。

不要把向量索引当作唯一入口。对工程知识库而言，很多问题是精确词、路径、API、错误码、commit、文件名，BM25 和结构化字段更稳定。

### 3.5 工具层

工具层回答：“Agent 如何访问 OKF？”

最小工具集：

```text
wiki_search(query, filters?) -> page hits
wiki_read(page_id or path, sections?) -> page content
wiki_links(page_id, direction?) -> related pages
wiki_sources(page_id or claim_id) -> original sources
wiki_validate(scope?) -> diagnostics
wiki_report_error(page_id, type, evidence) -> append Error Book
```

使用原则：

- search 用于定位入口。
- read 用于理解页面。
- links 用于多跳探索。
- sources 用于最终证据。
- validate 用于维护和 CI。
- report_error 用于沉淀失败案例。

### 3.6 评测层

评测层回答：“这个知识系统是不是真的有用？”

建议评测维度：

- Retrieval accuracy：能否找到正确页面。
- Evidence sufficiency：页面和来源是否足以回答问题。
- Citation support：引用是否支撑回答中的每个关键 claim。
- Multi-hop success：跨页面、多 skill、多文档推理是否正确。
- Cost：构建成本、查询 token、延迟、维护成本。
- Freshness：知识是否过期。
- Regression：新编译版本是否比旧版本退化。

评测样例来源：

- 真实用户问题。
- Issue 和 PR 评审问题。
- CI 失败日志。
- 文档里已有的使用示例。
- 人工构造的边界问题。

### 3.7 治理层

治理层回答：“谁可以改知识，怎么防漂移？”

治理对象：

- 真源归属。
- 页面 schema 变更。
- 编译器规则变更。
- 错误账本处理 SLA。
- 外部来源可信度。
- 敏感信息过滤。
- 生成内容审查。

最低要求：

- 每个页面有 source。
- 每个硬结论有 citation。
- 每个跨域关系有 reason。
- 每个自动生成页面可重现。
- 每个错误类型有处理动作。

### 3.8 演化层

演化层回答：“知识系统如何越用越好？”

演化闭环：

```text
用户问题/Agent失败
  -> 找不到页面、读错页面、引用不足、回答错误
  -> 记录 Error Book
  -> 添加 probe / regression case
  -> 修复 source 或编译规则
  -> 重建 Wiki
  -> 重新评测
  -> 发布新版本
```

没有演化层的 Wiki 会变成静态文档站；有演化层的 OKF 才能成为 agent 的长期记忆和工作底座。

## 4. LLM-Wiki 的系统模型

LLM-Wiki 可以拆成五个核心模块。

### 4.1 Document Compiler

输入：

- 原始文档。
- 代码与测试。
- 元数据。
- 领域词表。
- 页面模板。

输出：

- Wiki 页面。
- 页面索引。
- 链接图。
- source map。
- Error Book。

关键难点：

- 不丢关键事实。
- 不把示例误写成规则。
- 不把过期路径当作当前路径。
- 不把下游摘要当真源。
- 不生成无来源的 API 约束。

### 4.2 Knowledge Graph Lite

LLM-Wiki 不一定要完整知识图谱，但至少要有轻量关系图：

```text
Skill -> Reference
Skill -> Script
Skill -> Test
Skill -> Plugin
Agent -> Skill
Workflow -> Skill
Task -> Workflow
Concept -> Skill
Error Pattern -> Debug Skill
API -> Source Doc
Downstream Skill -> Source-of-Truth Skill
```

这个图主要服务导航和治理，不一定服务复杂图算法。

### 4.3 Retrieval Tools

工具设计要让 agent 分步工作：

1. `wiki_search("AscendC tiling matmul UB 切分")`
2. `wiki_read("skill/ascendc-tiling-design")`
3. `wiki_links("skill/ascendc-tiling-design", "out")`
4. `wiki_read("reference/ascendc-tiling-design/matmul/patterns")`
5. `wiki_sources("claim:matmul-ub-alignment")`

这样做比一次 top-k chunk 更可解释，因为每一步都有路径。

### 4.4 Error Book

Error Book 是 LLM-Wiki 区别于普通文档站的关键机制。

建议字段：

```yaml
- id: EB-20260620-001
  status: open
  severity: high
  type: missing_source
  page: skills/ascendc-tiling-design.md
  claim: "DAV_3510 UB 容量为 248KB"
  evidence: "页面未链接到 npu-arch 真源或 CANN 安装路径"
  detected_by: validate_wiki
  first_seen: 2026-06-20
  owner: ops
  action: "补充 source_of_truth 链接或移除硬结论"
```

错误类型建议：

| 类型 | 含义 | 处理方式 |
|---|---|---|
| broken_link | 页面链接不存在 | 修链接或删除引用 |
| missing_source | 关键结论缺来源 | 补 citation 或降级为假设 |
| source_drift | 下游复制真源内容并发生漂移 | 改为引用真源 |
| stale_page | 页面来源 commit 已落后 | 重编译或标记过期 |
| wrong_trigger | 页面触发条件过宽/过窄 | 修改 description 或 tags |
| over_summary | 摘要压缩导致关键条件丢失 | 改 schema，保留约束 |
| conflict | 两个来源给出冲突结论 | 标记版本/平台差异 |
| sensitive_leak | 页面含 token、私密路径、账号信息 | 立即移除并审计 |
| unverified_api | API 签名或行为未验证 | 链接官方文档或本地脚本查询 |

### 4.5 Compile-Evaluate-Refine

WiCER 类方法的核心启发是：编译不是一次完成，而是用 probe 发现丢失事实，再强制保留。

工程化流程：

1. 编译生成 Wiki。
2. 运行诊断问题集。
3. 对每个失败问题定位缺失事实。
4. 把缺失事实写入 Error Book。
5. 修改编译规则或页面模板。
6. 重编译并回归。

这比“人工感觉摘要不错”可靠得多。

## 5. 与 RAG、GraphRAG、Agentic Search 的细粒度区别

### 5.1 Vector RAG

典型流程：

```text
文档切 chunk -> embedding -> 向量库
用户问题 -> embedding -> top-k chunk -> 拼 prompt -> LLM 回答
```

优势：

- 实现简单。
- 对单事实查找、FAQ、短文档问答效果好。
- 构建成本低。
- 与现有向量数据库生态兼容。

问题：

- chunk 脱上下文。
- top-k 无法表达“先读目录再读细节”。
- 多跳推理依赖模型在有限上下文里拼接证据。
- 引用支持经常是 chunk 级，不是 claim 级。
- 更新和调试通常停留在索引层，缺少知识治理。

### 5.2 GraphRAG / LightRAG / HippoRAG

典型流程：

```text
文档 -> 实体/关系/摘要抽取 -> 图结构
问题 -> 查询图/社区摘要/关系路径 -> LLM 回答
```

优势：

- 更适合关系密集、跨文档综合。
- 可以显式利用实体关系。
- 对复杂问题比纯向量更稳定。

问题：

- 抽取质量决定上限。
- 图结构对人不一定可读。
- 关系抽取错误会系统性误导。
- 图更新和冲突处理复杂。
- 仍可能缺少完整页面语境。

### 5.3 Agentic Search

典型流程：

```text
Agent -> 搜索网页/文档 -> 打开结果 -> 继续搜索 -> 综合回答
```

优势：

- 适合最新信息。
- 可跨开放网络。
- 查询策略灵活。

问题：

- 结果不稳定。
- 成本高。
- 难沉淀成组织知识。
- 对企业内部资料的权限、版本、真源治理不够。

### 5.4 LLM-Wiki

典型流程：

```text
原始资料 -> 编译为 Wiki 页面/链接/索引/Error Book
Agent -> search -> read -> follow links -> check sources -> answer/report error
```

优势：

- 人和 agent 都可读。
- 结构稳定，可复用。
- 跨页面导航自然。
- 易做治理和评审。
- 可把失败变成 Error Book，再驱动修复。

问题：

- 初始设计成本高。
- 编译质量需要评测保障。
- 对高频变化资料需要增量更新。
- 查询 token 未必比 RAG 低。

## 6. OKF 页面设计详解

### 6.1 页面类型

建议标准页面类型：

| kind | 用途 | 示例 |
|---|---|---|
| `skill` | 描述一个可调用或内部参考 skill | `ascendc-tiling-design` |
| `reference` | 对应 skill 下的详细参考文档 | `reduction/patterns` |
| `task` | 用户目标或工作流入口 | “开发 AscendC 算子” |
| `concept` | 概念解释 | “真源 skill” |
| `api` | API 说明和来源 | `torch_npu.npu_fused_infer_attention_score` |
| `error` | 错误模式 | “aic error 内存越界” |
| `workflow` | 多步骤流程 | “PR 检视流程” |
| `plugin` | 插件安装和能力 | `ops-direct-invoke` |
| `agent` | 子 Agent 职责 | `torch-npugraph-ex` |
| `decision` | 架构决策记录 | “Wiki 生成产物是否提交” |

### 6.2 Frontmatter schema

推荐基础字段：

```yaml
---
id: skill.ascendc-tiling-design
title: AscendC Tiling Design
kind: skill
source:
  path: ops/ascendc-tiling-design/SKILL.md
  commit: <git-sha>
  line_start: 1
  line_end: 120
owners:
  - ops
tags:
  - ascendc
  - tiling
  - knowledge-source
status: active
stability: stable
generated_by: cannbot-knowledge-wiki@0.1
generated_at: 2026-06-20
---
```

可选字段：

```yaml
aliases:
  - tiling 设计
  - 多核切分
  - UB 切分
depends_on:
  - skill.npu-arch
consumed_by:
  - skill.ascendc-direct-invoke-template
related_tasks:
  - task.ascendc-op-development
validation:
  - tests/unit/skills/test-structure.sh
  - tests/unit/skills/test-content.sh
risk:
  - hardware-specific
  - source-drift
```

### 6.3 正文模板

```markdown
# <Title>

## Summary

一句话说清楚页面用途。

## When To Use

- 触发条件。
- 不适用条件。

## Core Knowledge

- 规则。
- 约束。
- 决策树。

## Workflow

1. 步骤。
2. 步骤。

## Source Of Truth

- 原始文件。
- 外部可信来源。
- 版本或 commit。

## Related Pages

- 上游真源。
- 下游消费者。
- 同域替代项。

## Validation

- 测试命令。
- 检查脚本。
- 人工审查点。

## Risks

- 过期风险。
- 硬件依赖。
- 未验证假设。
```

### 6.4 Claim schema

对于关键事实，可以维护 claim 级元数据：

```yaml
claims:
  - id: claim.dav3510.ub-size
    text: "DAV_3510 的 UB 容量按该 skill 当前说明为 248KB"
    source:
      path: ops/ascendc-tiling-design/SKILL.md
      section: "UB 切分策略"
    verification: "需要与 npu-arch 真源或 CANN 平台配置交叉确认"
    confidence: medium
```

claim 级引用比页面级引用更适合技术问答，因为模型回答里的每个硬结论都需要被独立支撑。

## 7. OKF 数据模型

### 7.1 实体

```text
Document
Page
Section
Claim
Source
Skill
Agent
Plugin
Task
Concept
API
Test
Script
ErrorPattern
Decision
```

### 7.2 关系

```text
Page derived_from Document
Claim supported_by Source
Skill has_reference Reference
Skill uses_script Script
Agent uses_skill Skill
Plugin installs_skill Skill
Task requires_skill Skill
Skill depends_on Skill
Skill is_source_of_truth_for Claim
Page links_to Page
Test validates Skill
ErrorPattern diagnosed_by Skill
Decision affects CompilerRule
```

### 7.3 最小 JSON 记录

```json
{
  "id": "skill.ascendc-tiling-design",
  "kind": "skill",
  "title": "AscendC Tiling Design",
  "source": {
    "path": "ops/ascendc-tiling-design/SKILL.md",
    "commit": "abc123"
  },
  "tags": ["ascendc", "tiling"],
  "links": [
    {"rel": "has_reference", "target": "reference.ascendc-tiling-design.reduction-patterns"},
    {"rel": "depends_on", "target": "skill.npu-arch"}
  ]
}
```

## 8. 编译器设计

### 8.1 输入扫描

第一版扫描规则：

```text
find . -name SKILL.md
find . -name AGENTS.md
find . -name quickstart.md
find . -path '*/references/*.md'
find . -path '*/scripts/*'
find . -path '*/.claude-plugin/plugin.json'
find tests -type f
```

### 8.2 解析策略

Markdown：

- 解析 frontmatter。
- 提取 H1/H2/H3。
- 提取链接。
- 提取代码块语言和命令。
- 提取表格。
- 提取 checklist。

JSON：

- 插件 manifest。
- hooks 配置。
- index 文件。

Shell/Python：

- 不做深度语义分析，只提取脚本名、入口参数、危险操作、依赖命令。

### 8.3 页面生成策略

规则优先：

- `SKILL.md` frontmatter 直接进入 skill 页面。
- `description` 进入 When To Use。
- “工作流程”章节进入 Workflow。
- “参考资料”章节进入 Related References。
- `disable-model-invocation: true` 标为 internal-reference。

LLM 可辅助：

- 生成一句话 summary。
- 抽取 tags。
- 建议 related pages。
- 识别风险。

LLM 不应直接决定：

- API 签名。
- 硬件参数。
- 命令是否安全。
- 测试是否通过。
- 许可证和权限。

### 8.4 增量编译

用 git diff 判断需要重编译的页面：

```text
changed SKILL.md -> rebuild skill page and dependent task pages
changed references/*.md -> rebuild reference page and parent skill links
changed AGENTS.md -> rebuild agent/plugin pages
changed tests/* -> rebuild validation index
changed compiler schema -> full rebuild
```

### 8.5 编译产物

建议目录：

```text
wiki/
├── _index.md
├── _graph.json
├── _search.jsonl
├── _sources.jsonl
├── _claims.jsonl
├── _error_book.yaml
├── skills/
├── references/
├── tasks/
├── concepts/
├── APIs/
├── workflows/
├── plugins/
└── agents/
```

是否提交生成产物取决于仓库策略：

- 小仓库：可以提交，方便浏览。
- 大仓库：只提交 schema 和脚本，CI 生成 artifact。
- CANNBot 这种 skill 仓库：建议初期只提交 PoC 子集产物，成熟后再决定是否全量提交。

## 9. 评测体系

### 9.1 检索评测

样例：

```yaml
- id: q001
  question: "如何设计 AscendC Reduction 算子的 tiling?"
  expected_pages:
    - skill.ascendc-tiling-design
    - reference.ascendc-tiling-design.reduction-patterns
  required_terms:
    - 多核切分
    - UB切分
    - Buffer规划
```

指标：

- top-1 命中率。
- top-3 命中率。
- 是否命中真源。
- 是否误命中下游摘要。

### 9.2 回答评测

检查：

- 是否先给结论。
- 是否引用原始来源。
- 是否区分已验证/未验证。
- 是否使用正确 skill。
- 是否遗漏硬约束。

### 9.3 编译评测

检查：

- 页面是否有 source。
- 链接是否有效。
- frontmatter 是否合规。
- 页面是否过长。
- 关键 claim 是否有 citation。
- Error Book 是否为空或有可解释项。

### 9.4 退化评测

每次编译后比较：

- 页面数量变化。
- broken links 数量。
- missing source 数量。
- query benchmark 分数。
- 关键页面 diff。
- claim 数量异常变化。

## 10. 安全和合规

### 10.1 敏感信息

禁止进入 Wiki：

- token、密码、私钥。
- 内部账号。
- 未公开客户数据。
- 机器唯一标识。
- 私有仓库 URL 中的凭据。
- 日志中的个人信息。

### 10.2 许可证

外部资料进入 OKF 时必须记录：

- 来源 URL。
- 许可证。
- 是否允许复制。
- 是否只允许摘要。
- 是否需要保留版权声明。

### 10.3 自动生成内容

自动生成页面要标记：

- generated_by。
- generated_at。
- source commit。
- review status。

未经审查的 LLM 摘要不能作为真源。

## 11. 失败模式清单

| 失败模式 | 表现 | 根因 | 修复 |
|---|---|---|---|
| Wiki 变成长文档集合 | 页面巨大，agent 仍然读不动 | 没有分层和链接 | 拆页面，建立 task/concept/reference |
| 摘要丢约束 | 回答漏 dtype、平台、版本限制 | 编译器过度压缩 | claim 级保留硬约束 |
| 真源漂移 | 多处复制同一规则但不一致 | 没有 source_of_truth | 标记上游真源，下游只引用 |
| Agent 乱跳 | 搜到相关但不正确页面 | tags/aliases 太宽 | 加 negative examples 和 filters |
| 成本过高 | 每次查询读大量页面 | 页面没有 section read | 支持按 section 读取 |
| 无法维护 | Error Book 一直增长 | 没有 owner/SLA | 分配 owner 和严重级别 |
| 虚假引用 | 引用页面存在但不支撑 claim | 只有页面级 citation | 做 claim-level citation |
| 最新知识缺失 | 外部 API 已变更 | 无 freshness 检查 | 周期性重新抓取/验证 |

## 12. 分阶段学习路线

### 第 1 阶段：概念

目标：知道 OKF 和 LLM-Wiki 要解决什么问题。

阅读：

- LLM-Wiki 论文摘要和方法。
- WiCER 摘要和 compilation gap。
- Vector RAG vs LLM-Compiled Wiki 的结论。
- OKBench 的动态知识评测思想。

输出：

- 一页术语表。
- 一张 RAG / GraphRAG / Agentic Search / LLM-Wiki 对比表。

### 第 2 阶段：建模

目标：能把一个知识库拆成 source、page、claim、link、test。

练习：

- 选 5 篇 Markdown。
- 手工写 5 个 Wiki 页面。
- 为每个硬结论补 source。
- 写 10 个检索问题。

### 第 3 阶段：编译器

目标：实现最小可用 Wiki 编译器。

功能：

- 扫描文件。
- 解析 frontmatter。
- 生成页面。
- 生成索引。
- 检查断链。
- 输出 Error Book。

### 第 4 阶段：Agent 使用

目标：让 agent 用 search/read/links/sources 回答问题。

要求：

- 回答前先定位页面。
- 技术结论回到原文。
- 证据不足时报告缺口。
- 失败写入 Error Book。

### 第 5 阶段：评测和治理

目标：把 OKF 变成可持续系统。

要求：

- benchmark 固化。
- CI 检查。
- owner 机制。
- 回归报告。
- 页面审查流程。

## 13. 推荐阅读资料

外部资料：

- Retrieval as Reasoning: Self-Evolving Agent-Native Retrieval via LLM-Wiki, arXiv:2605.25480.
- WiCER: Wiki-memory Compile, Evaluate, Refine Iterative Knowledge Compilation for LLM Wiki Systems, arXiv:2605.07068.
- Vector RAG vs LLM-Compiled Wiki: A Preregistered Comparison on a Small Multi-Domain Research, arXiv:2605.18490.
- OKBench: Democratizing LLM Evaluation with Fully Automated, On-Demand, Open Knowledge Benchmarking, arXiv:2511.08598.

内部落地时应优先读：

- 仓库 README。
- 开发规范。
- 测试门禁。
- 代表性 skill。
- 历史 Issue 和 PR。

## 14. 一句话判断标准

如果一个知识系统只能“搜到片段”，它还是 RAG；如果它能让 agent 明确地搜索、阅读、沿链接追证据、报告知识错误、驱动知识修复，它才开始接近 OKF / LLM-Wiki。
