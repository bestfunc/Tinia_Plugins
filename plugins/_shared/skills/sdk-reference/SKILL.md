---
name: sdk-reference
display_name: Tinia SDK 参考
description: tinia_runtime.Runtime 所有方法的完整参考 —— run.py 里必须用的 API
user-invocable: false
---

# Tinia SDK 参考（`tinia_runtime.Runtime`）

所有 Python 节点的 `run.py` 都长这样：

```python
import json
from tinia_runtime import Runtime

rt = Runtime.from_stdin()
rt.emit_progress(0, "开始")

# ... 业务逻辑 ...

rt.emit_output("result", handle)
rt.emit_done()
```

Runtime 由 Tinia 的 Python runner 注入 stdin（JSON 任务协议）+ 签发临时 API token 供插件节点调 Tinia 内部 API 用。

## 基础 API

### `Runtime.from_stdin() -> Runtime`

从 stdin 读任务 JSON 并构造 Runtime。**入口必调**。

任务 JSON 结构：
```json
{
  "graph_run_id": 123,
  "node_id": "n5",
  "node_type": "bestfunc/level_meter",
  "inputs": {
    "data": {"uri": "s3://bucket/xxx", "hash": "...", "type": "ProcessedDataset", "size": 1234567}
  },
  "params": {"threshold": 65, "weighting": "A"},
  "api_token": "短期 JWT",
  "server_url": "https://tinia..."
}
```

### `rt.task: dict`

原始任务字典。最常用：
- `rt.task["inputs"][port_key]` → 上游输入的 Handle
- `rt.task["params"]` → 用户配置的参数（对应 params.schema.json）
- `rt.task["node_id"]` / `rt.task["graph_run_id"]`

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

**手动**：item 里有 `local_uri` 字段时按 handle dict 拼好后传 fetch_blob：

```python
for item in ds["items"]:
    handle = {"uri": item["local_uri"], "hash": item.get("hash", ""),
              "size": item.get("size", 0), "mime": "audio/wav"}
    audio_bytes = rt.fetch_blob(handle)
```

> 写新音频分析节点时**优先 `AudioInput.iter_channels`**（多通道感知）；只关心混合信号或纯单声道场景才用 `load_audio`。两者都比手写 fetch + scipy.io.wavfile 解码安全。

### `rt.upload_blob(data: bytes, mime: str = "application/octet-stream") -> dict`

上传 bytes 到 blob store，返回新 handle（含 uri / hash / size / mime）。

```python
import json
out = {"type": "IndicatorData", "indicators": [{"name": "SPL_A", "value": 72.5, "unit": "dBA"}]}
handle = rt.upload_blob(json.dumps(out).encode(), "application/json")
rt.emit_output("result", handle)
```

### `rt.fetch_content_url(url: str) -> bytes`

直接从 URL 下载（不走 blob store hash 校验）。**只有 item 里真有 `content_url` 字段时才用** ——
当前 dataset/materialize 节点的 item 主要给 `local_uri` 字段，应优先用 `tinia_audio.load_audio` 或
拼 handle 走 `fetch_blob`。`fetch_content_url` 主要给老插件 / 外部 URL 场景用。

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
- `per_channel`（默认）：N 通道 → N 次 iter，输出 N 个独立 item
- `mix_down`：自动 mean，1 个 item
- `first_only` / `requires_single` / `multichannel_aware`：见 node-yaml

**关键收益**：
1. **不用写 mean(axis=1) 样板** —— SDK 收口，节点只关心算法
2. **ChannelMeta 自动穿透** —— 上游 channel_split / 数据源命名模板设的 calibration_db / unit 自动到 ChannelInput
3. **下游 indicator_viewer 自动多曲线** —— 输出 N items 直接画 N 条带 legend

**心理声学 / level_meter 类节点**用 ChannelMeta.calibration_db 优先于 params 的写法：

```python
effective_cal = ch.calibration_db if ch.calibration_db else params.get("calibration_db", 0)
```

依赖：声明 `numpy` 和 `scipy` 在 `runtime/requirements.txt`。tinia_audio / tinia_audio_input 走 PYTHONPATH 不用列。

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

## 可选：调 Tinia 内部 API

### 数据源插件节点需要访问自己的凭证时

通过 `rt.api_token`（短期 JWT）+ `rt.server_url` 访问：
- `POST {server_url}/api/v1/plugin-db/query` — 查插件自己的表
- 其他 Tinia API（按需）

插件数据源（`datasource_plugin` 模板）用这个机制存自己的凭证和数据源配置。

## 常见坑

1. **忘记 emit_done** → 节点永远 running
2. **emit_output 的 port_key 和 node.yaml 不一致** → 下游连不到
3. **handle 和 bytes 混用** → `fetch_blob(handle)` 要传 **handle dict**（来自 `rt.task["inputs"][port]` 或 `rt.upload_blob` 返回值）；不要传 `item["local_uri"]` 这种裸 string
4. **音频处理别自己解码** → 用 `tinia_audio.load_audio(rt, item)` 一行搞定 fetch+WAV 解码；自己写 fetch_blob + scipy.io.wavfile.read 容易踩 local_uri vs content_url 兼容性坑
5. **改了 requirements.txt 别只 dev_reload 一次** → reload 会自动检测 mtime 重装；但首次跑节点才触发 Prepare，要么测试节点跑一次让它装完，要么观察 server log 出现 `[python-runner] pip install for ...` 才算装完
6. **stdin 不要乱 print** → 所有 stdout 输出必须是 runner 约定格式的 JSON 事件；调试信息用 `emit_log` 或 stderr
7. **大数据别全塞 handle 的 JSON** → items 里引用 `blob_uri` / `local_uri`，真实字节存 blob store
