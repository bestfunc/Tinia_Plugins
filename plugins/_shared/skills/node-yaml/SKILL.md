---
name: node-yaml
display_name: node.yaml 字段速查
description: Tinia 节点清单（node.yaml）的所有字段含义 + 典型取值 + 默认行为
user-invocable: false
---

# `node.yaml` 字段速查

每个节点目录下必须有一个 `node.yaml`（大小写敏感），Tinia 启动/测试节点时读这个文件注册元数据。

## 最小有效例

```yaml
key: my_node
name: "我的节点"
sub_name: "My Node"
category: transform
version: "1.0"

inputs:
  required:
    data:
      type: AudioData
      label: "输入数据"

outputs:
  - key: result
    type: IndicatorData
    label: "分析结果"

params_schema: schemas/params.schema.json

runtime:
  type: python
  entrypoint: runtime/run.py
  requirements: runtime/requirements.txt
```

## 字段详解

### `key`（必填，string）

节点短 key，和**目录名**一致。小写字母/数字/下划线，字母开头。
注册后和命名空间拼成 full_key：`{namespace}/{key}`，如 `bestfunc/level_meter`。

**改 key = 新节点**：已有的分析流程引用了旧 key 会失配。改名不如删重建。

### `name`（string）

前端节点面板显示的名字。随便起，中文 OK。

### `sub_name`（string，可选）

副名（小字），显示在 `name` 旁边。设计用途：节点中文主名 + 英文副名（如 `name: 图表查看器 / sub_name: Chart Viewer`），或者主名 + 风味描述。
独立于多语言体系 —— 跟 `name` 一起总会显示（用户可在节点库切是否显示副名）。

### `category`（string，默认 `transform`）

影响前端节点面板分组。常用：`source` / `transform` / `analyzer` / `sink` / `viewer`。

### `version`（string，默认 `0.0.0`）

节点版本。插件级版本在 `tinia-repo.yaml` 里，一般 node.yaml 的 version 跟着插件走就行（例 `"1.0"`）。

### `description`（string，可选）

节点面板里的说明文本（显示 "说明" 按钮时才出）。

### `inputs`（map）

输入端口。分 3 类：

```yaml
inputs:
  required:       # 必接，没接节点启动失败
    data:
      type: AudioData
      label: "输入数据"
  optional:       # 可不接，run.py 里要判空
    mask:
      type: AnnotationLayer
      label: "段落遮罩"
  hidden:         # 内部用，不渲染到节点卡片（如 viewer 读 blob）
    …
```

**每个端口字段**：
- `type`：类型名（见 `types-reference`）
- `label`：显示名
- `default`：默认值（连线没连时注入）
- `enabled_when`：参数 key，只有该参数为真时此端口才启用
- `required`：true/false —— 写在 required: 下就隐式 true

### `outputs`（array，注意是数组不是 map）

```yaml
outputs:
  - key: result
    type: IndicatorData
    label: "分析结果"
  - key: spectrum
    type: FileBlob
    label: "频谱图"
```

### `dynamic_inputs`（object，可选）

动态端口（前端 +/- 按钮加端口）：

```yaml
dynamic_inputs:
  enabled: true
  prefix: in              # 端口 key = in_1, in_2, ...
  label: "特征源"          # 端口显示名模板
  port_type: FeatureMatrix # 推荐用 FeatureMatrix（超类型，自动接 IndicatorData）
  min_ports: 2
  max_ports: 10           # 0 = 无限制
```

**`port_type` 选 FeatureMatrix 还是 IndicatorData**：
- 想接"任何特征源" → `FeatureMatrix`（自动兼容单标量指标 + 多列特征矩阵）
- 强制只收单标量指标 → `IndicatorData`
- 通用透传节点 → `Any`

### `params_schema`（string，相对路径）

JSON Schema 文件路径。前端用 @rjsf/core 渲染表单，值传入 `rt.task["params"]`。

样例 `schemas/params.schema.json`：
```json
{
  "type": "object",
  "properties": {
    "weighting": {
      "type": "string",
      "title": "加权方式",
      "enum": ["A", "C", "Z"],
      "default": "A"
    },
    "threshold": {
      "type": "number",
      "title": "阈值 (dB)",
      "default": 65,
      "minimum": 0,
      "maximum": 200
    }
  }
}
```

### `runtime`（object，必填）

```yaml
runtime:
  type: python            # 目前仅支持 python
  entrypoint: runtime/run.py
  requirements: runtime/requirements.txt    # 可选，会被 pip install
  python: "3.10"          # 可选，指定解释器版本
```

### `channels_mode`（string，可选；音频分析节点用）

声明节点对**多通道音频输入**的展开策略。SDK `AudioInput.iter_channels()` 据此分发，节点开发者不写多通道路由代码。

```yaml
channels_mode: per_channel   # 分析节点推荐
```

| 值 | 行为 | 适用 |
|----|------|------|
| `per_channel` | N 通道 → N 次 iter，输出 N 个 item | **大多数分析节点**（fft / loudness / level_meter ...）|
| `mix_down` | 多通道 mean → 单通道，1 个 item | 显式只关心混合信号 |
| `first_only` | 只取 ch0，1 个 item | 单通道 only 节点 |
| `requires_single` | 多通道直接报错 | 严格接口（要求上游先 split/select）|
| `multichannel_aware` | 节点自己处理 (n_ch, n_samples) | channel_split / channel_select 这种通道操作节点 |

**不设 = `requires_single`**（通道语义 v2 起 fail-fast：不声明就不允许多通道输入，杜绝静默平均；单通道输入不受影响）。老文档说缺省 per_channel 已废止 — **所有处理多通道的节点必须显式声明**。详见 `sdk-reference` 的 AudioInput 章节。

### `accepts_quantities` / `emits_quantity`（可选；物理量契约）

```yaml
accepts_quantities: [sound_pressure]   # 节点接受的物理量集合;不声明 = 任意
emits_quantity: velocity               # 节点输出的物理量;不声明 = 透传输入
```

- **accepts_quantities**：算法仅对特定物理量有意义时声明（如心理声学节点只对声压有定义）。编辑器会沿连线上溯数据源，通道物理量不在集合内 → 节点黄色 ⚠ 警告（**告知不阻断** — 连线和运行照常）。合法值：sound_pressure / acceleration / velocity / displacement / voltage / current / force / strain / temperature / rpm。
- **emits_quantity**：改变量纲且**方向固定**的节点声明（如固定"加速度→速度"）。方向是运行参数的节点（如积分/微分方向可选）不用静态声明 — 在 runtime 里改写输出 `metadata.channels` 的 quantity/unit（参考官方 signal_math(信号数学运算)的做法）。

### `automl`（object，可选；声明节点在 AutoML 中的角色）

让节点显式声明能否参与 AutoML 调参/评估。AutoML 配置向导按这些字段过滤节点。

```yaml
automl:
  tunable: true        # 节点的 params 可加入搜索空间
  evaluable: true      # 节点的输出可作 features 拿去训练判别函数
  labelable: false     # 节点的输出可作 label 来源（带 attributes 的数据源类节点）
```

| 字段 | 何时为 true |
|------|-------------|
| `tunable` | 节点有数值参数想调参（如阈值、窗长、平滑系数） |
| `evaluable` | 节点输出 `FeatureMatrix`（features 端口）能拿去训分类器 |
| `labelable` | 数据源节点 / `attach_attributes` / `filter_node` 等能提供"分组维度" |

不设 = 不参与 AutoML（默认）。

### `ui`（object，可选）

```yaml
ui:
  result_view: ui/Viewer.tsx    # 节点运行后点"查看结果"打开的自定义视图
  params_form: ui/ParamsForm.tsx # 自定义参数表单（覆盖 params_schema 的默认渲染）
  icon: ui/icon.svg             # 节点卡片图标
```

`ui/Viewer.tsx` 签名：
```tsx
export default function Viewer({ runId, nodeId, outputs, nodeParams }: {
  runId: string; nodeId: string; outputs: any[]; nodeParams: any
}) { ... }
```

## 常见坑

- `outputs` 是**数组**不是 map，写错会解析失败
- `inputs` 分 required/optional/hidden 三层，最外层必须这三个 key 之一
- `params_schema` 路径相对节点目录（即 `nodes/<key>/`），不是项目根
- `runtime.requirements` 不写也行，但有新依赖必须加（否则 venv 里缺包）
- `ui.result_view` 引用的文件必须在 `nodes/<key>/ui/` 下，前端 vite 扫描路径硬编码
