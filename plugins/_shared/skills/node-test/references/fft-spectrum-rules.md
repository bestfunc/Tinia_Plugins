# FFT / 频谱节点参考值与判定规则

> 用于测试 `fft_spectrum`、`octave_analysis`、`spectrum_smooth` 节点。
> 核心原则：**喂已知特征的合成信号，看节点输出能不能精确还原已知特征**。

## 基础公式

| 量 | 公式 | 说明 |
|---|---|---|
| 频率分辨率 | `Δf = sr / n_fft` | sr=48k, n_fft=4096 → Δf = 11.72 Hz |
| sine amp=A 的单边谱峰 | `A/2`（线性幅度）| n_fft 是 2 的幂时严格 |
| sine amp=A 的 dB（ref=1.0）| `20*log10(A/2)` | A=1.0 → -6.02 dB；A=0.5 → -12.04 dB |
| sine amp=A 的 dB SPL（ref=20µPa）| `20*log10(A/(2*2e-5))` | A=1.0 → 87.96 dB SPL |
| 白噪声 PSD | `σ² / (sr/2)` | σ = amp（标准差），全频段平坦 |
| 粉噪声谱 | `∝ 1/f`（功率），即 dB 谱斜率 -10 dB/decade | |

## 窗函数特性

| 窗 | 主瓣宽 (bin) | 旁瓣峰值 (dB) | 幅度精度（最坏）| 适用场景 |
|---|---:|---:|---:|---|
| rectangular | 2 | -13 | -3.92 dB | 短瞬态、单 bin sine |
| hann | 4 | -32 | -1.42 dB | 通用默认 |
| hamming | 4 | -43 | -1.78 dB | 类似 hann，旁瓣略低 |
| blackman | 6 | -58 | -1.10 dB | 强旁瓣抑制 |
| flattop | 10 | -94 | **< 0.01 dB** | 测幅度精度（首选）|
| kaiser β=14 | ≈ 8 | -120 | < 0.5 dB | 可调，β 越大旁瓣越抑制 |

**幅度精度**：sine 频率落在 bin 之间时的最大幅度衰减。flattop 几乎为零（专为测幅度设计），rectangular 最差。

## fft_spectrum 测试方案

### 1. 验证频率定位

```
signal_generator (sine, 1000 Hz, amp=0.5, duration=2s, sr=48000)
  ↓
fft_spectrum (n_fft=4096, window=hann, scale=linear, ref_pressure=1.0, mode=average)
  ↓
spectrum_viewer
```

**期望**：
- 峰位置：1000 Hz ± Δf（11.72 Hz）
- 峰高（线性）：≈ 0.25（amp/2）
- 1000 Hz 不在 bin 中心（48000/4096 = 11.72，1000/11.72 = 85.33 → 落在 bin 之间）→ hann 窗会让峰值偏低 ≤ 1.42 dB

**精确测幅度**：把窗换成 flattop，期望峰值 ≈ 0.25 × 校正系数（flattop 的 NEBW 校正 ≈ 1.0，几乎无误差）。

### 2. 验证频率分辨率

```
signal_generator (sine, 1000 Hz, amp=0.5)  +  signal_generator (sine, 1010 Hz, amp=0.5)
  ↓ indicator_math (add) 或并行测两次
fft_spectrum (n_fft=8192)  ← Δf = 5.86 Hz，应能分辨
fft_spectrum (n_fft=2048)  ← Δf = 23.4 Hz，分不开
```

**期望**：n_fft=8192 看到两个独立峰，n_fft=2048 看到一个合并峰。

### 3. 验证窗函数旁瓣

```
signal_generator (sine, 1000 Hz, amp=1.0, duration=4s, sr=48000)
  ↓
fft_spectrum (n_fft=4096, window=hann/blackman/flattop, scale=dB, ref_pressure=1.0)
  ↓
spectrum_viewer
```

**期望**（看 1000 Hz 峰旁边的旁瓣最大值，相对于主峰）：
- hann：-32 dB
- blackman：-58 dB
- flattop：-94 dB（几乎看不到旁瓣）

### 4. 验证 dB / dBA / linear scale 一致性

```
signal_generator (sine, 1000 Hz, amp=1.0, sr=48000)
  ↓
fft_spectrum (scale=linear) → 峰高 = 0.5
fft_spectrum (scale=dB, ref_pressure=1.0) → 峰高 = 20*log10(0.5) = -6.02 dB
fft_spectrum (scale=dBA, ref_pressure=1.0) → 峰高 = -6.02 + 0 (1k 处 A=0) = -6.02 dB
fft_spectrum (scale=dB, ref_pressure=20u) → 峰高 = 20*log10(0.5/2e-5) = 87.96 dB
```

任一组合不符 = 节点 scale 转换错。

### 5. 验证白噪声平坦谱

```
signal_generator (white_noise, seed=42, amp=0.1, duration=10s, sr=48000)
  ↓
fft_spectrum (n_fft=4096, window=hann, mode=average, scale=dB, ref_pressure=1.0)
```

**期望**：
- 谱在 100 Hz~20 kHz 区间应近似平坦（±2 dB 涨落正常 — 短时统计抖动）
- duration 越长越平坦
- 平均水平 ≈ `10*log10(amp² / (sr/2)) + 10*log10(NEBW)` ≈ -57 dB（hann NEBW=1.5 bin）

### 6. 验证粉噪声 1/f

```
signal_generator (pink_noise, seed=42, amp=0.5, duration=10s, sr=48000)
  ↓
fft_spectrum (n_fft=8192, window=hann, mode=average, scale=dB)
```

**期望**：dB 谱应有 -10 dB/decade（每 10 倍频减 10 dB）的下行斜率。  
- 100 Hz vs 1000 Hz 应差 ≈ 10 dB
- 100 Hz vs 10000 Hz 应差 ≈ 20 dB

斜率不对 = pink_noise 算法错（最常见：用了 -3 dB/oct 而不是 -10 dB/dec，结果差不多但不严格）。

## octave_analysis 测试方案

### 1. 单一频率定位到正确频带

```
signal_generator (sine, 1000 Hz, amp=0.5, duration=3s, sr=48000)
  ↓
octave_analysis (fraction=3, weighting=Z)
  ↓
indicator_viewer
```

**期望**：1000 Hz 中心频带（其他频带能量 < -40 dB 相对最大值）。  
1/3 倍频程 1000 Hz 频带边界 ≈ 891-1122 Hz，所以 1000 Hz sine 完全落在带内。

### 2. 频带能量守恒（白噪声）

```
signal_generator (white_noise, seed=42, amp=0.1, duration=10s, sr=48000)
  ↓
octave_analysis (fraction=3, weighting=Z, summary_stat=sum_energy)
```

**期望**：dB 频带值应该按频率上升 +1 dB / 1/3 倍频程（因为带宽随频率正比上升）。
- 31.5 Hz 频带（带宽 ≈ 7.3 Hz）vs 1000 Hz 频带（带宽 ≈ 232 Hz）应差 ≈ 15 dB（10*log10(232/7.3)）
- A 计权下白噪声呈 A 曲线形状

### 3. 倍频程数验证

`fraction=1, f_min=20, f_max=20000` → 应有 10 个频带（中心：31.5, 63, 125, 250, 500, 1000, 2000, 4000, 8000, 16000）  
`fraction=3, f_min=20, f_max=20000` → 应有 31 个频带  
`fraction=12, f_min=20, f_max=20000` → 应有 121 个频带

数量对不上 = f_min/f_max 裁剪逻辑有问题。

## spectrum_smooth 测试方案

### 1. 平滑不应改变白噪声背景斜率

```
signal_generator (white_noise, seed=42, amp=0.1, duration=10s, sr=48000)
  ↓
fft_spectrum (n_fft=4096, scale=dB)
  ↓ 分两路
分支 A: 直接 → indicator_viewer
分支 B: spectrum_smooth (mode=octave_log_interp, fraction=3) → indicator_viewer
```

**期望**：B 应是 A 的"软化版" — 高频抖动消失，但平均水平相同（误差 < 0.5 dB）。如果 B 有系统性偏移，平滑算法有 bias。

### 2. 平滑后窄峰被削弱

```
signal_generator (sine, 1000 Hz, amp=0.5)
  ↓
fft_spectrum
  ↓
spectrum_smooth (fraction=3)
  ↓
indicator_viewer
```

**期望**：1000 Hz 处仍有峰，但比未平滑的低（窄峰能量被分散到周边频带）。具体：1/3 倍频程平滑会把窄峰摊平 ≈ 1/N 频带宽度，幅度下降 ≈ 10*log10(bin/带宽) dB。

### 3. 平滑后宽特征保留

```
signal_generator (pink_noise, seed=42, duration=10s)  → fft → smooth (fraction=3) → viewer
```

**期望**：1/f 斜率（-10 dB/dec）应几乎完整保留，平滑只去掉短时统计抖动。

## 常见失败诊断

| 现象 | 可能原因 |
|---|---|
| sine 峰位置偏移 > Δf | 节点用了错的 sr（比如硬编码 44.1k），或 zero-padding 没乘 2 |
| 峰值幅度比理论低 > 2 dB | 窗函数没补偿 NEBW；或 frequency 落在 bin 之间用了 hann |
| 旁瓣比表里高 10 dB | 窗函数实现错（最常见：hamming 误用为对称的 hann） |
| dB scale 全是负无穷 | 输入是纯 silence；或 ref_pressure 把 0 输入 log 出 -inf |
| dBA 比 dB 高 | scale 转换方向反了（应该减去 A 计权值在某些频段是正的，但 1k 以下都是减） |
| 白噪声谱不平 | n_fft 太小（统计抖动大）；或窗能量校正没做 |
| 粉噪声不是 -10 dB/dec | 算法用了 -3 dB/oct（≠ -10 dB/dec，差 ≈ 0.04 dB/oct，长频段累积明显） |
