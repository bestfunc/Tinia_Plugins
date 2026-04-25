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

```python
raw = rt.fetch_blob(rt.task["inputs"]["data"])
ds = json.loads(raw)  # 大多数 handle 存的是 JSON
for item in ds["items"]:
    # 处理单个 item
    audio_bytes = rt.fetch_content_url(item["content_url"])  # 或 item.blob_uri
```

### `rt.upload_blob(data: bytes, mime: str = "application/octet-stream") -> dict`

上传 bytes 到 blob store，返回新 handle（含 uri / hash / size / mime）。

```python
import json
out = {"type": "IndicatorData", "indicators": [{"name": "SPL_A", "value": 72.5, "unit": "dBA"}]}
handle = rt.upload_blob(json.dumps(out).encode(), "application/json")
rt.emit_output("result", handle)
```

### `rt.fetch_content_url(url: str) -> bytes`

直接从 URL 下载（item.content_url 经常是 MinIO 预签名，直接 GET 就行）。不走 blob store 的 hash 校验。

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
3. **handle 和 bytes 混用** → `fetch_blob(handle)` 要传 handle dict；不要传 bytes
4. **stdin 不要乱 print** → 所有 stdout 输出必须是 runner 约定格式的 JSON 事件；调试信息用 `emit_log` 或 stderr
5. **大数据别全塞 handle 的 JSON** → items 里引用 `blob_uri`，真实字节存 blob store
