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

### 节点的组成

```
nodes/level_meter/
├── node.yaml           ← 元数据：input/output 类型、参数 schema、display name
├── runtime/
│   ├── run.py          ← Python 算法本体，subprocess 跑
│   ├── requirements.txt ← Python 依赖（numpy / scipy 等）
│   └── .venv/          ← 自动创建的虚拟环境
└── ui/
    ├── ParamsForm.tsx  ← 前端参数配置面板
    ├── Viewer.tsx      ← 前端结果可视化
    └── ViewerLoader.tsx ← Viewer 的懒加载入口
```

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
- **未来 Production**：可配置（AWS S3、阿里云 OSS、Azure Blob、本地 NFS）

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

### Dashboard（多 Viewer 组合）

把多个 Viewer 组在一张布局里：

- 拖拽布局（react-grid-layout）
- 文本块 / 图片 / 切片器（slicer）
- 多 Viewer 共存
- 数据源是上游节点的 `dashboard_view` 输出

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

### 通道命名模板

为 Composite DataSource 提供"按模板自动命名通道"能力。例如模板 `mic_<seat>_<direction>` 自动把 wav 文件解析成 `mic_left_front` 等通道名。

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

## 8. Edition（部署形态）

Tinia daemon 启动时的标志：

| Edition | 用于 | 默认值差异 |
|---|---|---|
| `desktop` | 桌面单机版 | 内嵌 PostgreSQL + Python，setup wizard 引导 |
| `server` | 公司内网私有化 | 外部 PostgreSQL，无 setup wizard |
| `saas` | 多组织公网 | 多 Org 支持，组织管理 UI |

前端通过 `/api/v1/meta` 拿到 edition，按 edition 分叉 UI。

详见 `10-deployment-modes.md`。

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

Tinia 直接内置 MCP server（`/api/v1/mcp`），让 AI 客户端（Claude Code / Codex / Qwen CLI）能调用 65+ 工具完成全流程开发。

不是"加了一个 AI 助手按钮"，是"AI 是 first-class 客户"，跟人类用户平级。

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

## 12. AI Activity / Lock / Follow

### Activity Stream

前端右下角浮窗显示"AI 正在做什么"的实时事件流。

### Lock（流程锁 / 文件锁）

AI 改某个流程或文件时，UI 显示"AI 正在编辑"的光晕，防止人类同时改造成冲突。

### Follow（跟随导航）

AI 切到某个页面 / 文件时，UI 自动滚到那里，让用户跟得上 AI 在做什么。

详见 `Tinia/docs/ai-activity-architecture.md`。

---

## 下一步

- 7 件套架构怎么支持这些概念 → `02-architecture.md`
- 节点开发的具体步骤 → 用 `quickstart` / `create-node` skill
- MCP 工具列表 → `Tinia/docs/mcp-tool-reference.md`
- 部署形态差异 → `10-deployment-modes.md`
