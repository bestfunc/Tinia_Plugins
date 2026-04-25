---
name: create-node
display_name: 创建 Tinia 节点
description: 在现有 Tinia 插件项目里添加一个 Python 节点，生成骨架并实现 run.py
user-invocable: true
allowed-tools: mcp__tinia__dev_list_projects,mcp__tinia__dev_list_nodes,mcp__tinia__dev_create_node,mcp__tinia__dev_read_file,mcp__tinia__dev_write_file,mcp__tinia__dev_reload,mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__nodes_read_source
---

# 创建 Tinia 节点

## 流程

### 0. 【必做】先读官方相似节点的源码学风格

> ⚠ **跳过这步几乎必然写出风格不一致的 UI**。Tinia 主应用对节点视觉有强约定 ——
> 顶部不写长篇说明、不堆叠折叠面板、左右布局视图、用主应用 Tailwind token，
> 这些规则文档说不清楚，直接读源码 30 秒就懂。

按你即将写的节点类型，**至少读一个官方节点的 ParamsForm + Viewer 实现**：

```
nodes_list({namespace: "bestfunc"})              # 找类型相似的节点
  - analyzer 类  → 读 level_meter / fft_spectrum
  - transform 类 → 读 filter_node / convergent_trim
  - viewer 类    → 读 indicator_viewer / spectrum_viewer
  - source 类    → 读 dataset_node / materialize_node

nodes_describe(选中的 key)                       # 看 source_files 列表
nodes_read_source(key, "ui/ParamsForm.tsx")      # 抄风格、抄结构
nodes_read_source(key, "ui/Viewer.tsx") 或 ViewerLoader.tsx
nodes_read_source(key, "node.yaml")              # 顺带看官方 node.yaml 怎么组织
```

读完再开始下面的步骤。**不读直接写 = 用户大概率会让你重写**。

### 1. 确认上下文

- 用户没指定项目 → `dev_list_projects` 展示列表让用户选
- 已指定 → `dev_list_nodes(project_id)` 看已有节点，避免 key 冲突

### 2. 确认节点规格（不要跳过这一步）

一次性问清楚：

- **节点 key**（小写字母/数字/下划线，字母开头，如 `level_meter`）
- **节点中文名**（node.yaml 里的 name，如 "声级计"）
- **分类 category**（默认 transform，可选 source / transform / analyzer / sink / viewer）
- **输入端口**（端口 key / 类型 / 必填否）
  - 不确定类型就调 `nodes_list_types`，别瞎编
  - 常用组合：`data: AudioData` 最宽容 / `data: IndicatorData` 接分析结果
- **输出端口**（端口 key / 类型）
  - 分析节点一般输出 `IndicatorData`
  - 转换节点输出 `ProcessedDataset`
- **参数**（需要用户在前端填的配置），JSON Schema 格式，如
  - `threshold: number, default=65`
  - `weighting: enum ["A", "C", "Z"], default "A"`

### 3. 生成骨架

```
dev_create_node(project_id, key, with_viewer=false)
```

`with_viewer=true` 时会同时生成 `ui/Viewer.tsx`，对应 analysis_node 模板风格。没特殊需求用默认 false。

#### Viewer.tsx vs ViewerLoader.tsx —— 写哪个

节点的"运行结果视图"在两种文件之间二选一（**不要两个都写**）：

| 文件 | 何时用 | Props |
|---|---|---|
| `ui/Viewer.tsx` | **默认选择**：节点输出一种视图就够了（柱状图 / 表格 / 文本 / 单一图表） | `({ runId, nodeId, outputs })` |
| `ui/ViewerLoader.tsx` | 节点要根据用户选择**切换多种视图**（如指标查看器：列表 / 时序图 / 频谱图 三选一） | `({ runId, nodeId, outputs, nodeParams })` —— 自己 lazy load 子 viewer |

**规则**：
- 大部分新节点只需要 `Viewer.tsx`。前端会自动尝试 `viewer_loader` → fallback 到 `viewer`，所以**只写 `Viewer.tsx`** 完全 OK
- 不要两个都写 —— 两个都存在时前端优先用 ViewerLoader，Viewer 永远不被加载
- 不要为了"以后可能加多视图"提前写 ViewerLoader。需要时再加

#### outputs[] 字段（必看）

`Viewer.tsx` / `ViewerLoader.tsx` 收到的 `outputs` 是数组，**每一项的字段**：

| 字段 | 类型 | 含义 |
|---|---|---|
| `port_key` | string | 输出端口 key（如 `result`） |
| `type` | string | 类型 Kind（如 `Json`、`AudioData`） |
| `preview` | any | **首选**：小数据（< 64KB）后端已解析后直接给你的 JSON 对象，**不需要 fetch** |
| `url` | string | NodeOutputBlob 的 URL（带 `?token=` query），**fetch 直接拿原始 blob 内容** |
| `size_bytes` | number | 原始大小 |
| `truncated` | bool | true = preview 是结构摘要（不是完整数据），需走 url |
| `blob_uri` | string | 内部 minio:// URI，前端**不要**直接用 |

**优先级**：
1. `preview` 非空 → 直接用，零额外请求
2. `preview` 空或 `truncated=true` → fetch `url`

#### ⚠ 写 UI 前**必须**先读官方参考实现

Tinia 平台对节点 UI 有**强烈的视觉风格约定**（圆角、配色、按钮样式、表单组件、详情视图布局）。
你自己凭空写出来的 ParamsForm / Viewer **几乎一定不符合**主应用风格 —— 用户会觉得"这个节点画风跟其他节点不一样"。

**写 UI 前**：

```
1. nodes_list({namespace: "bestfunc"})  → 找一个跟你正在写的节点**类型相似**的官方节点
   - 写 analyzer  → 找 level_meter / fft_spectrum / wavelet_transform
   - 写 transform → 找 filter_node / convergent_trim
   - 写 viewer    → 找 indicator_viewer / spectrum_viewer
2. nodes_describe(key)                   → 看 source_files 列表（含 ui/*.tsx）
3. nodes_read_source(key, "ui/ParamsForm.tsx")  → 看官方表单怎么布局
4. nodes_read_source(key, "ui/Viewer.tsx") 或 ViewerLoader.tsx → 看官方视图怎么布局
```

**官方 ParamsForm 风格关键点**（看了源码就明白）：
- **不要自己写超长说明** —— 每个节点参数面板顶部有内置的节点名 + 描述区，节点说明走 ⓘ 帮助按钮（点击弹弹窗）
- **不要堆叠折叠面板** —— 简洁的字段+输入控件即可，必要时用 `<details>` 微折叠
- **不要自定义彩色按钮组** —— 用主应用统一的 input/select/checkbox

**官方 Viewer 风格关键点**：
- **左右布局**：左侧 item 列表（按 item_id/name 选择）+ 右侧主视图
- 主视图按数据特征支持多视图切换（层叠 / 平铺 / 竖铺 / 卡片 / 3D），由用户选
- 所有标题、按钮、卡片复用主应用 token（`text-text-primary` / `bg-card` / `border-border` 等）
- **不要自己定义颜色 hex** —— 全部用 Tailwind token

#### Viewer.tsx 标准模板（推荐：preview-first）

```tsx
import { useEffect, useState } from 'react'

interface ResultPayload {
  // 你的节点输出 JSON 形状
}

export default function Viewer({ outputs }: { runId: string; nodeId: string; outputs: any[] }) {
  const [data, setData] = useState<ResultPayload | null>(null)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const out = outputs.find((o: any) => o.port_key === 'result')   // 'result' 是你的输出端口 key
    if (!out) { setError('未找到 result 端口输出'); return }
    // 优先用 preview（无需 fetch）；preview 是结构摘要时（truncated=true）走 url
    if (out.preview && !out.truncated) {
      setData(out.preview)
      return
    }
    fetch(out.url).then((r) => r.json()).then(setData).catch((e) => setError(String(e)))
  }, [outputs])

  if (error) return <div className="p-4 text-sm text-danger">加载错误：{error}</div>
  if (!data) return <div className="p-4 text-sm text-text-muted">加载中...</div>

  return <div>...</div>
}
```

#### 易错点

- **绝对不要用 `out.url` 之外的字段去 fetch**。`out.blob_uri` 是 `minio://` 协议，浏览器不能直接访问
- **大小写敏感**：文件名必须是 `Viewer.tsx`（V 大写）。`viewer.tsx` 不会被 build 系统识别为 target
- 用户如果只看几个数字 / 一个图表，**99% 用 `preview` 就够**，不用 fetch
- 只有处理音频文件 / 大数据集 / 二进制内容时才用 `fetch(out.url)`

### 4. 改 node.yaml

读回 `nodes/<key>/node.yaml`，按第 2 步的规格改：

**改端口**（骨架默认 `data: AudioData` → `result: IndicatorData`，不对就改）：
```yaml
inputs:
  required:
    data:
      type: AudioData      # ← 按需改
      label: "..."

outputs:
  - key: result
    type: IndicatorData    # ← 按需改
    label: "..."
```

**改分类**：顶部 `category: analyzer`

**改参数 schema**：
- 骨架的 `schemas/params.schema.json` 默认只有一个 threshold
- 按需扩充（每个参数一个 properties 条目）

### 5. 写 run.py

骨架是：

```python
import json
from tinia_runtime import Runtime

rt = Runtime.from_stdin()
rt.emit_progress(0, "开始处理...")

data = rt.fetch_blob(rt.task["inputs"]["data"])
items = json.loads(data)
params = rt.task.get("params", {})

rt.emit_progress(50, "处理中...")

# TODO: 在这里实现节点逻辑
result = {
    "type": "IndicatorData",
    "indicators": [
        {"name": "result", "value": 42, "unit": "dB"}
    ]
}

handle = rt.upload_blob(json.dumps(result).encode(), "application/json")
rt.emit_output("result", handle)
rt.emit_done()
```

**改 TODO 区域** —— 根据用户描述的业务逻辑写。

**常见模式**（音频分析）：

```python
# 遍历 items 下载音频
for i, item in enumerate(items["items"]):
    audio_bytes = rt.fetch_content_url(item["content_url"])
    # 用 numpy/scipy/librosa 分析
    ...
    rt.emit_progress((i + 1) / len(items["items"]))
```

**加依赖**：如果 import 了新库（numpy / scipy / librosa），改 `runtime/requirements.txt`：
```
numpy
scipy
librosa
```

Tinia reload 时会自动 `pip install -r requirements.txt`。

### 6. 测试

```
dev_reload(project_id)
```

返回 `status: ok, node_count: N` 即成功。失败就看 error 字段，或走 `debug-node` skill。

成功后告诉用户：
> 已注册到你的个人命名空间 `{namespace}/{key}`。打开 Tinia Web UI 流程编辑器，节点面板里会看到一个带 DEV 标签的条目。拖到画布上连线，点 Run 看结果。

## 约束提醒

- 节点 key 一旦建好，目录名、node.yaml 里的 key、full_key 都绑定。**改 key 不等于改名，等于新建一个节点**。要改名的话请删旧的建新的
- `run.py` 必须以 `rt = Runtime.from_stdin()` 开头，以 `rt.emit_done()` 结束
- 每个输出端口 **必须 emit_output 一次**；漏一个会让下游连不到
- 未接的 optional 输入在 run.py 里要判空：`rt.task["inputs"].get("mask")`
- stdout 只能出 runtime 约定的事件 JSON —— 调试用 `rt.emit_log()` 或 `print(..., file=sys.stderr)`

## 相关 Skill

- 端口类型选型 → `types-reference`
- SDK 方法详情 → `sdk-reference`
- node.yaml 字段 → `node-yaml`
- 测试失败 → `debug-node`
