---
name: create-node
display_name: 创建 Tinia 节点
description: 在现有 Tinia 插件项目里添加一个 Python 节点，生成骨架并实现 run.py
user-invocable: true
allowed-tools: mcp__tinia__dev_list_projects,mcp__tinia__dev_list_nodes,mcp__tinia__dev_create_node,mcp__tinia__dev_read_file,mcp__tinia__dev_grep_files,mcp__tinia__dev_glob_files,mcp__tinia__dev_write_file,mcp__tinia__dev_edit_file,mcp__tinia__dev_reload,mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__nodes_read_source
---

# 创建 Tinia 节点

## ⛔ 完成定义（必读）

一个"完成"的 Tinia 节点 = 下面 6 件事**全部**做完，少一件就是没完成：

1. **参数齐全**：node.yaml + params.schema.json 列出**所有合理可调节项**，不要因为"用户没明说"就只写 1-2 个。业务类型常见参数下方有清单。
2. **node.yaml 端口正确**：input/output type 是真实存在的 type kind，不是瞎填
3. **run.py 实现完整**：handle 所有 emit_output + 进度上报
4. **`ui/ParamsForm.tsx` 必写**（不是可选）：默认文本框渲染体验差，凡是有枚举/布尔/范围/条件字段的节点**全部要写**自定义表单
5. **节点说明弹窗必写**：ParamsForm 顶部 HelpCircle 帮助按钮 → 弹窗里含：用途 / 输入输出 / 各参数详细说明 / 典型用法 / 已知限制
6. **跑通 dev_reload 无错**

> **常见反例（用户必让重写）**：
> - "用户没要求那么多参数 → 我只开 2 个" ❌ 业务上能调的就要给用户调
> - "默认表单足够 → 不写 ParamsForm" ❌ 只要参数有枚举/布尔，**必写**
> - "节点说明走 README → ParamsForm 不写帮助" ❌ 用户在画布编辑时看不到 README
> - "ParamsForm 顶部直接堆说明文字" ❌ 走 HelpCircle 弹窗，详见 `params-form` skill

## ⚠ 写文件工具选择

| 场景 | 用什么 |
|---|---|
| 创建新文件 / 完全重写 | `dev_write_file` |
| **改一两处文本 / 改一个函数 / 加一个 import** | **`dev_edit_file`** |

`dev_edit_file(path, old_string, new_string, replace_all=false)` 借鉴 Claude Code Edit：
- 找到 `old_string`（必须在文件里**唯一**出现，含空格/缩进/换行）→ 替换为 `new_string`
- 多处出现要么加更多上下文使其唯一，要么 `replace_all=true`
- 比 dev_write_file 整文件重传省 10~100 倍 token，且不会复制粘贴出错把别处改丢

**默认局部修改一律用 dev_edit_file**。dev_write_file 只在新建文件 / 节点骨架修改后第一次写完整内容时用。

**找代码 / 看大文件**（不要逐个 nodes_describe + nodes_read_source 翻官方节点）：

| 场景 | 用什么 |
|---|---|
| 知道函数名找用法 | `dev_grep_files("rt\\.fetch_blob", scope="official", glob="*.py")` |
| 找参考组件 | `dev_grep_files("HelpCircle", scope="official", glob="*.tsx")` |
| 找文件名 | `dev_glob_files(pattern="ParamsForm.tsx", scope="official")` |
| 看大文件局部 | `dev_read_file(path, offset=100, limit=50)` —— 1-indexed |

scope 可选 `dev` (默认) / `official` / `all`，返回路径前缀 `dev:<path>` 或 `official:<fullKey>/<rel>` 让你知道源头。**找参考实现优先 grep，不要遍历 nodes_describe**。

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

nodes_describe(选中的 key, fields=["meta","ui"])  # 仅取 source_files 清单（不拉 readme/yaml 省 token）
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

#### 参数 ⚠ 关键原则：宁多勿少

不要"只暴露用户明说的几个参数"。业务上每个能调节的环节都要开放成参数让用户在前端配。
**列完一遍**，每条带 default + range + description 写到 schema。

按业务类型对照检查（不是穷举，是兜底）：

| 节点类型 | 至少要开放的参数维度 |
|---|---|
| **声学/音频分析（FFT/小波/包络/频带能量/MFCC 等）** | 算法核心参数（如 wavelet / nperseg / window）+ 通道处理（mono_mix/per_channel/left/right）+ 重采样目标 + 时间窗裁剪（start/end）+ 预处理（去直流/预加重/高通/归一化）+ 输出格式（dB 转换/单位）+ 跳过空数据策略 |
| **滤波/变换** | 滤波类型（低通/高通/带通/带阻）+ 截止频率 + 阶数 + 滤波模式（filtfilt/lfilter）+ 边界处理 |
| **特征/统计** | 统计指标列表（mean/std/rms/peak/kurtosis/skewness/entropy/zcr 等）多选 + 时间窗大小 + 重叠率 + 归一化 |
| **数据筛选/采样** | 抽样策略（random/first/top/stratified）+ 数量/比例 + 排序字段 + 随机种子 + 阈值字段 + 阈值方向（>=/<=）+ 是否保留原顺序 |
| **机器学习/聚类** | 距离度量 + 聚类数 / eps / min_samples + 标准化方式 + 随机种子 + 迭代上限 |
| **段落/事件检测** | 检测算法 + 阈值（绝对/相对）+ 滞回 + 最小段长 + 合并间距 + 输出方向（time/sample） |

> 用户会感谢"参数多但默认合理"，烦"参数少必须改 run.py"。

参数 schema 写法（每条都要写 description，让 ParamsForm/帮助弹窗能用）：

```json
{
  "wavelet": {
    "type": "string",
    "enum": ["db4", "db8", "sym4", "sym8", "coif1", "coif3", "coif5", "bior2.2", "bior4.4", "haar"],
    "default": "db4",
    "description": "小波基。db/sym 通用、coif 平滑性好、bior 对称无相位失真、haar 速度最快"
  },
  "level": {
    "type": "integer",
    "minimum": 0,
    "maximum": 12,
    "default": 0,
    "description": "0=由 pywt.dwt_max_level 自动决定；分解出 1 个 cA + N 个 cD 频带"
  }
}
```

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
2. nodes_describe(key, fields=["meta","ui"]) → 看 source_files 列表（含 ui/*.tsx），省 token
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
- 时序数据用 `uplot`、复杂图表用 `echarts`、表格用原生 `<table>`，**不要手画 div 柱状图**
- 所有标题、按钮、卡片复用主应用 token（`text-text-primary` / `bg-card` / `border-border` 等）
- **不要自己定义颜色 hex** —— 全部用 Tailwind token

**详细的 Viewer 模板和反模式见 `tinia:result-view` skill**（按数据类型分别给完整模板：IndicatorData / FeatureMatrix / AnomalyResult / AudioData 等）。写 Viewer 之前**必看**那个 skill。

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

**音频分析节点加 `channels_mode`**（重要）：

```yaml
channels_mode: per_channel    # 默认行为，多通道源自动按通道展开成 N items
```

可选值：
- `per_channel`（默认推荐）：N 通道 → N 个独立 item
- `mix_down`：自动 mean 单通道
- `first_only` / `requires_single` / `multichannel_aware`：详见 `node-yaml` skill

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

### 6. 写 `ui/ParamsForm.tsx`（必做，不是可选）

> 默认表单（`<input type="text">`）对枚举/布尔/范围/条件字段全部失效，体验差。
> 凡是节点有任何枚举/布尔/范围/依赖字段（**几乎所有节点**）—— **必写**自定义表单。
>
> 详细的写法、Tailwind token、import 白名单、避坑清单见 `params-form` skill。

最小检查表（不达标 = 没完成）：

- [ ] 顶部一个 HelpCircle 帮助按钮 → 点击弹出"节点说明"弹窗
- [ ] 弹窗内含：用途 / 输入 / 输出 / 各参数详细说明 / 典型用法 / 已知限制
- [ ] schema 里的每个参数都有 UI 控件（不是只渲染了一两个）
- [ ] 枚举用 `<select>`、布尔用 `<input type="checkbox">`、有范围的数字用 `<input type="number" min max>`
- [ ] 字段间依赖用条件渲染（如选 X 才显示 Y）
- [ ] 用主应用 Tailwind token（`bg-input` / `text-text-muted` / `border-border` 等），不写 hex 颜色
- [ ] 不堆叠折叠面板，扁平排列即可

写完用 `dev_write_file` 落到 `nodes/<key>/ui/ParamsForm.tsx`。

### 7. 测试

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
