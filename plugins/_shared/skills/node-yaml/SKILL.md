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
  label: "指标"            # 端口显示名模板
  port_type: IndicatorData # 所有动态端口的类型
  min_ports: 2
  max_ports: 10           # 0 = 无限制
```

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
