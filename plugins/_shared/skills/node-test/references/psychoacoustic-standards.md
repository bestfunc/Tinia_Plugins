# 心理声学指标参考值（ISO 532 / DIN 45692 / ECMA-418）

> 用于测试 `loudness`、`sharpness`、`roughness`、`tonality`、`tnr` 节点。
> 心理声学测量**强依赖输入校准** —— 必须先理解校准约定，再谈期望值。

## 校准约定（critical）

Tinia 信号默认归一化到 `[-1, +1]`。心理声学算法需要知道"满量程对应多少 dB SPL"。  
节点参数 `calibration_db`（loudness/sharpness）或 `calibration_offset_db`（level_meter）就是干这个的。

**行业标准锚点**（基于麦克风校准器 Brüel & Kjær 4231 等）：

```
amp = 1.0 (满量程) + calibration_db = 94 dB SPL  ⇒  对应 1 Pa 声压
```

按这个换算其他幅度：

| amp | calibration_db | 实际声压级 |
|----:|---------------:|-----------:|
| 1.0 | 94 | 94 dB SPL（1 Pa）|
| 0.5 | 94 | 88 dB SPL |
| 0.1 | 94 | 74 dB SPL |
| 0.01 | 94 | 54 dB SPL |
| 0.001 | 94 | 34 dB SPL |

**测试模式**：固定 `calibration_db=94`，调 `amp` 模拟不同 SPL。

## loudness（响度，ISO 532-1 Zwicker）

### 标准锚点

ISO 532-1 的核心定义：**1 kHz 自由场 sine @ 40 dB SPL = 1 sone**。

| 输入 | 期望 N (sone) | 容差 |
|---|---:|---:|
| 1 kHz sine @ 40 dB SPL（amp=0.0063, calib=94, free）| 1.0 | ±5% |
| 1 kHz sine @ 60 dB SPL（amp=0.063, calib=94, free）| ≈ 4.0 | ±10% |
| 1 kHz sine @ 80 dB SPL（amp=0.63, calib=94, free）| ≈ 16.0 | ±10% |
| 1 kHz sine @ 100 dB SPL（amp=6.3，需 calib=74）| ≈ 64.0 | ±15% |
| silence | 0 | < 0.01 |
| 完全 white noise @ 60 dB SPL | ≈ 8 sone（宽带）| 算法依赖 |

**响度律**：每 +10 dB ≈ ×2 sone（Stevens' Law）。

### 测试方案

```
signal_generator (sine, 1000 Hz, amp=0.0063, duration=3s, sr=48000)
  ↓
loudness (method=zwst, field_type=free, calibration_db=94, summary_stat=mean)
  ↓
indicator_viewer
```

**期望**：单值 ≈ 1.0 sone（±5%）。  
不在容差内 → 检查 calibration_db 是否对、method 是否正确（zwtv 时变 vs zwst 稳态结果会差几个 %）。

### method 维度差异

| method | 适用 | 备注 |
|---|---|---|
| zwst | 稳态信号（常数 sine、稳定噪声）| 单值，最快 |
| zwtv | 时变信号（瞬态、调制）| 输出时变曲线 + summary_stat |
| ecma | ISO 532-3 Moore-Glasberg | 跟 zw 系会差 10-20%（不同感知模型）|

**测试时固定 method**，不同 method 不互相对照。

## sharpness（尖锐度，DIN 45692）

### 标准锚点

DIN 45692：**1 kHz narrowband noise（160 Hz 带宽）@ 60 dB SPL = 1 acum**。  
但单一 sine 也有近似值，可以用：

| 输入 | 期望 S (acum) | 容差 |
|---|---:|---:|
| 1 kHz sine @ 60 dB SPL（amp=0.063, calib=94）| ≈ 1.0 | ±15% |
| 1 kHz narrowband (FFT-bandlimited) @ 60 dB | 1.0 | ±5% |
| 4 kHz sine @ 60 dB SPL | ≈ 2.5 | ±15% |
| 8 kHz sine @ 60 dB SPL | ≈ 4.5 | ±20% |
| 200 Hz sine @ 60 dB SPL | < 0.5 | ±20% |
| white noise @ 60 dB SPL | ≈ 2.0 | 算法依赖 |
| pink noise @ 60 dB SPL | ≈ 1.5 | 算法依赖 |

**尖锐度律**：能量重心越往高频移，acum 越大。低频信号 sharpness 接近 0。

### 测试方案

```
# 测移频后尖锐度上升
signal_generator (sine, 1000 Hz, amp=0.063, duration=3s)  → sharpness (calib=94, weighting=din)  ← ≈ 1.0
signal_generator (sine, 4000 Hz, amp=0.063, duration=3s)  → sharpness (calib=94, weighting=din)  ← ≈ 2.5
signal_generator (sine, 8000 Hz, amp=0.063, duration=3s)  → sharpness (calib=94, weighting=din)  ← ≈ 4.5
```

如果"频率上升 → acum 不上升或下降" → 节点反了。

### weighting 维度

| weighting | 标准 | 区别 |
|---|---|---|
| din | DIN 45692（默认）| 工业最常用 |
| aures | Aures 1985 | 含响度依赖项，加权曲线略不同 |
| bismarck | Bismarck 1974 | 历史变体 |
| fastl | Fastl 1991 | 历史变体 |

不同 weighting 同输入差 10-30%。**测试固定 weighting=din** 跟标准对。

## roughness（粗糙度，Daniel & Weber 1997 / ECMA-418-2）

### 标准锚点

D&W 锚点：**1 kHz 载波 + 70 Hz AM (100% 调制) @ 60 dB SPL = 1 asper**。

粗糙度是对**调制频率 20-300 Hz** 范围最敏感的指标，70 Hz 是峰值。

| 输入 | 期望 R (asper) | 容差 |
|---|---:|---:|
| 1 kHz carrier + 70 Hz AM 100% @ 60 dB（DW 锚点）| 1.0 | ±20% |
| 1 kHz carrier + 4 Hz AM @ 60 dB | < 0.1 | 调制频率太低 |
| 1 kHz carrier + 70 Hz AM 50% @ 60 dB | ≈ 0.5 | 调制深度线性 |
| 1 kHz pure sine @ 60 dB | ≈ 0 | 无调制 |
| white noise @ 60 dB | 0.3-0.6 | 算法依赖 |

### 测试方案（需要 AM 信号 — signal_generator 当前不支持，需要变通）

signal_generator 直接没有 AM 模式，但可以**通过两路 sine 相乘** 用 `indicator_math (mul)`：

```
# 等效于 carrier × (1 + m·cos(2π·fmod·t))
# = carrier + m·carrier·cos(...)
# 但 indicator_math 是按 item_id 匹配，做信号相乘需要在频域

# 简化方案：用 chirp 扫调制频率，看 R 是否在 70 Hz 附近峰值
# 这是定性测试，不要求严格锚点
```

**实用做法**：定性测"调制频率 vs 粗糙度"曲线趋势：
1. 用 `signal_generator chirp` 直接扫 50-200 Hz 当 carrier（无调制） → roughness 应近零
2. 用真实音频测试集（含 AM 调制）→ 跟 ArtemiS/HEAD 对比

或者**等到 P3 把 AM 模式加进 signal_generator** 再做严格锚点测试。当前阶段：roughness 节点只需验证"silence → 0、纯 sine → 接近 0、复杂信号 → 非零"这种基础正确性。

### method 差异

| method | 标准 | 备注 |
|---|---|---|
| dw | Daniel & Weber 1997 | 工业默认 |
| ecma | ECMA-418-2 HMS | 较新，结果不一样 |

## tonality（音调性）

### 算法本质

`tonality` 度量"信号纯不纯"。三种 method：

| method | 范围 | sine 应给 | white noise 应给 |
|---|---:|---:|---:|
| sfm | 0~1（1=最纯）| ≈ 1.0 | ≈ 0 |
| crest | 0~∞ dB（高=尖峰）| 高（> 30 dB）| 低（≈ 5-10 dB）|
| peakiness | 0~∞ | 高 | 低 |

**核心断言**：sine → tonality 接近"纯"端；white noise → 接近"非纯"端。

### 测试方案

```
# 三种 stimulus 对照
signal_generator (sine, 1000 Hz, amp=0.5, duration=3s)
  → tonality (method=sfm, weighting=Z)  ← 期望 > 0.95

signal_generator (white_noise, seed=42, amp=0.1, duration=3s)
  → tonality (method=sfm, weighting=Z)  ← 期望 < 0.1

signal_generator (sine, 1000 Hz, amp=0.5)  +  signal_generator (white_noise, amp=0.05)  
  → indicator_math (add)
  → tonality (method=sfm)  ← 期望介于两者之间，靠近 sine 端（信噪比高）
```

如果 sine 给低值或 noise 给高值 → 算法实现可能 invert 了。

### peak detection 验证

`tonality` 还输出"检出的纯音频率列表"。测试：

```
signal_generator (sine, 1234 Hz, amp=0.5, duration=3s, sr=48000)
  → tonality (method=sfm, peak_prominence_db=10, peak_max_count=5)
  → indicator_viewer (展开 metadata)
```

**期望**：检出 1 个峰，频率 ≈ 1234 Hz（容差 ±FFT bin = sr/n_fft）。

如果检出 2 个峰（1234 + 镜像或谐波）→ peak_prominence_db 太低；如果一个都没检出 → 太高。

## tnr（音调噪声比 / Prominence Ratio，ECMA-418-1）

### 算法本质

TNR = 突出纯音的能量 / 周边背景噪声能量（dB）。  
PR = 纯音所在临界带能量 / 相邻临界带能量（dB）。

### 测试方案

**单纯音在白噪底**：

```
# 1 kHz 强纯音 (60 dB SNR) + white noise 背景
signal_generator (sine, 1000 Hz, amp=0.5, duration=3s, sr=48000)
  +
signal_generator (white_noise, seed=42, amp=0.0005, duration=3s)  ← 比 sine 低 60 dB
  ↓ indicator_math (add)
  ↓
tnr (method=tnr, analysis_mode=stationary, n_fft=8192, window=hann)
  ↓
indicator_viewer
```

**期望**：
- TNR > 30 dB（sine 远高于背景）
- 检出的纯音频率 ≈ 1000 Hz

### 弱纯音淹没在噪声

```
# 1 kHz sine 比噪声低 5 dB
signal_generator (sine, 1000 Hz, amp=0.005)  +  signal_generator (white_noise, amp=0.01)
  → tnr (prominence_only=true)
```

**期望**：可能不输出任何峰（被噪声掩盖）；prominence_only=true 时进一步过滤"显著"纯音。

### 多纯音

```
signal_generator (sine, 1000 Hz, amp=0.3)  +  signal_generator (sine, 3000 Hz, amp=0.2)  +  signal_generator (white_noise, amp=0.001)
  → tnr (analysis_mode=stationary)
```

**期望**：检出 2 个纯音 ≈ 1000 Hz, 3000 Hz；TNR 都 > 20 dB；3 kHz 的 TNR 比 1 kHz 低（amp 更小）。

## 心理声学测试通用注意

1. **duration 不能太短**：心理声学算法有时间常数（response time ≈ 100-500 ms）。≥ 2s 才稳定。silence 段算出的值不可信。

2. **method 必须明示**：换 method 结果可能差 30%。对照表里给的 method 是默认值（zwst / din / dw / sfm / tnr），不一致时偏差预期。

3. **calibration_db 是 first-class**：所有 dB SPL 期望都建立在已校准。如果默认 calibration_db=0（输入即 Pa），用户喂归一化信号时结果完全错（大 94 dB），但 silence 测试不受影响。

4. **field_type free vs diffuse**：自由场 vs 扩散场，1 kHz 以上有 1-3 dB 差异。锚点表全部基于 free。

5. **sample_rate 影响**：心理声学算法需要 sr ≥ 44.1k 才能完整覆盖人耳频段（20-20k）。sr=48000 是行业默认。

6. **AM / 复合信号合成限制**：当前 signal_generator 只生成单一波形，做 AM 调制需多路相加（用 indicator_math add）。严格 D&W 测试需等 AM 模式加入。
