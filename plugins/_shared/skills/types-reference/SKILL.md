---
name: types-reference
display_name: Tinia 类型体系
description: Tinia 节点图里的数据类型参考 — AudioData / IndicatorData / AnnotationLayer / Any / Dataset 等
user-invocable: false
---

# Tinia 类型体系

Tinia 节点之间传递的**不是数据本体**，而是 **Handle（blob 引用）+ 类型标记**。
类型系统保证端口连线的兼容性：目标端口声明 `AudioData`，源端口必须输出 `AudioData`（或其子类型）。

## 基础类型

| 类型 | 语义 | 典型 payload（JSON） |
|---|---|---|
| `Any` | 通配，任何类型都能连 | 任意 |
| `Dataset` | 数据列表（从数据源拉出来的 items） | `{items: [{id, name, content_url, media_type, ...}], total}` |
| `MaterializedDataset` | "已下载到 MinIO" 的数据集 | `{items: [{id, name, blob_uri, hash, duration_sec, ...}]}` |
| `ProcessedDataset` | 经过处理的数据集（切分 / 抽样后等） | 同 Materialized 结构 |
| `AudioData` | **超类型** —— 等价于"可用的音频数据"，兼容 MaterializedDataset / ProcessedDataset / AudioData | 下游统一按 items[].blob_uri 读 |
| `IndicatorData` | 分析输出（指标列表） | `{indicators: [{name, value, unit, ...}]}` |
| `AnnotationLayer` | 段落标注层（有效段检测等输出） | `{items: [{item_id, segments: [{start, end, label}]}]}` |
| `FileBlob` | 单个文件（CSV 导出、报告等） | Handle 自带 `uri / hash / size / mime` |

## 超类型兼容规则

当消费端声明为 **`AudioData`** 时，以下输出都可以连进来：
- `MaterializedDataset`
- `ProcessedDataset`
- `AudioData`

这就是为什么 `level_meter`、`fft_spectrum` 这些分析节点统一声明 `inputs.data: AudioData` —— 不管上游是直接物化的还是切分后的都能接。

**注意**：`AudioData` 是虚拟超类型，**没有节点真的输出 AudioData**。输出端应该用具体类型（大多数是 `ProcessedDataset`）。

## 怎么选输入类型

| 场景 | 建议类型 |
|---|---|
| 要直接读音频文件做 DSP | `AudioData`（最宽容） |
| 要读已处理过的音频（如 active_segment 切完的段） | `ProcessedDataset` |
| 接收另一个分析节点的输出做二次分析 | `IndicatorData` |
| 接收标注层做后续处理 | `AnnotationLayer` |
| 做通用透传（合并、复制、筛选等） | `Any` |

## 怎么选输出类型

- **分析节点**（产生指标）→ `IndicatorData`
- **转换节点**（音频→音频）→ `ProcessedDataset`
- **分割/选择节点** → `ProcessedDataset`（保持音频语义）
- **标注检测** → `AnnotationLayer`
- **导出节点**（CSV/报告） → `FileBlob`

## 动态端口

有些节点（如 feature_merge / dataset_merge）需要可变数量的输入。在 `node.yaml` 里声明：

```yaml
dynamic_inputs:
  enabled: true
  prefix: in       # 端口 key 前缀，最终是 in_1, in_2, ...
  label: "输入"
  port_type: IndicatorData
  min_ports: 2
  max_ports: 10
```

前端会让用户在节点卡片上 +/- 动态加端口。

## 查运行时真实结构

用 `nodes_describe` 看每个节点具体输出什么 JSON schema 其实不可靠（schema 描述的是参数，不是输出）。更可靠的办法是**让用户去 Web UI 跑一次，看流程快照里输出端口的 Schema 面板**——那是真实的 emit 结构。

## 超类型查表

直接调 `nodes_list_types` 得到：
```json
{
  "types": ["Any", "Dataset", "MaterializedDataset", "ProcessedDataset", "AudioData", "IndicatorData", ...],
  "super_types_doc": {
    "AudioData": "超类型：...",
    "Any": "通配：..."
  }
}
```
