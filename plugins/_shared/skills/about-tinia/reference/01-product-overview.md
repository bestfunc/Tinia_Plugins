# 产品总览

> 看完这篇，你能用 30 秒讲清"Tinia 是什么、给谁用、跟传统工具有何不同"。其他文档都是这一篇的展开。

---

## 一句话定位

**Tinia 是面向声学 / 振动 / NVH 领域的 AI-native 数据分析操作系统**。

工程师用它做的事，跟用 HEAD ArtemiS、Siemens Testlab 类似 —— 采集数据、跑算法、出报告 —— 但用的方式完全不同：**画一张节点流程图，AI 帮你搭、跑、改、写报告**。

更长的版本：

> Tinia is the AI-native operating system for industrial acoustics, vibration and NVH —— built for the era when every engineering decision needs an AI co-pilot.

---

## 产品矩阵（5 个 SKU）

Tinia 不是一个单品，是面向**学生 → 个人工程师 → 公司团队 → 工厂**的分层产品矩阵。

```
┌─────────────────────────────────────────────────────────────┐
│  Production（在线产线版，路线图 / 规划中）                    │
│  • 实时流处理 / 多机协同 / MES 集成 / 模型 OTA                │
│  • 面向：工厂在线监测（风电场、工业泵、电机厂等）             │
│  • 定价：项目制                                              │
├─────────────────────────────────────────────────────────────┤
│  ━━━━━━━━━━━━━━━━━━━ 以下四档均已交付 ━━━━━━━━━━━━━━━━━━━━━━━┤
│                                                              │
│  SaaS（云端多组织版）                                         │
│  • 公网托管 / 0 部署 / 多租户多组织 / 跨组织协作              │
│  • 面向：个人 / 小团队 / 早期客户                              │
│  • 定价：按 seat 订阅                                        │
│                                                              │
│  Server（团队私有化版）                                       │
│  • Web 部署在公司内网 / 多用户协作 / 团队管理                  │
│  • 面向：公司 NVH 部门、Tier1 测试团队、检测中心、合资 OEM    │
│  • 定价：按团队 seat 订阅 / 永久买断 + 实施服务                │
│                                                              │
│  Pro（个人桌面版）                                            │
│  • 完整离线分析工作站 / 多通道 / Order Tracking / 报告        │
│  • 面向：个人 NVH 工程师、独立 NVH 咨询顾问、自由职业          │
│  • 定价：按 seat 订阅 / 永久买断                              │
│                                                              │
│  Community（永久免费）                                        │
│  • 基础节点 + 单通道分析 + AI 辅助 + 限流程复杂度              │
│  • 面向：学生、研究者、个人入门                                │
│  • 定价：免费（节点商店上的商业节点单独收费）                  │
└─────────────────────────────────────────────────────────────┘
```

> ⚠ **SKU ≠ 代码 edition**：代码层只有 3 个部署 edition —— `desktop` / `server` / `saas`（见 `server/internal/config`）。Community 与 Pro 都跑在 `desktop` edition 上，区别在**是否激活**（桌面激活校验中间件 `EnforceDesktopActivation`）；Community / Pro / Production 是**商业 SKU / 打包概念**，不是独立的 edition flag。**Production 暂无对应 edition，是路线图项**。详见 `03-edition-comparison.md`。

**为什么这个矩阵能成立**：

- **Pro / Server / SaaS** 跟传统 NVH 软件相比，单 seat 价格更低，是因为云原生 + 订阅 + AI 增效让单 seat 边际成本接近零（传统软件基于桌面单机架构，边际成本结构不同）
- **Server / SaaS** 让公司客户能像传统大厂一样团队部署（私有化 or 云端托管），但获得 AI + 节点商店的新工作方式；SaaS 已支持多租户多组织、组织间数据严格隔离
- **Community** 是传统大厂不敢做的 —— 他们的核心客户是大企业，免费版会侵蚀付费版。Tinia 起步阶段没有存量包袱
- **Production** 是 SmartQuality（Bestfunc 现有产品线）客户的自然升级，路线图项目

详见 `03-edition-comparison.md`。

---

## 核心差异化（三条护城河）

跟传统 NVH 软件相比，Tinia 的三个"不可复制"的架构选择：

### 1. 节点流程 + AI 编排（不是脚本，是 DAG + Agent）

传统工具基本是"菜单驱动 + 模块化处理脚本"。Tinia 是**节点 DAG**：

- 每个分析能力是一个独立节点（声级、频谱、时频分析、Order Tracking…）
- 节点连成有向无环图，数据自动按拓扑流动
- AI 能直接读懂 DAG、改 DAG、新建节点 —— 工程师只要描述"我要看这段信号的频谱在转速 1000-3000 RPM 段的演化"，AI 就能搭流程
- HEAD ArtemiS 16 才刚加 Automation Project，本质仍是脚本化向导

### 2. MCP-native 协作 + SDK 外部调用（两条并行通路）

Tinia 对外开放两条互补通路：

**(a) MCP-native（AI 自动驾驶平台）**：主仓直接内置 MCP server（`/api/v1/mcp`），让 Claude Code、Codex、Qwen CLI 等 AI 客户端通过 OAuth 2.1 一键授权后**直接驱动 Tinia 完成节点开发 / 流程编排 / 数据上传 / 跑流程 / 发布**全流程。

- 工程师在 VSCode 里用 Claude Code 说"帮我写一个心声 roughness 节点"，AI 自动建项目 / scaffold / 写 run.py / 写 ParamsForm / 跑测试 / 发布到商店
- HEAD / Testlab 是 Windows 单机软件，有 SDK 但没有"AI 工程师助手"形态

**(b) Python SDK（外部程序调用平台算力）**：外部 Python 程序用 `tinia_sdk` 调用平台上已调好的分析 —— 传数据拿结果，**算法仍在平台 server 进程内执行**（没有第二个引擎）。超管在「SDK 管理」输入名称即生成可下载 SDK 包（凭据 license + 服务器地址已内置，零配置）。

- 三种调用：直接传节点类型 + 参数 / 节点表单「复制参数」串 / 引用平台流程里调好的节点（平台改参自动生效）；整条流程也能整体调用（「API 输入」+「API 输出」节点）
- 支持流式会话 / 实时数据流（边采边算、在线监测）、同机 Unix domain socket 直连 + 路径直传（大文件更快）、超管侧 SDK 调用分析（调用量 / 成功率 / 耗时 / Top 节点 / 失败排查）
- 这是 MCP 之外的"机器对机器"通路：让既有产线脚本 / 数据系统直接借用 Tinia 已经调好的分析能力

### 3. 开放节点生态（tinia-store）

Tinia 节点商店让外部开发者发布节点供他人订阅。商业节点采用 30% 平台分成（Apple App Store 路径）。

- 工程仿真领域目前没有真正的"App Store"。Ansys / Abaqus 有 user subroutine 但没有商店生态
- 学术界（高校 NVH 实验室）是天然的节点贡献者 —— 他们的研究算法可以直接变成商品

详见 `06-competitive-landscape.md`。

---

## 关键产品决策

### 是 AI 工程师，不是工程师工具

> 客户感知价值的转变：
> 传统："我买了 HEAD" → Tinia："我买了一个会工作的工程师"

这个叙事是 Tinia 的产品主线。功能层面 Tinia 会持续补齐传统工具的核心算法，但产品故事一定要从**"工具替代"转向"AI 工程师 + 节点商店"** —— 这是客户决定 5 年后用谁的根源。

### 离线 vs 在线的边界判断

- **现在**：Tinia 已是**离线 + 团队 + 云端**多形态分析平台，对标 HEAD/Testlab。产线级在线监测仍靠 SmartQuality（Bestfunc 现有产品）过渡
- **路线图**：把 Tinia 的流式执行引擎进一步"产线化"，推出 Tinia Production。底座已具备 —— 流式执行引擎（边算边推）、SDK 流式会话（实时数据流）、常驻执行池（热进程加速）都已交付，可作为在线场景的运行时基础
- 投资人买的是"产线工具"叙事（百亿+ TAM），不是"工程师工具"（中国 2-3 亿/年 TAM）。但短期必须先把离线 / 团队工具卖出去回血

> 历史说明：早期设计曾规划 `tinia-engine` / `tinia-runtime` 拆成独立仓库 / 独立引擎，该方案已**归档**（双引擎必然版本漂移、收益被"桌面单二进制 + headless"覆盖）。现状是单一执行引擎只在 tinia-server 进程内，节点 Python SDK 已并入主仓 `server/sdk/python/`（go:embed 注入）。

详见 `08-roadmap.md`。

### 团队 / 公司背景

Tinia 由具备多年工业声学 / 振动 / NVH 与 AI 经验的工程团队推进。公司有现有声学相关产品业务（SmartQuality 在线检测、Argus 工业 AI 运维平台），Tinia 是新一代核心产品。

具体团队规模 / 公司财务状态等信息**不在 skill 中暴露**，需要时通过销售渠道获取。

---

## 最常见的客户场景（速览）

| 客户类型 | 典型痛点 | Tinia 解法 | 详见 |
|---|---|---|---|
| 整车厂 / Tier1 NVH 工程师 | HEAD 太贵 + 不灵活、做新算法要等下个版本 | Pro：节点商店即时更新、AI 编排 | `scenarios/01-nvh-engineer-workflow.md` |
| 高校研究生 / 课题组 | 没经费买 HEAD、自己写 Python 又散乱 | Community + AI 辅助、研究成果发布到节点商店 | `scenarios/02-research-lab-usage.md` |
| Tier1 测试部门日常 | 一份报告做 8 小时，模板不能复用 | Pro：流程模板复用 + 报告导出 + AI 写解读 | `scenarios/03-tier1-supplier-test.md` |
| 风电场 / 工业泵 PdM | 自家产线没有 NVH 软件能用 | Production（路线图）：在线流处理 + MES 集成 | `scenarios/04-online-monitoring.md` |

---

## 你应该记住的 5 件事

1. **Tinia 不是工具替代品，是 AI 工程师 + 节点商店** —— 跟客户讲故事时一定要拔高到这一层，否则就被 HEAD 比下去
2. **5 个 SKU 产品矩阵**（Community / Pro / Server / SaaS / Production）覆盖学生到工厂全谱，按场景对应推荐；其中 Community/Pro/Server/SaaS **已交付**，Production 在路线图。注意 SKU ≠ 代码 edition（代码只有 desktop/server/saas 三个）
3. **MCP-native + SDK 两条通路** 是架构层面的领先，不是后期能加的功能 —— AI 自动驾驶（MCP）与外部程序调用算力（SDK，含实时流式会话）两手都有，对手短期追不上
4. **节点商店** 已内部跑通、在线发布订阅可用（@handle 命名空间隔离），对外开放是持续推进的主线，今天就可以跟客户讲
5. **不要被算法清单（134 项笛卡尔积）压住** —— HEAD 经过 30 年累积达到 134 项，Tinia 用 2 年到 40+ 官方节点 + AI 编排 + 节点商店长尾，**价值在工作方式而非算法数量**

---

## 下一步

- 想搞清架构 → `02-architecture.md`
- 想看版本差异 → `03-edition-comparison.md`
- 想看跟对手对比 → `06-competitive-landscape.md`
- 想看路线 → `08-roadmap.md`
