# LLM-Wiki / OKF 调研手册：面向 CANNBot Skills 的引入方案

日期：2026-06-20

## 0. 结论先行

LLM-Wiki 适合引入 `cannbot-skills`，但不建议作为传统 RAG 的替代品一次性重构全仓。更合理的做法是把它作为“知识组织层”：把现有 `SKILL.md`、`references/`、`scripts/`、`agents/`、`quickstart.md` 编译成可读、可链接、可校验的 Markdown Wiki，再给 agent 暴露 `wiki_search`、`wiki_read`、`wiki_links`、`wiki_validate` 等工具或约定入口。

它对 CANNBot 的核心收益不是“向量检索更准”，而是：

1. 降低大规模 skill 集合的导航成本。
2. 让 agent 能按任务链路逐步阅读，而不是一次塞入大量上下文。
3. 建立跨 skill 的依赖、真源、适用场景、验证命令和风险边界。
4. 通过 Error Book 持续记录断链、过期接口、重复真源、引用漂移、未验证结论等问题。

推荐先做 PoC：选 `ops/ascendc-tiling-design`、`model/model-infer-fusion`、`infra/cannbot-skill-reviewer`、`infra/gitcode-toolkit` 四类代表性 skill，生成 `wiki/` 原型和索引，不改变现有 skill 加载方式。

## 1. 名词辨析

### 1.1 LLM-Wiki 是什么

LLM-Wiki 是一种 agent-native retrieval 范式：把外部资料从“扁平 chunk + 向量索引”编译为“结构化、可读、互链、可维护的 Wiki 页面”。查询时，agent 不只拿 top-k 片段，而是像人读文档一样搜索、读页面、沿链接跳转、判断证据是否充分。

当前公开资料里，LLM-Wiki 主要有三条线：

- Tencent/微信团队 2026-05-25 论文：强调 Retrieval-as-Reasoning、Wiki 页面、双向链接、Error Book、`wiki_search` / `wiki_read` 工具接口。
- 小规模研究对比论文：指出 Wiki 在跨文档综合和 claim-level citation 支撑上有优势，但查询 token 成本未必低于 RAG。
- WiCER 论文：强调编译差距，提出 Compile-Evaluate-Refine 迭代，避免 LLM 编译 Wiki 时丢失关键事实。

### 1.2 OKF 是什么

“OKF”目前不是一个和 LLM-Wiki 绑定的稳定标准缩写。公开资料中至少有三种可能：

- Open Knowledge Foundation：开放知识基金会，关注开放数据、开放知识、开放许可。
- OKBench / Open Knowledge Benchmarking：面向动态知识问答的自动化 benchmark 生成框架。
- Open Knowledge Framework：如果按工程语义理解，可指“开放知识框架”，即把知识按开放格式、开放索引、可验证来源、可演化修正组织起来。

本手册后续采用第三种工程化理解：OKF = 面向 LLM/Agent 的开放知识组织框架。不要把它误认为已经有统一行业标准。

## 2. LLM-Wiki 与 RAG / GraphRAG / Agentic Search 的区别

| 维度 | 传统 Vector RAG | GraphRAG / LightRAG / HippoRAG | Agentic Search | LLM-Wiki |
|---|---|---|---|---|
| 知识单元 | chunk | 实体、关系、社区摘要、图索引 | 外部网页/文档结果 | Wiki 页面、目录索引、别名、标签、事实、来源、链接 |
| 查询方式 | 一次 top-k 检索 | 图增强检索 | 多轮搜索/浏览 | 搜索、读页、沿链接遍历、证据充分性判断 |
| 可读性 | 低，chunk 脱上下文 | 中，取决于图和摘要 | 高但不稳定 | 高，人和 agent 都能读 |
| 维护方式 | 重建索引或增量索引 | 重抽取实体/关系 | 临时查询为主 | 编译、校验、修复、Error Book 累积 |
| 强项 | 单事实查找、低成本 | 关系/社区级综合 | 最新信息探索 | 多跳推理、跨 skill 导航、可解释知识库 |
| 弱项 | 多跳和关系追踪弱 | 图构建复杂，摘要可能丢细节 | 结果不稳定，难沉淀 | 构建成本高，编译质量和页面设计决定上限 |

关键差异：LLM-Wiki 不是“更好的向量库”，而是把检索从 lookup 变成 reasoning substrate。它更像给 agent 准备一个可维护的知识工作台。

## 3. 核心优势

### 3.1 对 agent 更友好

CANNBot 的使用方式天然是 agent 工作流：先识别任务，再调用 skill，再按 references 深入。LLM-Wiki 的页面、目录、链接与这种工作方式一致。

### 3.2 支持跨 skill 组合

现有仓库已有大量知识类、模板类、调试类、测试类、工具类 skill。单个 `SKILL.md` 是渐进披露入口，但跨 skill 依赖关系不够显性。LLM-Wiki 可以生成：

- skill 到 reference 的依赖图。
- skill 到验证命令的映射。
- task 到 skill 组合路径。
- 真源 skill 与下游消费 skill 的关系。

### 3.3 可做知识治理

Error Book 可以记录并闭环：

- Markdown 链接失效。
- reference 被移动但 skill 未更新。
- 多个 skill 复制同一规则导致漂移。
- CANN / torch_npu / GitCode API 结论缺少可信来源。
- skill 描述过宽导致误触发。
- 长文件违反渐进式披露。

### 3.4 人机共享

向量库通常只服务模型；Wiki 页面既能服务模型，也能服务维护者和 reviewer。对 CANNBot 这种社区 skill 仓库，这是很关键的治理优势。

## 4. 风险与反例

LLM-Wiki 不适合替代所有 RAG：

- 单点 API 查找、错误码查找、命令速查，BM25/向量检索可能更便宜。
- Wiki 编译可能丢细节，尤其是参数约束、边界条件、版本差异。
- 页面太长会变成“换皮长上下文”，失去低成本优势。
- 如果没有验证器，链接和事实会随着仓库变化退化。
- 如果 agent 没有读页/跳转策略，Wiki 结构不会自动转化为效果。

因此推荐混合架构：Wiki 负责结构与导航，BM25/向量负责全文召回，原始文件负责最终证据。

## 5. CANNBot Skills 仓库适配性分析

### 5.1 仓库现状

本次已下载并阅读 `https://gitcode.com/cann/cannbot-skills` 到 `/tmp/cannbot-skills`。观察到：

- 仓库定位：面向 CANN 开发的可复用 Skills 模块。
- 目录结构：`ops/`、`model/`、`graph/`、`infra/`、`plugins-official/`、`plugins-community/`。
- 当前 `SKILL.md` 数量：109 个。
- 规范要求：可信来源、渐进式披露、简洁精炼、单一职责、可组合性、真源 skill 单向依赖。
- 测试体系：已有 skill 结构、内容、链接、唯一性等门禁。

这些特点非常适合 LLM-Wiki，因为现有仓库已经在用 Markdown 和分层 references 表达知识，只缺少统一的“编译索引、关系图、错误账本、查询入口”。

### 5.2 不建议的做法

不要做这些：

- 不要把所有 reference 合并成一个巨型文档。
- 不要把 Wiki 生成结果反向写回每个 `SKILL.md`。
- 不要只做 embeddings 后宣称引入 LLM-Wiki。
- 不要让 LLM 无校验地改写 CANN 技术事实。
- 不要破坏现有 skill 目录、frontmatter、测试和插件安装逻辑。

### 5.3 推荐架构

新增一个不直接触发的基础设施 skill：

```text
infra/cannbot-knowledge-wiki/
├── SKILL.md
├── scripts/
│   ├── build_wiki.py
│   ├── validate_wiki.py
│   └── search_wiki.py
├── references/
│   ├── page-schema.md
│   ├── compile-policy.md
│   └── error-book-taxonomy.md
└── wiki/                 # 可选：生成产物；也可放到 .cache 或 docs/generated
    ├── _index.md
    ├── skills/
    ├── tasks/
    ├── concepts/
    ├── APIs/
    └── error_book.yaml
```

`SKILL.md` 设置：

- `name: cannbot-knowledge-wiki`
- `disable-model-invocation: true`
- 定位为内部参考/基础设施，不直接响应用户触发。

### 5.4 Wiki 页面类型

建议页面 schema：

```yaml
---
title: ascendc-tiling-design
kind: skill
source: ops/ascendc-tiling-design/SKILL.md
owners:
  - ops
tags:
  - tiling
  - ascendc
  - knowledge-source
updated_from_commit: <git-sha>
---
```

页面正文固定分区：

1. Summary：一句话定位。
2. When to use：触发条件，从 description 抽取。
3. Workflow：步骤摘要，不替代原文。
4. References：列出链接到的 references。
5. Related skills：按链接、引用、同域、任务流推断。
6. Source of truth：指出真源或上游。
7. Validation：相关测试/脚本。
8. Risks：可信源不足、硬件依赖、版本假设。

## 6. PoC 实施计划

### 阶段 1：只读编译

目标：不改动现有 skill 行为，只生成 wiki。

输入范围：

- `ops/ascendc-tiling-design/SKILL.md`
- `model/model-infer-fusion/SKILL.md`
- `infra/cannbot-skill-reviewer/SKILL.md`
- `infra/gitcode-toolkit/SKILL.md`

产出：

- `wiki/_index.md`
- `wiki/skills/*.md`
- `wiki/concepts/*.md`
- `wiki/error_book.yaml`

验收：

- 所有源文件路径可回溯。
- 所有 Markdown 链接可验证。
- 每个页面有来源和更新时间。
- 不生成没有来源支撑的 CANN 技术结论。

### 阶段 2：检索工具

先不用向量库，做轻量版：

- `search_wiki.py query`：BM25/关键词搜索页面标题、aliases、tags、summary。
- `read_wiki.py path`：读页面。
- `links_wiki.py path`：列出相关页面。
- `validate_wiki.py`：断链、重复标题、缺 source、缺 validation。

等页面质量稳定后再加 embeddings。

### 阶段 3：Agent 使用约定

在相关 agent 或 plugin quickstart 中加入约定：

1. 任务不明确时先查 `wiki/_index.md`。
2. 命中 skill 后读该 skill 页面。
3. 需要技术细节时回到原始 `references/`。
4. 输出结论必须引用原始文件，而不是只引用 wiki 摘要。
5. 发现断链或过期结论写入 Error Book。

### 阶段 4：CI 门禁

加入轻量测试：

```bash
python infra/cannbot-knowledge-wiki/scripts/build_wiki.py --check
python infra/cannbot-knowledge-wiki/scripts/validate_wiki.py
```

门禁不应阻塞普通 skill 开发太多；初期可以只 warn，稳定后对断链、无 source、重复 page id 设 error。

## 7. 学习路线

### 第 1 天：理解范式

阅读：

- LLM-Wiki 论文摘要、Introduction、Method。
- Vector RAG vs LLM-Compiled Wiki 摘要。
- WiCER 摘要。

掌握：

- retrieval-as-lookup vs retrieval-as-reasoning。
- compilability / composability / evolvability。
- Error Book 的作用。

### 第 2 天：对比 RAG 和图检索

做一张对比表：

- Vector RAG
- BM25
- GraphRAG
- LightRAG
- HippoRAG
- Agentic Search
- LLM-Wiki

重点不是背算法，而是判断每类系统的知识单元、查询接口、维护成本和失败模式。

### 第 3 天：读 cannbot-skills

重点读：

- `README.md`
- `AGENTS.md`
- `docs/STANDARDS.md`
- `tests/unit/skills/test-structure.sh`
- `tests/unit/skills/test-content.sh`
- 2 个知识类 skill
- 1 个工具类 skill
- 1 个治理类 skill

目标：理解当前 skill 生态的治理边界。

### 第 4-5 天：设计页面 schema

不要先写复杂代码。先手工为 4 个代表 skill 写页面，验证 schema 是否够用。

检查问题：

- 页面是否比原 skill 更容易导航？
- 是否保留了可信来源？
- 是否能表达跨 skill 关系？
- 是否容易被 agent 用 search/read/link 三个动作遍历？

### 第 6-7 天：实现编译器原型

实现最小功能：

- 遍历 `SKILL.md`。
- 解析 frontmatter。
- 提取 Markdown 链接。
- 生成 skill 页面和 `_index.md`。
- 输出断链和缺字段到 `error_book.yaml`。

不做 LLM 自动总结也可以先跑通；摘要可先用规则提取。

## 8. CANNBot 引入后的预期收益指标

建议用这些指标验证是否真的有收益：

- 首次定位正确 skill 的轮次减少。
- 长任务中错误引用 reference 的次数减少。
- 新贡献者理解某类 skill 的时间减少。
- `SKILL.md` 断链、重复真源、描述误触发问题减少。
- 多 skill 组合任务中，agent 能明确说明为什么使用这些 skill。
- PR 审查中，reviewer 能更快定位规范依据。

## 9. 最小 PR 方案

如果要向 `cannbot-skills` 提 PR，建议拆成三步：

1. PR-1：新增 `infra/cannbot-knowledge-wiki` 内部 skill，仅包含设计文档和 schema，不生成全仓产物。
2. PR-2：新增只读脚本，支持对 4 个示例 skill 生成 wiki，测试只覆盖这 4 个。
3. PR-3：扩大到全仓，接入 CI warn 门禁，讨论是否提交生成产物。

这样风险最低，也符合仓库现有的渐进披露和治理风格。

## 10. 资料来源

- Retrieval as Reasoning: Self-Evolving Agent-Native Retrieval via LLM-Wiki, arXiv:2605.25480, 2026-05-25.
- Vector RAG vs LLM-Compiled Wiki: A Preregistered Comparison on a Small Multi-Domain Research, arXiv:2605.18490, 2026-05-18.
- WiCER: Wiki-memory Compile, Evaluate, Refine Iterative Knowledge Compilation for LLM Wiki Systems, arXiv:2605.07068, 2026-05-08.
- OKBench: Democratizing LLM Evaluation with Fully Automated, On-Demand, Open Knowledge Benchmarking, arXiv:2511.08598, 2025-10-31.
- cannbot-skills 本地克隆：`/tmp/cannbot-skills`，读取文件包括 `README.md`、`AGENTS.md`、`docs/STANDARDS.md`、`docs/skills-usage.md`、代表性 `SKILL.md` 和测试脚本。
