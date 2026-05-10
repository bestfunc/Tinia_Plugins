---
name: filter-design
display_name: 设计滤波流程
description: 帮用户设计 NVH / 声学滤波流水线 — 选 iir_filter / fir_filter / weighting_filter 三个节点的合适组合，给出场景化参数。覆盖去工频、抗混叠、人耳计权、频段隔离等典型需求
user-invocable: true
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__datasource_list,mcp__tinia__datasource_describe,mcp__tinia__flow_create,mcp__tinia__flow_list,mcp__tinia__flow_describe,mcp__tinia__flow_open,mcp__tinia__flow_batch_edit,mcp__tinia__flow_add_node,mcp__tinia__flow_set_node_params,mcp__tinia__flow_connect,mcp__tinia__flow_replace_node,mcp__tinia__flow_auto_layout,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_run_status,mcp__tinia__flow_node_output_preview,mcp__tinia__flow_node_logs
---

# filter-design —— 设计滤波流程

## 何时用

- 用户说 "做个滤波 / 滤一下噪声 / 滤工频 / 滤高频" 等
- 用户提到 "A 计权 / dBA / 计权" 等声学专有词
- 用户说 "去掉 XX Hz / 保留 XX Hz / 频段隔离"
- 用户描述 NVH 流水线（频率分析、心理声学、噪声评价）需要前置滤波
- 已建好流程但出现 "信号有工频干扰 / 高频噪声 / 没计权" 等问题，要插入滤波节点

## 三个滤波节点（决策树）

```
┌─ 用户要做的事 ────────────┐    ┌─ 选哪个节点 ─────────────────────┐
│ 通用过滤（lp/hp/bp/bs）   │ →  │ iir_filter（默认推荐 ⭐）         │
│ 严格线性相位需求          │ →  │ fir_filter                       │
│ A/C/Z 计权（声学）        │ →  │ weighting_filter                 │
└──────────────────────────┘    └──────────────────────────────────┘
```

### iir_filter（首选 — 90% 滤波场景）

通用 IIR 滤波。lp / hp / bp / bs 四种类型 × Butter / Bessel / Cheby1 / Cheby2 / Elliptic 五种设计法 × 1–12 阶。**带 zero_phase（filtfilt 双向滤波）选项消除相位失真**。

| 场景 | 推荐参数 |
|---|---|
| 去工频 50/60Hz | `mode=highpass, design=butterworth, order=4, cutoff_high=50`（或 60）|
| 抗混叠 | `mode=lowpass, design=butterworth, order=8, cutoff_low=0.4*sr` |
| 语音频段 | `mode=bandpass, design=butterworth, order=4, cutoff_low=300, cutoff_high=3400` |
| 振动主能量 | `mode=lowpass, design=butterworth, order=4, cutoff_low=1000` |
| 陷波（去单频干扰）| `mode=bandstop, design=butterworth, order=4, cutoff_low=48, cutoff_high=52`（50Hz 周围）|
| 陡峭过渡 | `design=elliptic, order=8, ripple_db=0.5, attenuation_db=80` |
| 离线分析（消相位失真）| 任意配置 + `zero_phase=true` |

### fir_filter（精确相位需求）

严格线性相位 FIR。代价：要更多 taps（通常 257–1023）才能达到 IIR 4–8 阶的过渡陡度。

**何时强烈推荐 fir_filter 而非 iir_filter**：
- 心理声学预处理（loudness / sharpness 之前）
- 音质评价（任何对时域波形敏感）
- 用户说 "保持波形 / 不要相位失真"

| 场景 | 推荐参数 |
|---|---|
| 去工频（精确）| `mode=highpass, taps=511, window=hamming, cutoff_high=50` |
| 抗混叠（精确）| `mode=lowpass, taps=257, window=hamming, cutoff_low=8000` |
| 心理声学带通 | `mode=bandpass, taps=1023, window=kaiser, kaiser_beta=8, ...` |
| 极陡阻带 | `taps=2049, window=blackmanharris` |

> **默认勾选 `compensate_delay`**：FIR 自带 (taps-1)/2 群延迟，补偿后输出时间戳跟输入对齐。**关闭它仅当下游节点要求严格因果性**（罕见）。

### weighting_filter（声学专属）

IEC 61672 标准 A / C / Z / B 计权。**几乎只在声学评价流水线里用**。

| 场景 | 推荐参数 |
|---|---|
| 噪声评价 / dBA 测量 / 设备声功率 | `weighting=A` ⭐（默认）|
| 高声压 / 冲击 / 爆破（dBC）| `weighting=C` |
| 不需计权但要 metadata 链路 | `weighting=Z`（passthrough）|
| 复测旧数据（IEC 已淘汰）| `weighting=B`（仅向后兼容）|

> sr ≥ 48 kHz 时 8 kHz 内是 IEC Class 1；关注 16 kHz 内容请用 sr ≥ 96 kHz。

## 标准 NVH 流水线（推荐起手式）

```
原始 WAV (MaterializedDataset)
  ↓
iir_filter (highpass 20Hz, butter 4)         去 DC 漂移 + 工频低端
  ↓
iir_filter (lowpass 0.4×sr, butter 8)        抗混叠
  ↓
weighting_filter (A)                         模拟人耳响应（dBA 链路）
  ↓
分析节点（level_meter / loudness / fft_spectrum / active_segment / ...）
```

**节点天然串联** —— 输入输出都是 `AudioData`，不需要中间转换。每经过一级，输出 NPZ 的 `metadata.filters_applied` 数组追加自身条目，下游节点 / Viewer 可读取显示。

## 标准动作链

```
1. nodes_describe iir_filter weighting_filter fir_filter
   ↑ 如果不熟参数，先 describe 一次拿到精确字段名

2. 跟用户确认场景 — 关键问题：
   - 数据采样率？（决定 cutoff 上限 + 计权精度）
   - 想要的频段范围？（决定 mode + cutoff）
   - 需要严格相位吗？（IIR + zero_phase / FIR / IIR 单向）
   - 需要 dBA / dBC 吗？（决定要不要加 weighting_filter）

3. flow_batch_edit 一次性加节点 + 连线
   - 优先用 batch（事务性，失败回滚）而不是逐个 add
   - 节点连线顺序：data_source → iir_filter (HPF) → iir_filter (LPF) → weighting_filter → 分析节点

4. flow_run + flow_wait_run

5. flow_node_output_preview 看每一级输出
   - 检查滤波后的样本是否在合理范围（不应被 NaN / Inf）
   - 检查 RMS 减小了多少（高通去掉 DC 后 RMS 通常稍降）
```

## 决策细则（AI 必看）

### 选 IIR 还是 FIR？

```
默认选 IIR。
仅当用户明确说 "保持波形 / 心理声学 / 不能相位失真" 才用 FIR。
要消相位失真又想要 IIR 的速度 → IIR + zero_phase=true。
```

### 阶数 / Taps 怎么定？

```
IIR：默认 order=4。要更陡过渡用 6-8。要极陡用 elliptic + order=8。
FIR：默认 taps=257。要更陡用 511/1023。极陡用 2049+ blackmanharris。
```

### 何时启用 zero_phase（IIR）？

```
默认 false（实时友好）。
离线分析 + 用户关心时间戳对齐 / 段落起止 → true。
零相位等效阶数翻倍，order=4 实际相当于 8 阶衰减。
```

### Bessel 何时用？

```
IIR + 要保留波形形状（最线性相位）→ Bessel。
但 Bessel 的过渡很缓，要陡过渡且要相位 → 选 FIR 更合适。
```

## 常见用户需求 → 节点配方

| 用户描述 | 节点配方 |
|---|---|
| "去掉 50Hz 工频" | `iir_filter`（hp 50Hz butter 4）或 `iir_filter`（bandstop 48-52Hz） |
| "去掉低频噪声" | `iir_filter`（hp，cutoff 看用户场景，振动 5-20Hz / 语音 80-100Hz） |
| "去掉高频噪声" | `iir_filter`（lp，cutoff 看场景） |
| "做个 A 计权" | `weighting_filter`（A） |
| "保留语音频段" | `iir_filter`（bp 300-3400Hz butter 4） |
| "我想测 dBA" | `weighting_filter`（A）→ `level_meter` |
| "相位不能变" | `fir_filter` 或 `iir_filter + zero_phase=true` |
| "提取共振峰" | `iir_filter`（窄带 bp，比如 cutoff_low=900 cutoff_high=1100） |
| "分析前先滤掉直流" | `iir_filter`（hp 1-5Hz butter 2） |

## 测试与验证

跑完滤波后**必看的检查**：

1. **flow_node_output_preview <iir_filter_node_id>** —— 看 NPZ samples 是否合理
2. **接 fft_spectrum 看频谱** —— 确认通带保留、阻带衰减（最直观）
3. **看 metadata.filters_applied** —— 通过 nodes_describe 输出 / dashboard 等能看到滤波链记录

### 用 signal_generator 严格验证（推荐）

如果要**精确验证滤波器频响 / 衰减是否符合理论**（不只是肉眼看），用 `signal_generator` 节点合成已知特征信号作为输入：

```
signal_generator (chirp 20-20000Hz log) → <被测滤波节点> → fft_spectrum → spectrum_viewer
```

或最快的冲激法：

```
signal_generator (impulse, amp=1.0) → <被测滤波节点> → fft_spectrum
                                            ↑ spectrum 直接 = 滤波器完整频响 H(f)
```

完整测试方法（4 种 stimulus / 5 种模板 / 失败排查）见专用 skill：**`/skill:node-test`**。

## 边界情况告知

如果遇到下面情况，**主动告诉用户**：

- **截止频率 ≥ Nyquist**（采样率/2）→ 节点会跳过该 item，要么降低 cutoff，要么提高 SR
- **bandpass / bandstop 的 cutoff_low ≥ cutoff_high** → 设计失败，修正参数
- **zero_phase=true 但信号太短**（< ~3 × order）→ 节点自动降级单向 + warning，告诉用户考虑减小 order
- **FIR taps > 信号长度** → 跳过该 item，要么减小 taps 要么用更长信号
- **B 计权（已淘汰）** → 提醒用户 IEC 61672 已弃用，除非复测旧数据否则用 A 或 C

## 与其他 skill 的关系

- 跟 `flow-author` 互补：flow-author 教搭整个流程，filter-design 专治"流程里的滤波环节"。如果用户场景已经在 flow-author 范畴里走了，但卡在"加什么滤波"，调本 skill。
- 跟 `node-test` 互补：filter-design 帮用户**搭**滤波链（业务用途）；node-test 帮用户**测**滤波节点对不对（开发期 / 验证期）。用户说"测下这个滤波器" → 跳到 `/skill:node-test`。
- 跟 `result-view` / `params-form`：那两个是开发新节点用的，本 skill 是教 AI 用现有 3 个滤波节点。
