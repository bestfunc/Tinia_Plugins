---
name: node-test
display_name: 测试节点正确性
description: 用 signal_generator 合成已知特征的测试信号（chirp/impulse/sine/noise），自动搭测试图验证 DSP / 分析节点的频响、衰减、相位、输出值是否符合理论。开发新滤波器 / FFT / 分析节点时的标配验证流程
user-invocable: true
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__flow_create,mcp__tinia__flow_list,mcp__tinia__flow_describe,mcp__tinia__flow_open,mcp__tinia__flow_batch_edit,mcp__tinia__flow_add_node,mcp__tinia__flow_set_node_params,mcp__tinia__flow_connect,mcp__tinia__flow_replace_node,mcp__tinia__flow_auto_layout,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_run_status,mcp__tinia__flow_node_output_preview,mcp__tinia__flow_node_logs
---

# node-test —— 用合成信号测节点正确性

## 何时用

- 用户说 "测一下 X 节点 / 验证 X 节点 / 这个滤波器对不对 / 频响对不对"
- 用户刚改完节点代码（在 dev studio）想立刻验证
- 调用 dev_reload 装载新版本节点之后
- 怀疑某个节点输出不对，想用对照信号定位问题
- 需要 regression test（同输入永远同输出）

## 核心思路 — 行业 4 大测试法

DSP / 信号节点的"正确性"不是字符串相等，而是看**信号特征**。用合成信号（特征已知）→ 跑节点 → 对比理论值。比真实音频测试更精确、可重复、可断言。

| 方法 | 输入 stimulus | 看什么 | 推荐场景 |
|---|---|---|---|
| **冲激法** ⭐ | `impulse` (amp=1) | spectrum 直接 = 完整频响 | 任何 LTI 滤波器（最快） |
| **扫频法** ⭐ | `chirp` 20–20kHz log | spectrum 看通带 / 阻带 / 过渡 | 频响曲线最直观 |
| **点验法** | `sine` 单一频率 | level_meter 看衰减分贝 | 验证特定截止点 |
| **噪声法** | `white_noise` (固定 seed) | PSD 比 = 频响 | 统计验证 / regression |

## 标准动作链

```
1. nodes_describe <要测的节点>
   ↑ 拿到节点的输入/输出类型 + 参数
   ↑ 同时 nodes_describe signal_generator 看 stimulus 模式

2. 跟用户确认要测什么：
   - 节点类型？（滤波器 / FFT / level / loudness / 自定义）
   - 期望特征？（如 lowpass 1kHz 4 阶 → 1k 处 -3dB，2k -24dB ...）
   - 用哪种测试法？默认推荐：滤波器用扫频，分析节点用 sine

3. flow_batch_edit 一次性搭测试图
   - signal_generator (合适 stimulus) → 被测节点 → 验证节点（fft_spectrum / level_meter / spectrum_viewer / indicator_viewer）
   - 别忘 flow_auto_layout 自动布局

4. flow_run + flow_wait_run

5. flow_node_output_preview 各级看输出
   - 重点看最末验证节点（spectrum / level）
   - 拿数值跟理论对比
   - 给用户 PASS / FAIL 结论 + 关键数字

6. 失败时排查：
   - flow_node_logs 看节点 stderr
   - 上游各节点 preview 看是否中途坏了
   - 检查 stimulus 参数是否合理（如 chirp_end ≥ Nyquist 会混叠）
```

## 测试模板（按节点类型选）

### 模板 1：测滤波器频响（IIR/FIR/计权）

```
signal_generator (chirp, 20-20000Hz, log, 5s, sr=48k, amp=0.5)
  ↓
<被测滤波节点>
  ↓
fft_spectrum
  ↓
spectrum_viewer
```

**预期看到**：通带平坦、阻带衰减、过渡带斜率符合阶数（每阶 6 dB/oct）。

**或更快的冲激法**：

```
signal_generator (impulse, 2s, sr=48k, amp=1.0)
  ↓
<被测滤波节点>
  ↓
fft_spectrum
  ↓
spectrum_viewer
```

冲激响应的 FFT **直接就是**滤波器频响 H(f)，最干净。

### 模板 2：测衰减是否准确（点验）

```
# 测 highpass 50Hz：5Hz sin 应被强衰减、1kHz sin 应几乎不变
signal_generator (sine, freq=5, 2s, amp=1.0)    → <hp_filter> → level_meter   预期：< -40 dB
signal_generator (sine, freq=1000, 2s, amp=1.0) → <hp_filter> → level_meter   预期：≈ 0 dB
```

跑两次（或两份并行流），分别看 level_meter 输出。

### 模板 3：测 A 计权曲线

```
signal_generator (chirp 20-20000Hz log, 10s, sr=96000)
  ↓
weighting_filter (A)
  ↓
fft_spectrum
  ↓
spectrum_viewer
```

**预期**：1 kHz 处 0 dB；100 Hz 处 ≈ -19 dB；20 Hz 处 ≈ -50 dB（IEC 61672 标准曲线）。

### 模板 4：测分析节点（FFT / level / loudness）

```
# 测 FFT 是否准确：1 kHz 单音应在 spectrum 1 kHz 处出现窄峰
signal_generator (sine, 1000Hz, 1s, amp=0.5, sr=48000)
  ↓
fft_spectrum
  ↓
spectrum_viewer  ← 1 kHz 处 应有 ≈ -6 dB 峰（amp=0.5 → 20log(0.5) = -6dB）
```

```
# 测 level_meter：白噪声 RMS 应等于 amp（标准差）
signal_generator (white_noise, seed=42, amp=0.1, 5s)
  ↓
level_meter   ← RMS 应 ≈ 0.1（即 -20 dB FS）
```

```
# 测 loudness 节点：1 kHz @ 满量程应 = 标定的 phon 值
signal_generator (sine, 1000Hz, 2s, amp=1.0)
  ↓
loudness   ← 应 ≈ 标定的"1 kHz 满幅 → ? phon"
```

### 模板 5：Regression 测试（固定 seed）

```
signal_generator (white_noise, seed=42, amp=0.5, 10s)  ← seed > 0 永远同输出
  ↓
<被测节点>
  ↓
flow_node_output_preview  ← 跟"上次已知好"的输出对比
```

每次改完节点跑一遍，输出 hash 不变 = 没回归。

## 决策细则（AI 必看）

### 选哪种 stimulus？

```
用户要测什么 → 选什么 stimulus

频响 / 频率特性    → chirp（扫频，看完整频响）or impulse（最快，spectrum = h(f)）
某频点的精确衰减   → sine（单一频率，level_meter 读 dB）
谐波失真           → square / sawtooth（含丰富谐波，spectrum 看谐波是否被滤掉）
统计 / regression  → white_noise / pink_noise（seed 固定）
分析节点 PSD       → sine（已知频率峰）or pink_noise（已知 1/f 谱）
心理声学 / loudness → sine 1kHz @ 已知 amp（对照标准 SPL）
去 DC / 高通验证   → step（DC = 直流）or sine 5Hz（极低频）
基线对照           → silence（确认下游对零信号不报错 / 不出虚假峰）
```

### sample_rate 怎么定？

```
默认 48000（行业通用）。
要测 16 kHz 以上响应（如 A 计权 16k 段精度）→ 用 96000 或 192000。
要省时间 + 内容低频 → 用 22050 或 16000 也行。
计权节点（weighting_filter）：sr ≥ 48k 是 IEC Class 1，96k 才严格 Class 0。
```

### 用 fft_spectrum 还是 spectrum_viewer？

```
fft_spectrum：算频谱（数值 IndicatorData），下游可被 level_meter 等再分析
spectrum_viewer：可视化（纯展示，给人眼看的）

测试流程通常 signal_generator → 节点 → fft_spectrum → spectrum_viewer，
让用户在 viewer 里直接看曲线对照。
```

## 常见用户需求 → 测试方案

| 用户描述 | 测试方案 |
|---|---|
| "我刚改了 iir_filter 你测一下" | 模板 1：chirp → iir_filter → fft → viewer |
| "weighting_filter A 计权对不对" | 模板 3：chirp 96k → weighting_filter A → fft → viewer |
| "高通去工频是否真的去掉 50Hz" | 模板 2：sine 50Hz vs sine 1kHz 各一份对照 |
| "fft_spectrum 准不准" | 模板 4 FFT：sine 1kHz → fft → 看 1k 处峰位置和高度 |
| "我这个节点输出有问题" | 默认 sine 1kHz → 节点 → preview 看输出形态 |
| "做 regression test" | 模板 5：white_noise seed=42 + preview hash 对比 |

## 失败时怎么排查

调 `flow_node_output_preview` 各级看：

1. **signal_generator 输出**：是否真的是 sine / chirp 等期望波形？
   - 看 metadata.synthetic = true 确认是合成
   - 看 active_samples 前几个值（如 sine 0Hz 应该是 0、cos 应该是 amp）
2. **被测节点输出**：跟上一级对比，看是否符合滤波/分析行为
3. **如果节点 emit_error**：调 `flow_node_logs` 看 stderr / traceback
4. **如果输出全是 NaN/Inf**：检查 stimulus 是否激发了数值病态（如 chirp 终点 ≥ Nyquist）

## 边界提醒（主动告诉用户）

- **chirp_end ≥ sr/2**：超 Nyquist 会混叠，自动 warning 但仍生成
- **白/粉噪声 seed=0**：每次结果不同，做 regression 必须设 > 0
- **多通道**（channels > 1）：当前所有通道相同信号；要不同信号需多个 generator + merge
- **超长 / 超大文件**：duration × sr × channels × 4 = 字节数，1 GB 以上会很慢
- **silence**：真零信号，下游 fft 可能出 -inf dB（log(0)），告诉用户预期

## 与其他 skill 的关系

- **filter-design** 教用户怎么搭滤波链；**node-test** 教怎么验证滤波链对不对。常一起用
- **flow-author** 是搭通用流程；**node-test** 专做"测试图"（stimulus + assert）
- **debug-node** 是节点 reload 失败排查；**node-test** 是节点跑通了但输出不对的排查
