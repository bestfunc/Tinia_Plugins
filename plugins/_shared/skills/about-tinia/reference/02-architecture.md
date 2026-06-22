# 多仓架构（现役 7 仓）

> Tinia 不是单一仓库，是一组互相解耦的子项目协作运行。这种切分是有意为之 —— 既是工程合理性，也是商业护城河（架构层面的领先，对手短期内难以追上）。
>
> **重要演进**：早期设想的"engine / runtime / graph_spec 独立成仓 + 独立 pip 包"路线已**归档不再投入**（理由见下文「历史：归档的双引擎设想」）。执行**永远发生在 tinia-server 进程内**（无第二引擎）；节点 Python SDK 已并入主仓 `server/sdk/python/`，通过 Go `go:embed` 注入。当前现役为下列 7 个仓。

---

## 总览图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                 用户层                                     │
│   开发同事 / 销售 / 客户 — 用 Web / 桌面 app / Claude Code / 外部程序       │
└──────────────────────────────────────────────────────────────────────────┘
              ↕ Web/桌面          ↕ MCP（AI 驱动）        ↕ Python SDK（程序调用）
┌──────────────────────────────────────────────────────────────────────────┐
│                              主平台（Tinia）                               │
│   Go + React + Wails，三种部署 edition（saas/server/desktop）共享同一份代码 │
│   ▸ 流程编辑器 / 数据源 / 看板 / AutoML 调参 / 设置 / 商店订阅 UI          │
│   ▸ 多租户：Org / Member / Seat / Activation                              │
│   ▸ 执行内核：DAG 调度 + Python subprocess + 常驻执行池(HotPool) + blob     │
│   ▸ 节点 Python SDK（server/sdk/python，go:embed 注入 PYTHONPATH）         │
│   ▸ MCP server（/api/v1/mcp）— AI 客户端接入点                            │
│   ▸ SDK API（/api/v1/sdk）— 外部程序调用 + 流式会话                       │
│   ▸ 内置 PostgreSQL + Python runtime（桌面单机版用）                      │
└──────────────────────────────────────────────────────────────────────────┘
        ↑ 注册节点              ↑ 节点 catalog/分成          ↑ 身份/授权/seat
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  Tinia_nodes         │  │  Tinia_Store         │  │  Bestfunc_Passport   │
│  （官方节点）        │  │  （商店服务）        │  │  （账号+授权控制面） │
│  bestfunc 命名空间   │  │  Go + React 独立部署 │  │  Gin+GORM+Postgres   │
│  约 41 个 Python 节点 │  │  • 节点发布审批      │  │  • OAuth / JWKS      │
│  跟主平台同步发布    │  │  • 用户订阅          │  │  • userinfo /        │
│  • 声学/振动/心理声学 │  │  • 节点商业分成      │  │    entitlement /seat │
│  • 流式逐帧 / 多尺度  │  │  • 实例激活(store_url)│  │  • 身份提供方(IdP)   │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
        ↑ scaffold/发布           ↑ 浏览/管理订阅              ↑ Tinia 为接入方之一
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  Tinia_Cli           │  │  Tinia_Plugins       │  │  tinia-release       │
│  （开发者 CLI）      │  │  （Claude Code plugin）│  │  （构建产物）        │
│  Go binary           │  │  marketplace 仓库     │  │  • Win/macOS 安装包  │
│  • scaffold 节点项目 │  │  • 4 个变体 plugin    │  │  • latest.json       │
│  • 本地测试          │  │  • 共享 skills        │  │    自动更新源        │
│  • tinia login 授权  │  │  • 本地 MCP bundle    │  │  • 当前约 1.35.x     │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

---

## 现役 7 个仓库各自的职责

### 1. **Tinia** —— 主仓库 / 业务平台 + 执行内核

- **仓库**：`https://github.com/bestfunc/Tinia`
- **语言**：Go（后端） + React + TypeScript（前端） + Wails v2（桌面包装）
- **职责**：
  - 用户 / 组织 / 权限 / 商店订阅 / 凭据管理（业务逻辑层）
  - 流程编辑器、数据源管理、看板编辑器、AutoML 调参、开发者工作室（UI 层）
  - **执行内核**（`server/internal/nodes/`、`internal/graph/`）：接收流程 JSON、按 DAG 调度、Python subprocess + venv 隔离、blob 存储抽象、事件流（SSE / WS）、**常驻执行池 HotPool**
  - **节点 Python SDK**（`server/sdk/python/`）：唯一事实源，通过 `go:embed` 嵌进 server 二进制，fork 节点时注入 `PYTHONPATH` 指向嵌入版
  - **MCP server**（`/api/v1/mcp`，AI 接入层）—— 核心差异化之一
  - **SDK API**（`/api/v1/sdk`，外部程序调用 + 流式会话）—— 另一条对外通路
  - 桌面单机版的全部胶水（嵌入式 PostgreSQL / Python runtime / setup wizard / 自动更新）
- **三种部署 edition**（代码层）：`saas` / `server` / `desktop`
  - 解析优先级：env `TINIA_EDITION` > 构建期 ldflags `config.DefaultEdition` > 兜底 `server`
  - 桌面构建用 `-X .../config.DefaultEdition=desktop`
- **关系**：是用户感知到的"Tinia 应用"本体，也是唯一执行引擎；其他仓库是它的周边或接入方

> **商业 SKU vs 部署 edition**：代码里只有 3 个 edition（saas/server/desktop）。Community / Pro / Production 是**商业打包概念，不是 edition flag**。Community 与 Pro 的差异在代码里是「desktop 是否激活 + 激活校验中间件」，并非不同 edition。详见 `03-edition-comparison.md`。

### 2. **Tinia_nodes** —— 官方节点

- **仓库**：`https://github.com/bestfunc/Tinia_nodes`
- **命名空间**：`bestfunc`（如 `bestfunc/level_meter`、`bestfunc/fft_spectrum`）
- **结构**：每个子目录一个节点，含 `node.yaml` / `runtime/run.py` / `requirements.txt` / `ui/ParamsForm.tsx` / `ui/Viewer.tsx` / `ui/Help.tsx`（节点说明，vite glob 自动注册）
- **当前节点数**：约 41 个（覆盖信号源 / 预处理 / 声学·振动·心理声学分析 / 特征工程 / 可视化全链路）
- **不携带 SDK**：早期各节点仓自带 SDK 副本导致协议漂移，现已统一到主仓 `server/sdk/python/`；节点仓内 `sdk/python/README.md` 仅为迁移说明占位
- **版本独立**：跟主仓 Tinia 版本独立递增（`feature/v1.N` → `latest`）
- **近期主线**：
  - 声级计/频谱/倍频程**无缝逐帧**（流式会话跨窗状态延续，v1.22）—— 与主仓 SDK 流式会话端到端对应
  - 多尺度谱分析 `scale_space_spectrum` + GPU 共享 sidecar（v1.23）
  - 振动节点族（时域统计 / 包络解调 / 信号数学运算 + 人体振动计权）
  - 全节点补 SDK 说明 tab + `ui/Help.tsx` 节点说明组件
- **重点节点**（部分）：
  - 声学：`level_meter`、`fft_spectrum`、`octave_analysis`、`modulation_spectrum`、`scale_space_spectrum`
  - 振动/旋转机械：`order_tracking`、`envelope_demod`、`time_stats`
  - 心理声学：`loudness`、`roughness`、`sharpness`、`tonality`、`tnr`、`fbank_extract`
  - 特征工程：`feature_merge`、`feature_normalize`、`spec_limit_check`、`cluster_explore`、`zscore_anomaly`
  - 查看器：`spectrum_viewer`、`indicator_viewer`、`chart_viewer`、`matrix_view`

> **内置节点 vs Python 节点**：`dataset_node` / `dataset_merge` / `materialize_node` / `dashboard_node` / `csv_export` / `filter_node` 等是**主仓 Go 内置节点**（`server/internal/nodes/builtin/`），不在 Tinia_nodes；`channel_split` / `channel_select` 才在 Tinia_nodes。

### 3. **Tinia_Cli** —— 开发者命令行

- **仓库**：`https://github.com/bestfunc/Tinia_Cli`
- **语言**：Go（跨平台 binary）
- **职责**：scaffold 新节点项目 / 本地测试 / 打包提交到 Store / `tinia login`（v1.31 起在桌面窗口内完成授权）/ `tinia sdk install`（拷一份本地 SDK 副本做 IDE 类型提示，仅本地用、不进发布）
- **目标用户**：节点开发者（学术研究者 / 独立工程师 / 公司内部节点维护者）
- **跟主仓关系**：CLI 通过 HTTP API 跟主仓通信，本身不嵌入执行引擎

### 4. **Tinia_Plugins** —— Claude Code 插件市场

- **仓库**：`https://github.com/bestfunc/Tinia_Plugins`
- **是什么**：给 Claude Code / Codex / Qwen CLI 装的 plugin marketplace
- **提供 4 个变体 plugin**（按用户的 Tinia 部署形态选）：
  - `tinia` —— SaaS 版（接 SaaS 公网域名）
  - `tinia-onprem` —— 私有化版（接客户内网域名）
  - `tinia-desktop` —— 桌面单机版（接 `localhost:18720`，装后 AI 客户端可免 OAuth 连本机）
  - `tinia-local` —— 本地开发（主仓贡献者用，接 `localhost:18722`）
- **每个变体含**：`mcpServers` 配置（OAuth 自动授权 MCP connector）、20+ AI skill、共享的本地 MCP bundle
- **跟主仓关系**：plugin 装在 AI 客户端那一侧，连接到任意 Tinia 部署的 MCP 端点

### 5. **Tinia_Store** —— 节点商店

- **独立服务**：跟主仓物理分离，可独立部署
- **语言**：Go（后端） + React（前端）
- **职责**：节点发布审批（公开 / 私有组织池）、用户订阅管理、商业节点平台分成、实例激活（主仓 desktop 走 `store_url` 完成激活，已支持 `seat_id` 用户级 seat 模型）
- **跟主仓关系**：主仓 desktop 启动 setup wizard 走激活流程到 Store；SaaS / Server 通过 Store API 拉订阅的节点 bundle；Store 自己也是独立 web app

### 6. **Bestfunc_Passport** —— 账号 + 授权控制面（身份提供方 IdP）

- **状态**：真实独立仓（Gin + GORM + Postgres，dev 端口 18725/18726）
- **定位**：**bestfunc 全产品的账号 + 授权控制面**，Tinia 是接入方之一
- **职责**：对外提供 OAuth / JWKS（离线验签）/ userinfo / entitlement / seat
- **跟主仓关系**：Tinia 侧 `server/internal/auth/sso.go` 等接入；身份 / 授权 / seat 正逐步由 Passport 承担（与 Store 的「节点 catalog / 分成」职责分工）

### 7. **tinia-release** —— 发布产物

- **仓库**：`https://github.com/bestfunc/tinia-release`
- **不是代码仓**：纯产物，存最新版的 Windows / macOS 桌面安装包 + `latest.json`（自动更新源，当前约 `1.35.x`）
- 桌面客户端启动时 GET `latest.json` 检查是否有新版

---

## 三条对外通路（理解架构的关键）

Tinia 不只是"自己的 Web/桌面 UI"，对外还有两条并行的程序化通路：

| 通路 | 面向 | 入口 | 干什么 |
|---|---|---|---|
| **Web / 桌面 UI** | 人 | 浏览器 / Wails 窗口 | 交互式编排、看板、调参 |
| **MCP** | AI 客户端 | `/api/v1/mcp`（JSON-RPC 2.0 over HTTP） | "AI 自动驾驶平台"：搭流程 / 跑测试 / 改节点代码 |
| **Python SDK** | 外部程序 | `/api/v1/sdk`（License 鉴权 + 可选 UDS 同机直连） | "外部程序调用平台算力"：传数据拿结果，算法仍在平台执行 |

> MCP 与 SDK 是两条不同通路：MCP 让 AI 驱动平台开发与编排；SDK 让外部 Python 程序把平台当算力后端调用。详见 `11-mcp-ai-integration.md`、`04-key-concepts.md` 的 SDK 章节。

---

## 数据流（一个典型流程跑起来）

用户在 Web UI 点"运行流程"后会发生什么：

```
1. 前端 POST /api/v1/graph-runs/run
        ↓
2. 主仓 internal/graph 执行器拓扑排序 DAG（流式执行：边算边推，整链路并发）
        ↓
3. 对每个节点：
   a. 主仓拉节点定义（从 namespace 注册表）
   b. 启动 Python subprocess（或复用常驻执行池里的热进程），
      PYTHONPATH 注入主仓 go:embed 的 tinia_runtime
   c. 通过 stdin 发 task JSON（含 inputs 的 blob handle）
   d. 节点 run.py 调 fetch_blob(handle) 拉数据
   e. 算法跑完，调 upload_blob(bytes) → 拿到 output handle
   f. emit_output("port_name", handle) → stdout 写一行 JSON
        ↓
4. 主仓监听 subprocess stdout，转发为 SSE / WebSocket 事件给前端
        ↓
5. 前端实时更新节点状态（pending → running → completed），进度条带分母（X/N）
        ↓
6. 全部完成后，前端可点节点的"查看视图"，加载 Viewer.tsx 渲染 outputs
        ↓
   命中"输入指纹"缓存时直接复用结果（带命中标记 + 节省时间显示）
```

**关键设计**：

- **数据走 blob，不走 inline**：上下游通过 handle 引用，server 不解析内容
- **节点完全隔离**：每个节点独立 venv + 独立 subprocess，崩溃不影响其他节点
- **事件流式**：前端能看到节点进度实时变化，而非"提交完等结果"
- **常驻执行加速**：高频 / 实时调用复用热进程，省掉每次 fork + import 的固定开销（详见下文）

---

## 常驻执行池（HotPool）

> 源：`server/internal/nodes/hotpool.go`。

分析节点进程可**常驻待命**（`tinia_runtime.serve()`），只加载一次库，第二次起省掉 fork + import 的固定开销（从约 530ms 启动降到纯计算时间）。

- 按 node fullKey 分桶（per-node venv 隔离），SDK 高频 / 实时调用直接复用热进程
- 超管「常驻执行」页：查看运行状态、调进程上限 / 空闲回收、勾选需预热（预热白名单）的节点
- 适合 SDK 流式会话、实时直调等需要低延迟的场景

---

## SDK 通路与流式会话在架构里的位置

> 源：`server/internal/sdkapi/{middleware,handler,stream}.go`、`server/sdk/client-python/tinia_sdk/`、`docs/sdk-design.md`。

**核心前提**：SDK 是**纯通路、没有第二个引擎**——执行永远在 tinia-server 进程内，SDK 只负责"上传 raw bytes → 调用 → 下载结果"。

- **鉴权**：License（不是 api_key）。凭据打包进 SDK 包里的 `license.json`（`license_id` + `secret`），请求头 `Bearer <license_id>.<secret>`；server 端 `LicenseAuth` 查 `sdk_licenses` 表校验。超管「SDK 管理」输入名称即生成可下载 SDK 包（server 地址 + 凭据已内置，零配置）。
- **连接**：`connect(server_url=None, license_path=None, socket_path=None, use_uds=None)`；`server_url` 缺省取 `license.json` 内的地址（desktop 包打本机地址）。**同机调用可走 Unix domain socket（UDS）直连 + 路径直传**，大文件显著更快。
- **三种调用方式**：① 直接传节点类型 + 参数；② 用节点表单「复制参数」拿到的参数串（自带节点类型）；③ 引用平台流程里调好的节点（平台改参自动生效）。整条流程也能整体调用（放「API 输入」+「API 输出」节点）。
- **流式会话 / 实时数据流**（v1.35，`stream.go`）：JWT `session_token` 门控 push / recv / close / keepalive，可持续往流程推数据、实时取回计算结果，适合在线 / 边采边算场景；实时直调跳过缓存、走内存中转，配合常驻执行池响应更快。节点侧（Tinia_nodes v1.22）通过声明 `_stream_continuous` 维护跨窗状态实现无缝逐帧，二者端到端组成实时流计算。
- **SDK 调用分析**（v1.35）：超管可查看每个 SDK 的调用量、成功率、耗时、Top 节点、最近失败，用于排查与监控。

---

## 部署 vs 仓库

容易混淆：**部署形态**（saas/server/desktop）跟**仓库**不是一一对应的。同一份主仓代码可以编译/配置出三种 edition，区别在：

- 启动方式：`--desktop` 走桌面（含 `--setup` / `--window` / `--no-window`，并以 `daemon` 子命令常驻）；无参数 = runServer，edition 由 `TINIA_EDITION` 环境变量决定 server / saas（**没有 `--saas` flag**）
- 数据库：内嵌 PostgreSQL（desktop）/ 外部 PostgreSQL（server/saas）
- Edition flag：`cfg.Edition` 决定 UI 是否显示组织管理、商店等
- 激活：desktop / server / saas 各自走不同激活流程

详见 `10-deployment-modes.md`。

---

## 为什么是多仓（不是 1 个大仓库）

这是**架构层面的领先**，是 Tinia 长期演进力的核心：

| 拆分理由 | 价值 |
|---|---|
| 主平台 vs nodes | 节点能独立迭代，不绑主平台版本；为节点商店做铺垫 |
| 主平台 vs cli | CLI 是节点开发者的入口，跟最终用户分离 |
| 主平台 vs plugins | AI 客户端的 plugin 在 AI 那侧，不嵌主平台 |
| 主平台 vs store | 让商店能独立部署、独立运维、按平台级服务演进 |
| 主平台 vs passport | 身份 / 授权 / seat 作为 bestfunc 全产品控制面，跨产品复用 |

对标 ArtemiS / Siemens Testlab 这类基于桌面单进程的成熟商业软件 —— 要扩展到 AI agent 接口 / 节点商店 / 外部 SDK 调用 / 云端协同需要重写大量基础设施。Tinia 的多仓切分让这些扩展是顺理成章的演进。

---

## 历史：归档的双引擎设想

早期设计文档曾设想 7 件套含独立的 `tinia-engine`（执行引擎抽离成 binary / Go library）+ `tinia_runtime`（独立 pip 包）+ `Tinia_Graph_Spec`（流程 JSON 规范）。这三者已在 `docs/sdk-design.md §0.1` 明确**归档、不再投入**，理由：

- 双引擎（平台内 + 独立）必然产生协议漂移，维护成本高
- 收益已被「桌面单二进制 + headless 模式 + SDK 纯通路」覆盖

现状：执行只在 tinia-server 内；节点 SDK 唯一源在主仓 `server/sdk/python/`（go:embed）。本地不存在这三个目录。提及历史时使用"已归档 / 不再规划"措辞，不要写成"待抽离"。

---

## 下一步

- 详细看每个部署模式 → `10-deployment-modes.md`
- 看技术栈选型理由 → `09-tech-stack.md`
- 看 MCP-native 怎么实现 → `11-mcp-ai-integration.md`
- 看节点生态规划 → `12-node-ecosystem.md`
