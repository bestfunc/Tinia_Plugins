# 7 件套架构

> Tinia 不是单一仓库，是 7 个互相解耦的子项目协作运行。这种切分是有意为之 —— 既是工程合理性，也是商业护城河（架构层面的领先，5 年内对手追不上）。

---

## 总览图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                 用户层                                    │
│         开发同事/销售/客户 — 用 Web / 桌面 app / Claude Code              │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↕
┌──────────────────────────────────────────────────────────────────────────┐
│                              主平台（Tinia）                              │
│   Go + React + Wails，三种 edition（saas/server/desktop）共享同一份代码    │
│   ▸ 流程编辑器 / 数据源 / 看板 / 设置 / 商店订阅 UI                       │
│   ▸ 多租户：Org / Member / Seat / Activation                              │
│   ▸ MCP server（/api/v1/mcp）— AI 客户端接入点                            │
│   ▸ 内置 PostgreSQL + Python runtime（桌面单机版用）                      │
└──────────────────────────────────────────────────────────────────────────┘
              ↓ 调用                              ↑ 注册节点
┌──────────────────────────────┐    ┌──────────────────────────────────────┐
│  tinia-engine（执行引擎）     │    │  Tinia_nodes（官方节点）              │
│  Go binary / Go library      │←───│  bestfunc/* 命名空间，跟主平台同步发布  │
│  • DAG 调度                   │    │  • level_meter / fft_spectrum         │
│  • Python subprocess 管理     │    │  • order_tracking / cepstrum 等       │
│  • Blob 存储抽象              │    │  • 多通道架构 + 通道命名模板          │
│  • 事件流输出（SSE / WS）      │    └──────────────────────────────────────┘
└──────────────────────────────┘                  ↑ pip 依赖
              ↑ stdio JSON RPC                    │
┌──────────────────────────────┐    ┌──────────────────────────────────────┐
│  tinia_runtime（Python SDK）  │    │  Tinia_Store（商店服务）              │
│  pip package                  │    │  独立部署，Go + React                  │
│  • get_input / set_output     │    │  • 节点发布审批                        │
│  • upload_blob / fetch_blob   │    │  • 用户订阅                            │
│  • progress / log             │    │  • Tinia 实例激活授权（OAuth + Seat）  │
└──────────────────────────────┘    │  • 商业节点 30% 平台分成               │
              ↑ import                └──────────────────────────────────────┘
┌──────────────────────────────┐                ↑ 发布提交
│  Tinia_Cli（开发者 CLI）      │                │
│  Go binary                    │────────────────┘
│  • scaffold 节点项目          │
│  • 本地测试                   │    ┌──────────────────────────────────────┐
│  • 打包提交到 Store           │    │  Tinia_Plugins（Claude Code plugin）  │
└──────────────────────────────┘    │  marketplace 仓库                      │
                                     │  • 4 个变体 plugin（按部署形态）        │
                                     │  • 共享 skills + 本地 MCP bundle      │
                                     └──────────────────────────────────────┘
                                                  ↑ Claude Code 装
                                     ┌──────────────────────────────────────┐
                                     │  tinia-release（构建产物）            │
                                     │  • Windows / macOS 桌面安装包          │
                                     │  • latest.json 自动更新源              │
                                     └──────────────────────────────────────┘
```

---

## 7 个仓库各自的职责

### 1. **Tinia** —— 主仓库 / 业务平台

- **仓库**：`https://github.com/bestfunc/Tinia`
- **语言**：Go（后端） + React + TypeScript（前端） + Wails v2（桌面包装）
- **职责**：
  - 用户 / 组织 / 权限 / 商店订阅 / 凭据管理（业务逻辑层）
  - 流程编辑器、数据源管理、看板编辑器（UI 层）
  - MCP server（AI 接入层）—— 这是核心差异化
  - 桌面单机版的全部胶水（嵌入式 PostgreSQL / Python runtime / setup wizard / 自动更新）
- **三种部署 edition**：
  - `saas`：多租户公网，`tinia-saas.bestfunc.com`
  - `server`：单组织私有化，`t.bestfunc.com`
  - `desktop`：单用户桌面单机版
- **关系**：是用户感知到的"Tinia 应用"本体；其他 6 个仓库都是它的 dependency 或周边

### 2. **tinia-engine** —— 纯执行引擎（规划中，待抽离）

- **状态**：当前嵌在主仓 `server/internal/nodes/` 里；2027 路线图规划独立成 binary / Go library
- **职责**：
  - 接收流程 JSON，按 DAG 调度节点
  - Python subprocess 管理 + venv 隔离
  - Blob 存储抽象（local file / MinIO / S3 三选一）
  - 事件流输出（节点开始 / 进度 / 完成 / 失败）
- **不做的事**：数据库、用户系统、流程持久化、UI、商店、AI MCP（全部留给主平台）
- **为什么独立**：让三方业务平台能嵌入"可视化数据处理"而不用整个 Tinia
- **详见**：`docs/framework-design.md`

### 3. **tinia_runtime** —— Python SDK

- **当前**：在 `Tinia_nodes/sdk/python/tinia_runtime/`，节点开发者通过相对路径 import
- **未来**：独立 pip package 发布
- **职责**：节点 `run.py` 用的 API
  - `get_input(port)` / `set_output(port, handle)`
  - `upload_blob(bytes)` / `fetch_blob(handle)`
  - `emit_progress(value, message)` / `emit_log(level, msg)`
- **跟 engine 通信**：stdio JSON RPC（每行一个 JSON 事件）

### 4. **Tinia_nodes** —— 官方节点

- **仓库**：`https://github.com/bestfunc/Tinia_nodes`
- **命名空间**：`bestfunc/*`（如 `bestfunc/level_meter`、`bestfunc/fft_spectrum`）
- **结构**：每个子目录一个节点，含 `node.yaml` / `runtime/run.py` / `requirements.txt` / `ui/ParamsForm.tsx` / `ui/Viewer.tsx`
- **当前节点数**：33（覆盖核心声学 / 振动算法，HEAD 134 项笛卡尔积清单的子集 + 部分心理声学）
- **版本独立**：跟主仓 Tinia 版本独立递增
- **重点节点**（部分）：
  - 声级类：`level_meter`、`indicator_viewer`
  - 频谱类：`fft_spectrum`、`spectrum_smooth`、`spectrum_viewer`、`octave_analysis`
  - 时域：`weighting_filter`、`fir_filter`、`signal_generator`
  - 音频处理：`audio_segment_split`、`active_segment`、`convergent_trim`
  - 心理声学：`fbank_extract`、`roughness`（在 mosqito 基础上封装）
  - 看板/Composite：`dataset_node`、`channel_split`、`channel_select`

### 5. **Tinia_Cli** —— 开发者命令行

- **仓库**：`https://github.com/bestfunc/Tinia_Cli`
- **语言**：Go
- **职责**：
  - `tinia init` —— scaffold 新节点项目
  - `tinia dev` —— 本地热加载跑节点
  - `tinia pack` —— 打包成 bundle
  - `tinia publish` —— 提交到 Store
- **目标用户**：节点开发者（学术研究者 / 独立工程师 / 公司内部节点维护者）
- **跟主仓关系**：CLI 通过 HTTP API 跟主仓通信，本身不嵌入 engine

### 6. **Tinia_Plugins** —— Claude Code 插件市场

- **仓库**：`https://github.com/bestfunc/Tinia_Plugins`
- **是什么**：给 Claude Code / Codex / Qwen CLI 装的 plugin marketplace
- **提供 4 个变体 plugin**（按用户的 Tinia 部署形态选）：
  - `tinia` —— SaaS 版（接 `tinia-saas.bestfunc.com`）
  - `tinia-onprem` —— 私有化版（接 `t.bestfunc.com`）
  - `tinia-desktop` —— 桌面单机版（接 `localhost:18720`）
  - `tinia-local` —— 本地开发（Tinia 主仓贡献者用，接 `localhost:18722`）
- **每个变体含**：
  - `mcpServers` 配置（OAuth 自动授权 MCP connector）
  - 20+ AI skill（quickstart / create-node / debug-node / publish-plugin / 等）
  - 共享的本地 MCP bundle（如 `tinia-file-mcp` 多文件上传）
- **跟主仓关系**：plugin 装在 AI 客户端那一侧，连接到任意 Tinia 部署的 MCP 端点

### 7. **Tinia_Store** —— 节点商店

- **独立服务**：跟主仓物理分离，可独立部署
- **语言**：Go（后端） + React（前端）
- **职责**：
  - 节点发布审批（公开 / 私有组织池）
  - 用户订阅管理
  - 商业节点 30% 平台分成
  - Tinia 实例激活授权（颁发 OAuth token + 管理 Seat）
- **跟主仓关系**：
  - 主仓 desktop 启动时，setup wizard 走 OAuth 流程到 Store 完成激活
  - 主仓 SaaS / Server 通过 Store API 拉订阅的节点 bundle
  - Store 自己也是一个独立的 web app，用户用浏览器登录管理订阅

### 附：**tinia-release** —— 发布产物

- **仓库**：`https://github.com/bestfunc/tinia-release`
- **不是代码仓**：纯产物，存最新版的 Windows / macOS 桌面安装包 + `latest.json`（自动更新源）
- 桌面客户端启动时 GET `latest.json` 检查是否有新版

---

## 数据流（一个典型流程跑起来）

用户在 Web UI 点"运行流程"后会发生什么：

```
1. 前端 POST /api/v1/graph-runs/run
        ↓
2. 主仓 server/internal/graph/executor.go 拓扑排序 DAG
        ↓
3. 对每个节点：
   a. 主仓拉节点定义（从 namespace 注册表）
   b. 启动 Python subprocess，PYTHONPATH 注入 tinia_runtime
   c. 通过 stdin 发 task JSON（含 inputs 的 blob handle）
   d. 节点 run.py 调 fetch_blob(handle) 拉数据
   e. 算法跑完，调 upload_blob(bytes) → 拿到 output handle
   f. emit_output("port_name", handle) → stdout 写一行 JSON
        ↓
4. 主仓监听 subprocess stdout，转发为 SSE / WebSocket 事件给前端
        ↓
5. 前端实时更新节点状态（pending → running → completed）
        ↓
6. 全部完成后，前端可点节点的"查看视图"，加载 Viewer.tsx 渲染 outputs
```

**关键设计**：

- **数据走 blob，不走 inline**：上下游通过 handle 引用，daemon 不解析内容
- **节点完全隔离**：每个节点独立 venv + 独立 subprocess，崩溃不影响其他节点
- **事件流式**：前端能看到节点进度实时变化，而非"提交完等结果"

---

## 部署 vs 仓库

容易混淆：**部署形态**（saas/server/desktop）跟**仓库**不是一一对应的。同一份主仓代码可以编译出三种 edition，区别在：

- 启动参数：`--desktop` / 无参数 / `--saas`
- 数据库：内嵌 PostgreSQL（desktop）/ 外部 PostgreSQL（server/saas）
- Edition flag：cfg.Edition 决定 UI 是否显示组织管理、商店等
- License：desktop / server / saas 各自要走不同激活流程

详见 `10-deployment-modes.md`。

---

## 为什么是 7 件套（不是 1 个大仓库）

这是**架构层面的领先**，是 Tinia 长期演进力的核心：

| 拆分理由 | 价值 |
|---|---|
| 主平台 vs engine | 让 engine 能给三方业务平台用，扩大适用场景 |
| engine vs runtime | runtime 是 Python 生态，节点作者不用懂 Go |
| 主平台 vs nodes | 节点能独立迭代，不绑主平台版本；为节点商店做铺垫 |
| 主平台 vs cli | CLI 是节点开发者的入口，跟最终用户分离 |
| 主平台 vs plugins | AI 客户端的 plugin 在 AI 那侧，不嵌主平台 |
| 主平台 vs store | 让商店能独立部署、独立运维、按平台级服务演进 |

HEAD ArtemiS / Siemens Testlab 是基于桌面单机架构的成熟商业软件，功能集成在单进程内 —— 要扩展到 AI agent 接口 / 节点商店 / 云端协同需要重写大量基础设施。Tinia 的 7 件套切分让这些扩展是顺理成章的演进。

---

## 下一步

- 详细看每个部署模式 → `10-deployment-modes.md`
- 看技术栈选型理由 → `09-tech-stack.md`
- 看 MCP-native 怎么实现 → `11-mcp-ai-integration.md`
- 看节点生态规划 → `12-node-ecosystem.md`
