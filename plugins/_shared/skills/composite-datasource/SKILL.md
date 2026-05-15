---
name: composite-datasource
display_name: 合成多通道数据源
description: 把已有的 N 个 mono 数据源 stack 成一个多通道 composite 数据源 — 工业 NVH 常见场景（N 麦阵列各录 mono / 多次采集 mono 横拼对比）。AI 帮用户选源 + 配对方式（自动 / 手工 / CSV）+ 通道命名模板 → 创建 → 触发合成。下游分析节点零感知，用起来跟普通多通道数据源一样。
user-invocable: true
allowed-tools: mcp__tinia__datasource_list,mcp__tinia__datasource_describe,mcp__tinia__datasource_create,mcp__tinia__datasource_update,mcp__tinia__datasource_delete,mcp__tinia__datasource_materialize,mcp__tinia__datasource_list_files,mcp__tinia__channel_template_list,mcp__tinia__channel_template_create,mcp__tinia__channel_template_apply,mcp__tinia__flow_create,mcp__tinia__flow_batch_edit,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_node_output_preview,Bash
---

# composite-datasource —— 合成多通道数据源

## 何时用

用户表达类似下面的需求时调本 skill：

- "把 Mic_FL 和 Mic_FR 这两个数据源合成立体声"
- "5 麦阵列各录的 mono 文件想合成 5 通道做 NVH 分析"
- "多次采集分别保存的 mono 想横着拼成多通道做对比"
- "我有几个单通道数据源，能不能合一下"
- "合成多通道数据源" / "composite datasource"

如果用户是**测试**数据源功能（验证 channel_template + ChannelMeta 全链路），用 [datasource-test](../datasource-test) skill。

## 核心概念

**composite datasource = 数据源层的"虚拟视图"**：不存独立文件，存配置 + 一次性合成结果。
- 创建时只配置（source_ids + pair_by + ...），**不立即合成** —— 用户主动调 `datasource_materialize` 触发
- 合成时在 server 端读 N 个 mono blob → numpy stack → 写多通道 wav 到 `datasource_uploads`
- 后续 `dataset_node` / `materialize_node` / 分析节点**完全透明**（看到的就是普通多通道数据源）
- 源数据源仍独立存在，可单独使用 / 分析；composite 是另一个独立对象，不影响源

**为什么不自动合成**：
- 用户上传源文件可能分多次（先传 ch0 再传 ch1）
- 系统没法判断"传完了" → 自动合成会浪费计算
- 让用户主动按按钮（或 AI 调 `datasource_materialize`）= 时机用户掌控

## 核心流程

```
1. 跟用户对齐意图：要合的源 + 配对方式 + 通道命名（可选）
        ↓
2. datasource_list 看用户当前数据源 → 确认每个源都是 mono + sr 一致
        ↓
3. (可选) channel_template_create 建/选模板（如果用户要给通道命名）
        ↓
4. datasource_create({
     name: "...",
     source_kind: "composite",
     channel_template_id: tpl_id (可选),
     composite_config: {
       source_ids: [...],
       pair_by: "item_id_prefix" | "index",
       length_mode: "truncate_min"
     }
   })
        ↓ 返回 ds_id（状态：待合成）
5. datasource_materialize(datasource_id=ds_id)
        ↓ 后端 stack + 写 uploads（状态：已合成）
6. (可选验证) datasource_list_files 看生成的多通道 wav
        ↓
7. (可选) flow_create + dataset_node + fft_spectrum 跑端到端，
        看 _meta.channel_label 是模板里定义的名字
```

## 反问规则（缺信息时主动问）

AI 不应该瞎猜，下面信息缺一项就反问用户：

| 缺什么 | 反问示例 |
|--------|---------|
| 不知道哪些源 | "你想合哪几个数据源？我看到你有 Mic_FL / Mic_FR / Accel_X / Accel_Y / Accel_Z 5 个 mono 数据源，全部合成 5 通道，还是只合前 2 个？" |
| 不知道配对方式 | "源里的 item 怎么对齐？(A) 按文件名公共前缀（推荐，如 rec_001.wav 自动配成同组）(B) 按位置（按文件名排序后第 1 配第 1，要求各源 item 数相等）" |
| 不知道是否要命名通道 | "要不要给通道起业务名（如 Mic_FL / Mic_FR / Accel_X...）？分析输出里 _meta.channel_label 会用这些名字。或者不命名，下游用 ch0/ch1/...？" |
| 通道顺序不确定 | "通道顺序按你给的数据源 id 列表为准（第一个 = ch0，第二个 = ch1...）。例 [12, 13, 14] = (ch0=Mic_FL, ch1=Mic_FR, ch2=Accel_X)。这样对吗？" |

## 配对方式选择决策

三种模式，按优先级判断：

```
用户给了 CSV 配对文件？
├─ 是 → pair_by = "manual"，AI 读 CSV → 转 groups → 调 datasource_create
│       （详见下面"AI 处理 CSV 配对文件"章节）
└─ 否 → 看用户 mono 文件命名规范：
        ├─ 有公共前缀（rec_001.wav, rec_002.wav）→ pair_by = "item_id_prefix"（推荐）
        ├─ 各源文件名乱但顺序对齐 → pair_by = "index"
        └─ 命名乱 + 顺序也乱 + 文件不多 → pair_by = "manual"，
            AI 反问用户每组该怎么配，构造 groups 数组
```

**默认 `item_id_prefix`** —— 大多数 NVH 录音都按 session ID 命名，前缀匹配最稳。

**Manual 模式 schema**：

```json
{
  "name": "...",
  "source_kind": "composite",
  "channel_template_id": 5,
  "composite_config": {
    "source_ids": [12, 13],
    "pair_by": "manual",
    "groups": [
      {
        "name": "rec_001.wav",
        "channels": [
          {"src_ds_id": 12, "src_item_id": "5"},
          {"src_ds_id": 13, "src_item_id": "8"}
        ]
      }
    ]
  }
}
```

注意：
- `src_item_id` 必须是 **string** 形式的 datasource_uploads 行 id（不是 filename）
- 用 `datasource_list_files(datasource_id)` 拿到的 `id` 字段，转 string 传过去
- `groups[].channels[]` 顺序 = ch0..chN（跟 channel_template.channels 必须对齐）

## AI 处理 CSV 配对文件

用户场景："这是配对 CSV，帮我合成"。AI 应该：

1. **读 CSV** —— 用 Bash `cat <path>` 拿全文（小文件直接读，大文件 head 看 header + 抽样）
2. **解析 header** —— 第一行通常是 `group_name,ch0_source,ch0_item,ch1_source,ch1_item,...`
3. **datasource_list** 拿所有 ds 的 `name → id` 映射
4. **对每个用到的 source ds 调 `datasource_list_files`** 拿 `filename → id` 映射
5. **逐行解析数据** —— 对每行：
   - `group_name` → `groups[i].name`
   - `chN_source` → 在 ds 列表里按 name 查 id → `groups[i].channels[N].src_ds_id`
   - `chN_item` → 在该 ds 的 files 里按 filename 查 id → `groups[i].channels[N].src_item_id`（转 string）
6. **datasource_create** 用 manual 模式 + groups
7. **datasource_materialize** 触发合成

**Bash 读 CSV 示例**：

```bash
cat /path/to/composite.csv
# 或大文件先看结构：
head -5 /path/to/composite.csv
```

AI 在内存里 parse 即可（Python `csv` 模块 / 简单 split 都行），不用执行外部 csv 解析工具。

**Critical**：CSV 里 `chN_item` 是文件名（如 `r001.wav`），后端 manual 模式需要的是 datasource_uploads 行的数字 id。AI 必须做这个映射，**不能直接把文件名传给 `src_item_id`** —— 否则 materialize 时报"找不到 item"。

**典型 CSV 示例**：

```csv
# Tinia Composite DataSource Pairing v1
group_name,ch0_source,ch0_item,ch1_source,ch1_item
rec_001,Mic_FL_recordings,r001.wav,Mic_FR_recordings,r001.wav
rec_002,Mic_FL_recordings,r002.wav,Mic_FR_recordings,r002.wav
```

注意支持 `#` 注释行 + 空行（解析时跳过）。如果用户的 CSV 第一列不是 `group_name`，反问用户列名映射。

## 长度对齐选择

```
源都是同一次采集（应该等长）？
├─ 是 → "strict"（严格相等，不一致报错让用户排查）
└─ 否 → "truncate_min"（截断到最短，最常见的妥协）
```

特殊情况下用 `pad_zeros`（不足补 0），但分析容易出 artifact，不推荐默认。

## 通道命名模板

跟普通 datasource 用同一个模板系统（[datasource-test](../datasource-test) §"通道命名模板"）。

**关键约束**：模板的 `n_channels` 必须 = composite 的源数（即 source_ids 长度）。
- 创建 composite 时如果指定 channel_template_id，AI 应该先 `channel_template_list` 找通道数匹配的模板
- 没合适模板时帮用户建一个：`channel_template_create({n_channels: N, channels: [{name, unit, calibration_db}, ...]})`

## 典型对话脚本

```
用户："我有 Mic_FL_recordings 和 Mic_FR_recordings 两个数据源，每个都是 12 个 mono 文件
       命名是 rec_001 ~ rec_012。帮我合成立体声做 FFT 分析，左通道叫 Mic_L 右叫 Mic_R。"

AI：
1. datasource_list → 找到 Mic_FL_recordings (id=12) 和 Mic_FR_recordings (id=13)
2. 各 datasource_describe 验证是 mono + sr 一致（如果 describe 不返回 channel 数，
   datasource_list_files 看 size_bytes 估算）
3. channel_template_list({n_channels: 2}) → 看有没有现成 (Mic_L, Mic_R) 模板
4. 没有 → channel_template_create({
     name: "立体声 (Mic_L/Mic_R)",
     n_channels: 2,
     channels: [
       {name: "Mic_L", unit: "Pa", calibration_db: 94, color: "#3b82f6"},
       {name: "Mic_R", unit: "Pa", calibration_db: 94, color: "#ef4444"}
     ]
   }) → tpl_id=N
5. datasource_create({
     name: "stereo_LR_composite",
     source_kind: "composite",
     channel_template_id: N,
     composite_config: {
       source_ids: [12, 13],
       pair_by: "item_id_prefix",
       length_mode: "truncate_min"
     }
   }) → ds_id=M
6. datasource_materialize({datasource_id: M})
   → 报告：{groups_total: 12, groups_created: 12}
7. （可选）datasource_list_files(M) 让用户看到 12 个合成文件
8. （可选）跑 flow 验证 _meta.channel_label = Mic_L / Mic_R
```

## 典型对话脚本：CSV 模式

```
用户："/Users/me/recordings/composite_pair.csv 这是配对文件，帮我合成 5 通道做分析。"

AI:
1. Bash: cat /Users/me/recordings/composite_pair.csv
   → 拿到 CSV 文本
   → 解析 header: group_name, ch0_source, ch0_item, ch1_source, ch1_item, ...
   → 数据行（10 组）

2. datasource_list → 拿到 ds name → id 映射：
   {"Mic_FL_recordings": 12, "Mic_FR_recordings": 13, "Accel_X_recordings": 14, ...}

3. 对每个用到的 source ds 调 datasource_list_files：
   datasource_list_files(12) → {"r001.wav": 100, "r002.wav": 101, ...}
   datasource_list_files(13) → {"r001.wav": 200, "r002.wav": 201, ...}
   ... (5 个 source ds → 5 次调用，结果 cache 在 AI 内存里)

4. 解析 + 映射：CSV 每行的 chN_source/chN_item 转成 (src_ds_id, src_item_id)
   构造 groups 数组

5. (可选) channel_template_create({n_channels: 5, channels: [...]}) → tpl_id

6. datasource_create({
     name: "测试 5 通道_composite",
     source_kind: "composite",
     channel_template_id: tpl_id,
     composite_config: {
       source_ids: [12, 13, 14, 15, 16],   // 用到的源（自动从 groups 里 collect）
       pair_by: "manual",
       length_mode: "truncate_min",
       groups: [
         {name: "rec_001.wav", channels: [{src_ds_id: 12, src_item_id: "100"}, ...]},
         {name: "rec_002.wav", channels: [{src_ds_id: 12, src_item_id: "101"}, ...]},
         ...
       ]
     }
   }) → composite_ds_id

7. datasource_materialize(composite_ds_id)
   → 报告 {groups_total: 10, groups_created: 10}

8. datasource_list_files(composite_ds_id) 给用户看合成结果
```

## 失败排查

| 现象 | 可能原因 | 处理 |
|------|---------|------|
| `datasource_create` 报 "composite_config.source_ids 不能为空" | 自动模式时 source_ids 没传 | 反问用户要哪些源 |
| `datasource_create` 报 "composite_config.groups 不能为空" | manual 模式时 groups 没传 | 检查 CSV 解析或反问用户配对结构 |
| `datasource_create` 报 "groups[i] 通道数=N 与第一组=M 不一致" | manual 模式各组通道数对不齐 | 检查 CSV 行长度，可能某行多了/少了列 |
| `datasource_materialize` 报 "ch0 sr=44100 != ch1 sr=48000" | 源 sr 不一致 | 告诉用户必须一致；可建议先转码或选其他源 |
| 报 "expect mono, got 2 channels" | 源里有立体声/多通道文件混进来了 | composite 只接 mono；告诉用户哪个源不是 mono |
| 报 "audio_format=3 not supported" | 源是 float32 wav | 当前只支持 PCM int16；告诉用户先转码 |
| 报 "ch0 长度=480000 与 ch1=479000 不一致 (length_mode=strict)" | 严格对齐但源长度不等 | 改 length_mode="truncate_min" 重 update + 重 materialize |
| 报 "配对后没有可合成 group" (item_id_prefix) | 各源文件命名前缀不重合 | 让用户检查命名 / 改 pair_by="index" |
| 报 "ds=12 items=10 与 ds=11 items=12 不等" (index) | 各源 item 数不等 | 让用户对齐 / 改 pair_by="item_id_prefix" |
| 报 "group=X ch0: ds=12 找不到 item=r001.wav" (manual) | src_item_id 传了文件名而非 id | 用 datasource_list_files 把 filename 映射成 id 字符串 |
| 跑分析后 _meta.channel_label 还是 ch0/ch1 | datasource 没设 channel_template_id；或 composite 没 materialize 成功 | datasource_describe 检查 channel_template_id 字段 + composite_config.materialized |

## 决策细则（AI 必看）

### 通道顺序的语义

`source_ids: [12, 13, 14]` 表示：
- ch0 ← datasource 12 的对应 item
- ch1 ← datasource 13 的对应 item
- ch2 ← datasource 14 的对应 item

`channel_template.channels` 数组按 ch0..chN 顺序解释。所以 `source_ids` 的顺序 = template channels 的顺序，**两边必须对齐**。如果用户说"左通道是 Mic_FR 右通道是 Mic_FL"，要把 source_ids 也按这个顺序排（不是按字典序）。

### 不能嵌套合成

composite 的 source_ids 必须是 **`source_kind=upload` 的数据源**，不能再嵌套 composite。AI 调 datasource_list 后应该过滤 `source_kind === 'upload'` 才进入候选。

### 重新合成的语义

源数据源加文件 / 改文件后，composite **不会自动跟着变**（这是 v1.23 的设计选择 — materialize-first）。要刷新：再调一次 `datasource_materialize`，会清掉旧合成结果重做。

### channel_template_id 必须在 datasource 上而非 composite_config 里

后端 schema 设计：composite 用的 channel_template_id 跟普通 upload datasource 用的是**同一个字段** `datasource.channel_template_id`，不在 composite_config 里。所以 datasource_create 时 channel_template_id 跟 composite_config 同级传：

```json
{
  "name": "...",
  "source_kind": "composite",
  "channel_template_id": 5,           ← 这里
  "composite_config": {                ← 不在这里面
    "source_ids": [...],
    "pair_by": "..."
  }
}
```

### 名称建议带 _composite 后缀

如 `driving_2026_5ch_composite` / `stereo_LR_composite` —— 让用户在数据源列表里一眼看出是合成的。前端列表项也会自带 ▣ "合成" 徽章，但名字带后缀方便搜索。

## composite vs signal_generator：什么时候用哪个

两条路径都能产出"多通道音频"给下游分析节点，但适用场景完全不同：

```
用户的诉求？
├─ 合并已有真实 mono 录音 → composite-datasource（本 skill）
│  - 工业 NVH 阵列录音 / 多次采集对比
│  - 数据本身是采集来的
│
└─ 需要可控的合成测试信号（已知频率/相位/相关性） → signal_generator 节点
   - 测分析节点的多通道能力（fft / loudness / etc）
   - 没有真实采集数据，要造测试用例
   - 通道间相位差 / 频率扫描 / 噪声相关性等可控场景
```

### signal_generator 多通道用法（v1.23 Phase E1 加强）

`signal_generator` 的多通道支持有两层：

**默认模式**（所有通道相同信号）：
```json
{
  "channels": 4,
  "mode": "sine",
  "frequency": 1000,
  "channel_names": "Mic_FL,Mic_FR,Accel_X,Accel_Y",
  "channel_units": "Pa,Pa,g,g",
  "channel_calibration_db": "94,94,100,100"
}
```

**逐通道独立配置**（v1.23+，相位差/混合源测试）：
```json
{
  "channels": 4,
  "per_channel_config": true,
  "per_channel_specs": [
    {"mode": "sine", "frequency": 1000, "amplitude": 0.5, "phase_deg": 0},
    {"mode": "sine", "frequency": 1000, "amplitude": 0.5, "phase_deg": 90},
    {"mode": "white_noise", "noise_seed": 42, "amplitude": 0.1},
    {"mode": "silence"}
  ],
  "channel_names": "L,R,Noise,Silent"
}
```

`per_channel_specs[i]` 的字段缺失会沿用顶层 spec 的对应值（如 `phase_deg` 不传 = 用顶层 phase_deg）。`silence` 模式可以单独让某通道全零，方便对照分析。

典型用例：
- **通道间相位差测试**：ch0 phase=0 / ch1 phase=90 → 验证 fft 输出的相位
- **频率扫描阵列**：ch0..ch4 各设不同 freq → 验证 fft per_channel 不混淆
- **信噪比测试**：ch0=信号 / ch1=白噪声 → 验证 SNR 类指标
- **混合验证**：ch0=sine / ch1=chirp / ch2=silence → 一次跑覆盖多场景

AI 调时用 flow_add_node + flow_batch_edit 设置 signal_generator 节点 params 即可，跟其他节点用法一致。

## 与其他 skill 的关系

| skill | 关系 |
|-------|------|
| **composite-datasource**（本）| 实际工作 — 帮用户合成多通道数据源（真实录音）|
| **datasource-test** | 测试导向 — 验证 datasource + 通道模板 + ChannelMeta 全链路 |
| **flow-author** | 用 composite 数据源搭分析流程（dataset_node 直接用 ds_id，跟普通 upload 一样）|
| **datasource-plugin** | 不相关（那是开发"接入外部数据源"的插件，跟"合成"是两件事）|
| **signal_generator 节点** | 合成测试信号的替代路径（无真实录音时用）—— 详见上面"composite vs signal_generator"|

## 端到端验证模板

要确认合成 + ChannelMeta 链路完全打通时跑这个：

```
1. AI 调 composite-datasource 流程创建 + materialize 出 composite_id
2. flow_create + 节点：
   - dataset_node(datasource_id=composite_id)
   - materialize_node
   - fft_spectrum
   - indicator_viewer
3. flow_run → flow_wait_run → flow_node_output_preview(fft_spectrum)
4. 验证 fft 输出 N items，每 item 的 _meta.channel_label 是模板里的业务名
   （Mic_L / Mic_R / Accel_X 等，不是 ch0/ch1/ch2）
```

如果 _meta.channel_label 还是 ch0/ch1，回头查：
- composite 的 channel_template_id 是不是设了
- channel_template 里 channels[i].name 是不是非空
- composite_config.materialized 是不是 true（没 materialize 的话上层节点也跑不通）

成功后 datasource → 节点 → 分析输出 _meta.channel_label 全链路 = 通。
