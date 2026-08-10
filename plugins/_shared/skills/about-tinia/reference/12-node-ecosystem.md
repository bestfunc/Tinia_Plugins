# 节点生态 & 商店

> 节点是 Tinia 的基本能力单元，节点商店是 Tinia 三大护城河之一。给销售讲生态故事 + 给开发者讲怎么贡献。

---

## 节点是什么

DAG 的最小执行单元，一个"做某件事的能力"：

- 算法节点：`level_meter` / `fft_spectrum` / `order_tracking`
- 数据源节点：`dataset_node` / `signal_generator`
- 处理节点：`weighting_filter` / `audio_segment_split`
- 可视化节点：`spectrum_viewer` / `indicator_viewer`

每个节点是独立的代码 + UI 单元，可以独立发布、订阅、卸载、更新。

详见 `04-key-concepts.md`。

---

## 当前节点清单（截至 2026 H1）

> Tinia 官方节点（`bestfunc/*` namespace）。其中 **Tinia_nodes 仓库内的 Python 节点约 41 个**，外加主仓 daemon 内置的 Go 节点（数据源 / 数据流 / 看板等）。覆盖从信号生成、预处理、声学 / 心理声学 / 振动分析、特征工程到可视化的完整链路。
>
> ⚠ 标记系统 v1（diffgram）数据源节点位于独立仓库 `Tinia_nodes_diffgram`，不在官方分析节点集合内。

### 数据源 / 数据流（builtin Go 节点）

> ⚠ 这一类是 daemon 内置的 Go 节点（不在 `Tinia_nodes/` 仓库），跟用户开发的 Python 节点统一暴露在 `bestfunc/*` namespace。

| 节点 | 说明 | 动态端口 |
|---|---|---|
| `dataset_node` | 通用数据集加载（接 datasource）| ❌ |
| `signal_generator` | 合成信号（白噪声 / 正弦 / 扫频，多通道）| ❌ |
| `materialize_node` | 把 lazy 数据集固化到 blob | ❌ |
| `dataset_merge` | **合并多个数据集** | ✅ 动态 in_N（2-8）|
| `dashboard_node` | **看板节点**，接多个 viewer 的 dashboard_view 输出 | ✅ 动态 in_N（1-16）|
| `attach_attributes_node` | 把外部属性表附加到数据集 items | ❌ |
| `filter_node` | 通用 item 过滤器 | ❌ |
| `sample_node` | 数据集随机 / 顺序采样 | ❌ |
| `csv_export` | 把数据集 items 导出 CSV | ❌ |
| `print_console` | 调试节点，把任意输入打印到 stderr | ❌ |

### 信号源（source）

| 节点 | 说明 |
|---|---|
| `signal_generator` | 信号发生器（11 种波形 / 多通道 / chirp 扫频 / 可复现噪声种子）|

### 信号处理 / 数据准备（preprocess）

| 节点 | 说明 |
|---|---|
| `audio_segment_split` | 按时间 / 段长切割音频 |
| `active_segment` | 有效段检测（v2 重构，输出 AnnotationLayer + 阈值可视化）|
| `convergent_trim` | 收敛剔除（去除异常段）|
| `weighting_filter` | 频率计权滤波（A/B/C + 已吸收原 `iso_weighting`）|
| `fir_filter` | FIR 数字滤波器 |
| `iir_filter` | IIR 数字滤波器（巴特沃斯 / 切比雪夫等）|
| `signal_math` | 信号数学运算（合并了原 `vibration_convert`）|
| `channel_split` | 多通道拆单通道 |
| `channel_select` | 选择部分通道 |

### 频域 / 谱分析（analysis）

| 节点 | 说明 |
|---|---|
| `fft_spectrum` | FFT 频谱 |
| `octave_analysis` | 倍频程 / 三分之一倍频程（含 Tailored Octave）|
| `modulation_spectrum` | 调制谱分析（NVH / 声品质，自定义表单标杆）|
| `scale_space_spectrum` | 多尺度谱分析（输出尺度σ×频率×时间谱栈 + 跨尺度统计，GPU 加速）|
| `order_tracking` | 阶次跟踪（旋转机械）|
| `spectrum_smooth` | 频谱平滑 |

### 振动 / 旋转机械（analysis）

| 节点 | 说明 |
|---|---|
| `envelope_demod` | 包络解调（振动 / 轴承故障）|
| `time_stats` | 时域统计（RMS / Crest / Kurtosis 等）|

### 时域 / 时频特征（analysis / feature）

| 节点 | 说明 |
|---|---|
| `fbank_extract` | 频谱特征提取（filter bank）|
| `st_features` | 结构张量特征（谱图纹理特征）|
| `attribute_extract` | 属性提取（输出 AttributeTable）|
| `acoustic_fingerprint` | 声学指纹（高维特征矩阵，质检链路的特征来源）|
| `audio_emit` | 波形输出（把 AudioData 降采样成 vs_time 指标）|

### 心理声学指标（analysis）

| 节点 | 说明 |
|---|---|
| `loudness` | 响度（Zwicker 标准）|
| `sharpness` | 尖锐度 |
| `roughness` | 粗糙度（基于 mosqito）|
| `tonality` | 音调性 |
| `tnr` | TNR（Tone-to-Noise Ratio）|

### 指标计算 / 整理（analysis / transform / feature）

| 节点 | 说明 | 动态端口 |
|---|---|---|
| `level_meter` | 声级计（dBA / dBC / dBZ + IEC 61672）| ❌ |
| `indicator_math` | 指标数学运算（加减乘除 / 阈值判断等）| ❌ |
| `indicator_merge` | 多源指标合并 | ✅ in_N（2-8）|
| `feature_merge` | 特征聚合（FeatureMatrix 超类型核心枢纽）| ✅ in_N（2-8）|
| `feature_normalize` | 特征归一化（Z-score / Min-Max 等）| ❌ |
| `annotation_merge` | 标注合并（2~16 层 AnnotationLayer）| ✅ in_N（2-16）|
| `spec_limit_check` | 限值检查（阶梯上下限判定，独立 limit 端口）| ❌ |

### 动态端口节点 — 重要常识

部分节点支持**任意数量输入端口**（声明 `dynamic_inputs.prefix="in"` + 范围）。AI 用 MCP 调流程时**连到 `in_N` 就自动激活第 N 个端口**，不需要"加端口"操作。

5 个常见动态端口节点：
- `dataset_merge`（数据集合并）
- `dashboard_node`（看板）
- `indicator_merge` / `feature_merge` / `annotation_merge`

更多详见 `flow-author` skill 的"动态端口节点"专章。

### 可视化（viewer）

| 节点 | 说明 |
|---|---|
| `indicator_viewer` | 指标查看器（4 布局 + 限值叠加阶梯虚线 + 音频指针） |
| `spectrum_viewer` | 频谱查看器（2D/3D 热力图 + 音频联动 + 多尺度谱栈模式） |
| `chart_viewer` | **通用图表查看器**（v1.26+）—— 接行列表格做柱状/散点/折线/箱线/直方 5 种图 + 数据表（排序/筛选/列显隐/导出 CSV），可接「原始音频」按 item_id 关联真实标签 |
| `matrix_view` | 透视矩阵（features×attributes join 热力矩阵）|

> 多数 viewer 同时实现「看板协议」适配（顶栏分享、tile/card 可拖拽、属性面板），可被前端看板直接消费。

### 高级分析 / 异常检测（feature）

| 节点 | 说明 |
|---|---|
| `cluster_explore` | 聚类探索（KMeans / DBSCAN / HDBSCAN / GMM + PCA / UMAP / t-SNE 降维）|
| `zscore_anomaly` | Z-score 异常检测（v2.0.0 **fit/apply 双模式**：不接 baseline 输入=建基线，接了=套基线判定。已吸收并替代旧的 `baseline_stats`）|
| `score_predictor` | **评分器**（v2.0.0）—— 把高维特征降成一个分。训练 / 推理 / 评估三模式；监督组 5 种算法（LR / LDA / 二阶多项式 / 决策树 / GBDT）+ 单类组 3 种（knn_pit / shrinkL2 / maxz，只有 OK 样本也能冷启动）|
| `model_artifact` | **制品库读写口**（接输入=存、不接=取），双层版本号 + active 上线指针 |
| `flow_record` | **检测记录**（判定结果落库，供查询 / 统计 / 回填真值），透传不阻断 |
| `feature_pool_write` / `feature_pool_read` | **特征池**（攒特征供重训，跳过最慢的提特征步骤）|

### 仪表盘

| 节点 | 说明 |
|---|---|
| `dashboard_view` 输出 | 多 Viewer 组合给 Dashboard 用 |

### 实时 / 流式逐帧（v1.22+ 旗舰能力）

> 三大基础分析节点实现**跨窗状态延续**，从"逐窗独立批处理"升级为"无缝实时逐帧"，与主仓 SDK 流式会话（v1.35）端到端配套。

| 节点 | 流式增强 |
|---|---|
| `level_meter` | A/C 计权滤波器 `zi` 跨窗延续 + 滚动 Leq + 滚动 percentile |
| `octave_analysis` | 每频带滤波器 `zi` 跨窗延续 + 滚动平均谱 |
| `fft_spectrum` | STFT 残留 carry 跨窗无缝 + 线性功率滚动平均谱 |

- emit 新增 `rolling_*` 字段；`_stream_continuous` 标志门控，批量行为零回归。
- 运行时基座是 SDK 的 **ChunkRuntime（SV2）**——分块流式调度 + `upstream_total` 进度条。节点作者只需声明 `_stream_continuous` 并维护跨窗状态，会话生命周期由 server 侧 SDK 流式会话调度（见 `04-key-concepts.md` / SDK 章节）。
- `loudness` 因依赖 mosqito 黑盒无法做到无缝，保持"准实时逐窗"。

### AutoML 链路（v1.25+ 主线能力）

| 模块 | 说明 |
|---|---|
| 调参引擎 | Optuna 贝叶斯优化驱动，支持搜索空间 / 评估函数声明 / Top-K 结果 |
| 训练 / 验证拆分 | 真 holdout（不再是 val 子集自跑 CV 蒙到 1.0），gap > 15% ⚠️ 过拟合警告 |
| 区分度诊断 | PCA 散点 + 混淆矩阵 + per-class 指标 + 特征 × 样本热力图 + 错分明细 |
| 限值直评 | 节点自带判定列 / 连续分值时（node.yaml 声明 `eval_columns`）直接拿它跟真实标签比，不用再训分类器 |
| 副作用隔离 | 写库 / 发消息类节点声明 `skip_side_effects_in_trial`，搜参时跳过写入但照常透传，不污染生产库 |

> **判别函数已从 AutoML 里剥离**（v1.45）。AutoML 只管定参 —— 告诉你哪组参数效果好；
> 判别函数变成独立的通用节点 `score_predictor`（评分器），模型走制品库。
> 原先的「判别函数 tab / 复制 JSON / → 创建评分节点」入口全部移除：
> 声学指纹这类没有可调参数的链路，现在直接搭"评分器 → 制品库"，根本不经过 AutoML。

### 节点开发节奏

- 节点列表持续扩充，每月新增数个；单节点各自维护 SemVer（如 `feature_merge` 已到 3.0.0、`active_segment` 2.x，多数稳定节点在 1.x）。
- 每个节点 `ui/Help.tsx` 内嵌「更新履历」——GraphEditor 在节点版本号旁渲染 HelpCircle 图标，点击弹模态查看说明 + 参数 + 算法 + 履历。
- 详细规划节点路线见下方"规划节点路线图"
- 跟 HEAD 134 项笛卡尔积清单对比 → 详见 `06-competitive-landscape.md`

---

## 规划节点路线图

> ⚠ 以下节点**尚未实现**，按 2026 H2 三波交付 + 2027 规划。具体优先级和时间可能调整。

### 2026 H2 第一波（5-6 月）—— "这是专业 NVH 工具"

| 节点 | 说明 | 价值 |
|---|---|---|
| `stft` | 短时傅里叶变换 | 时频联合分析基础 |
| `spectrogram` | 频谱图（基于 STFT 的可视化）| NVH 工程师每天必用 |
| `psd_welch` | Welch 法功率谱密度估计 | 平稳信号谱估计标准方法 |
| ~~`audio_player`~~ | ✅ 已落地：`spectrum_viewer` 内置音频联动指针 | — |
| ~~`time_domain_stats`~~ | ✅ 已落地为 `time_stats` 节点 | — |
| ~~物理量+单位系统贯通~~ | ✅ 已落地：通道物理量语义（声压 / 加速度 / 速度等 11 种 + 传感器灵敏度自动换算，v1.32）+ 独立「通道模板」页（v1.34）| — |
| **报告导出**：PDF / Word / PPT，4 个内置模板 | | 不是节点，是平台级导出能力 |
| **多运行 overlay 对比** | | 不是节点，是 Viewer 增强 |

### 2026 H2 第二波（7-9 月）—— Order Tracking 战役

> 旋转机械分析核心，汽车 NVH 客户的"够用门槛"。

| 节点 | 说明 | 价值 |
|---|---|---|
| `tach_input` | 转速通道输入（Tachometer 信号解析）| RPM 处理基础 |
| `rpm_resample` | 角度域重采样（按转速插值到等角度采样）| 阶次分析前置 |
| `order_spectrum` | 阶次谱（频率轴 → 阶次轴）| 旋转机械经典分析 |
| `campbell_plot` | Campbell 图（阶次 vs RPM 三维）| NVH 演化可视化 |
| ~~`order_tracker`~~ | ✅ 已落地为 `order_tracking` 节点（阶次跟踪）| — |
| `runup_detect` | Run-up / Coast-down 工况识别 | 自动从信号中识别启停段 |
| `psychoacoustic_vs_rpm` | 心声指标 vs RPM（在现有 loudness / sharpness / roughness 接入 RPM 横轴）| 心声-工况联合 |
| `can_signal_input` | CAN 总线信号接入（简版）| 整车信号集成 |
| `obd_input` | OBD-II 信号接入 | 整车信号集成 |

### 2026 H2 第三波（10-11 月）—— System 大类 + PdM 抢跑

| 节点 | 说明 | 价值 |
|---|---|---|
| `frf_estimate` | 频响函数估计（H1 / H2 / Hv 三个估计器）| 系统辨识基础 |
| `coherence` | 相干函数 | 系统线性度判断 |
| `cross_spectrum` | 互谱 | 信号对分析 |
| `auto_spectrum` | 自谱 | 信号自相关分析 |
| ~~`envelope_spectrum`~~ | ✅ 已落地为 `envelope_demod`（包络解调，轴承故障检测）| — |
| `spectral_kurtosis` | 谱峭度 | 故障频段识别 |
| `cepstrum` | 倒谱 | 传动系统 / 语音分析 |
| `iso10816_judge` | ISO 10816 振动等级判定 | 工业振动标准合规 |

### Should Have（视进度补）

| 节点 | 说明 | 状态 |
|---|---|---|
| ~~`modulation_spectrum`~~ | ✅ 已落地（调制谱分析，自定义表单标杆）| 已发布 |
| `fluctuation_strength` | 心声波动强度 | 已有研究材料，产品化成本低 |
| `wavelet_transform` | 小波变换 | 同上 |

### 2026 H2 不做（Won't Have）

明确告知客户**短期不做**的：

| 类别 | 节点 | 不做原因 |
|---|---|---|
| **EMA / 模态分析** | `ema_modal` 等系列 | 重型功能，2027 Q1 评估 |
| **TPA / 传递路径分析** | `tpa_analysis` 等 | 同上 |
| **建筑声学** | `rt60` / `sti` 等 | 不是核心客户群 |
| **Beamforming / 声学相机** | 相关节点 | 需要硬件配合，不做纯软件 |

### 2027 主线 1 — Tinia Production 配套节点（规划中）

> 在线产线版（**Production，路线图 / 规划中**）的新节点形态（流处理 + 集成）。
>
> 💡 注意：底层**实时数据流能力已部分先行落地**——SDK 流式会话（v1.35）+ 声级计 / 频谱 / 倍频程跨窗无缝逐帧（v1.22）已让"边采边算"在 Pro/Server 上可用；下列偏向"产线集成 / 设备接入 / 推送"方向的节点仍在规划。

| 节点 | 说明 |
|---|---|
| `realtime_stream_input` | 实时传感器流输入（MQTT / OPC UA / Modbus 等）|
| `online_window` | 在线滑动窗口处理 |
| `threshold_alert` | 阈值告警节点（超限触发通知）|
| `email_notify` / `sms_notify` / `webhook_call` | 告警推送节点 |
| `mes_writer` | 写入 MES / ERP 系统 |
| `database_writer` | 通用数据库写入（PostgreSQL / TimescaleDB 等）|
| `dashboard_realtime` | 实时仪表盘输出 |
| `model_inference` | 模型推理节点（PyTorch / ONNX 等）|
| `model_ota_pull` | 从云端拉取最新模型 |

### 2027 主线 3 — 商店外部贡献节点（设想）

商店开放后，预期来自高校 / 独立开发者的节点类型：

| 类别 | 例子 | 来源 |
|---|---|---|
| **行业垂类** | 风电特有指标 / 工业泵特有指标 / 高铁特有指标 | 高校 + 行业研究院 |
| **先进算法** | 论文里的新心声指标 / 新故障检测方法 | 学术发布 |
| **标准方法** | GB / ISO / SAE 等标准节点（按编号） | 检测中心 / 独立顾问 |
| **特定硬件接入** | 某品牌传感器 / 数采卡 SDK 节点 | 硬件厂商 / 集成商 |

商店生态形成后，节点数量会按月递增。

### 路线节点总览（按时间线）

```
2026 H1（已完成）：约 41 个 Tinia_nodes Python 节点 + 主仓内置 Go 节点
                  含心声指标全家、振动节点族、多尺度谱、调参/评分链路、看板可视化
   ↓
2026 H2 W1-W2：第一波（STFT / spectrogram / Welch PSD）+ 基础设施增强（多已先行落地）
   ↓
2026 H2 W3-W4：第二波（Order Tracking 战役，order_tracking 已先落地）
   ↓
2026 H2 W5-W6：第三波（System + PdM，envelope 已落地）
   ↓
2026 Q4：累计节点持续增长
   ↓
2027 H1：Tinia Production 配套节点（规划中）
   ↓
2027 H2：商店开放，外部贡献节点持续涌入
   ↓
2027 Q4 目标：商店上节点数量持续增长
```

详细 H2 三波交付时间表见 [`08-roadmap.md`](./08-roadmap.md)。

---

## 节点商店（Tinia_Store）

### 是什么

独立部署的 web 服务（`tinia.bestfunc.com`），让节点开发者把自家节点发布出去给其他人订阅安装。

### 当前状态（2026 H1）

- ✅ 商店基础架构上线
- ✅ Tinia 主仓的"商店浏览 / 订阅" UI 上线
- ✅ Tinia 主仓的"我的提交 / 审批历史" UI 上线
- ✅ 商店端的"运营审批 + 详情查看" UI 上线
- ✅ 实例激活 + Seat 体系上线
- 🔜 对外开放注册（当前仅内部用，2026 H2 内部跑通 → 2027 对外）

### 节点发布的两种范围

| 范围 | 谁能看 / 装 | 适合 |
|---|---|---|
| **public**（公开）| 所有 Tinia 用户 | 通用算法、研究成果 |
| **org_only**（私有组织池）| 仅某 Org 成员 | 公司内部专有算法 / 工艺 |

### 节点发布的两种付费

| 模式 | 说明 |
|---|---|
| **免费** | 所有人可装；开发者无直接收益（但累积下载量 / 评分是简历加分）|
| **付费** | 一次性买断 / 月订阅 / 年订阅；商店分成 30%，开发者拿 70% |

### 发布流程

```
1. 开发者在 DevStudio / CLI 写节点 → 自己跑测试
2. 调 plugin_publish_submit → 上传 bundle 到商店
3. 商店运营审批：算法正确性 + 性能 + 安全（不调外部 API 泄露数据）
4. 通过 → 上架；拒绝 → 反馈意见 → 开发者改 → 重提
5. 用户浏览商店 → 订阅
6. 主仓后台拉 bundle → 自动安装到 plugins/
7. 节点可用，UI 出现在节点选择器里
```

### 节点版本管理

- 节点有 `version` 字段（semver）
- 用户订阅默认锁定主版本（如 1.x），新 minor / patch 自动更新
- 大版本（2.0）需要用户确认升级（可能破坏兼容）

---

## 开发者激励

### 三类节点贡献者

| 贡献者 | 动机 |
|---|---|
| **学术研究者**（高校 NVH 实验室） | 研究成果获得行业曝光 + 简历 + 微小副业收入 |
| **独立工程师 / 资深咨询顾问** | 把"师傅经验"产品化 + 被动收益 |
| **公司内部节点维护者** | 公司内 org_only 发布给团队用，统一标准 |

### 收益模型

**免费节点**：

- 累积下载量 → 简历加分
- 商店 leaderboard → 行业认可
- 间接：作者可能因此获得咨询 / 培训机会

**付费节点**：

- 30% 平台分成，开发者 70%
- 节点商店为开发者提供独立的销售渠道
- 商店统一处理支付 / 发票 / 客服

### 早期开发者扶持计划（规划中）

- "Top 10 商业节点"专区推广
- 给关键学术开发者签独家协议
- 培训 + 文档 + 技术支持
- 商店运营协助：节点 SEO / 推荐位 / 评分管理

---

## 节点商店的护城河逻辑

```
研究者发布算法 ──→ 商店上架 ──→ Pro 用户订阅
   ↑                                  │
   │                                  ↓
   └─── 影响力 + 收益 ←── 反哺 Pro 用户的能力
```

正循环一旦形成：

- Tinia Pro 比 HEAD/Testlab 算法更新更快（社区贡献）
- 用户碰到新需求能直接从商店找（不用等大厂下个版本）
- 算法生态越累积，对手追赶成本越高（前期 1000 个节点 vs 后期 10000 个）

### 为什么对手追不上

- HEAD / Testlab 的商业模式是"卖大厂自己的软件"，开放商店会侵蚀模块收入
- Ansys / Abaqus 有 user subroutine（自定义代码扩展）但没有"商店"概念
- 真正的 App Store 需要支付 / 分成 / 审批 / 推荐算法的完整基础设施

详见 `06-competitive-landscape.md`、`07-pricing-business-model.md`。

---

## 节点开发者保护

商业化前必须解决：**节点源码完全可读 → 商业开发者不敢来**。

### 当前状态

- Python 节点 `run.py` 是裸源码，用户能直接 cat 看
- 前端 viewer `.tsx` 也是裸源码

### Phase A 保护（v1.25 上线前必做）

- Python `.py` → `.pyc` + bundle 加密
- 前端 viewer minify + 删 sourcemap
- Plugin 激活定期校验（撤销订阅 → 节点立即拒载）

### Phase B 保护（商业节点真正涌入前）

- 商店服务端 Cython 编译流水线（Linux/Mac/Windows × Python 3.11/3.12 矩阵）
- 分发的是 `.so` / `.pyd` 二进制（IDA 级别才能逆）

### Phase C 保护（生态扩大后）

- Remote 节点架构：核心算法跑商店服务端，本地只是壳
- 类似 OpenAI API 模式，算法 0 离开服务器

详见 `Tinia/docs/plugin-protection-design.md`。

---

## 节点开发：从想法到上线

完整流程（用 Tinia + Claude Code）：

### 1. 想法

> "我想做一个'传动系统轴承故障检测'节点"

### 2. 调研

- 看现有节点：`mcp:nodes` 工具列出 `bestfunc/*` 看有没有类似的
- 看商店：浏览同类型节点的实现思路

### 3. 开发

- 用 Claude Code + `quickstart` skill：
  - dev_create_project（新建项目）
  - dev_create_node（scaffold 节点）
  - 写 run.py（Claude 帮写算法 / 引用文献）
  - 写 ParamsForm.tsx + Viewer.tsx
- 整个过程 AI 主导，开发者审 + 调

### 4. 测试

- dev_reload 让 Tinia 装载
- 用 `flow-author` skill 搭测试流程
- flow_run 跑测试，看 outputs 是否符合预期

### 5. 打包

- `pack-and-publish` skill：dev_bump_version → dev_export → 打包

### 6. 提交

- `publish-plugin` skill：plugin_publish_submit → 提交商店
- 等审批
- 通过 → 上架

### 7. 后续

- 用户订阅 → 自动安装
- 用户评分 / 反馈 → 开发者迭代
- 收益按月结算

---

## 跟客户讲节点生态

### "你们节点比 HEAD 少很多怎么办"

> "短期是少，长期会比 HEAD 多。
>
> HEAD 30 年只能自己开发算法，团队规模决定上限。
> Tinia 节点商店是开放生态，全行业开发者贡献。
>
> 2 年后 Tinia 节点数会超过 HEAD，5 年后远超。"

### "怎么保证节点质量"

> "三层保障：
> 1. 商店审批：算法正确性 + 性能 + 安全
> 2. 用户评分：差节点会被自然淘汰
> 3. 官方认证标记：'Bestfunc Certified' 节点是我们额外验证过的"

### "我们想发自家算法节点，怎么发"

> "两种路径：
> 1. **公开发布**：所有人可见可装，我们运营审批通过即上架
> 2. **私有组织池**：只在你们公司内部可见，发布前我们辅助走内部审批
>
> 商业节点 30% 平台分成。我们提供 SDK + 开发者文档 + 技术支持。"

### "节点会不会过气没人维护"

> "商店有'活跃度'指标：作者多久更新一次、用户最近一次安装时间。
>
> 长期无人维护的节点会被标记 → 推荐用替代节点。
>
> 公司用户可以付费'托管节点维护'，让我们帮接管。"

---

## 跟开发者讲节点生态

### "我有研究算法想发，怎么开始"

1. 装 Tinia Desktop（Community 免费版）
2. 装 Claude Code + tinia-desktop plugin
3. 跟 Claude 说"/quickstart" → 跟着步骤走
4. 半天能跑出第一个节点
5. 满意后 plugin_publish_submit 提交商店

### "我能不能用我自己的代码（不想用 SDK）"

- Tinia 节点 SDK（`tinia_runtime`）是必需的 —— get_input / set_output 是 daemon 跟节点通信的桥梁
- 但 SDK 很薄，只是 IO 包装，算法本体完全是你自己的代码
- 可以用任何 Python 库（numpy / scipy / mosqito / pytorch / 自家算法库）
- **SDK 唯一事实源在主仓 `server/sdk/python/`**，通过 go:embed 嵌进 server 二进制；节点 fork 时由 server 注入 `PYTHONPATH` 指向嵌入版，节点本身**不携带也不依赖物理 SDK 目录**（消除了过去各节点仓自带 SDK 副本导致的协议漂移）。本地开发可 `tinia sdk install --target ./sdk/python/` 拷一份做 IDE 类型提示（gitignore，不进发布）。

> ⚠ 这里的"节点 SDK（`tinia_runtime`）"是给**节点作者写 run.py** 用的运行时契约；跟下文给**外部程序调用平台算力**用的 `tinia_sdk` 客户端是两条不同的东西，别混淆。

### 外部程序调用平台：`tinia_sdk` 通路（v1.33+）

> 这是"节点生态"之外的另一条对外能力——把平台上已调好的分析当成 API 给外部 Python 程序调用。

- 超管「SDK 管理」输入名称即生成可下载 SDK 包，**凭据（license）+ 服务器地址已内置，零配置**；鉴权走打包进 SDK 的 `license.json`（`license_id` + `secret`），不是 api_key。
- 调用方式：① 直接传节点类型 + 参数；② 用节点表单「复制参数」拿到的参数串；③ 引用平台流程里调好的节点（平台改参自动生效）；整条流程也能整体调用（「API 输入」+「API 输出」节点）。
- 传数据便利：直接传文件路径（wav/csv/npz/tdms）或内存数组自动上传；同机调用自动走本地直连 + 路径直传（甚至 Unix domain socket），大文件显著更快。
- **流式会话 / 实时数据流（v1.35）**：可持续往流程推数据、实时取回计算结果，适合在线 / 边采边算场景；实时直调跳过缓存、走内存中转，响应更快。
- **SDK 调用分析（v1.35）**：超管可查看每个 SDK 的调用量、成功率、耗时、Top 节点、最近失败。
- 配套加速：**常驻执行池（v1.35）**让分析节点进程常驻待命（只加载一次），SDK 高频 / 实时调用直接复用热进程，省掉每次 fork+import 的固定开销；超管「常驻执行」页可配上限 / 空闲回收 / 预热白名单。
- 节点帮助新增「SDK 说明」标签页，官方 40 个节点已全部补齐示例代码。

详见 `04-key-concepts.md` 的 SDK / 常驻执行章节。

### "节点商店分成怎么计算"

- 用户支付 → 商店收 30% → 开发者拿 70%
- 月度结算到开发者账户
- 跨币种（USD / CNY 等）自动转换
- 详情见商店开发者后台

---

## 下一步

- 看节点的具体技术结构 → `04-key-concepts.md` 的 Node 部分
- 看节点开发实战 → `quickstart` / `create-node` skill
- 看 Tinia_Cli 的命令行开发 → `Tinia_Cli/README.md`
- 看节点保护设计 → `Tinia/docs/plugin-protection-design.md`
- 看商店 API → `Tinia_Store/README.md`
