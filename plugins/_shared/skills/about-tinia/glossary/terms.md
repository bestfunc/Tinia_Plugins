# Tinia 术语表

> 所有其他文档第一次出现某术语时会链回这里。看不懂某词，直接在本页 Ctrl+F。

按字母顺序，中英混排。

---

## A

### Activation（激活）

Tinia 桌面实例与商店组织绑定的过程，绑定后从 Community 解锁为 Pro。激活基于"组织（Org）→ 席位（Seat）→ 实例（Instance）"三层模型：

- 组织买若干 Seat（席位）
- 组织管理员把 Seat 分给具体用户（seat 已支持 user-level `seat_id` 模型）
- 用户在某台机器（Instance）上用自己被分到的 Seat 完成激活
- 激活后该机器解锁 Pro 功能直到 Seat 过期或被回收
- 离线宽限期：网络不通时给 7 天宽限继续可用

激活走 `store_url`（商店）。代码层由桌面激活校验中间件 `EnforceDesktopActivation` 实现 —— Community 与 Pro 同属 `desktop` edition，区别只在「是否已激活」。Community 版无需激活，开箱即用（功能受限）。

> 身份 / OAuth / JWKS / entitlement / seat 等"授权控制面"正逐步由独立的 Bestfunc Passport 仓承担（见 Passport 条目），Store 侧主要负责节点 catalog / 分成。

### Activity Stream（活动流）

前端 UI 实时显示"AI 正在做什么"的事件流。AI 通过 MCP 调用工具时（如创建项目 / 写文件 / 跑流程），daemon 把事件推到 `GET /api/v1/mcp/activity/stream`（SSE），前端在右下角浮窗里展示。不是 MCP 协议的一部分，是 Tinia 自家可视化。

### Audio Player（音频播放器）

Tinia 的音频联动播放能力（由 `spectrum_viewer` / 播放器 widget 提供）。能播放上游节点输出的 `AudioData`，时长从 `materialize_node` 输出的元信息读取。

### AutoML / 调参

平台内置的超参搜索 + 评估能力。节点参数通过 schema 里的 `x-tinia-tunable` 声明可搜范围，平台据此对节点做超参搜索。已交付完整闭环：4 步配置向导 + 训练/验证拆分 + 过拟合警告 + 学习曲线 + 参数重要性 + Top-K + 区分度诊断；调参自动 fork 快照、早停（MedianPruner）、CMA-ES、智能建议、特征自适应筛选；「限值直评」+「相似度筛选」+「继续探索」+「再跑一次」，调参上限放宽到 2000。节点可单独勾「调参 / 评估」角色，任务同组织共享。

**AutoML 只管定参** —— 它回答"哪组参数效果好"，不回答"用什么判别函数"。v1.45 起判别函数从
AutoML 剥离成独立的通用节点（见「评分器」），模型走制品库；原先的判别函数 tab / 一键生成评分节点
入口已移除。副作用节点（写库 / 发消息）可声明 `skip_side_effects_in_trial`，搜参时跳过写入但照常透传。

### 评分器 / 制品库 / 检测记录

把高维特征落到实际检测的三件套（v1.45）：

- **评分器**（`score_predictor`）：把多维特征降成一个分。训练 / 推理 / 评估三模式 —— 不接模型端口就训练、接了就推理。监督组 5 种算法，单类组 3 种（只有 OK 样本也能冷启动）。
- **制品库**（`model_artifact`）：模型存哪、怎么管版本。双层版本号（大版本 / 子版本）+ active 上线指针 —— **换模型 = 移动指针，检测流程零改动**。
- **检测记录**（`flow_record`）：每次判定结果落库，供事后查询、统计、人工复核回填真值，真值回流后再重训。

配套的**特征池**（`feature_pool_write` / `read`）攒的是特征而不是音频 —— 提特征是整条链最慢的一步，
存下来后重训几秒出新模型，不用把历史音频重跑一遍。

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

### Channel Template（通道模板）

为多通道数据源提供"按模板自动命名 + 物理量语义"的能力。每通道带名 / 单位 / 校准 dB + 物理量类型（声压 / 加速度 / 速度等 11 种）+ 传感器灵敏度自动换算，整链路传递；通道校准自动套用。独立「通道模板」页支持 CSV 通道模板（列映射 / 时间序列开关）。

### Community（社区版）

商业 SKU 中的免费层（桌面单机），底层是未激活的 `desktop` edition。面向学生 / 研究者 / 个人工程师，提供基础能力。**已交付。** 详见 `reference/03-edition-comparison.md`。

> 注意：Community 是商业 SKU，不是 edition flag。代码里只有 desktop / server / saas 三个 edition，没有 `community` edition 常量。

### Cache（节点输出缓存）

按"输入指纹"自动缓存节点输出。命中时跳过执行、显示节省时间，支持单节点缓存开关；标记系统数据集结果也可缓存。「不可缓存」图标仅在手动关掉缓存时显示。配合流式执行引擎，缓存键时序问题已修复。SDK 实时直调可跳过缓存走内存中转。

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

Tinia 后端守护进程，由主仓 `cmd/server/main.go` 编译出的**单一二进制**提供。运行形态：

- `--desktop`：桌面单机模式，内嵌前端 + PostgreSQL + Python runtime + setup wizard（同 binary 以 `daemon` 子命令常驻 + Wails 窗口接管）
- 无参数：runServer，跑 server / saas 部署；具体 edition 由 `TINIA_EDITION` 环境变量决定（没有 `--saas` flag）
- `--setup`：强制重跑 setup wizard

### Dashboard（看板）

把多个视图 + 富媒体组件组合在一张可布局、可分享报告里的能力，由主仓内置 `dashboard_node` 驱动。看板编辑器支持流式 + 自由布局、整屏排版、批量编辑（多选同类组件统一改属性/删除/调尺寸）。组件类型：视图、章节标题、文本（富文本 + 插图 + 全屏编辑）、Item 列表、维度筛选器（slicer）、单图/多图切换、HTML 组件（嵌入上传的 HTML 文件）、视频组件（看板内播放 + 全屏 + 拖动进度）。分享链接固化内容快照，不随运行记录清理失效。配置层与数据层分离（数据冻结快照，布局跟随作者最新编辑）。详见 `reference/04-key-concepts.md`。

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

### Edition（部署形态标志）

Tinia daemon 的部署形态标志，代码里**只有 3 个**（`config.go` 的 `EditionDesktop` / `EditionServer` / `EditionSaas`）：

- `saas`：多组织 SaaS，跑在公网（`tinia-saas.bestfunc.com`），靠 `TINIA_EDITION=saas` 启用
- `server`：单组织私有化部署，公司内网（`t.bestfunc.com`），无参数启动的默认兜底
- `desktop`：单用户桌面单机版（`--desktop`）

edition 解析优先级：env `TINIA_EDITION` > 编译期 ldflags `config.DefaultEdition` > 兜底 `server`。通过 `/api/v1/meta` 端点对前端暴露，前端按 edition 分叉 UI。

> Edition（部署层，3 个）≠ SKU（商业层，5 档）。Community / Pro 都跑在 desktop edition 上，区别在是否激活；Production 是路线图 SKU，没有对应 edition。详见 `reference/04-key-concepts.md` 第 8 节。

### Engine（tinia-engine，已归档）

> **历史概念，已归档不再投入。** 早期设想把执行引擎抽成独立 binary / Go library（`bestfunc/Tinia_Engine`）供第三方嵌入。现状：因"双引擎必然版本漂移、收益被桌面单二进制 + headless 覆盖"，`Tinia_Engine` / `Tinia_Runtime` / `Tinia_Graph_Spec` 三仓已归档。**执行永远发生在 tinia-server 进程内，没有第二个引擎。** 详见 `reference/02-architecture.md`、`Tinia/docs/sdk-design.md`。

---

## F

### FeatureMatrix / AttributeTable（特征 / 属性类型）

类型系统中用于特征工程的类型：`FeatureMatrix`（特征矩阵，`feature_merge` 等特征类节点的核心枢纽）、`AttributeTable`（属性表，`attribute_extract` 输出）。配合 `AnnotationLayer`（段落标注）构成标注 / 属性 / 特征侧链路。

### Framework（Tinia Framework，已归档）

> **历史概念，已归档。** 早期设想 = Tinia Engine + Runtime + 节点规范（tinia-graph-spec）三件套的统称，给业务平台提供解耦的"加载节点 + 运行流程"能力。随 Engine / Runtime / Graph_Spec 三仓归档而废弃。现状是单二进制内执行 + 节点 SDK 并入主仓 go:embed。

---

## G

### GPU sidecar（共享 GPU 计算）

主仓 `internal/gpucompute` 提供的 GPU 加速能力。节点通过 IPC 复用共享的 torch sidecar 做 GPU 计算（如 `scale_space_spectrum` 多尺度谱），未 provision GPU 时自动回退 numpy。

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

### HotPool（常驻执行池 / Resident Execution Pool）

分析节点的 Python 进程常驻待命（`tinia_runtime.serve()`），只加载一次库。SDK 高频 / 实时调用直接复用热进程，省掉每次 fork 进程 + import 库的固定开销（约 530ms → 纯计算时间）。按 node fullKey 分桶（per-node venv 隔离）。超管「常驻执行」页可查看运行状态、调进程上限 / 空闲回收、勾选预热白名单。server 侧实现 `server/internal/nodes/hotpool.go`。

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

旋转机械振动分析的经典方法。把振动信号按转速重采样到角度域，得到与转速无关的频谱（阶次谱）。已作为官方节点 `bestfunc/order_tracking` 交付。

---

## P

### Passport（Bestfunc Passport）

bestfunc 全产品的账号 + 授权控制面（身份提供方），Tinia 是接入方之一。独立仓（Gin + GORM + Postgres，独立库 `bestfunc_passport`，dev 端口 18725/18726），对外提供 OAuth / JWKS（离线验签）/ userinfo / entitlement。Tinia 侧经 `server/internal/auth/sso.go` 等接入。身份 / OAuth / JWKS / entitlement / seat 等授权控制面正逐步由 Passport 承担，Store 侧主要负责节点 catalog / 分成。

仓库：`https://github.com/bestfunc/Bestfunc_Passport`

### PdM（Predictive Maintenance，预测性维护）

工业设备健康监测、提前预警故障。Tinia Pro 第三波 + Production 的目标场景之一（风电、工业泵、电机厂）。

### Pro（Tinia Pro）

商业 SKU 中的**个人付费版**（桌面单机，底层是已激活的 `desktop` edition）。面向个人 NVH 工程师 / 独立顾问 / 自由职业，提供完整离线分析工作站、多通道、报告导出、Order Tracking。**已交付。**

跟 Server 的差异：Pro 是**个人**，Server 是**团队**（详见 `reference/03-edition-comparison.md`）。

### Production（Tinia Production）

商业 SKU 中的**在线产线版**（边缘 + 云端）。设想面向工厂在线产线部署，提供实时流处理、多机协同、MES 集成、模型 OTA。**路线图 / 规划中**，是目前唯一明确未交付的主线 SKU，没有对应 edition flag。

### Plugin（插件）

= Bundle，节点包的发布形态。开发者发布到 Tinia Store，其他用户订阅安装。

---

## R

### Runtime（tinia_runtime）

节点开发者写 `run.py` 用的 Python SDK（含 `tinia_audio` / `tinia_audio_input` / `tinia_features` 等）。提供 `get_input` / `set_output` / `log` / `progress` / `upload_blob` / `fetch_blob` 等方法，以及端口类型契约（AudioData / IndicatorData / FeatureMatrix / AnnotationLayer / AttributeTable）、quantity 契约、错误分级、ChunkRuntime 分块流式 endpoints。

**唯一事实源已从节点仓迁出，统一到主仓 `server/sdk/python/`，由 server 通过 go:embed 嵌入二进制**，fork 节点时注入 `PYTHONPATH` 指向嵌入版 —— 不再是独立仓、也不是独立 pip 包（消除早期各节点仓自带副本导致的协议漂移）。节点开发者本地可 `tinia sdk install --target ./sdk/python/` 拷一份做 IDE 类型提示（gitignore，不进发布）。

---

## S

### SDK（Python SDK 通路）

让外部 Python 程序用 `tinia_sdk` 调用平台上已调好的分析 —— 传数据拿结果，**算法仍在平台 server 进程内执行，没有第二个引擎**。超管「SDK 管理」输入名称即生成可下载的零配置 SDK 包（凭据 + 服务器地址内置）。三种调用方式：① 直接传节点类型 + 参数；② 用「复制参数」串；③ 引用平台流程里调好的节点（平台改参自动生效）。整条流程也能整体调用（「API 输入」+「API 输出」节点）。传数据可直接给文件路径（wav/csv/npz/tdms）或内存数组；同机调用走本地直连（UDS）+ 路径直传。鉴权见 SDK License。已交付（主线 v1.33）。详见 `reference/04-key-concepts.md`、`Tinia/docs/sdk-design.md`。

> SDK 与 MCP 是两条并行通路：MCP 让 AI 驱动平台，SDK 让外部程序调用平台算力。

### SDK License（SDK 凭据）

SDK 包的鉴权凭据。打包进 SDK 的 `license.json`（`license_id` + `secret`），请求头 `Bearer <license_id>.<secret>`，server 端 `internal/sdkapi/middleware.go` 的 `LicenseAuth` 查 `sdk_licenses` 表校验。**不是 api_key 概念。** 超管「SDK 管理」生成、「SDK 调用分析」查看每个 SDK 的调用量、成功率、耗时、Top 节点、最近失败。

### Streaming Session（SDK 流式会话 / 实时数据流）

SDK 的流式能力：可持续往流程推数据、实时取回计算结果，适合在线 / 边采边算场景。实时直调跳过缓存、走内存中转，响应更快（server 侧 `internal/sdkapi/stream.go`，JWT session_token + push/recv/close/keepalive）。节点侧配套「跨窗状态延续」（声级计 / FFT / 倍频程等声明 `_stream_continuous`，滤波器状态、STFT 残留、滚动统计跨窗无缝），从逐窗批处理升级为无缝实时逐帧。已交付（主线 v1.35）。

### SaaS

商业 SKU 中的**多组织云端版**（公网托管，底层 `saas` edition）。一个实例托管多个 Org，按 seat 订阅，组织间数据严格隔离 + 顶部组织切换器 + 角色按组织独立。**已交付。**

### Seat（席位）

激活授权单位。一个 Org 购买若干 Seat，分给 Member 后 Member 才能在自己机器上激活 Tinia 实例。

### Server（团队私有化版）

商业 SKU 中的**团队付费版**（公司内网 web 部署，底层 `server` edition）。**已交付。**面向公司 NVH 部门 / Tier1 测试团队 / 检测中心 / 合资 OEM / 国企。单 Org 多用户，多人协作，含团队管理 + 流程模板库 + 凭据共享 + 报告模板组织级 + 审计日志 + 实施服务。

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
- 商业节点收费 + 平台分成
- 节点 catalog；激活授权走 `store_url`

> 身份 / OAuth / JWKS / entitlement / seat 等授权控制面正逐步由独立的 Bestfunc Passport 承担（见 Passport 条目），Store 侧聚焦节点 catalog / 分成。

仓库：独立 `Tinia_Store`，部署在 `tinia.bestfunc.com`。

---

## T

### tinia-cli

= CLI，见上。

### tinia-engine

= Engine（已归档），见上。

### tinia-graph-spec

> **历史概念，已归档。** 曾设想做 Tinia 流程 JSON / 节点 manifest / 事件协议的跨语言规范文档（`Tinia_Graph_Spec`，曾到 v0.3.0）。随双引擎方案放弃而归档。

### tinia_runtime

= Runtime，见上（已并入主仓 go:embed）。

### tinia_sdk

外部程序调用平台算力用的 Python SDK 客户端（区别于节点内用的 `tinia_runtime`）。`connect(server_url, license_path, socket_path, use_uds)` 连接平台，凭据从打包进 SDK 的 `license.json`（`license_id` + `secret`）读，鉴权头 `Bearer <license_id>.<secret>`，server 端 `LicenseAuth` 查 `sdk_licenses` 表校验。支持同机 Unix domain socket 直连 + 路径直传。见 SDK 条目。

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
