---
name: sdk-reference
display_name: Tinia SDK 参考
description: tinia_runtime（Runtime / ChunkRuntime）完整 API 参考 —— 节点 run.py 必用；附 tinia_sdk（外部程序调用平台）速查
user-invocable: false
---

# Tinia SDK 参考（`tinia_runtime`）

> **本 skill 讲的是节点内 `run.py` 用的 `tinia_runtime`。**
> 如果你要写的是**外部 Python 程序调用 Tinia 平台**（把数据喂给平台上调好的节点/流程、拿回结果，或开实时流式会话），那是另一条线 `tinia_sdk`（`connect()` / `run_node` / `open_stream` …）—— 见文末「另一条线：`tinia_sdk`」一节。两者不要混。

## 两个 Runtime 入口（V2 流式架构）

`tinia_runtime` 提供两个入口，节点按自己的 manifest 选一个，**入口选错会立即报错**：

| 入口 | 何时用 | 读输入 | 写输出 |
|------|--------|--------|--------|
| `Runtime` | 默认 batch；不声明 streaming 的简单节点 | `fetch_blob(handle)` 一次性拉 | `upload_blob` + `emit_output` |
| `ChunkRuntime` | **新分析/处理节点推荐**；manifest 声明 `streaming.can_stream_downstream=true` 的节点必用 | `iter_input(port)` 边收 chunks 边 yield | `open_output(port).write_item()` + `close()` |

`ChunkRuntime` 是 `Runtime` 的子类：基础 API（fetch_blob / upload_blob / emit_* / extract_provenance / pick_device …）全继承，只多了 `iter_input` / `open_output` 这套流式 I/O。**新写节点优先用 `ChunkRuntime`** —— 即便下游不 streaming，它也会退化成等价 V1 行为（一次性拉 final blob、close 时一次性上传），节点代码一份通吃，没有 streaming 损失。

### ChunkRuntime 标准模板（与 create-node skill 一致）

```python
import json
from tinia_runtime import ChunkRuntime

def main():
    rt = ChunkRuntime.from_stdin()
    p = rt.task.get("params") or {}

    # 打开输出端口（header 放所有 item 共享的元数据）
    out = rt.open_output("result", header={"indicator": "...", "unit": "dB"},
                         node_type="Any", inherit_total_from="data")

    idx = 0
    for items_chunk in rt.iter_input("data"):      # 边收边处理
        for item in items_chunk:
            out.write_item(process(item))          # 边产边写
            idx += 1
            total = rt.upstream_total("data")
            rt.emit_progress(min(idx / total, 1.0) if total else 0,
                             f"{idx}/{total} 项" if total else f"已处理 {idx} 项")

    out.close()        # flush + 上传完整 final blob + emit_output（V2 时同时 fan-out done）
    rt.emit_done()

if __name__ == "__main__":
    main()
```

Runtime 由 Tinia 的 Python runner 注入 stdin（JSON 任务协议）+ 临时 token，供节点拉/推 blob、调 Tinia 内部端点用。

### 常驻执行池 / serve 模式对节点作者透明

平台有「常驻执行池（HotPool）」让 worker 进程常驻、省掉每次 fork+import numpy/scipy 的开销。**这对节点作者完全透明：你不需要写任何 serve 适配代码。**

- ✅ 保持标准写法即可：
  ```python
  if __name__ == "__main__":
      main()
  ```
- ❌ **不要**再写 `if os.environ.get("TINIA_SERVE")=="1": serve(main) else: main()` —— 这种历史写法**已废弃**。常驻池由平台的 `_serve_launcher.py` 透明驱动（它按路径 import 节点模块、取 `main()` 交给内部 `serve()` 循环），那时 `__name__ != "__main__"`，那段分支本来也不会执行，写不写都一样。

`from_stdin()` 每个 task 重新解析 stdin（凭据/token 不跨 task 残留），所以同一份 `main()` 既能一次性执行，也能进池循环，零额外代码。

## 基础 API（`Runtime` / `ChunkRuntime` 共有）

### `Runtime.from_stdin()` / `ChunkRuntime.from_stdin()`

从 stdin 读任务 JSON 并构造 Runtime。**入口必调**。按节点是否 streaming 选对应的类。

任务 JSON 结构（节选）：
```json
{
  "graph_run_id": 123,
  "node_id": "n5",
  "node_type": "bestfunc/level_meter",
  "mode": "v1",
  "inputs": {
    "data": {"uri": "minio://bucket/blobs/xx/HASH", "hash": "...", "type": "ProcessedDataset", "size": 1234567}
  },
  "params": {"threshold": 65, "weighting": "A"},
  "fetch_token": "短期 token",
  "fetch_url": "...", "upload_url": "..."
}
```

### `rt.task: dict`

原始任务字典。最常用：
- `rt.task["inputs"][port_key]` → 上游输入的 Handle（dict）
- `rt.task["params"]` → 用户配置的参数（对应 params.schema.json）
- `rt.task["node_id"]` / `rt.task["graph_run_id"]`

> `Runtime` 的低层字段是 `rt.token`（= task 的 `fetch_token`）/ `rt.fetch_url` / `rt.upload_url`，由 SDK 内部用，节点一般不直接碰。

## Blob 读写

### `rt.fetch_blob(handle) -> bytes`

从 handle 下载数据（自动走 Tinia 的 blob store，小文件内存、大文件流式）。返回 bytes。

**handle 是 dict**（含 `uri` / `hash` / `size` / `mime`），**不是裸 string**。
直接传 `rt.task["inputs"][port_key]` 给它就行。

```python
# ✅ 正确
raw = rt.fetch_blob(rt.task["inputs"]["data"])
ds = json.loads(raw)

# ❌ 错误：传裸 uri 字符串
raw = rt.fetch_blob(some_item["local_uri"])  # 报错 "handle 缺少 hash 字段"
```

### 处理 MaterializedDataset 的 item（音频文件等）

dataset/materialize 节点输出的 `items[]` 里每条字段是 **`local_uri`**（minio://），
**不是** `content_url`。处理方法二选一：

**多通道节点（推荐）**：用 `AudioInput.iter_channels(rt, item)` —— 自动按 manifest channels_mode 展开通道。详见下面的 `tinia_audio_input.AudioInput` 章节。

**单声道老节点**：用 `tinia_audio.load_audio(rt, item)` —— 多通道自动 mean down，方便快速写：

```python
from tinia_audio import load_audio

raw = rt.fetch_blob(rt.task["inputs"]["data"])
ds = json.loads(raw)
for item in ds["items"]:
    audio, sr = load_audio(rt, item)   # numpy float32, sample_rate（多通道已 mix down）
    # ... 用 audio / sr 做你的分析
```

**手动**：item 里有 `local_uri` 字段时按 handle dict 拼好后传 fetch_blob（`fetch_blob` 会从 uri 末段提取 hash，所以 hash 缺省也能用）：

```python
for item in ds["items"]:
    handle = {"uri": item["local_uri"], "hash": item.get("hash", ""),
              "size": item.get("size", 0)}
    audio_bytes = rt.fetch_blob(handle)
```

> 写新音频分析节点时**优先 `AudioInput.iter_channels`**（多通道感知）；只关心混合信号或纯单声道场景才用 `load_audio`。两者都比手写 fetch + scipy.io.wavfile 解码安全。

### `rt.upload_blob(data, node_type="", content_type="application/octet-stream") -> dict`

上传 bytes 到 blob store，返回新 handle（含 uri / hash / size 等）。

⚠ **签名注意：第二个位置参数是 `node_type`，不是 mime。** mime 在第三参 `content_type`，**务必关键字传**：

```python
import json
out = {"indicators": [{"name": "SPL_A", "value": 72.5, "unit": "dBA"}]}

# ✅ 正确：node_type 走第二位，mime 用关键字 content_type=
handle = rt.upload_blob(json.dumps(out).encode(), node_type="IndicatorData",
                        content_type="application/json")
rt.emit_output("result", handle)

# ❌ 错误：把 mime 当第二位置参数 —— 它会被当成 node_type，前端类型识别全乱
handle = rt.upload_blob(json.dumps(out).encode(), "application/json")
```

- `node_type`：输出端口的数据类型（IndicatorData / FeatureMatrix / ProcessedDataset / Any …），写进 blob 的 X-Type 头供前端 viewer + 类型校验用。
- `content_type`：HTTP MIME（绝大多数情况是 `application/json`）。

> **`Runtime`（batch 节点）**用 `upload_blob` + `emit_output` 一次性出结果。
> **`ChunkRuntime`（流式节点）**不直接调 upload_blob —— 用 `open_output().write_item()` + `close()`，由 `ChunkEmitter` 在 close 时内部调 `upload_blob(..., node_type=...)` 上传完整 final blob。下面是流式 I/O 详解。

## 流式 I/O（`ChunkRuntime`）—— 新节点首选

### `rt.iter_input(port) -> Iterator[list[dict]]`

流式读上游 port，**一段一段** yield `list[dict]`（一个 chunk 的 items）。无论 server 是 V1 还是 V2 调度，**节点代码同一份**：

```python
for items_chunk in rt.iter_input("data"):
    for item in items_chunk:
        out.write_item(process(item))
```

- V1 调度 / 该 port 不在 streaming 链路上：内部 `fetch_blob` 拉完整 final blob，整段当 1 个 chunk yield（等价老 `json.loads(fetch_blob)`）。
- V2 调度 + 该 port 是 streaming 输入：long-poll chunks 端点，上游边推边收。
- 上游失败 → 抛 `UpstreamFailedError`；上游 done → 正常结束循环。
- 没有该输入 → 空迭代（不报错）。

> 它会自动适配 `items` / `rows` 两种顶层字段（FeatureMatrix 用 rows，其余用 items），节点不用自己判断。

### `rt.upstream_total(port) -> int`

拿上游某 port 的预期总 item 数，给进度条当分母。**要在 `iter_input` 拿到第一个 chunk 之后才有值**（未知时返回 0，节点应显示「已处理 N 项」无分母）：

```python
total = rt.upstream_total("data")
rt.emit_progress(min(idx / total, 1.0) if total else 0,
                 f"{idx}/{total} 项" if total else f"已处理 {idx} 项")
```

### `rt.open_output(port, header=None, node_type="Any", inherit_total_from=None) -> ChunkEmitter`

打开输出端口，拿到一个 `ChunkEmitter`。节点用 `write_item` 一行行写，`close` 时上传完整 final blob。

- `header`：放所有 items 共享的元数据（indicator / unit / fraction …），不在 per-item 里重复。`total` 字段由 SDK 在 close 时自动回填。
- `node_type`：输出类型；FeatureMatrix 时 final blob 顶层字段是 `rows`，其余是 `items`（SDK 自动选，与 FeatureBuilder.build() / 主仓 Go 端一致）。
- `inherit_total_from`：1:1 节点便利 —— 第一个 chunk push 时自动用 `upstream_total(该 port)` 当本端口预期 total，下游进度条就有分母。非 1:1 节点别用（要么处理完调 `set_total(n)`，要么不调）。

### `ChunkEmitter` 方法

```python
out = rt.open_output("result", header={"indicator": "octave_A", "unit": "dB", "fraction": 3},
                     node_type="Any", inherit_total_from="data")
for items_chunk in rt.iter_input("data"):
    for item in items_chunk:
        out.write_item({                       # 累积进 final blob；V2 时攒满 chunk_size 自动 push 下游
            "item_id": ch.label,
            "value": value,
            "vs_freq": {"freqs": valid_centers, "values": [...]},
        })
out.close()        # flush 末段 chunk + 上传完整 final blob + emit_output("result", handle)
```

- `out.write_item(item)`：写一个 item。永远累积到 final blob；V2 streaming 输出端口同时攒到 `chunk_size` 自动 push 给下游。
- `out.set_total(n)`：声明本端口预期总 item 数（**必须在第一次 write_item 之前**调）。
- `out.flush_chunk()`：按业务边界手动 flush 当前 buffer（可选）。
- `out.close()`：flush 末段 + 上传完整 final blob（内部走 `upload_blob(..., node_type=...)`）+ `emit_output`。**每个 open 的端口必须 close**。
- 支持 `with rt.open_output(...) as out:` —— 正常退出自动 close；异常时（V2）通知 server 该端口上游失败，让下游 raise。

> final blob 的字节跟 V1 节点 `json.dumps({...header, "items":[...], "total":N})` **完全一致** —— V1/V2 cache_key 互通。chunks 只是链路加速副产物，不入库、不改输出语义。

### `tinia_audio` helper（老 API — 单声道场景）

`from tinia_audio import load_audio, compute_stat, downsample_list`

封装了：
- `load_audio(rt, item) -> (numpy.ndarray, sample_rate)` — 自动处理 local_uri/content_url 兼容 + WAV 解码 + **多通道自动 mean(axis=1)** 单声道
- `compute_stat(values, stat)` — rms / mean / std / peak / kurtosis / skewness 等单 channel 统计
- `downsample_list(arr, max_len=2000)` — 给 viewer 展示用的均匀降采样

⚠️ **多通道注意**：`load_audio` 默认 mix-down 单声道。要按通道独立分析（NVH 双耳响度 / 多麦阵列等），用下面的 `AudioInput` 抽象。

### `tinia_audio_input.AudioInput` —— 多通道感知输入（v1.11+，**新节点首选**）

```python
from tinia_audio_input import AudioInput

for ch in AudioInput.iter_channels(rt, item):
    # ch.samples 1D ndarray, ch.sr int
    # ch.name / ch.unit / ch.calibration_db / ch.index / ch.total_channels
    spectrum = compute_fft(ch.samples, ch.sr)
    results.append({
        "item_id": ch.label,            # 自动加通道后缀（如 _Mic_FL），单通道源不加
        "name": ch.display_name,        # 给视图层（如 "rec_001 [Mic_FL]"）
        "value": ...,
        "_meta": ch.to_meta_dict(),     # 自动展开 channel_label/source_channel/source_n_ch/...
    })
```

**展开行为由节点 yaml 的 `channels_mode` 决定**（详见 node-yaml skill）：
- `per_channel`：N 通道 → N 次 iter，输出 N 个独立 item（分析节点推荐）
- `mix_down`：自动 mean，1 个 item
- `first_only` / `requires_single`（**不声明时的缺省** — fail-fast）/ `multichannel_aware`：见 node-yaml

**ChannelInput v2 字段**（通道语义 v2 起，物理量锚点）：

| 字段 | 含义 |
|------|------|
| `ch.quantity` | 物理量类型（sound_pressure / acceleration / velocity / ...；空 = 未声明）|
| `ch.db_reference` | dB 计算参考值（按 quantity 自动派生 ISO 1683 标准值；0 = 无标定）|
| `ch.is_calibrated` | 信号是否具备绝对物理量级（已应用灵敏度/标定，或源文件本身是物理量）|
| `ch.unit` | 物理单位（"Pa" / "m/s²" / ...）|
| `ch.bit_depth` | 源文件位深（wav 16/24/32；0 = 未知）|

**输出 dB 的节点统一写法**（viewer 据此诚实标轴 "dB SPL" vs "dB (rel.)"）：

```python
ref = ch.db_reference or 1.0           # 无标定 → 相对 dB(ref=1)
level = 20 * np.log10(rms / ref)
# _meta 用 ch.to_meta_dict() 自动带出 db_reference / calibrated 供 viewer 标轴
```

**注意 samples 已是物理量**：value_kind=physical 的源（振动 CSV/TDMS 等）SDK 直通不归一化；
raw_voltage 的源按模板灵敏度换算。节点**不要再自行缩放**。

**关键收益**：
1. **不用写 mean(axis=1) 样板** —— SDK 收口，节点只关心算法
2. **ChannelMeta 自动穿透** —— 上游 channel_split / 数据源命名模板设的 calibration_db / unit 自动到 ChannelInput
3. **下游 indicator_viewer 自动多曲线** —— 输出 N items 直接画 N 条带 legend

**心理声学 / level_meter 类节点**用 ChannelMeta.calibration_db 优先于 params 的写法：

```python
effective_cal = ch.calibration_db if ch.calibration_db else params.get("calibration_db", 0)
```

依赖：声明 `numpy` 和 `scipy` 在 `runtime/requirements.txt`。tinia_audio / tinia_audio_input 走 PYTHONPATH 不用列。

### `tinia_features.FeatureBuilder` —— 产出 FeatureMatrix（v1.13+，分析节点必用）

输出多列特征时用这个收集器，**自动**做以下事：
- 按 item_id 归并多通道结果到同行（一个 item 不同通道展开成多列）
- 输出 `columns` 列表（按字典序）
- 输出 `labels`（英文 key → 中文显示名，给前端 chart_viewer / AutoML 显示用）
- 输出 `feature_direction`（low/high/both，给异常检测节点用）
- 自动透传 `_provenance` 字段（数据溯源）

```python
from tinia_features import FeatureBuilder, stats_from_series, TIME_STATS_LABELS

# 节点顶部常量
FEATURE_LABELS = {
    "value": "响度",
    "value_overall": "总响度",
    **TIME_STATS_LABELS,  # 提供 vs_time_mean/max/p95 等常见统计量的中文
}

# run.py 末尾
fb = FeatureBuilder(
    labels=FEATURE_LABELS,                     # 必传，否则 chart_viewer 显示英文 key
    direction={"value": "low", "vs_time_max": "low"},  # 可选，异常检测节点用
)
for items_chunk in rt.iter_input("data"):
    for src_item in items_chunk:
        for ch in AudioInput.iter_channels(rt, src_item):
            features = {
                "value": compute_loudness(ch.samples, ch.sr),
                **stats_from_series(vs_time, prefix="vs_time_"),
            }
            fb.add(
                source_item_id=src_item["item_id"],
                channel_label=ch.channel_short,     # ⚠️ 必须 channel_short，不能用 ch.label
                features=features,
                provenance=Runtime.extract_provenance(src_item),
                name=src_item.get("name"),
            )

# fb 内部已做完聚合，一次性上传。features 端口流式化收益有限，
# 走 upload_blob + emit_output（node_type 用 FeatureMatrix）即可。
h = rt.upload_blob(json.dumps(fb.build()).encode(), node_type="FeatureMatrix")
rt.emit_output("features", h)
```

> `FeatureBuilder` **没有** `build_streaming` 方法 —— 它只有 `add()` 和 `build()`。features 端口的输出是 `upload_blob(json.dumps(fb.build()).encode(), node_type="FeatureMatrix")` + `emit_output("features", h)`。即便节点主输出端口用 `ChunkRuntime` 流式，features 端口仍这么写（见 level_meter run.py 实例）。

**铁律 — `channel_label` 必须用 `ch.channel_short`**：
- `ch.channel_short`：单通道返回 `""`，多通道返回 `"ch0"` / `"ch1"` —— 这才是设计意图
- `ch.label`：单通道会返回 item_id 本身 —— 错用会让 features 列变成 `<item_id>.<feature>`，每个样本一组独立列，AutoML 训练时矩阵爆炸 + 对角线非零异常

**输出 dict 结构**（`fb.build()`）：
```python
{
    "columns": ["energy", "value", "vs_time_max", ...],
    "labels": {"energy": "能量", "value": "响度", "vs_time_max": "时序-最大", ...},
    "feature_direction": {"value": "low", ...},
    "rows": [
        {"item_id": "001", "name": "...", "features": {"energy": 0.5, "value": 31.8, ...}, "_provenance": {...}}
    ],
    "total": N
}
```

## 错误分级（ConfigError + emit_warning）

平台约定（错误分级，主仓 channel-semantics 设计 §11.6）：**配置类错误告知用户但不阻断运行**。

```python
from tinia_runtime import ConfigError

config_errs = 0
last_config_err = None
for item in items:
    try:
        for ch in AudioInput.iter_channels(rt, item):
            ...
    except ConfigError as e:    # 配置类(缺采样率/通道模式不符) — SDK reader 抛的就是它
        config_errs += 1
        last_config_err = e
        rt.emit_log("warn", f"{item_id}: {e}")
    except Exception as e:      # 数据类(单文件损坏) — skip 即可
        rt.emit_log("warn", f"{item_id}: {e}")

# 收尾三件套：
if idx > 0 and ok_count == 0:
    raise RuntimeError(f"全部 {idx} 项处理失败,最后错误: {last_err}")   # 0 结果必须失败
if config_errs > 0:
    rt.emit_warning(f"{config_errs}/{idx} 项因配置问题被跳过: {last_config_err}")
```

- `ConfigError(ValueError)`：同一配置下重复 N 个 item 都会失败的错误。自己的节点遇到"参数/配置不对"也应抛它（继承 ValueError，向后兼容）。
- `rt.emit_warning(message)`：与 emit_log("warn") 的区别 — warning 写进运行记录，run 完成后前端在节点上显示黄色 ⚠ 角标（"完成了但你应该知道"）。可多次调用，server 换行累积。
- 全部失败仍要 raise — "0 结果假成功"比失败更糟。

## 进度 & 输出事件

每个方法都会写一行 JSON 到 stdout（runner 监听并 relay 到前端/DB）。

### `rt.emit_progress(ratio: float, message: str = "")`

0.0 ~ 1.0 进度。节点卡片边框会按这个填充。

**message 必须简短**（建议 ≤ 25 个字符）—— 它会显示在节点卡片底部一行，前端 max-width 260px。
长字符串虽然会被 truncate，但完全不显示有用信息更糟。

✅ 好：`"处理 4/9"` / `"FFT 计算中"` / `"训练 epoch 12/50"`
❌ 差：`f"处理 {i}/{total}: {item['name']}"` —— item.name 是文件名，常常 80+ 字符
       会变成 `"处理 4/9: HPFAN-0415-…"` 用户看不到关键的进度数字

### `rt.emit_output(port_key: str, handle: dict)`

**必须对每个 output 端口调一次**。handle 来自 `rt.upload_blob`。缺一个端口下游会没法连。

### `rt.emit_done()`

**必须最后调**。不然 runner 认为进程异常终止。

### `rt.emit_log(level: str, text: str)`

打日志到 Tinia 节点运行记录（level: info / warn / error）。用户在 Web UI 运行详情里能看到。

## 错误处理

### 捕获异常，emit_error

```python
try:
    # ... 业务 ...
    rt.emit_output("result", handle)
    rt.emit_done()
except Exception as e:
    rt.emit_error(str(e))  # 会让节点状态变 failed，下游 skip
    raise  # 可选，让 runner 看到非零退出码
```

直接抛异常也会被 runner 捕获，但**建议显式 `emit_error`**，错误信息才会显示在 Web UI 上。

## GPU helper（manifest 声明 `gpu` 的节点用）

- `rt.pick_device(prefer="auto") -> str`：自动探测返回 `"cuda"` / `"mps"` / `"cpu"`（CUDA → Apple MPS → CPU；无 torch 直接 cpu）。会自己 emit_log 报告选中 device，节点不必再打日志。`prefer="cpu"` 强制 CPU。
- `rt.require_gpu(prefer="auto") -> str`：`gpu: required` 节点用 —— 拿不到 GPU 直接抛清晰中文 `ConfigError` 终止。
- `rt.gpu_call(kernel, params, timeout=180.0) -> dict | None`：调 daemon 的共享 GPU sidecar（torch 集中在 sidecar，节点自身 venv 不装 torch）。sidecar 未就绪/出错统一返回 `None`，让调用方本地 CPU 回退（绝不抛错中断节点）。

## 数据源凭据（数据源插件节点用）

### `rt.get_datasource() -> dict | None`

节点参数含 `datasource_id` 时，Go 端自动查 DB 解密凭据注入到 task。节点用这个方法拿到：

```python
ds = rt.get_datasource()
if ds:
    cred = ds["credential"]      # {"host": ..., "project": ..., "client_id": ..., "client_secret": ...}
    cfg = ds["config"]           # 数据源配置（如 default_directory）
```

返回结构含 `id` / `name` / `type` / `config` / `credential_type` / `credential`；节点没有 datasource_id 参数时返回 `None`。**这就是数据源插件访问自己凭据的唯一机制 —— 没有 `rt.api_token` / `rt.server_url` / `plugin-db/query` 这些东西**（历史文档里有，已不存在）。

## 常见坑

1. **忘记 emit_done** → 节点永远 running
2. **emit_output 的 port_key 和 node.yaml 不一致** → 下游连不到（`ChunkRuntime` 用 `open_output(port)` 的 port 同理）
3. **handle 和 bytes 混用** → `fetch_blob(handle)` 要传 **handle dict**（来自 `rt.task["inputs"][port]` 或 `rt.upload_blob` 返回值）；不要传 `item["local_uri"]` 这种裸 string
4. **upload_blob 第二参是 node_type 不是 mime** → mime 必须关键字传 `content_type=`
5. **音频处理别自己解码** → 用 `AudioInput.iter_channels` 或 `tinia_audio.load_audio(rt, item)`；自己写 fetch_blob + scipy.io.wavfile.read 容易踩 local_uri 兼容性坑
6. **别写 serve 适配代码** → 保持 `if __name__=="__main__": main()`；常驻池透明接管，`if TINIA_SERVE==1: serve(main)` 已废弃
7. **stdin 不要乱 print** → 所有 stdout 输出必须是 runner 约定格式的 JSON 事件；调试信息用 `emit_log` 或 stderr
8. **大数据别全塞 handle 的 JSON** → 用 `ChunkRuntime`（`iter_input` + `open_output().write_item()`）边收边写边产，避免 `results.append(...) + json.dumps(...)` 累积模式的内存翻倍

---

## 另一条线：`tinia_sdk`（外部程序调用平台）

> ⚠ 上面整篇讲的 `tinia_runtime` 是**节点内 `run.py`** 用的。如果你写的是**外部 Python 程序**，要把数据交给 Tinia server 上已调好的节点/流程执行、同步拿回结果，用的是**另一个包 `tinia_sdk`**，API 完全不同。

`tinia_sdk` 从平台「SDK 管理」生成，授权凭据 + server 地址打包在内，零配置：

```python
import tinia_sdk
tinia = tinia_sdk.connect()                          # 地址/凭据自动读取

# 跑单个节点（数据支持 文件路径 / numpy 数组 / 已上传句柄）
result = tinia.run_node("bestfunc/time_stats", "test.wav", params={"frame_ms": 0})
print(result.json("result"))

# 用平台「复制参数」拿到的 TNP1 预设串跑（串里自带节点类型）
result = tinia.run_preset("TNP1:eyJ0eXBlIjoi...", "test.wav")

# 跑整个流程（流程里放一个「API 输入」+ 一个「API 输出」节点）
result = tinia.run_flow(graph_id=12, input=data)
```

主要 API：`connect()` / `TiniaClient.run_node` / `run_preset` / `run_flow` / `upload` / `upload_signal`。

### 流式会话（实时数据流，v1.35）—— 只在 `tinia_sdk` 这侧

```python
with tinia.open_stream(graph_id=1, sr=51200) as s:
    s.push(frame_array)                 # 推一帧/一窗（numpy 1D 单通道 / 2D 通道×采样点），返回帧序号
    for res in s.recv_iter():           # 持续 yield 结果 item 直到会话收尾
        dba = res.get("dba")
# 离开 with 自动 close；也可手动 s.recv(timeout=30) -> (items, done) / s.close() / s.keepalive()
```

`StreamSession`（由 `open_stream` 创建）：`push` / `recv` / `recv_iter` / `close` / `keepalive`。内部 long-poll + token 临近过期自动续签，适配 7×24 长会话。

> **流式会话是「外部喂实时数据流给平台流程」的能力，属于 `tinia_sdk`。** 节点内 `run.py` 的 `ChunkRuntime` 流式 I/O（`iter_input` / `open_output`）是平台内部链路加速，**两者不是一回事，不要混淆**。节点要参与流式会话只需保持标准写法（如 level_meter 用 `params["_stream_continuous"]` 跨窗延续状态），平台透明驱动。

**去哪查 `tinia_sdk` 细节**：`Tinia/server/sdk/client-python/`（`README.md` + `tinia_sdk/client.py`）。异常类型：`NodeError` / `LicenseRevokedError` / `AuthError` / `TiniaTimeoutError` / `TiniaError`。
