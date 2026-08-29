---
name: node-test
display_name: 测试节点正确性
description: 用 signal_generator 合成已知特征的信号（chirp/impulse/sine/noise/真阶跃），自动搭测试图验证 DSP / 分析节点（滤波器、FFT、心理声学、声级计、段落检测、多通道分析等）的输出是否符合行业标准（IEC 61672 / ISO 532 / DIN 45692 / ECMA-418）。覆盖单通道精度、多通道独立分析、系统阶跃响应。开发新节点 / 改完代码立刻验证 / 写 regression test 都用这个。
user-invocable: true
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__flow_create,mcp__tinia__flow_list,mcp__tinia__flow_describe,mcp__tinia__flow_open,mcp__tinia__flow_batch_edit,mcp__tinia__flow_add_node,mcp__tinia__flow_set_node_params,mcp__tinia__flow_connect,mcp__tinia__flow_replace_node,mcp__tinia__flow_auto_layout,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_run_status,mcp__tinia__flow_node_output_preview,mcp__tinia__flow_node_logs,mcp__tinia__flow_run_cancel,mcp__tinia__flow_run_retry,mcp__tinia__flow_fork
---

# node-test —— 用合成信号测节点正确性

## 何时用

- "测一下 X 节点 / 验证 X 节点对不对"
- 用户刚改完节点代码（dev studio）想立刻验证
- 调用 `dev_reload` 装载新版本节点之后
- 怀疑某节点输出不对，想用对照信号定位
- 写 regression test（同输入永远同输出）

## 核心范式 —— AI 是"判断者"，节点只负责生成 + 计算

```
signal_generator (已知特征) → <被测节点> → 查看器 (输出指标)
                                               ↓
                                            AI 读输出 → 查 references/ → 判 PASS/FAIL → 写报告
```

**判断逻辑不进节点**。节点只生成数据 + 算指标，AI 在 skill + reference 文档的指引下做"期望值查表 → 容差对比 → 失败诊断"。
- 优点：测试可解释（AI 能说"为什么 FAIL、怎么修"）；可演化（加新节点不需要写新断言节点）；零运行时成本（不污染生产流程）。

## 参考资料（按需 Read）

测某类节点前，先 Read 对应 reference 拿"期望值表 + 测试方案 + 容差"。

| 节点类别 | reference 文件 |
|---|---|
| `weighting_filter` / `level_meter` / `octave_analysis (A/C)` / `fft_spectrum (dBA)` | `references/weighting-curves.md` |
| `fft_spectrum` / `octave_analysis` / `spectrum_smooth` | `references/fft-spectrum-rules.md` |
| `loudness` / `sharpness` / `roughness` / `tonality` / `tnr` | `references/psychoacoustic-standards.md` |
| `iir_filter` / `fir_filter` | 见下面"模板：测滤波器频响"（不需读 reference）|
| `active_segment` / `zscore_anomaly` | 见下面"段落 / 异常检测"|
| `indicator_math` / `indicator_merge` | 见下面"指标节点"|
| `channel_split` / `channel_select`（多通道）| 见下面"模板 8：多通道节点"|

**Read 的时机**：用户说"测 loudness" → 先 Read psychoacoustic-standards.md → 再搭流程。  
**别一次全 Read**：浪费 token。只读涉及的那一份。

## 行业 4 大测试法（适用所有 DSP 节点）

| 方法 | stimulus | 看什么 | 适用 |
|---|---|---|---|
| **冲激法** ⭐ | `impulse` (amp=1) | spectrum 直接 = 频响 H(f) | LTI 滤波器（最快）|
| **扫频法** ⭐ | `chirp` 20-20k log | spectrum 通带/阻带/过渡 | 频响曲线（最直观）|
| **点验法** | `sine` 单一频率 | level_meter / 数值峰值 | 验证特定频点 |
| **噪声法** | `white_noise` (固定 seed) | PSD 比 = 频响 | regression / 统计验证 |

## 标准动作链

```
1. nodes_describe <要测的节点> + nodes_describe signal_generator
   ↑ 拿到节点输入/输出类型 + 关键参数

2. 跟用户对齐：
   - 节点类型？（滤波器 / FFT / 心理声学 / level / 段落 ...）
   - 期望特征？（如 lowpass 1k 4 阶 → 1k 处 -3dB）
   - 用哪种测试法？（默认推荐：滤波器→冲激/扫频；分析节点→单 sine）

3. 按节点类别 Read references/X.md（如需要）拿期望值表

4. flow_batch_edit 一次性搭测试图（带 flow_auto_layout）

5. flow_run + flow_wait_run

6. flow_node_output_preview 各级看输出
   - 重点看末端查看器
   - 跟 reference 表对照
   - 给用户【AI 测试报告】(见下面"输出格式")

7. 失败排查：
   - flow_node_logs 看 stderr / traceback
   - 上游各节点 preview 看是否中途坏
   - 检查 stimulus 参数（如 chirp_end ≥ Nyquist 会混叠、calibration_db 没设）
```

## 测试模板

### 模板 1：滤波器频响（IIR / FIR）

```
signal_generator (chirp, 20-20000 Hz, log, 5s, sr=48k, amp=0.5)
  ↓
<被测滤波节点>
  ↓
fft_spectrum
  ↓
spectrum_viewer
```

**判定**：通带平坦、阻带衰减、过渡带斜率符合阶数（每阶 6 dB/oct）。

**或更快的冲激法**：

```
signal_generator (impulse, 2s, sr=48k, amp=1.0)
  ↓
<被测滤波节点>      ← ⚠ 仅 zero_phase=false（单向 sosfilt）能这样测
  ↓
fft_spectrum
  ↓
spectrum_viewer
```

冲激响应的 FFT **直接就是**频响 H(f)，最干净 —— **但不要在 zero_phase=true 时用**：

`scipy.signal.sosfiltfilt`（即 zero_phase）在两端做镜像反射 padding 近似零相位。
impulse 在 sample[0] 时反射后 sample[-1] 也 = 1，**两端各出一个伪冲激** → 频谱失真。

实测案例：4 阶 Butterworth LP fc=1k + zero_phase=true：
- 期望 fc=-6dB / 2k=-48dB（filtfilt 等效 8 阶）
- 实测 1k=-53dB / 2k=-59dB / 3k=-63dB → 滚降假象

测 zero_phase 频响**只能用 chirp 或 white_noise PSD 平均**（模板 1 的扫频法适用）。

### 模板 2：滤波器衰减点验

```
# 测 highpass 50Hz：5Hz sin 应被强衰减、1kHz sin 应几乎不变
signal_generator (sine, 5 Hz)    → <hp_filter> → level_meter   预期：< -40 dB
signal_generator (sine, 1000 Hz) → <hp_filter> → level_meter   预期：≈ 0 dB
```

### 模板 3：计权节点

→ Read `references/weighting-curves.md`，按那里的 chirp 法测试。

### 模板 4：FFT / 频谱节点

→ Read `references/fft-spectrum-rules.md`，按那里的 6 个测试场景。

### 模板 5：心理声学节点

→ Read `references/psychoacoustic-standards.md`。**关键**：必须设校准 —— 推荐**直接在 sg 上配 `channel_calibration_db="94"`**（校准跟数据走，所有下游分析节点自动用），分析节点 calibration_db params 留空。老方法（节点 params 配 calibration_db）仍可用，但不如 sg 配方便。

### 模板 6：段落 / 异常检测

```
# active_segment：用 burst 信号验证检出
signal_generator (silence, 1s) + signal_generator (sine 1k, 2s) + signal_generator (silence, 1s)
  → indicator_math (add) 或 audio_segment_split 拼接
  → level_meter   ← 输出 IndicatorData
  → active_segment (threshold=适中)
  → annotation 输出应有 1 段，时间戳 ≈ [1s, 3s]
```

```
# zscore_anomaly（v2.0.0 双模式）：一个节点跑两遍，训练一遍、推理一遍
# ① 训练：不接 baseline 输入 → 从 baseline 端口出基线制品
signal_generator (white_noise, seed=42)  → 提取特征 → zscore_anomaly       → baseline
# ② 推理：把①的 baseline 接进来 → 套基线判 ok/anomaly
signal_generator (sine, 5kHz amp=0.5)    → 提取特征 → zscore_anomaly + baseline → result
```

判定：silence 段不应被检出有效段；纯 sine 在 white noise 基线下 z-score 应远超阈值。

> 别再找 `baseline_stats` —— 它已经被并进 `zscore_anomaly` 的训练模式删掉了。
> **接不接 `baseline` 输入**就是 fit / apply 的开关。

### 模板 7：指标节点（indicator_math / indicator_merge）

```
# indicator_math add：两个 sine 频谱相加
signal_generator (sine 1k) → fft_spectrum → A
signal_generator (sine 2k) → fft_spectrum → B
indicator_math (A + B, mode=add) → 应该 1k 和 2k 都有峰
```

```
# indicator_math sub：原始 - 平滑 = 突出度（pyfar 风格 tone prominence）
signal_generator (sine 1k + white noise) → fft_spectrum → 原始
                                          → spectrum_smooth (1/3 oct) → 平滑
indicator_math (原始 - 平滑) → 1k 处应有显著正峰，其他频段 ≈ 0
```

判定：算术运算结果跟手算一致；item_id 匹配 + 频率轴插值正确。

### 模板 8：多通道节点（channel_split / channel_select）

✅ **现行规则（通道语义 v2，fail-fast）**：节点 manifest **不声明 `channels_mode` = `requires_single`** —— 多通道输入直接报错，不静默平均。多通道自动按通道展开成 N 个独立 item，是因为分析节点（fft_spectrum / loudness / level_meter / sharpness / tonality / tnr / octave_analysis / order_tracking / modulation_spectrum / roughness / time_stats 等）**显式声明了 `channels_mode: per_channel`**，不是默认行为。滤波器类（weighting_filter / iir_filter / fir_filter）声明 `multichannel_aware`（n 进 n 出，节点自己处理 (n_ch, n_samples)）。详见 node-yaml skill 的 `channels_mode` 表。

显式声明 `per_channel` 的分析节点，多通道源能直接接、自动按通道展开成 N 个 item，下游 indicator_viewer 自动多曲线 + 通道命名。

```
# 直接接：sg(channels=2) → fft → 2 条独立频谱
signal_generator (sine 1k, channels=2)
  → fft_spectrum
  → indicator_viewer  ← 自动 2 条频谱（rec_001 [ch0], rec_001 [ch1]）
```

判定：fft 输出 N items（item_id 加通道后缀，name 加 [ch0]）；每 item 的 _meta 含 `channel_label / source_channel / source_n_ch`。

```
# 心理声学也一样：sg(channels=2) → loudness → 2 个独立 sone
signal_generator (sine 1k, channels=2)
  → loudness
  ← 输出 2 个 item，每通道独立 sone 值
```

#### ChannelMeta 穿透（推荐写测试用）

sg 加了 `channel_names / channel_units / channel_calibration_db` 参数，让通道元信息从源头流到所有分析节点。下游的 _meta.channel_label 和 calibration_db 会用这些值，**不再是默认 ch0/0**。

```
# 双耳录音 + 校准穿透
signal_generator (sine 1k, channels=2,
  channel_names="Mic_L,Mic_R",
  channel_units="Pa,Pa",
  channel_calibration_db="94,94")
  → loudness   # ch.calibration_db=94 自动生效，节点 params.calibration_db 留空
  ← 输出 2 items：name=[Mic_L]/[Mic_R]，value 是真 sone（94 dB SPL 校准过）
```

```
# 多麦阵列模拟
signal_generator (sine 1k, channels=5,
  channel_names="Mic_FL,Mic_FR,Accel_X,Accel_Y,Accel_Z",
  channel_units="Pa,Pa,g,g,g",
  channel_calibration_db="94,94,100,100,100")
  → fft_spectrum
  ← 5 items，name 用业务名，_meta 含完整 channel_label/unit/calibration_db
```

判定：下游 _meta.channel_label 应是 sg 配的业务名（"Mic_L" 而非 "ch0"）；calibration_db 字段应等于 sg 配的值。

#### channel_split / channel_select 何时仍有用

虽然声明了 `per_channel` 的分析节点已经覆盖大部分场景，这两个节点在以下情况仍有用：

```
# channel_split：通道分到不同分支独立处理（不同 channel 走不同链路）
sg (channels=2) → channel_split → ch0 → lowpass 1k → fft_spectrum
                                → ch1 → highpass 1k → fft_spectrum
                                → indicator_merge 左右对比
```

```
# channel_select：取一个通道喂给 channels_mode=requires_single 的节点
sg (channels=4) → channel_select (channels=R) → <严格单声道 only 的节点>
```

```
# regression：select 对单通道输入应保持原样
signal_generator (sine 1k, channels=1) → channel_select (channels=0) → fft_spectrum
preview hash 应跟没经过 select 的等价流程一致
```

#### ⚠️ channel_select mix_mode=sum 振幅理解（容易踩的坑）

`channel_select` 的 `mix_mode=sum` **本身不除以 N**（`out = selected.sum(axis=0)`，
意图是"工程 sum 不归一化"），但 sum 后峰值 > 1.0 时 **下游 SDK AudioInput 会做
peak 归一化到 [-1, 1]** 防溢出 —— 这导致 fft / level_meter 看到的振幅会被压缩。

**统计预期**：

| 场景 | 时域 peak | SDK 归一化后 | fft 主峰（amp=0.5 输入）|
|------|----------|-------------|------------------------|
| sum N 个**同相同频** sine（H 轮）| N × amp = 2.0（N=4，amp=0.5）| → 1.0 | -7.7 dB（amp 1.0 + hann scallop）|
| sum N 个**异频** sine（S3）| ≈ √N × amp ≈ 1.0（统计期望，瞬时可能略 > 1）| → 0.5/peak | 实测 -15.5 dB |
| sum 异频不超 1.0 | ≤ 1.0 | 不动 | -13.6 dB（跟单 sine 一致）|

**两个判定要点**：

1. 看 `channel_select` 节点的 logs —— sum + peak > 1.0 时节点会 `emit_log("warn", ...)`
   告诉你"sum 后峰值 X.XX > 1.0，下游 load_audio 会归一化"。看到这条 warning =
   绝对幅度不可信，**只能看相对结构**（频率/相位/通道独立性）
2. 振幅精度场景**应该用 `mix_mode=mean`**（除以 N，确保 ≤ 1.0，幅度可信）

`mean` vs `sum` 的语义选择：
- **mean**：物理意义 = 多通道平均（取平均场量），振幅可信，下游 dB 准
- **sum**：工程上"叠加场量"（多麦贡献叠加），振幅会受 SDK 归一化干扰，**适合定性看结构**而非定量看 dB

### 模板 9：阶跃响应（系统识别）

要测被试系统的"阶跃响应"（看上升时间 / 过冲 / 振铃）必须用真 unit step，**不是** DC 全程恒值：

```
signal_generator (mode=step, step_at_s=0.5, duration=2s, sr=48000, amp=1.0)
  ↓
<被测系统>（如 lowpass 1kHz）
  ↓
preview 时域波形

期望：t<0.5s 输出 0；t≥0.5s 输出爬升到 1，可能有过冲和振铃
```

⚠️ 旧用法 `step_at_s=0`（默认）= 全程 DC 恒值，**测不到阶跃响应**。要测响应必设 `step_at_s > 0`。

### 模板 10：Regression（固定 seed）

```
signal_generator (white_noise, seed=42, amp=0.5, 10s)  ← seed > 0 永远同输出
  ↓
<被测节点>
  ↓
flow_node_output_preview  ← 跟"上次已知好"的输出对比
```

每次改完节点跑一遍，输出数值不变 = 没回归。

## 选 stimulus 决策表

```
要测什么 → 选什么 stimulus

频响 / 频率特性          → chirp（看完整曲线）or impulse（最快，spectrum = h(f)）
                            ⚠ 测 zero_phase=true 滤波器只能用 chirp（impulse 会被 filtfilt 边界反射污染）
某频点的精确衰减         → sine（单频，level_meter 读 dB）
谐波失真                 → square / sawtooth（含丰富谐波）
统计 / regression        → white_noise / pink_noise (seed > 0)
分析节点 PSD             → sine（已知频率峰）or pink_noise（已知 1/f 谱）
心理声学 / loudness      → sine 1kHz @ 已知 SPL（在 sg 配 channel_calibration_db="94"）
心理声学 / sharpness     → 多个不同频率 sine 对照（频率高 → acum 高）
心理声学 / tonality      → sine（→1）vs white_noise（→0）对照
心理声学 / tnr           → sine + white_noise 加和（已知 SNR）
去 DC / 高通验证         → step (step_at_s=0)（DC 全程恒值）or sine 5Hz（极低频）
系统阶跃响应             → step (step_at_s=0.5)（真 unit step，看上升时间/过冲/振铃）
段落检测                 → silence + burst + silence 拼接
基线对照                 → silence（确认下游对零信号不报错 / 不出虚假峰）
多通道按通道独立分析     → channels=N 直接接 per_channel 分析节点（自动展开 N item）；要分支不同链路才用 channel_split
取一个通道分析           → channel_select (channels=L 或 0)，输出仍 AudioData
```

## sample_rate 怎么定

```
默认 48000（行业通用）
要测 16 kHz 以上响应（A 计权 16k 段精度）→ 用 96000 或 192000
内容低频 + 想省时间                    → 22050 或 16000 也行
计权节点                               → sr ≥ 48k 是 IEC Class 1，96k 才严格 Class 0
心理声学                               → ≥ 44100（覆盖完整人耳带宽）
```

## AI 测试报告输出格式（给用户）

测试跑完，**用以下格式给用户报告**（不要只说 "PASS" 或贴 JSON）：

```markdown
## 🧪 测试报告：<节点名> v<版本>

### 测试方案
- **方法**：<冲激/扫频/点验/噪声>
- **stimulus**：signal_generator (mode=..., freq=..., amp=..., sr=..., calibration_db=...)
- **流程**：sg → 节点 → fft → viewer
- **参考标准**：<IEC 61672 / ISO 532-1 / 自定义...>

### 结果对照
| 测试点 | 期望 | 实测 | 偏差 | 判定 |
|---|---:|---:|---:|:---:|
| 1 kHz @ A 计权 | 0.0 dB | 0.02 dB | +0.02 | ✅ |
| 8 kHz @ A 计权 | -1.1 dB | -1.3 dB | -0.2 | ✅ |
| 16 kHz @ A 计权 | -6.6 dB | -8.1 dB | -1.5 | ⚠️ |

### 总体结论
- **整体 PASS**：12/13 测试点在容差内
- **关注**：16 kHz 偏差 -1.5 dB，超出 IEC Class 1 容差。原因：sr=48k 双线性变换在 Nyquist 附近压缩（理论限制）。建议测计权 16 kHz 段时把 sr 提到 96k。

### 流程链接
flow_id: `xxx`，可在 DevStudio 中复跑
```

3 个要点：
1. **写人话** —— 不只是"PASS/FAIL"，给"为什么、怎么修"
2. **数字带单位** —— dB / sone / acum / Hz，别只给浮点
3. **解释偏差** —— 偏差 > 容差时给出"算法原因 vs 节点 bug"判断

## 常见用户需求 → 测试方案

| 用户描述 | 方案 |
|---|---|
| "测一下我刚改的 iir_filter" | 模板 1：chirp → iir → fft → viewer |
| "weighting_filter A 计权对不对" | Read weighting-curves.md → chirp 96k → A → fft 对表 |
| "高通是否真的去掉 50Hz" | 模板 2：sine 50Hz vs sine 1kHz 对照 |
| "fft_spectrum 准不准" | Read fft-spectrum-rules.md → 6 个验证场景 |
| "loudness 节点对不对" | Read psychoacoustic-standards.md → 1 kHz @ 40 dB SPL → 应 1 sone |
| "sharpness 是不是高频更尖锐" | Read psychoacoustic-standards.md → 1k vs 4k vs 8k 对照 |
| "tonality 区分纯音和噪声" | Read psychoacoustic-standards.md → sine vs white_noise 对照 |
| "tnr 检出纯音" | Read psychoacoustic-standards.md → sine + noise 加和测 |
| "active_segment 检测准不准" | 模板 6：silence + burst + silence 拼接 |
| "测多通道分析" / "立体声 L/R 分别处理" | 模板 8：channel_split 或 channel_select |
| "测系统阶跃响应" / "看过冲和振铃" | 模板 9：step + step_at_s > 0 |
| "做 regression test" | 模板 10：white_noise seed=42 → preview 对比 |

## 失败时排查清单

按顺序问 AI 自己：

1. **stimulus 对不对**？`flow_node_output_preview signal_generator` 看 metadata.synthetic、active_samples 前几个值
2. **校准对不对**？心理声学 / level_meter 类节点需要 calibration —— 优先在 sg 上配 `channel_calibration_db`（自动穿透到所有下游），或者 fallback 在节点 params 单独配 `calibration_db`。两个都没设时归一化信号会被当 94 dB SPL 太大
3. **sr 够不够**？测 8k 以上响应 sr 至少 48k，测 16k 至少 96k（双线性变换限制）
4. **FFT 参数**？n_fft 太小 → 频率分辨率不够；窗函数不对 → 旁瓣干扰
5. **节点 method 一致**？loudness zwst vs zwtv、sharpness din vs aures，差 10-30%
6. **节点本身 emit_error**？`flow_node_logs` 看 stderr / traceback
7. **输出全 NaN/Inf**？检查 stimulus 是否激发数值病态（chirp_end ≥ Nyquist 混叠、silence 喂 log）
8. **多通道结果跟单通道一样 / 多通道直接报错**？检查节点 manifest `channels_mode` —— 不声明 = `requires_single`（多通道 fail-fast 报错）；要按通道自动展开成 N item，节点得显式声明 `channels_mode: per_channel`。若节点声明了 per_channel 却仍是单 item，可能上游本来就是单通道，或节点 run.py 没按 per_channel 改造。严格单声道节点（requires_single）要分析多通道，上游用 `channel_split` / `channel_select` 兜底。
9. **测阶跃响应没看到爬升**？step 默认 `step_at_s=0` 是全程 DC 不是真阶跃，必须设 `step_at_s > 0`

## 边界提醒（主动告诉用户）

- **chirp_end ≥ sr/2**：超 Nyquist 会混叠（自动 warning 但仍生成）
- **white/pink noise seed=0**：每次结果不同，做 regression 必设 > 0
- **多通道**：当前所有通道相同信号；要不同信号需多个 generator + merge
- **超长文件**：duration × sr × channels × 4 = 字节数，1 GB 以上很慢
- **silence**：真零信号，下游 fft 可能出 -inf dB（log 0），告诉用户预期
- **心理声学短信号**：duration < 2s 时 response time 内值不稳定，建议 ≥ 3s

## 与其他 skill 的关系

- **filter-design** 教搭滤波链；**node-test** 教验证滤波链对不对（搭配用）
- **flow-author** 搭通用流程；**node-test** 专做"测试图"（stimulus + assert）
- **debug-node** 节点 reload 失败排查；**node-test** 节点跑通了但输出不对的排查
- **create-node** 写新节点；**node-test** 写完后立刻测（推荐工作流：写 → reload → test）
