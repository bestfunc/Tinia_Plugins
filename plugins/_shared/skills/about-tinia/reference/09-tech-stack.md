# 技术栈

> Tinia 各仓库用了什么语言 / 框架 / 库，以及为什么这么选。给技术销售 / 新员工 / 评估 Tinia 的工程师看。

---

## 速览

```
┌─────────────────────────────────────────────────────────┐
│ 主仓 Tinia                                              │
├─────────────────────────────────────────────────────────┤
│ 后端：Go 1.22+ / Gin / GORM / PostgreSQL                 │
│ 前端：React 18 / TypeScript / Vite / Tailwind            │
│       React Flow / uPlot / ECharts / zustand            │
│ 桌面：Wails v2（WebView2 + WKWebView）                   │
│ 嵌入：fergusstrange/embedded-postgres /                  │
│       python-build-standalone / 节点 Python SDK(go:embed)│
│ 执行：Python subprocess + 常驻执行池(HotPool) + GPU      │
│       共享 sidecar（internal/gpucompute，未配置则 numpy） │
│ blob：MinIO（SaaS/Server）/ 本地文件（Desktop）           │
│ MCP：自研 JSON-RPC 2.0 over HTTP（兼容 Anthropic 协议）  │
│ SDK：/api/v1/sdk（License 鉴权 + UDS 直连 + 流式会话）   │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Tinia_nodes 节点仓                                        │
├─────────────────────────────────────────────────────────┤
│ Python 3.11 / numpy / scipy / matplotlib                 │
│ 心理声学：mosqito                                          │
│ 前端：每节点 React TSX（运行时 esbuild）                  │
│ 不携带 SDK（统一用主仓 go:embed 版）                      │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Tinia_Store                                               │
├─────────────────────────────────────────────────────────┤
│ Go + Gin + PostgreSQL + MinIO（跟主仓共享底座架构）       │
│ 前端：React + Vite + Tailwind                             │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Bestfunc_Passport（账号 + 授权控制面 / IdP）             │
├─────────────────────────────────────────────────────────┤
│ Go + Gin + GORM + PostgreSQL                             │
│ OAuth / JWKS(离线验签) / userinfo / entitlement / seat   │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Tinia_Cli                                                  │
├─────────────────────────────────────────────────────────┤
│ Go / 跨平台 binary（mac/win/linux）                       │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Tinia_Plugins                                             │
├─────────────────────────────────────────────────────────┤
│ Markdown skill（无运行时）                                │
│ 本地 MCP bundle：TypeScript → esbuild 成单 .js            │
└─────────────────────────────────────────────────────────┘
```

> 节点 Python SDK（`tinia_runtime`）的唯一事实源在主仓 `server/sdk/python/`，通过 `go:embed` 嵌进 server 二进制并在 fork 节点时注入 `PYTHONPATH`。早期"runtime 独立成 pip 包 / engine 独立成仓"的设想已归档（见 `02-architecture.md`「历史：归档的双引擎设想」），不要再写成"待抽离 / 未来独立 pip 包"。

---

## 后端：Go

### 主仓选 Go 的理由

| 维度 | Go | Python | Rust | Java |
|---|---|---|---|---|
| 单 binary 部署 | ✅ | ❌（依赖管理痛苦）| ✅ | ❌（JVM）|
| 启动速度 | ms 级 | 100ms+ | ms 级 | s 级 |
| 内存占用 | 低 | 中 | 低 | 高 |
| 并发模型 | goroutine 简单 | asyncio 复杂 | async/await 学习陡 | 线程 |
| 工程师招聘 | 中（K8s/Docker 圈很多）| 易 | 难 | 易 |
| HTTP server | net/http + Gin 成熟 | FastAPI / Django | actix-web | Spring |

**关键决策**：

- Tinia 桌面单机要打成单 exe / app bundle，Python 不行（GIL + 依赖）
- 主仓后端兼任 API server + DAG 调度 + Python subprocess 管理 → Go 的 goroutine 天生适合
- 团队有 Go 经验

### 关键 Go 库

| 库 | 用途 |
|---|---|
| `gin-gonic/gin` | HTTP framework |
| `gorm.io/gorm` | ORM |
| `golang-jwt/jwt` | JWT |
| `wailsapp/wails/v2` | 桌面 app 框架 |
| `getlantern/systray` | 系统托盘 |
| `fergusstrange/embedded-postgres` | 桌面单机版的内嵌 PostgreSQL |
| `gorilla/websocket` | WebSocket（事件流）|
| `go:embed` | 嵌入前端 dist / migrations / 节点 Python SDK |
| `internal/gpucompute` | GPU sidecar / 共享 torch sidecar（节点经 IPC 复用，未 provision 时 numpy 回退） |
| `internal/nodes/hotpool` | 常驻执行池（节点进程常驻，复用热进程加速）|

---

## 前端：React + TypeScript + Vite

### 选 React 的理由

- 生态最大：React Flow / uPlot / ECharts 等关键库都有
- 团队熟悉：转移成本低
- TypeScript 严格类型：跟 Go 后端的强类型对齐

### 关键前端库

| 库 | 用途 |
|---|---|
| `@xyflow/react` (React Flow) | 流程编辑器画布（DAG 可视化）|
| `uplot` | 高性能时序 / 频谱画图（百万点流畅）|
| `echarts` | 仪表盘、复杂可视化 |
| `react-grid-layout` | Dashboard 拖拽布局 |
| `monaco-editor` | DevStudio 代码编辑器（VSCode 同款）|
| `zustand` | 状态管理（比 Redux 轻）|
| `tailwindcss` | 原子化 CSS |
| `lucide-react` | 图标库 |
| `react-router-dom` v6 | 路由 |
| `esbuild-wasm` | 浏览器内 esbuild（DevStudio 的 TSX 热加载用）|

### 为什么不用 SvelteKit / Vue / Solid

- React Flow 是流程编辑器事实标准，其他框架没有同级别替代
- 团队 React 经验
- 复杂业务 UI（看板编辑器 / 数据源页 / 设置页）React 库齐全

---

## 桌面：Wails v2（不是 Electron）

### Wails 是什么

Go 语言的桌面 app 框架。把 Go binary + 前端打包成单 exe / app bundle，UI 用系统原生 WebView（Windows WebView2 / macOS WKWebView）。

### 跟 Electron 对比

| 维度 | Electron | Wails v2 |
|---|---|---|
| WebView | 内置 Chromium（~200MB） | 系统 WebView（~5MB binary）|
| 主进程语言 | Node.js | Go |
| 安装包大小 | 通常 100MB+ | 通常 10-50MB |
| 内存占用 | 大（独立 Chromium）| 小（共享系统 WebView）|
| API 调用前端 | IPC | 双向 binding 简洁 |
| 跨平台 | mac/win/linux | mac/win/linux |

### Tinia 用 Wails 的关键决策

- **不喜欢 Electron 200MB+ 安装包**：声学工程师下载耐心有限
- **后端是 Go**：用 Wails 复用同语言，没必要再加 Node.js 层
- **WebView2 / WKWebView 已足够**：Tinia 不用 Chromium 独占特性

### 桌面版的内嵌依赖

| 内嵌物 | 库 | 大小 |
|---|---|---|
| 前端 dist | go:embed | ~5MB（minified）|
| Migrations SQL | go:embed | ~100KB |
| PostgreSQL | fergusstrange/embedded-postgres | 首次下载 ~50MB，本地解压 |
| Python runtime | python-build-standalone（PBS） | 首次下载 ~50MB |
| Tinia_nodes bundle | 打包内嵌 | ~5MB |

**总安装包**：约 30-50MB（不含首次下载的 PG/Python）。

详见 `10-deployment-modes.md`。

---

## 节点 Runtime：Python 3.11

### 选 Python 的理由

- 声学 / 科学计算生态绝对中心：numpy / scipy / matplotlib / librosa / mosqito 等都是 Python
- 节点作者大多是工程师 / 研究者，Python 比 Go / Rust 普及度高几个量级
- 节点开发门槛低 → 节点生态能起来

### 版本锁定 3.11

- numpy / scipy 等关键库的稳定 ABI
- python-build-standalone 提供官方编译版（不需要客户机有 Python）
- 跟节点 wheel 兼容性矩阵简单

### Python runtime 管理

- 每个节点自动创建独立 venv（`runtime/.venv/`）
- venv 共享 base interpreter，仅装节点 requirements.txt 里的依赖
- 自动从清华 PyPI 镜像加速（中国用户）

### 节点跟 server 通信

- 通过 stdio JSON RPC：server 启 subprocess → stdin 喂任务 JSON → stdout 收事件 JSON
- 数据走 HTTP API（`/api/v1/internal/blobs`）上传 / 下载 blob
- 节点契约（端口类型 / quantity 通道语义 / 错误分级 emit_warning 等）由主仓 go:embed 的 SDK 提供

### 常驻执行（HotPool）与流式

- 节点进程可常驻待命（`tinia_runtime.serve()`），第二次起省 fork + import（约 530ms → 纯计算）
- SDK 的 ChunkRuntime 给节点提供分块流式 endpoints；节点声明 `_stream_continuous` 即可维护跨窗状态，配合主仓 SDK 流式会话组成端到端实时数据流计算

---

## 存储

### 元数据：PostgreSQL

| 用途 | 表 |
|---|---|
| 用户 / 组织 / 权限 | users / orgs / org_members / org_permissions |
| 流程定义 / 运行 | graphs / graph_runs / node_runs / events |
| 数据源 / 凭据 | datasources / credentials |
| 节点 / 商店订阅 | plugins / subscriptions / approvals |
| 多租户底座 | namespaces / activations / seats |
| SDK 通路 | sdk_licenses（SDK 凭据 + 调用记录）|
| MCP 审计 | mcp_audit_logs |

**Migration 工具**：自研，按版本号顺序执行 `server/migrations/*.sql`。

**桌面单机版**：内嵌 PostgreSQL（fergusstrange/embedded-postgres），首次启动自动下载二进制 + 初始化。

### 大数据：MinIO / 本地文件

| 部署 | blob 存储 |
|---|---|
| SaaS / Server | MinIO（S3 兼容对象存储） |
| Desktop | 本地文件（`<data_dir>/blobs/`）|

抽象层：`internal/blob/Store` interface，背后可换 MinIO / S3 / 本地，节点代码透明。

---

## MCP（AI 接入）

### 协议

- 当前实现：JSON-RPC 2.0 over HTTP（POST `/api/v1/mcp`）
- 协议版本：claim `2024-11-05`（Anthropic 早期版本号）
- 实际形态：Streamable HTTP 最简退化形态（每个 POST 直接返 JSON 响应，无 SSE upgrade）
- 兼容：Claude Code / Codex / 大部分 MCP client

### 鉴权

- OAuth 2.1 + PKCE + 动态客户端注册（DCR）+ loopback
- 首次授权一键 OAuth，token 缓存到 client 本地；scope 由 user_group `permissions.mcp.<module>` 决定，超管全开
- 不需要手动管 API key；审计落 `mcp_audit_logs`

### Tools 数量

- 当前 70+ 个 tool（含约 25 个 `dev_*`），覆盖 **8 个模块**：`dev / nodes / graphs / data / data_write / templates / assistant / plugins`
- 每个模块对应一个 scope `mcp:<module>`（共 8 个 scope），模块分组通过 OAuth scope 管理
- 工具覆盖项目操作 / 文件读写 / 节点脚手架 / 编译 / 重载 / 导出 / 分析流程 / 数据源 / 通道模板 / SDK 管理等

详见 `11-mcp-ai-integration.md`、`Tinia/docs/mcp-spec.md`。

---

## SDK（外部程序调用）

### 协议与鉴权

- 入口：`/api/v1/sdk`，Python 客户端 `tinia_sdk`（`server/sdk/client-python/`）
- 鉴权：**License（不是 api_key）**。凭据打包进 SDK 包里的 `license.json`（`license_id` + `secret`），请求头 `Bearer <license_id>.<secret>`；server 端 `internal/sdkapi/middleware.go LicenseAuth` 查 `sdk_licenses` 表校验
- `connect(server_url=None, license_path=None, socket_path=None, use_uds=None)`；缺省 server_url 取 `license.json` 内地址；**同机可走 Unix domain socket（UDS）直连 + 路径直传**，大文件显著更快
- **没有第二个引擎**：SDK 只上传 raw bytes + 调用 + 下载，执行永远在 tinia-server 进程内

### 流式会话

- `internal/sdkapi/stream.go`：JWT `session_token` 门控 push / recv / close / keepalive，可持续推数据、实时取回结果；实时直调跳过缓存、走内存中转，配合常驻执行池更快

### SDK 调用分析

- 超管可查看每个 SDK 的调用量、成功率、耗时、Top 节点、最近失败

详见 `Tinia/docs/sdk-design.md`。

---

## 商店：Tinia_Store

### 跟主仓共享技术栈

- 同样 Go + Gin + PostgreSQL + MinIO
- 同样 React + Vite + Tailwind
- 部分核心库（auth / oauth provider / blob store）从主仓抽出共享

### 独立服务的理由

- 独立部署 / 独立运维 / 平台级演进
- 商店是平台级服务，跟具体 Tinia 实例解耦
- 多个 Tinia 实例（不同公司 / 不同 edition）共享一个商店

> **Store 与 Passport 分工**：Store 负责节点 catalog / 发布审批 / 订阅 / 商业分成 / 实例激活（`store_url`）；身份 / OAuth / JWKS / entitlement / seat 正逐步由独立的 **Bestfunc_Passport** 承担（bestfunc 全产品的账号 + 授权控制面，Tinia 为接入方之一，dev 端口 18725/18726）。

---

## Passport：Bestfunc_Passport（身份 / 授权控制面）

- Go + Gin + GORM + PostgreSQL，独立库 `bestfunc_passport`
- 对外提供 OAuth / JWKS（离线验签）/ userinfo / entitlement / seat
- Tinia 侧 `server/internal/auth/sso.go` 等接入；定位为 bestfunc 全产品复用的身份提供方（IdP），不是 Tinia 专属

---

## CI / 构建

| 类型 | 工具 |
|---|---|
| 主仓后端 | `go build` + `go test` |
| 主仓前端 | `npm run build`（Vite）|
| 桌面打包 | `wails build`（mac/win） + NSIS（Windows installer） |
| Windows 签名 | EV Code Signing |
| macOS 签名 | codesign + notarize |
| 自动更新 | `tinia-release/latest.json` 拉取检测 |
| 测试机 | 内部远程 Windows agent（argus 平台）|

---

## 第三方依赖审计（合规 / 法务）

对外发布时需要给客户提供 NOTICES.md：

- **Apache-2.0 / MIT 库**：占绝大多数
- **GPL / AGPL**：避免（会污染主仓闭源部分）
- **数据库 / 存储**：PostgreSQL（PostgreSQL License）、MinIO（AGPL —— 注意只用 server，不嵌入 client）

第三方依赖审计是商业化前置任务，详见 `docs/commercialization-todo.md` §E4。

---

## 性能基线

### Tinia daemon（桌面单机）

- 启动：5-15 秒（含 PostgreSQL 启动 + Python venv 检查）
- 内存：200-500MB（取决于运行的节点数）
- 流程跑：几个节点 + 几 MB 数据 → 秒级；多通道 + 几百 MB → 分钟级

### 前端

- 首屏加载：1-2 秒（dist 约 1.5MB gzip）
- 流程编辑器：100+ 节点流畅
- Viewer：百万点频谱用 uPlot 流畅

---

## 选型反向决策记录（可被问到）

| 问题 | 决策 | 理由 |
|---|---|---|
| 为什么不用 Electron | Wails | 安装包小 + Go 主进程一致 |
| 为什么不用 FastAPI | Gin | 团队 Go 经验 + 单 binary 部署 |
| 为什么不用 Vue 3 | React | React Flow 是流程编辑器事实标准 |
| 为什么不嵌入 Python（不靠子进程）| subprocess | 隔离失败、版本管理简单、GIL 不影响 daemon |
| 为什么 PostgreSQL 不是 SQLite | PG | 多租户 + JSONB + 桌面版用 embedded-postgres 兼容性同 |
| 为什么自研 MCP 不是用 SDK | 自研 | Go 没成熟 MCP SDK；自己实现 ~600 行代码 |
| 为什么 plugin marketplace 不是 npm | 独立 store | npm 是代码生态，不是行业 plugin 商店 |

---

## 下一步

- 各仓库的代码组织结构 → `02-architecture.md`
- 桌面单机版怎么打包 / 跨平台 → `10-deployment-modes.md`
- MCP 协议细节 → `11-mcp-ai-integration.md`
