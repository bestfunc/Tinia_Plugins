---
name: 修改节点端口与参数
description: 给现有节点增/改/删端口或参数时的兼容性注意事项
user-invocable: true
allowed-tools: mcp__tinia__dev_read_file, mcp__tinia__dev_write_file, mcp__tinia__dev_reload, mcp__tinia__nodes_describe
---

# 修改节点端口与参数

修改端口 / 参数会影响**所有已存在的引用该节点的分析流程**。在改之前先和用户对齐影响面。

## 改什么兼容性如何

| 改动 | 影响 |
|---|---|
| **加一个 optional 输入** | ✅ 兼容，旧流程不受影响 |
| **加一个 required 输入** | ⚠️ 旧流程会启动失败（required 没接） |
| **删一个 optional 输入** | ✅ 兼容；已连的线会变灰但不报错 |
| **删一个 required 输入** | ⚠️ 旧流程能跑但逻辑可能错 |
| **改输入 type**（放宽，如 AudioData → Any） | ✅ 兼容 |
| **改输入 type**（收窄，如 Any → AudioData） | ❌ 原来连了 Dataset 的会失配 |
| **加一个输出** | ✅ 兼容 |
| **删一个输出** | ❌ 下游引用这个输出的会断 |
| **改输出 type** | ❌ 下游类型校验会失败 |
| **改端口 key**（从 data 改成 input） | ❌ **和删了等效**，所有旧引用失败 |
| **加一个参数** | ✅ 兼容（用默认值填充） |
| **删一个参数** | ✅ 兼容（run.py 记得处理 None） |
| **改参数 key** | ❌ 旧流程里填的值丢失 |
| **改参数 type** | ⚠️ 数值 ↔ 字符串要迁移 |

## 流程

### 1. 先了解影响面

`nodes_describe(key=当前节点的 full_key)` 看当前 inputs/outputs 的 schema。
问用户："这个节点是否已经被分析流程用过？如果用过，下面的改动可能让 N 个已存在的流程失配。"

### 2. 保持兼容的改法

**加输入**：用 optional 不用 required
```yaml
inputs:
  required:
    data:
      type: AudioData
  optional:        # 新加的放这里
    mask:
      type: AnnotationLayer
      label: "段落遮罩（可选）"
```
run.py 里判空：
```python
mask_handle = rt.task["inputs"].get("mask")
if mask_handle:
    mask = json.loads(rt.fetch_blob(mask_handle))
```

**加参数**：给默认值
```json
{
  "new_param": {
    "type": "number",
    "title": "新参数",
    "default": 0   // ← 必须有 default，旧流程的 params 没这个字段也能工作
  }
}
```
run.py：`params.get("new_param", 0)`

**加输出**：直接加，下游没连的话没影响
```yaml
outputs:
  - key: result
    type: IndicatorData
  - key: debug_info   # 新加
    type: FileBlob
```
run.py 里也记得 emit_output 新端口（不 emit 会让下游"没东西能连"，但不会让流程挂）。

### 3. 破坏性改动（不兼容）

如果必须做破坏性改动（如改 key / 删端口 / 改类型）：

1. **跟用户确认** —— 影响面明确
2. **让用户调 `dev_bump_version` 升版本**（哪怕是小改，也要让旧 graph 能区分）
3. 提示用户："升版后旧流程引用的是老版本节点，可能需要手工在流程编辑器里重新连线"

## 交互示例

**用户**：我想给 level_meter 加个 "权重" 参数。

你：
1. `dev_read_file(project_id, "nodes/level_meter/schemas/params.schema.json")` 看现有 params
2. 建议加在 properties 里，带 default：
   ```json
   "weighting": {
     "type": "string",
     "title": "加权方式",
     "enum": ["A", "C", "Z"],
     "default": "A"
   }
   ```
3. `dev_write_file` 写回
4. 改 run.py 读这个参数：`params.get("weighting", "A")`
5. `dev_reload` 测试

因为加了默认值，旧流程会用 "A" 默认值，**兼容**。

## 相关 Skill

- 类型选型 → 「Tinia 类型体系」
- 字段完整定义 → 「node.yaml 字段速查」
- 测试改动 → 用 `dev_reload` + 「调试节点运行错误」
