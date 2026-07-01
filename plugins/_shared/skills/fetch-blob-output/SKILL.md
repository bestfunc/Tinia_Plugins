---
name: fetch-blob-output
display_name: 拉取 blob 输出并解码导出（fetch_blob → npz/矩阵 → CSV）
description: 节点输出里 FeatureMatrix / 频谱矩阵端口的真实数值在内层 blob（numpy .npz）里，flow_node_output_preview 只给「信封」（每行的 local_uri）。本 skill 教怎么用 fetch_blob 把 local_uri 拉到本机、np.load 解码、导出 CSV。用户说「导出某节点矩阵/频谱数据」「把 mod_matrix / depth_matrix 存成 csv」「拿 blob 里的原始数值」时用。
user-invocable: false
allowed-tools: mcp__tinia__flow_describe,mcp__tinia__flow_node_output_preview,mcp__tinia__flow_runs_list,mcp__tinia-file__fetch_blob,Bash
---

# 拉取 blob 输出并解码导出

## 背景：数值不在 preview 里，在内层 blob 里

`flow_node_output_preview` 对 **FeatureMatrix / 二维矩阵端口**（如 `modulation_spectrum`
的 `mod_matrix` / `depth_matrix`，`spectrum_viewer` 的频谱矩阵等）只返回**信封**：
每行是 `{ item_id, name, local_uri, hash, size, metadata }`，**真实的数值矩阵在
`local_uri` 指向的内层 blob 里**，preview 不会解引用它。

这些矩阵 blob 是 `np.savez_compressed(...)` 出来的 **numpy `.npz`**（zip），
常见数组键：
- `fbank` —— 主矩阵，`(n_frames/n_bands, n_features/n_modfreq)`，多为 dB 域
- 频率轴/频带轴数组，如 `mod_freqs`、`band_centers`（键名随节点而定，`z.files` 列出来看）

> 有的端口（如 `modulation_spectrum` 的 `result`）把谱线**内联**在 JSON 里（`vs_freq.values`），
> 那种直接从 preview `mode=full` 就能拿，不需要本流程。**只有值落在 `local_uri` blob 时才用 fetch_blob。**

## 标准流程

### 1. 定位 run + 端口 + 要导出的 item

```
flow_describe(flow_id, fields=["nodes","last_run"])           # 拿 last_run.id + 节点 class_type
flow_node_output_preview(run_id, node_id, mode="summary")     # 看端口、行数、每行 local_uri + metadata
```

在 summary 里记下目标端口某行的 `local_uri`（形如
`minio://tinia-blobs/blobs/70/5c0b...`）和它的 `metadata`（`shape` / `unit` /
`n_modfreq` 等，导出表头要用）。

### 2. 用 fetch_blob 把内层 blob 拉到本机

```
fetch_blob(blob_uri="minio://tinia-blobs/blobs/70/5c0b...")
```

返回：
```json
{ "format": "npz", "size_bytes": 875, "sha256": "705c...",
  "saved_path": "/tmp/tinia-blob-705c....npz",
  "format_hint": "numpy .npz…" }
```
- `saved_path` 是落到本机磁盘的文件（默认系统临时目录，AI context 不过大数据）。
- `format` 已嗅探好（npz / npy / wav / parquet / json）。
- 想指定落点：`save_path="/Users/.../Downloads/xxx.npz"`。

### 3. 用 Python 解码 + 导出 CSV

```python
import io, csv, os, numpy as np
z = np.load(SAVED_PATH)          # fetch_blob 返回的 saved_path
print(z.files)                   # 先看有哪些数组
fbank = z["fbank"]               # 主矩阵 (T, F)，多为 dB
# 频率轴键名看 z.files（modulation_spectrum 是 mod_freqs / band_centers）
xaxis = z["mod_freqs"] if "mod_freqs" in z.files else np.arange(fbank.shape[1])

out = os.path.expanduser("~/Downloads/节点_端口.csv")
with open(out, "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["x"] + [f"row_{i}" for i in range(fbank.shape[0])])
    for j in range(fbank.shape[1]):
        w.writerow([round(float(xaxis[j]), 4)] +
                   [round(float(fbank[i, j]), 3) for i in range(fbank.shape[0])])
print("written", out)
```

导出到**用户下载目录**用 `os.path.expanduser("~/Downloads/...")`。

## modulation_spectrum 具体示例（mod_matrix 端口）

`mod_matrix` 内层 npz：`fbank`=`(n_bands, n_modfreq)` dB 矩阵，`band_centers`=频带中心(Hz)，
`mod_freqs`=调制频率轴(Hz)。单频带时 `fbank` 是 `(1, 35)`，导出即「调制频率 → dB」一列。
`depth_matrix` 端口结构同形，`fbank` 值是调制深度%（不是 dB）。

## 边界与排错

- **只支持 MinIO 后端**（start-dev.sh 的 docker MinIO / 自建 MinIO）。桌面单机版 blob 存在
  本地文件系统，不是 `minio://` scheme，fetch_blob 会明确报错 —— 那种情况改从节点 preview
  的内联字段拿，或让用户在有 MinIO 的 dev 环境跑。
- MinIO 连接参数来自 env（`TINIA_MINIO_ENDPOINT` / `_ACCESS_KEY` / `_SECRET_KEY` /
  `_BUCKET`），缺省 `localhost:9000` / `minioadmin` / `tinia-blobs`。连不上先确认 docker
  MinIO 在跑（`docker ps | grep minio`，S3 API 端口 9000）。
- `HTTP 403 SignatureDoesNotMatch` → access/secret key 不对或时钟偏差大。
- `HTTP 404 NoSuchKey` → uri 抄错，或该 run 的 blob 已被 GC/清理。
- 返回的 `sha256` 应与 `local_uri` 末段 hash 一致（内容寻址），不一致说明取错对象。
```
