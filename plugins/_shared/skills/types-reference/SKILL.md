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
| `IndicatorData` | 分析输出（单标量指标） | `{indicator: "loudness", items: [{item_id, name, value, _provenance, ...}]}` |
| `FeatureMatrix` | 分析输出（多列特征矩阵） | `{columns: [...], labels: {key: 中文名}, rows: [{item_id, name, features: {col: val}}]}` |
| `AttributeTable` | 属性/标签表（从 items 提取字段、聚类标签等输出） | `{columns: [...], rows: [{item_id, <alias>: val, ...}], total, source_total}` |
| `BaselineStats` | 基线统计阈值（`zscore_anomaly` 训练模式输出，给异常检测当参考） | `{columns: [...], feature_direction: {col: "high"/"low"/"both"}, thresholds: {col: {mean, std, min, max, count, p5, p95, ...}}, sample_count}` |
| `AnomalyResult` | 异常检测结果（zscore_anomaly 输出） | `{summary: {total, ok_count, anomaly_count, feature_names, method, thresholds}, items: [{item_id, name, status: "ok"/"anomaly", severity, n_triggers, triggers, features}], views: {distribution, spectrum_ref, audio_ref}}` |
| `AnnotationLayer` | 段落标注层（有效段检测等输出） | `{items: [{item_id, segments: [{start, end, label}]}]}` |
| `FileBlob` | 单个文件（CSV 导出、报告等） | Handle 自带 `uri / hash / size / mime` |
| `Table` | 通用二维表 | `{columns, rows}` 风格 |
| `ItemList` | 通用 items 列表（轻量，不要求音频语义） | `{items: [...], total}` |
| `Json` / `Number` / `String` / `Bool` | 标量/任意 JSON 端口（参数透传、小配置块、可选引用 handle 等） | 对应原生 JSON 值 |

## 超类型兼容规则

**两条超类型链**：

1. 消费端 `AudioData` 接受：`MaterializedDataset` / `ProcessedDataset` / `AudioData`
   - 适用：`level_meter`、`fft_spectrum` 等分析节点 `inputs.data: AudioData` —— 不管上游是直接物化的还是切分后的都能接
   - `AudioData` 是虚拟超类型，**没有节点真的输出 AudioData**。输出端应该用具体类型（大多数是 `ProcessedDataset`）

2. 消费端 `FeatureMatrix` 接受：`IndicatorData` / `FeatureMatrix`
   - 适用：`feature_merge` `port_type: FeatureMatrix` 同时接两类特征源；`score_predictor.features` 也能接
   - 单标量指标（IndicatorData）视为一列的特征矩阵自动兼容

## FeatureMatrix 的 labels 字段（中英文解耦）

`FeatureMatrix` 输出结构：

\`\`\`json
{
  "columns": ["value", "vs_freq_peak_freq", "vert_strength_mean"],
  "labels":  { "value": "响度", "vs_freq_peak_freq": "Bark 谱-峰值频率", "vert_strength_mean": "竖向能量均值" },
  "rows": [
    { "item_id": "001", "name": "...", "features": { "value": 31.8, "vs_freq_peak_freq": 12.4, ... } }
  ]
}
\`\`\`

**铁律**：
- `columns` / `features` dict 的 key **永远用英文标识符**（机器内部用，对接 AutoML / 评分预测要求稳定）
- `labels` 字段是可选的，做"英文 key → 中文显示名"映射；前端 chart_viewer / AutoML 诊断 / 评分器公式视图都会用它显示中文
- SDK 的 `FeatureBuilder(labels=FEATURE_LABELS)` 一行配置自动产出 labels 字段，**新节点必须传**

## 怎么选输入类型

| 场景 | 建议类型 |
|---|---|
| 要直接读音频文件做 DSP | `AudioData`（最宽容） |
| 要读已处理过的音频（如 active_segment 切完的段） | `ProcessedDataset` |
| 接收单值指标做二次分析 | `IndicatorData` |
| 接收多列特征矩阵（或聚合 / 评分预测 / 通用消费） | `FeatureMatrix`（自动也接 IndicatorData） |
| 接收基线阈值做异常判定 | `BaselineStats`（如 zscore_anomaly.baseline） |
| 接收属性/标签表做染色、分组、二次筛选 | `AttributeTable`（如 cluster_explore / matrix_view） |
| 接收标注层做后续处理 | `AnnotationLayer` |
| 接收原始 items 做字段提取（不要求音频语义） | `Any`（如 attribute_extract.dataset，兼容任何上游） |
| 做通用透传（合并、复制、筛选等） | `Any` |

## 怎么选输出类型

- **分析节点（单标量指标，如声级、平均响度）** → `IndicatorData`
- **分析节点（多维特征矩阵，如结构张量 13 维特征 / 多种统计量）** → `FeatureMatrix`（必须传 `labels=FEATURE_LABELS`）
- **转换节点**（音频→音频）→ `ProcessedDataset`
- **分割/选择节点** → `ProcessedDataset`（保持音频语义）
- **标注检测** → `AnnotationLayer`
- **属性提取 / 聚类标签**（从 items 抽字段或打标签，输出 `{columns, rows}`）→ `AttributeTable`（如 `attribute_extract.table` / `cluster_explore.clusters`）
- **基线统计**（算出每列特征的 mean/std/分位阈值，供异常检测当参考）→ `BaselineStats`（如 `zscore_anomaly.baseline`，训练模式的产出）
- **异常检测**（拿特征 + 基线判 ok/anomaly，带 severity 与分布视图）→ `AnomalyResult`（如 `zscore_anomaly.result`）
- **导出节点**（CSV/报告） → `FileBlob`
- **items + attributes 风格**（如 `score_predictor.scored`）→ 类型 `Any`，结构 `{items:[{item_id, attributes:{score, predicted}, features:{...}}], labels:{...}}` —— chart_viewer 自适应解析

> 分析链路常见接法：`分析节点→FeatureMatrix` → `zscore_anomaly` →`AnomalyResult`；`attribute_extract`（Any→`AttributeTable`）的标签可喂给 `cluster_explore` / `matrix_view` 做染色分组。

> **`zscore_anomaly` 是 fit/apply 双模式节点**（v2.0.0 起，吸收了已删除的 `baseline_stats`）：
> 不接 `baseline` 输入 = **训练**，用当前这批数据建基线并从 `baseline` 端口发出来；
> 接了 `baseline` 输入 = **推理**，套已有基线判 ok/anomaly。所以不再需要"先 baseline_stats 再 zscore"两个节点。
> 这也是平台的**可训练节点范式** —— 你自己写训练类节点时照这个来：同一个节点，
> 有制品输入就 apply、没有就 fit，别拆成两个节点。

## 动态端口

有些节点（如 feature_merge / dataset_merge）需要可变数量的输入。在 `node.yaml` 里声明：

```yaml
dynamic_inputs:
  enabled: true
  prefix: in       # 端口 key 前缀，最终是 in_1, in_2, ...
  label: "输入"
  port_type: FeatureMatrix  # 推荐：兼容 IndicatorData + FeatureMatrix 两类特征源
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
  "types": ["Any", "Dataset", "MaterializedDataset", "ProcessedDataset", "AudioData", "IndicatorData",
            "ItemList", "FileBlob", "Table", "FeatureMatrix", "BaselineStats", "AnomalyResult",
            "AttributeTable", "AnnotationLayer", "Number", "String", "Bool", "Json"],
  "super_types_doc": {
    "AudioData": "超类型：...",
    "Any": "通配：..."
  }
}
```
