---
name: 创建 Tinia 节点
description: 在现有 Tinia 插件项目里添加一个 Python 节点，生成骨架并实现 run.py
user-invocable: true
allowed-tools: mcp__tinia__dev_list_projects, mcp__tinia__dev_list_nodes, mcp__tinia__dev_create_node, mcp__tinia__dev_read_file, mcp__tinia__dev_write_file, mcp__tinia__dev_reload, mcp__tinia__nodes_list_types
---

# 创建 Tinia 节点

## 流程

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

返回 `status: ok, node_count: N` 即成功。失败就看 error 字段，或走「调试节点运行错误」skill。

成功后告诉用户：
> 已注册到你的个人命名空间 `{namespace}/{key}`。打开 Tinia Web UI 流程编辑器，节点面板里会看到一个带 DEV 标签的条目。拖到画布上连线，点 Run 看结果。

## 约束提醒

- 节点 key 一旦建好，目录名、node.yaml 里的 key、full_key 都绑定。**改 key 不等于改名，等于新建一个节点**。要改名的话请删旧的建新的
- `run.py` 必须以 `rt = Runtime.from_stdin()` 开头，以 `rt.emit_done()` 结束
- 每个输出端口 **必须 emit_output 一次**；漏一个会让下游连不到
- 未接的 optional 输入在 run.py 里要判空：`rt.task["inputs"].get("mask")`
- stdout 只能出 runtime 约定的事件 JSON —— 调试用 `rt.emit_log()` 或 `print(..., file=sys.stderr)`

## 相关 Skill

- 端口类型选型 → 「Tinia 类型体系」
- SDK 方法详情 → 「Tinia SDK 参考」
- node.yaml 字段 → 「node.yaml 字段速查」
- 测试失败 → 「调试节点运行错误」
