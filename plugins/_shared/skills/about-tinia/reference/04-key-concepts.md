# 核心概念

> 不懂这些就理解不了 Tinia。给开发同事、新员工、技术销售用。每个概念配"它解决什么问题"+"在产品里怎么体现"。

---

## 概念地图

```
                           ┌──────────────┐
                           │   流程 Graph  │
                           │   (DAG 图)    │
                           └──────┬───────┘
                                  │ 由若干
                                  ↓
            ┌────────────────────────────────────────┐
            │       节点 Node（DAG 的顶点）            │
            │  ┌──────────┐  ┌──────────┐  ┌──────┐ │
            │  │ run.py   │  │ ParamsForm│  │Viewer│ │
            │  │  (算法)   │  │  (参数 UI)│  │(结果)│ │
            │  └──────────┘  └──────────┘  └──────┘ │
            └────────────────────────────────────────┘
                                  │ 通过
                                  ↓
                       ┌──────────────────┐
                       │  Blob 句柄 Handle  │
                       │  (节点间传数据)    │
                       └──────────────────┘
                                  │ 引用
                                  ↓
                       ┌──────────────────┐
                       │  实际数据 (blob)  │
                       │ local file/MinIO │
                       └──────────────────┘
```

---

## 1. 节点（Node）

### 是什么

DAG 的最小执行单元。一个具体的"做某件事的能力"：

- `level_meter` —— 算声级
- `fft_spectrum` —— 做 FFT 出频谱
- `order_tracking` —— 阶次跟踪
- `signal_generator` —— 生成测试信号

> 两类节点要分清：
> - **主仓内置节点（Go 实现）**：`dataset_node` / `dataset_merge` / `materialize_node` / `dashboard_node` / `csv_export` / `filter_node` 等，编译在 server 二进制里（`server/internal/nodes/builtin/`），不在 Tinia_nodes 仓。内置 Go 节点同样注册在 `bestfunc` 命名空间（如 `bestfunc/dataset_node`），"内置"指实现方式（Go vs Python），不代表换了命名空间。
> - **Tinia_nodes Python 节点**：`level_meter` / `fft_spectrum` / `channel_split` / `channel_select` 等分析/处理/可视化节点（约 41 个，namespace `bestfunc`），以 subprocess 形式运行。

### 节点的组成

```
nodes/level_meter/
├── node.yaml           ← 元数据：input/output 类型、参数 schema、display name
├── runtime/
│   ├── run.py          ← Python 算法本体，subprocess 跑
│   ├── requirements.txt ← Python 依赖（numpy / scipy 等）
│   └── .venv/          ← 自动创建的虚拟环境
└── ui/
    ├── ParamsForm.tsx  ← 前端参数配置面板（≥4 参数或含方法选择必须自定义 + 预设）
    ├── Viewer.tsx      ← 前端结果可视化
    ├── ViewerLoader.tsx ← Viewer 的懒加载入口
    └── Help.tsx        ← 节点说明 + 参数表 + 算法 + 更新履历（含「SDK 说明」标签页）
```

> Python SDK（`tinia_runtime` 等）不再随节点仓分发，已统一到主仓 `server/sdk/python/`，由 server 通过 go:embed 嵌入并在 fork 节点时注入 `PYTHONPATH`，节点本身不携带物理 SDK 目录（消除跨仓库版本漂移）。

### 节点的命名空间

每个节点有完整 key `<namespace>/<name>`：

- `bestfunc/level_meter` —— 官方节点
- `acme-corp/proprietary_indicator` —— 某公司私有节点
- `dr-li-research/wavelet_kurtosis` —— 某研究者发布的节点

详见 `glossary/terms.md` 的 Namespace 条目。

### 节点的输入输出（IO）

每个节点声明自己的 input port + output port：

- **类型化**：input 期待 `AudioData`，output 是 `IndicatorData`，跨类型连接编辑器会报错
- **多通道感知**：节点声明 `channels_mode: per_channel`（每通道独立跑）或 `aggregated`（同时拿全部通道）
- **可选 / 必选**：input port 可标 optional，连不连都能跑

---

## 2. 流程（Graph） / DAG

### 是什么

由若干节点 + 边组成的有向无环图（Directed Acyclic Graph）。

- 节点为顶点
- "上游 output 连到下游 input" 为边
- 不能循环（A 输出 → B 输入 → ... → A 输入 ❌）

### 跟"脚本"的区别

```
脚本（命令式）：
    audio = load("test.wav")
    spectrum = fft(audio)
    plot(spectrum)
    # 一步步执行，逻辑顺序由代码决定

DAG（声明式）：
    [load_node] → [fft_node] → [plot_node]
    # 声明节点 + 连线，执行顺序由引擎按拓扑序自动决定
```

**DAG 的好处**：

- AI 能读懂：节点列表 + 边列表是结构化数据，LLM 容易理解和修改
- 自动并行：无依赖关系的节点能并行跑（如两个分支同时算频谱和声级）
- 可视化天然：直接画成流程图
- 可复用：流程可保存为 JSON 模板，下次换数据源就重跑

### 流程怎么存

```json
{
  "nodes": [
    { "id": "n1", "class_type": "bestfunc/dataset_node", "params": { "datasource_id": 5 } },
    { "id": "n2", "class_type": "bestfunc/fft_spectrum", "params": { "window": "hann" } },
    { "id": "n3", "class_type": "bestfunc/spectrum_viewer", "params": {} }
  ],
  "edges": [
    { "from": "n1.audio", "to": "n2.input" },
    { "from": "n2.spectrum", "to": "n3.input" }
  ]
}
```

### 流程的运行实例（GraphRun）

每次"运行"流程会创建一个 GraphRun 实例：

- 记录每个节点的状态（pending / running / completed / failed / cached / skipped）
- 记录每个节点的输出 blob handle
- 实时事件流推给前端（进度、错误、log）
- 支持取消（kill 所有 subprocess）

跑完后能在 `/graphs/:id/runs/:runId` 历史回放。

---

## 3. 类型系统

Tinia 节点间传递数据有几种核心类型：

### AudioData（音频超类型）

包含原始音频数据 + 元信息（采样率、通道数、时长、physical unit）：

```
AudioData {
  blob_handle: Handle,    // 实际 PCM 数据
  sample_rate: 48000,
  channels: 2,
  duration_s: 10.5,
  physical_unit: "Pa",    // 物理量
  channels_meta: [
    { name: "front_left", unit: "Pa" },
    { name: "front_right", unit: "Pa" }
  ]
}
```

**子类型**（都兼容 AudioData 输入）：

- `MaterializedDataset`：已固化到 blob 的数据集
- `ProcessedDataset`：滤波 / 切割后的数据集

### IndicatorData（指标数据）

一组带语义的数值，用于分析结果：

```
IndicatorData {
  blob_handle: Handle,    // NPZ / Parquet
  items: [
    { item_id: "freq_1000_2000", value: 75.3, unit: "dB" },
    { item_id: "freq_2000_4000", value: 68.1, unit: "dB" }
  ],
  metadata: {...}
}
```

### AnnotationLayer（标注层）

段落标注，用于"这段音频在 2.3-4.1s 是异常"等：

```
AnnotationLayer {
  segments: [
    { start: 2.3, end: 4.1, label: "abnormal" }
  ]
}
```

### Any

通用透传类型，给少量"不关心数据是什么"的节点用。

---

## 4. Blob 句柄（Handle）

### 是什么

二进制数据的引用，不是数据本身。

```
Handle {
  uri: "minio://tinia/blobs/ab/c123def456...",
  hash: "abc123def456...",
  kind: "AudioData",
  mime: "audio/wav",
  byte_size: 12345678
}
```

### 为什么不直接传数据

如果上游节点输出 10GB 数据：

- ❌ 直接传：每个节点之间复制 10GB，内存爆 + 慢
- ✅ 传 handle：每个节点只传一个 JSON 引用，需要时各自 fetch

### blob 物理存在哪

- **桌面单机**：`<data_dir>/blobs/<前缀2>/<hash>.bin`
- **SaaS / Server**：MinIO（S3 兼容对象存储）

blob 存储走统一抽象层，未来可扩展更多后端（如其他对象存储 / NFS），这部分属路线图设想。

### blob 引用计数

- 一个 blob 可能被多个 GraphRun 引用
- 删除 GraphRun 时不直接删 blob，等没人引用了才回收
- 防止"我以为没用了，结果其他流程还在用"

---

## 5. Dashboard / Viewer

### Viewer（单节点结果可视化）

每个节点结果有自己的 Viewer：

- `spectrum_viewer`：频谱图
- `indicator_viewer`：指标表格
- 自定义节点可自带专属 Viewer（用 React + uPlot / ECharts 等画图库）

### Dashboard（看板：多组件组合的可分享驾驶舱）

把多个视图 + 富媒体组件组在一张可布局、可分享的报告里。从早期"组合视图的报告"已成长为整屏排版的驾驶舱：

- **布局**：流式拖拽布局 + 自由布局；整屏排版（把所有组件缩到一屏直接拖动重排、自由缩放）
- **组件类型**：视图组件、章节标题、文本（富文本 + 插图 + 全屏编辑）、Item 列表、维度筛选器（slicer）、单图/多图切换、HTML 组件（嵌入上传的 HTML 文件）、视频组件（看板内播放 + 全屏 + 拖动进度）
- **批量编辑**：Ctrl/⌘ 多选同类组件统一改属性 / 删除 / 调尺寸；视图组件可把一份显示配置一键套用到所有选中
- **配置层 / 数据层分离**：数据冻结为快照，布局跟随作者最新编辑
- **分享**：分享链接固化内容快照，不随运行记录清理而失效，长期可访问
- **性能**：大数据看板先显示框架再按需加载，首屏更快
- 由主仓内置的 `dashboard_node` 驱动，消费上游 viewer 的看板协议输出

### Slicer（切片器）

跨 Viewer 的联动过滤器：

- 例：一个 Slicer 选了"RPM 1500-2000"，所有 Viewer 自动按此过滤显示

---

## 6. 数据源（DataSource）

### 是什么

声学数据的入口。流程的第一步通常是 `dataset_node` 接数据源。

### 支持类型

| 类型 | 说明 |
|---|---|
| **本地文件** | wav / mat / unv / hdf5 等，桌面单机版直接拖进来 |
| **Diffgram** | 外部数据平台对接 |
| **信号发生器** | 内置 synthetic 生成器（白噪声 / 正弦 / 扫频等），给测试用 |
| **Composite DataSource** | 多个独立源按"配对模板"组合成虚拟源 |

### Composite DataSource 三种模式

| 模式 | 适合场景 |
|---|---|
| **模式 A：快捷合并** | 文件名按规则配对（mic_*.wav 跟 rpm_*.csv） |
| **模式 B：CSV 导入** | 表格指定配对关系，复杂场景 |
| **模式 C：手工配对** | UI 拖拽指定，灵活但慢 |

### 通道模板（Channel Template）

为多通道数据源提供"按模板自动命名 + 物理量语义"能力。例如模板 `mic_<seat>_<direction>` 自动把 wav 文件解析成 `mic_left_front` 等通道名。已升级为物理量语义：每通道带名 / 单位 / 校准 dB + 物理量类型（声压 / 加速度 / 速度等 11 种）+ 传感器灵敏度自动换算，整链路传递；通道校准自动套用。独立「通道模板」页支持 CSV 通道模板（列映射 / 时间序列开关）。

---

## 7. 命名空间（Namespace）

### 是什么

节点的归属与可见性隔离机制。

| 类型 | 例子 | 谁可见 |
|---|---|---|
| **official** | `bestfunc/level_meter` | 所有人 |
| **org** | `acme-corp/internal_metric` | 仅 acme-corp 组织内 |
| **user** | `dev-li/test_node` | 仅 dev-li 自己（DevStudio 开发期间）|

### 为什么要 namespace

- 防冲突：两个开发者写同名节点（`my_filter`）不会撞
- 隔离：组织私有节点不应被其他组织看到
- 商业模型：节点商店的"私有节点"靠 namespace 实现

### Bare key 兼容

历史上节点引用是 bare key（如 `level_meter`），后来加 namespace 后保持向后兼容 —— bare key 自动 fallback 到 `bestfunc/level_meter`。

详见 `Tinia/docs/node-namespace-design.md`。

---

## 8. Edition（部署形态）vs SKU（商业形态）

这是两个不同维度，容易混淆：

**部署 Edition（代码层，只有 3 个）** —— Tinia daemon 启动时确定，由 `TINIA_EDITION` 环境变量或编译期 ldflags（`config.DefaultEdition`）决定，兜底为 `server`：

| Edition | 用于 | 默认值差异 |
|---|---|---|
| `desktop` | 桌面单机版（`--desktop` 启动） | 内嵌 PostgreSQL + Python，setup wizard 引导，默认端口 18720 |
| `server` | 公司内网私有化（无参数启动） | 外部 PostgreSQL + MinIO，默认端口 18721 |
| `saas` | 多组织公网（`TINIA_EDITION=saas`） | 多 Org 支持，组织管理 UI，组织间数据隔离 |

> 代码里只有这三个 edition 常量（`EditionDesktop` / `EditionServer` / `EditionSaas`）。**没有 `production` / `community` / `pro` 这样的 edition flag。**

**商业 SKU（打包/授权层，五档）** —— 面向客户的产品形态，与 edition 不是一一对应：

| SKU | 底层 edition | 状态 |
|---|---|---|
| Community（免费桌面） | desktop（未激活/受限） | 已交付 |
| Pro（个人付费桌面） | desktop（已激活） | 已交付 |
| Server（团队私有化） | server | 已交付 |
| SaaS（多租户云） | saas | 已交付 |
| Production（产线版） | — | 路线图（规划中） |

Community 与 Pro 在代码里同属 desktop edition，区别在「桌面是否激活」（激活校验中间件 `EnforceDesktopActivation`），不是不同 edition flag。

前端通过 `/api/v1/meta` 拿到 edition，按 edition 分叉 UI。

详见 `10-deployment-modes.md`、`03-edition-comparison.md`。

---

## 9. Activation / Seat / Org（多租户）

### Organization（组织）

多租户基本单位。一个 Org 包含若干 Member / Seat / Activation。

### Seat（席位）

授权单位。Org 买 N 个 Seat，分给 N 个 Member，每个 Member 在自己机器上激活。

### Activation（实例激活）

把一台 Tinia 安装绑定到一个 Seat 的过程：

1. 用户启动 Tinia → 弹激活向导
2. OAuth 登录商店
3. 选择要绑哪个 Org（必须是 Member 且有未用 Seat）
4. 完成绑定，Seat 状态变成"已使用"
5. 下次启动直接生效，无需再激活

### 续期 / 撤销

- 激活有有效期（年订阅 = 1 年；买断 = 永久）
- Org 管理员可以撤销已分配 Seat（用户下次启动失效）
- 离线宽限期：网络不通时给 7 天宽限继续可用

---

## 10. MCP-native

### 是什么

Tinia 直接内置 MCP server（`/api/v1/mcp`），让 AI 客户端（Claude Code / Codex / Qwen CLI）能调用 70+ 工具（覆盖 8 个模块）完成"开发节点 → 搭流程 → 跑测试 → 改代码"全流程闭环。

不是"加了一个 AI 助手按钮"，是"AI 是 first-class 客户"，跟人类用户平级。

> MCP 接入与 SDK 通路（见下）是两条并行通路：MCP 面向"AI 自动驾驶平台"，SDK 面向"外部程序调用平台算力"。

详见 `11-mcp-ai-integration.md`。

---

## 11. DevStudio / 节点开发 / Plugin

### DevStudio

主仓内置的节点开发 IDE。让节点作者直接在浏览器写代码、跑测试、提交到 Store。

### Dev Project

一个节点开发的工作空间。包含若干节点、共享 SDK、版本快照。

### Plugin / Bundle

= 节点包的发布形态。开发者 publish 后，用户从 Store 订阅，主仓自动安装到 `plugins/` 目录。

---

## 12. Python SDK 通路 / 流式会话 / 调用分析

### SDK 通路（外部程序调用平台算力）

让外部 Python 程序用 `tinia_sdk` 调用平台上已调好的分析 —— 传数据拿结果，**算法仍在平台 server 进程内执行（没有第二个引擎）**。

- 超管「SDK 管理」输入名称即生成可下载的 SDK 包，凭据（`license.json`，含 `license_id` + `secret`）和服务器地址已内置，零配置。
- 鉴权走 license（不是 api_key）：请求头 `Bearer <license_id>.<secret>`，server 端查 `sdk_licenses` 表校验。
- 三种调用方式：① 直接传节点类型 + 参数；② 用节点表单「复制参数」拿到的参数串（自带节点类型）；③ 引用平台流程里调好的节点（平台改参自动生效）。整条流程也能整体调用（放「API 输入」+「API 输出」节点）。
- 传数据便利：直接传文件路径（wav/csv/npz/tdms）或内存数组自动上传；单输入节点连端口名都不用写。同机调用自动走本地直连（Unix domain socket）+ 路径直传，大文件显著更快。
- 节点帮助新增「SDK 说明」标签页，官方 40 个节点已全部补齐示例代码。

### 流式会话 / 实时数据流

SDK 新增流式会话，可持续往流程推数据、实时取回计算结果，适合在线 / 边采边算场景。实时直调跳过缓存、走内存中转，响应更快（server 侧 `internal/sdkapi/stream.go`，JWT session_token + push/recv/close/keepalive）。

节点侧的配套是「跨窗状态延续」：声级计 / FFT 频谱 / 倍频程等基础分析节点的滤波器状态、STFT 残留、滚动统计跨窗无缝延续（节点声明 `_stream_continuous`），从"逐窗独立批处理"升级为"无缝实时逐帧"。节点作者只需维护跨窗状态，不用自己管会话生命周期。

### SDK 调用分析

超管可查看每个 SDK 的调用量、成功率、耗时、Top 节点、最近失败，用于排查与监控。

---

## 13. 常驻执行池（Resident Execution Pool / HotPool）

### 是什么

分析节点的 Python 进程可常驻待命（`tinia_runtime.serve()`），只加载一次库。SDK 高频 / 实时调用直接复用热进程，省掉每次 fork 进程 + import 库的固定开销（约 530ms → 纯计算时间）。

- 按 node fullKey 分桶（per-node venv 隔离），第二次起复用。
- 超管「常驻执行」页：查看运行状态、调进程上限 / 空闲回收、勾选需预热（预热白名单）的节点。

server 侧实现：`server/internal/nodes/hotpool.go`。

---

## 14. 节点输出缓存

按"输入指纹"自动缓存节点输出。命中时跳过执行、显示节省时间，支持单节点缓存开关；标记系统数据集结果也可缓存，「不可缓存」图标仅在手动关掉缓存时显示。配合流式执行引擎，缓存键时序问题已修复。

---

## 15. AI Activity / Lock / Follow

### Activity Stream

前端右下角浮窗显示"AI 正在做什么"的实时事件流。

### Lock（流程锁 / 文件锁）

AI 改某个流程或文件时，UI 显示"AI 正在编辑"的光晕，防止人类同时改造成冲突。

### Follow（跟随导航）

AI 切到某个页面 / 文件时，UI 自动滚到那里，让用户跟得上 AI 在做什么。

详见 `Tinia/docs/ai-activity-architecture.md`。

---

## 下一步

- 多仓库架构怎么支持这些概念 → `02-architecture.md`（注：`tinia-engine` / `tinia_runtime` 作为独立仓的设想已归档，执行只在 server 内、节点 SDK 已并入主仓）
- 节点开发的具体步骤 → 用 `quickstart` / `create-node` skill
- MCP 工具列表 → `Tinia/docs/mcp-tool-reference.md`
- 部署形态差异 → `10-deployment-modes.md`
