# Tinia 术语表

> 所有其他文档第一次出现某术语时会链回这里。看不懂某词，直接在本页 Ctrl+F。

按字母顺序，中英混排。

---

## A

### Activation（激活）

Tinia 实例与商店组织绑定的过程。Pro / Production 部署后需要激活才能解锁全功能。激活基于"组织（Org）→ 席位（Seat）→ 实例（Instance）"三层模型：

- 组织买若干 Seat（席位）
- 组织管理员把 Seat 分给具体用户
- 用户在某台机器（Instance）上用自己被分到的 Seat 完成激活
- 激活后该机器解锁 Pro 功能直到 Seat 过期或被回收

Community 版无需激活，开箱即用。

### Activity Stream（活动流）

前端 UI 实时显示"AI 正在做什么"的事件流。AI 通过 MCP 调用工具时（如创建项目 / 写文件 / 跑流程），daemon 把事件推到 `GET /api/v1/mcp/activity/stream`（SSE），前端在右下角浮窗里展示。不是 MCP 协议的一部分，是 Tinia 自家可视化。

### Audio Player（音频播放器）

Tinia 内置的音频播放节点 + 前端 widget。能播放上游节点输出的 `AudioData`，时长从 `materialize_node` 输出的元信息读取。

---

## B

### Blob

二进制大对象（Binary Large OBject）。Tinia 流程中节点之间传递大数据用 blob 句柄而非 inline JSON：

- 上游节点把数据上传到 daemon 拿到 `Handle{uri, hash, kind, mime, byte_size}`
- 下游节点拿到 handle 后调 `fetch_blob(handle)` 取实际字节
- 大数据物理存在 local file（桌面单机版的 `<data_dir>/blobs/`）或 MinIO（SaaS / Server 版）
- daemon 不解析内容，只做存储 + 引用计数

### Bundle（节点包）

一个 Plugin 编译后的安装包。开发者在 DevStudio 写完节点 → 提交到 Store → 商店审批 → 用户订阅 → 主仓拉 bundle → 解压到 `<data_dir>/plugins/<bundle>/`。Bundle 内包含：

- 节点元数据（`plugin.json` / `node.yaml`）
- Python 算法（`runtime/run.py` + `requirements.txt`）
- 前端组件（`ui/ParamsForm.tsx` / `ui/Viewer.tsx`）
- 共享 SDK 依赖

---

## C

### CLI（Tinia_Cli）

`tinia` 命令行工具，给节点开发者用。能 scaffold 新节点项目、本地跑测试、打包提交到 Store。

仓库：`https://github.com/bestfunc/Tinia_Cli`

### Community（社区版）

Tinia 5 个 SKU 中的免费层（桌面单机）。面向学生 / 研究者 / 个人工程师，提供基础节点 + 单通道分析 + AI 辅助 + 流程复杂度限制。详见 `reference/03-edition-comparison.md`。

### Composite DataSource（组合数据源）

把多个独立数据源（如多支麦克风 + 转速通道）按"配对模板"组合成一个虚拟数据源给下游使用的能力。支持三种模式：

- 模式 A：快捷合并（按名称规则自动配对）
- 模式 B：CSV 导入（按表格指定配对关系）
- 模式 C：手工配对（UI 拖拽指定）

---

## D

### DAG（Directed Acyclic Graph，有向无环图）

Tinia 流程的本质：节点为顶点、数据流向为边的有向无环图。Executor 按拓扑序调度节点执行，节点完成后把 outputs 通过 blob handle 传给下游。

不是脚本（不是"先做 A 再做 B"的命令式编程），是声明式的"A 的 output 接到 B 的 input"，调度由 engine 决定。

### Daemon

Tinia 后端守护进程，由主仓 `cmd/server/main.go` 编译出的二进制提供。三种运行模式：

- `--desktop`：桌面单机模式，内嵌前端 + PostgreSQL + Python runtime + setup wizard
- 无参数：API 模式，跑 SaaS / 私有化部署
- `--setup`：强制重跑 setup wizard

### Dashboard（看板）

把多个 Viewer 的输出组合在一张布局里的能力。用户可以从节点的 dashboard_view 输出连到看板节点的输入，看板编辑器（`DashboardEditor`）支持拖拽布局 + 文本块 / 图片 / 切片器（slicer）/ 多 viewer 共存。

### Datasource（数据源）

声学数据的入口。Tinia 支持的数据源类型包括：

- 本地文件（wav / mat / unv 等）
- Diffgram（外部数据平台）
- 信号发生器（synthetic generator，用于测试）
- 组合数据源（Composite，见上）

### Desktop（桌面版）

Tinia 的桌面单机部署形态。用 Wails v2 包装 Go binary + React 前端 + 嵌入式 PostgreSQL + python-build-standalone。Windows / macOS 双击安装即用，所有数据在本机。

### DevStudio（开发者工作站）

Tinia 主仓内置的节点开发 IDE。让节点开发者直接在浏览器里写代码、跑测试、打包、提交到 Store。基于 monaco editor + 实时 reload。

---

## E

### Edition（版本类型）

Tinia daemon 的部署形态标志：

- `saas`：多组织 SaaS，跑在公网（`tinia-saas.bestfunc.com`）
- `server`：单组织私有化部署，公司内网（`t.bestfunc.com`）
- `desktop`：单用户桌面单机版

通过 `/api/v1/meta` 端点对前端暴露，前端按 edition 分叉 UI。

### Engine（tinia-engine）

Tinia 的纯执行引擎，从主仓抽出来的独立 binary / Go library。只负责"加载节点 + 跑流程"，不带数据库、用户、权限、商店等业务能力。让三方业务平台可以嵌入"可视化数据处理"能力而不用整个 Tinia。详见 `reference/02-architecture.md`。

---

## F

### Framework（Tinia Framework）

= Tinia Engine + Runtime + 节点规范（tinia-graph-spec）三件套的统称。给业务平台提供"加载节点 + 运行流程"能力的解耦层。详见 `docs/framework-design.md`。

---

## G

### Graph（流程）

= DAG，Tinia 中的具体一个流程实例。Graph 由若干 Node + Edge 组成，保存为 JSON。用户在 GraphEditor 里画图，运行后产生 GraphRun。

### GraphEditor（流程编辑器）

主仓前端的核心页面（`/graphs/:id/edit`）。基于 React Flow 实现拖拽式节点编排。

### GraphRun（流程运行实例）

一次 Graph 执行的运行记录。包含每个节点的执行状态（pending / running / completed / failed / cached / skipped）、进度、错误、输出 handle 等。前端用 `/graphs/:id/runs/:runId` 查看历史回放。

---

## H

### Handle（数据句柄）

= Blob Handle。形如 `{uri, hash, kind, mime, byte_size}`，节点之间通过它传递大数据。

### HEAD ArtemiS

德国 HEAD acoustics 公司的旗舰 NVH 分析软件。30+ 年历史，C++ 桌面单机版，欧美和中国汽车 / 风电 / 测试行业广泛使用。Tinia Pro 直接对标产品。

---

## I

### Indicator（指标）

Tinia 类型系统中的一种数据类型。表示一组带语义的数值（如声级 dBA、心理声学 loudness / sharpness / roughness 等）。分析节点常输出 `IndicatorData`，看板节点能跨节点对比。

### Instance（实例）

一个安装好 Tinia 的具体部署（一台机器 / 一个公司私有化实例）。Activation 把 Instance 跟 Org 的 Seat 绑定。

---

## M

### MCP（Model Context Protocol）

Anthropic 主导的 AI 工具协议标准。Tinia 的差异化亮点之一是 **MCP-native** —— 主仓直接内置 MCP server（`/api/v1/mcp`），让 Claude Code / Codex / Qwen CLI 等 AI 客户端通过 OAuth 2.1 一键授权后直接驱动 Tinia 完成插件项目从创建到发布的全流程。

详见 `reference/11-mcp-ai-integration.md`。

### MaterializedDataset

Tinia 类型系统中"已固化到 blob 的数据集"。一般是 `materialize_node` 节点的输出，把上游 lazy 拉取的 dataset 固化下来供后续节点反复使用。

---

## N

### Namespace（命名空间）

Tinia 的多租户隔离与节点归属机制。三种 namespace：

- `official`：官方节点（如 `bestfunc`），所有人可见可用
- `org`：组织私有节点，仅该 Org 内可见
- `user`：用户个人节点，仅该用户可见（DevStudio 开发期间用）

节点完整 key 形如 `bestfunc/level_meter`，前缀就是 namespace。

### Node（节点）

DAG 的最小执行单元。一个 Node 包含：

- 元数据：input/output schema、parameters、display name
- Runtime：Python `run.py`（subprocess 跑算法）
- UI：React `ParamsForm.tsx`（参数配置面板）+ `Viewer.tsx`（结果可视化）

参见 `reference/04-key-concepts.md`。

### node.yaml

节点的元数据声明文件，描述 input/output 类型、参数 schema、display name、runtime 类型等。Engine 通过它装载节点。

### NVH（Noise, Vibration, Harshness）

汽车工业术语：噪声、振动、声振粗糙度。Tinia 的核心目标用户群是 NVH 工程师。

---

## O

### Org（Organization，组织）

Tinia 的多租户基本单位。一个 Org 包含若干 Member、若干 Seat、若干 Activation。SaaS 版本支持多 Org，Server / Desktop 版本是单 Org。

### Order Tracking（阶次跟踪）

旋转机械振动分析的经典方法。把振动信号按转速重采样到角度域，得到与转速无关的频谱（阶次谱）。Tinia Pro 第二波交付的核心能力（2026 Q3）。

---

## P

### PdM（Predictive Maintenance，预测性维护）

工业设备健康监测、提前预警故障。Tinia Pro 第三波 + Production 的目标场景之一（风电、工业泵、电机厂）。

### Pro（Tinia Pro）

Tinia 5 个 SKU 中的**个人付费版**（桌面单机）。面向个人 NVH 工程师 / 独立顾问 / 自由职业，提供完整离线分析工作站、多通道、报告导出、Order Tracking。2026 H2 主交付。

跟 Server 的差异：Pro 是**个人**，Server 是**团队**（详见 `reference/03-edition-comparison.md`）。

### Production（Tinia Production）

Tinia 5 个 SKU 中的**在线产线版**（边缘 + 云端）。面向工厂在线产线部署，提供实时流处理、多机协同、MES 集成、模型 OTA。2027 规划。

### Plugin（插件）

= Bundle，节点包的发布形态。开发者发布到 Tinia Store，其他用户订阅安装。

---

## R

### Runtime（tinia_runtime）

节点开发者写 `run.py` 用的 Python SDK。提供 `get_input` / `set_output` / `log` / `progress` / `upload_blob` / `fetch_blob` 等方法。跟 engine 通过 stdio JSON RPC 通信。

---

## S

### SaaS

Tinia 5 个 SKU 中的**多组织云端版**（公网托管）。一个实例托管多个 Org，按 seat 订阅。**规划中**，当前未正式商业化。

### Seat（席位）

激活授权单位。一个 Org 购买若干 Seat，分给 Member 后 Member 才能在自己机器上激活 Tinia 实例。

### Server（团队私有化版）

Tinia 5 个 SKU 中的**团队付费版**（公司内网 web 部署）。面向公司 NVH 部门 / Tier1 测试团队 / 检测中心 / 合资 OEM / 国企。单 Org 多用户，多人协作，含团队管理 + 流程模板库 + 凭据共享 + 报告模板组织级 + 审计日志 + 实施服务。

跟 Pro 的差异：Pro 是**个人桌面**，Server 是**团队 web + 私有化部署**。功能内核一样，差异在协作能力。详见 `reference/03-edition-comparison.md`。

部署形态：客户内网服务器，外部 PostgreSQL + MinIO；可 docker-compose 单机 / K8s 部署 / 空气隔离环境。

### Setup Wizard（设置向导）

Tinia 桌面版首次启动的引导流程。4 步：

1. 选数据目录（DataDir）
2. 自动配置（启动嵌入式 PostgreSQL、解压 Python runtime、装节点）
3. 创建管理员账号
4. 商店激活（OAuth → 选组织 → 消耗 seat）

### Siemens Testlab

德国西门子（前 LMS）的 NVH 分析与测试集成软件。跟 HEAD ArtemiS 并列的传统主流工具。

### Store（Tinia_Store）

Tinia 节点商店，独立部署的服务。负责：

- 节点发布审批
- 用户订阅
- 商业节点收费 + 30% 平台分成
- Tinia 实例激活授权

仓库：独立 `Tinia_Store`，部署在 `tinia.bestfunc.com`。

---

## T

### tinia-cli

= CLI，见上。

### tinia-engine

= Engine，见上。

### tinia-graph-spec

Tinia 流程 JSON / 节点 manifest / 事件协议的规范文档。让多语言、多实现可以兼容。

### tinia_runtime

= Runtime，见上。

### tinia-store

= Store，见上。

### Tinia_nodes

Tinia 官方节点仓库（`https://github.com/bestfunc/Tinia_nodes`）。所有 `bestfunc/` namespace 的节点都在这里维护。

### Tinia_Plugins

Tinia 的 Claude Code Plugin marketplace 仓库（`https://github.com/bestfunc/Tinia_Plugins`）。提供 4 个变体 plugin（tinia / tinia-onprem / tinia-desktop / tinia-local）+ 共享 skills + 本地 MCP bundle。

---

## V

### Viewer（可视化器）

节点结果的前端可视化组件。每个节点可以有自己的专属 Viewer（如 `spectrum_viewer` 画频谱）或落到通用 Viewer。

### View / Tab View

在桌面版的流程编辑器里，点节点的"查看视图"按钮打开的 view tab。同一编辑器内可开多个 view tab，编辑器状态保持不丢。

---

## W

### Wails

Go 语言的桌面 app 框架，类似 Electron 但用原生 WebView（Windows WebView2 / macOS WKWebView）而非 Chromium。Tinia 桌面版用 Wails v2 把 Go binary + React 前端打包成单 exe / app bundle。

---

## 缩写速查

| 缩写 | 全称 | 解释 |
|---|---|---|
| AAC | Advanced Audio Coding | 音频压缩格式 |
| ARR | Annual Recurring Revenue | 年度经常性收入（商业指标） |
| BPMN | Business Process Model and Notation | 业务流程建模语言（SmartQuality 用） |
| CMS | Condition Monitoring System | 状态监测系统（风电场常用） |
| DAG | Directed Acyclic Graph | 有向无环图（流程） |
| FRF | Frequency Response Function | 频响函数（实验模态用） |
| MCP | Model Context Protocol | AI 工具协议 |
| MES | Manufacturing Execution System | 制造执行系统（产线集成） |
| NVH | Noise, Vibration, Harshness | 噪声、振动、粗糙度 |
| OEM | Original Equipment Manufacturer | 整车厂 |
| OTA | Over The Air | 在线远程更新 |
| PdM | Predictive Maintenance | 预测性维护 |
| PSD | Power Spectral Density | 功率谱密度 |
| PoC | Proof of Concept | 概念验证 |
| RPM | Revolutions Per Minute | 转速 |
| SBOM | Software Bill of Materials | 软件依赖清单（合规用） |
| SOW | Statement of Work | 工作说明（合同附件） |
| STFT | Short-Time Fourier Transform | 短时傅里叶变换 |
| TPA | Transfer Path Analysis | 传递路径分析 |
| TAM | Total Addressable Market | 可寻址市场总量 |
| Tier1 | Tier 1 Supplier | 整车厂一级供应商 |
