---
name: about-tinia
display_name: Tinia 产品百科
description: Tinia 声学/振动/NVH 数据分析平台的方方面面 —— 给开发同事 / 销售同事 / 业务同事查阅。涵盖产品定位、架构、商业模式、对比、客户场景、技术栈、路线图、术语表。能根据业务需求给出使用建议。
user-invocable: true
---

# Tinia 产品百科

这是 Tinia 的**自助百科**。无论你是开发同事、销售同事、业务同事，还是给客户做评估的 AI，都能从这里查到全部产品知识，并根据业务场景给出合理建议。

不写代码 / 不调用工具。如果需要写代码，去用 `quickstart` / `create-node` / `flow-author` 等开发类 skill。

---

## 我是谁？我从哪里看起？

### 🧑‍💼 我是销售同事 / 客户成功，要见客户

按这个顺序看：

1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 一句话定位 / 三层产品矩阵 / 核心差异化
2. **[reference/06-competitive-landscape.md](./reference/06-competitive-landscape.md)** — 跟 HEAD ArtemiS / Siemens Testlab 等传统 NVH 工具的对比
3. **[reference/05-target-customers.md](./reference/05-target-customers.md)** — 客户画像（NVH 工程师 / 研究院 / Tier1 / 风电…）+ 他们的痛点
4. **[reference/07-pricing-business-model.md](./reference/07-pricing-business-model.md)** — 三层定价模型、商业模式
5. **[faq/03-sales-talking-points.md](./faq/03-sales-talking-points.md)** — 销售内部话术 + 异议处理 *(内部参考)*
6. **[scenarios/](./scenarios/)** — 4 个典型客户场景的完整流程示例

### 🧑‍💻 我是开发同事 / 新员工，要上手

按这个顺序看：

1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 知道 Tinia 是什么
2. **[reference/02-architecture.md](./reference/02-architecture.md)** — 7 件套架构（主仓 / engine / runtime / nodes / cli / plugins / store）
3. **[reference/09-tech-stack.md](./reference/09-tech-stack.md)** — Go / React / Wails / Python / PG / MinIO 等选型
4. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — 节点 / 流程 / 数据类型 / blob / DAG 等
5. **[faq/02-developer-questions.md](./faq/02-developer-questions.md)** — 开发者最常问的问题
6. **[reference/11-mcp-ai-integration.md](./reference/11-mcp-ai-integration.md)** — MCP-native 是怎么回事

### 🧐 我是潜在客户 / 投资人 / 行业研究者，要做评估

按这个顺序看：

1. **[reference/01-product-overview.md](./reference/01-product-overview.md)**
2. **[reference/03-edition-comparison.md](./reference/03-edition-comparison.md)** — Community / Pro / Production 三层差异
3. **[reference/06-competitive-landscape.md](./reference/06-competitive-landscape.md)** — 与传统工具对比
4. **[reference/08-roadmap.md](./reference/08-roadmap.md)** — 2026 H2 三波交付 + 2027 三主线
5. **[faq/01-customer-questions.md](./faq/01-customer-questions.md)** — 客户最常问（数据安全 / 迁移 / 定价等）

### 📚 我只是想查个术语 / 概念

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`

| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 一句话定位 / 三层产品矩阵 / 核心差异化三条 |
| [02-architecture.md](./reference/02-architecture.md) | 7 件套架构 + 仓库关系图 + 数据流 |
| [03-edition-comparison.md](./reference/03-edition-comparison.md) | Community / Pro / Production 功能与目标用户差异 |
| [04-key-concepts.md](./reference/04-key-concepts.md) | 节点 / 流程 / 类型系统 / blob / DAG / 看板 / namespace |
| [05-target-customers.md](./reference/05-target-customers.md) | 客户画像 + 行业切片 + 痛点对照 |
| [06-competitive-landscape.md](./reference/06-competitive-landscape.md) | HEAD ArtemiS / Siemens Testlab / Brüel & Kjær 对比 |
| [07-pricing-business-model.md](./reference/07-pricing-business-model.md) | 三层定价模型、Store 30% 分成、商业护城河 |
| [08-roadmap.md](./reference/08-roadmap.md) | 2026 H2 三波交付 + 2027 三主线 + Won't Have |
| [09-tech-stack.md](./reference/09-tech-stack.md) | Go/React/Wails/Python/PostgreSQL/MinIO 选型理由 |
| [10-deployment-modes.md](./reference/10-deployment-modes.md) | SaaS / 私有化 / 桌面单机 / 未来 Production 部署差异 |
| [11-mcp-ai-integration.md](./reference/11-mcp-ai-integration.md) | MCP-native 是怎么回事 + Claude/Codex/Qwen CLI 接入 |
| [12-node-ecosystem.md](./reference/12-node-ecosystem.md) | 官方节点清单 / Store 生态规划 / 开发者激励 |

### Scenarios（场景层）— `scenarios/`

| 文件 | 内容摘要 |
|---|---|
| [01-nvh-engineer-workflow.md](./scenarios/01-nvh-engineer-workflow.md) | 整车 NVH 工程师的典型工作流 |
| [02-research-lab-usage.md](./scenarios/02-research-lab-usage.md) | 高校实验室 / 研究院教学 + 科研场景 |
| [03-tier1-supplier-test.md](./scenarios/03-tier1-supplier-test.md) | Tier1 测试部门日常 + 报告交付 |
| [04-online-monitoring.md](./scenarios/04-online-monitoring.md) | 风电场 / 工业泵在线监测（2027 路线） |

### FAQ（话术层）— `faq/`

| 文件 | 内容摘要 |
|---|---|
| [01-customer-questions.md](./faq/01-customer-questions.md) | 客户最常问的高频问题 + 回答 |
| [02-developer-questions.md](./faq/02-developer-questions.md) | 开发者常见问题（如何写节点 / 调试 / 发布） |
| [03-sales-talking-points.md](./faq/03-sales-talking-points.md) | 销售内部话术 + 异议处理 *(内部参考，不对外发)* |

### Glossary（术语层）— `glossary/`

| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部术语表（DAG / blob / seat / edition / dashboard / viewer / namespace / 等等） |

---

## 当你被问到的时候

> **客户**：你们跟 HEAD ArtemiS 比有什么不一样？
> **你**：先看 `reference/06-competitive-landscape.md`，里面有完整对比表，可直接引用关键 3 点（MCP-native / DAG / 节点商店）。

> **同事**：什么是 blob 句柄？
> **你**：直接给 `glossary/terms.md` 里的 blob 条目。

> **客户**：我们是新能源 Tier1 的 NVH 测试部门，Tinia 适合我们吗？
> **你**：去 `scenarios/03-tier1-supplier-test.md` 看完整场景，然后基于客户具体情况给出 Pro 还是 Community 的建议。

---

## 维护说明（给写 skill 的人看）

- 内容跨多文件，但都基于以下事实来源：
  - 主仓代码（`Tinia/`） + `Tinia/docs/`
  - `Tinia_nodes/` 节点清单
  - `Tinia_Cli/` / `Tinia_Plugins/` / `Tinia_Store/`
  - 内部产品路线图与战略稿（**不在 skill 中暴露具体数字**）
- 凡涉及"客户名 / 营收数字 / 团队规模 / 内部决策辩论"一律不写入 skill 任何文档
- 凡涉及"未来路线"用 *计划 / 路线图 / 设想* 等词修饰，不用 *承诺 / 一定 / 必须* 等词
- 跨文档引用一律相对路径，方便 GitHub 渲染
