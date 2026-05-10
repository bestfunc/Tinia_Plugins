---
name: datasource-test
display_name: 测试数据源 + 通道命名模板
description: 端到端测试 Tinia 数据源功能 — 创建数据源、生成多通道 wav 上传、创建通道命名模板、应用模板到数据源、用流程节点验证 ChannelMeta 全链路传播。覆盖 Phase B 数据源页 UI + Phase C 通道命名模板系统。AI 在本机用 numpy/scipy 生成测试 wav 后 base64 上传，不依赖外部资源。
user-invocable: true
allowed-tools: mcp__tinia__datasource_list,mcp__tinia__datasource_describe,mcp__tinia__datasource_create,mcp__tinia__datasource_update,mcp__tinia__datasource_delete,mcp__tinia__datasource_upload_files,mcp__tinia__datasource_list_files,mcp__tinia__datasource_delete_file,mcp__tinia__channel_template_list,mcp__tinia__channel_template_create,mcp__tinia__channel_template_apply,mcp__tinia__channel_template_delete,mcp__tinia__flow_create,mcp__tinia__flow_batch_edit,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_node_output_preview,Bash
---

# datasource-test —— 测试数据源 + 通道命名模板

## 何时用

- 用户说"测一下数据源 / 上传 / 通道命名模板"
- 验证 Phase B（数据源页 UI 重构）/ Phase C（通道命名模板系统）功能
- 端到端测试 ChannelMeta 全链路（datasource → 节点 → 分析输出 _meta.channel_label）
- 写回归测试（CRUD + 模板应用 + 删除级联）

## 核心流程

```
1. datasource_create("test_datasource_xxx")  ← 建测试数据源
        ↓
2. AI 在 Bash 里用 Python 生成 wav 字节（numpy + scipy.io.wavfile）
        ↓
3. base64 编码 → datasource_upload_files
        ↓
4. channel_template_create({n_channels: N, channels: [...]})  ← 建模板
        ↓
5. channel_template_apply(datasource_id, template_id)
        ↓
6. datasource_describe → 验证 channel_template_id 已设
        ↓
7. （可选）flow_create + 跑 fft → 看 _meta.channel_label 是否对
        ↓
8. cleanup：datasource_delete + channel_template_delete
```

## 生成测试 wav 的标准 Python（必看）

AI 用 Bash 跑 Python，**不要写到磁盘**，直接 base64 输出。这是模板代码：

```python
import io, base64, numpy as np
from scipy.io import wavfile

# 参数
sr = 48000
duration_s = 2.0
freq = 1000.0
amp = 0.5
n_channels = 5

# 生成单通道 sine
t = np.arange(int(duration_s * sr)) / sr
mono = (amp * np.sin(2 * np.pi * freq * t) * 32767).astype(np.int16)

# 多通道：所有通道相同（也可每通道不同 freq 模拟阵列）
if n_channels == 1:
    samples = mono
else:
    samples = np.tile(mono[:, np.newaxis], (1, n_channels))  # WAV 是 (n_samples, n_ch)

# 写到 BytesIO 不落盘
buf = io.BytesIO()
wavfile.write(buf, sr, samples)
content_base64 = base64.b64encode(buf.getvalue()).decode()
print(content_base64)
```

调 Bash：

```bash
python3 -c "<上面的代码>" | tr -d '\n'
```

把输出捕获到变量给 `datasource_upload_files`。

## 测试模板

### 模板 1：最简链路（双通道命名穿透）

```
1. ds_id = datasource_create("test_dual_ch")
2. 生成 2 通道 sine 1k @ 60dB SPL：
   sr=48000, n_channels=2, freq=1000, amp=0.063
3. datasource_upload_files(ds_id, [{filename:"stereo.wav", content_base64: "..."}])
4. tpl_id = channel_template_create({
     name: "test_dual_LR",
     n_channels: 2,
     channels: [
       {name:"Mic_L", unit:"Pa", calibration_db:94, color:"#3b82f6"},
       {name:"Mic_R", unit:"Pa", calibration_db:94, color:"#ef4444"}
     ]
   })
5. channel_template_apply(ds_id, tpl_id)
6. datasource_describe(ds_id) → 验证 channel_template_id == tpl_id
7. cleanup: datasource_delete(ds_id), channel_template_delete(tpl_id)
```

判定：5 步全部 OK + 第 6 步返回的 channel_template_id 等于 tpl_id。

### 模板 2：5 通道驾驶舱完整流程（NVH 工业场景）

```
1. ds_id = datasource_create("driving_2026_test")
2. 用 Bash + Python 生成 5 通道 wav：
   - ch0/1: sine 1kHz amp=0.063（模拟双麦）
   - ch2/3/4: white_noise amp=0.01（模拟三轴加速度）
   - sr=48000, duration=3s
3. datasource_upload_files
4. tpl_id = channel_template_create({
     name: "驾驶舱 5 通道",
     n_channels: 5,
     channels: [
       {name:"Mic_FL", unit:"Pa", calibration_db:94},
       {name:"Mic_FR", unit:"Pa", calibration_db:94},
       {name:"Accel_X", unit:"g", calibration_db:100},
       {name:"Accel_Y", unit:"g", calibration_db:100},
       {name:"Accel_Z", unit:"g", calibration_db:100}
     ],
     is_org_default: true
   })
5. channel_template_apply
6. （可选）建 graph：dataset_node(ds_id) → fft_spectrum → indicator_viewer
   验证 fft 输出 5 items，_meta.channel_label = Mic_FL/Mic_FR/Accel_X/...
7. cleanup
```

判定：fft 输出 5 items + 每 item 的 _meta.channel_label 是业务名（不是 ch0/ch1/...）+ source_n_ch=5。

### 模板 3：CRUD 完整路径 + 删除级联

```
1. ds_id = datasource_create("temp_ds")
2. 上传 1 个 wav
3. datasource_list_files → 应有 1 个文件
4. tpl_id = channel_template_create(...)
5. channel_template_apply
6. datasource_update(ds_id, name="renamed_ds") → 重命名
7. datasource_describe → 验证 name="renamed_ds" + channel_template_id=tpl_id
8. channel_template_delete(tpl_id)
9. datasource_describe(ds_id) → 验证 channel_template_id=null（FK ON DELETE SET NULL 自动清空）
10. datasource_delete(ds_id)
11. datasource_list → 应不在列表
```

判定：每步状态符合预期 + 删除模板后 datasource 自动 unset。

### 模板 4：权限测试（org 隔离）

仅当超管或多 org 环境可测：

```
1. 用户 A（org=1）建模板 M_A
2. channel_template_list（用户 A）→ 应见 M_A
3. 切换用户 B（org=2）
4. channel_template_list（用户 B）→ 应**不见** M_A（org 私有）
5. 用户 B channel_template_apply(ds_b, M_A.id) → 应失败"无权使用模板"
```

## 失败排查

| 现象 | 可能原因 |
|------|---------|
| `datasource_upload_files` 返回 skipped > 0 | base64 字符串里有换行 / 空格 → 用 `tr -d '\n'` 清理 |
| 文件大小 = 30 字节但是合法 wav | 可能 numpy stack 维度搞反，应该 (n_samples, n_ch) 不是 (n_ch, n_samples) |
| `channel_template_create` 报 "channels 长度必须等于 n_channels" | 数组长度跟 n_channels 不一致，检查 |
| `datasource_describe` 没看到 channel_template_id | 当前 describe 只在 v1.23+ 后端返回这个字段，旧版可能没 |
| 应用模板后 fft 输出仍是 ch0/ch1（不是业务名）| dataset_node 可能还没改造（Phase D 范围）— ChannelMeta 当前在 datasource 层有，但还没传到节点输出。先验 datasource 自身有 channel_template_id 就算 PASS |

## 决策细则（AI 必看）

### 文件大小限制

`datasource_upload_files` 走 MCP request body，单次建议总 < 50MB。大文件用前端 UI 拖拽（浏览器多 part 上传）。WAV 估算：
- 单声道 48k 1秒 ≈ 96KB
- 5 通道 48k 3秒 ≈ 1.4MB
- 5 通道 48k 30秒 ≈ 14MB

测试场景几乎不会超 5MB。

### 上传后是否要等？

`datasource_upload_files` 是同步的，返回时已写入 blob + DB。立即可用。

### channel_template_create 的 is_org_default

`is_org_default=true` 时，同 org 同 n_channels 的其他 default 会被自动 unset（一个 default 唯一）。多次创建 default 不会冲突，但旧的会失效。

### 测试用名前缀（避免污染）

测试创建的 datasource / template 名建议加前缀 `test_` 或 `_temp_`，方便事后批量清理。

### Cleanup 顺序

删除顺序：先 datasource 后 template（因为 datasource.channel_template_id 是 FK，模板被删后 datasource 字段自动 SET NULL；反过来 datasource 没删时模板可删，但留个孤儿引用）。

## 与其他 skill 的关系

- **node-test** 是测节点；**datasource-test** 是测数据源 / 模板
- **flow-author** 搭流程；本 skill 不直接搭流程（只准备数据），需要测端到端时调 flow_create 串起来
- 测多通道分析（A4 范畴）应该用 signal_generator 节点（更快，不用上传）；本 skill 主要测**数据源管理本身**和**模板系统**
