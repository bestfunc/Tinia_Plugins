# 产品路线图

> Tinia 2026 H2 + 2027 的产品规划。给客户讲"什么时候有什么"用，**不写具体收入/客户数目标**（那些是内部 KPI 不对外）。

---

## 当前阶段（2026 上半年完成 / 已就绪）

**v1.x（已发布）**：

- ✅ 节点 DAG 流程引擎（执行 + 调度 + 事件流）
- ✅ 33+ 官方节点（声学、振动、心理声学、滤波等核心算法）
- ✅ React 前端流程编辑器（基于 React Flow）
- ✅ 多通道架构（per_channel / aggregated 双模 + 通道命名模板）
- ✅ Composite DataSource（多源数据虚拟组合）
- ✅ 数据源管理（本地文件 / Diffgram / 信号发生器）
- ✅ 看板编辑器（多 Viewer 组合 + 切片器 + 文本块）
- ✅ DevStudio 节点开发 IDE
- ✅ MCP-native（OAuth 2.1 + 65+ 工具）
- ✅ 节点商店 v1（内部使用，外部开发者下半年开放）
- ✅ Tinia_Store 实例激活体系（OAuth + Seat + 续期）
- ✅ Tinia 桌面单机版 Wails 打包（Windows / macOS）
- ✅ Tinia_Cli 节点开发命令行
- ✅ Tinia_Plugins（Claude Code marketplace 4 个变体 plugin）
- ✅ 多视图 tab + 状态保持（GraphEditor + GraphRun + Dashboard 都进 TabShell）

详见 `Tinia/docs/todo.md`、`client/src/data/changelog.ts`。

---

## 2026 H2（5-12 月，7 个月）—— 三波交付

> 目标：让 Tinia Pro v1.0 "专业感拉满"，能跟传统 NVH 工具同台竞争。Community 版同步发布。

### 第一波（5-6 月，2 个月）—— "这是专业 NVH 工具"

让用户第一眼觉得"这是真的 NVH 工具"，不需要等所有功能。

| 能力 | 状态 | 价值 |
|---|---|---|
| **STFT / Spectrogram + 音频联播** | 计划 | NVH 工程师每天必用，缺得最明显 |
| **物理量 + 单位系统** | 计划 | Pa / g / m/s 全链路贯通，calibration 完整 |
| **报告导出（PDF / Word / PPT）** | 计划 | 4 个内置模板，可定制 |
| **多运行 overlay 对比** | 计划 | 同一指标多次实验叠图 |
| **PSD / Welch + 时域统计套件** | 计划 | RMS / Crest / Kurtosis 等 |

**交付后可以**：开始约客户演示 —— 不需要等所有功能。

### 第二波（7-9 月，3 个月）—— Order Tracking 战役

> 这是汽车 NVH 客户的"够用门槛"。做完才算真正进入 NVH 工具市场。

| 能力 | 说明 |
|---|---|
| **RPM 通道 + 角度域重采样** | 旋转机械分析基础 |
| **Order Spectrum / Campbell / Order Tracker** | 三大经典阶次分析 |
| **Run-up / Coast-down 工况识别** | 自动从信号中识别启停段 |
| **心声指标 vs RPM** | 把现有 loudness / sharpness / roughness 接入 RPM 横轴 |
| **CAN / OBD 信号接入（简版）** | 整车信号集成 |

### 第三波（10-11 月，2 个月）—— System 大类 + PdM 抢跑

| 能力 | 说明 |
|---|---|
| **FRF (H1/H2/Hv) + Coherence** | 频响函数三个估计器 + 相干函数 |
| **Cross / Auto Spectrum** | 互谱 / 自谱 |
| **Envelope Spectrum + Spectral Kurtosis** | PdM 突破，轴承故障检测 |
| **Cepstrum** | 倒谱，传动系统分析 |
| **ISO 10816 振动等级判定** | 工业振动标准 |

### Community 版同步

- ✅ tinia-store 上线（节点市场，先内部用，后期开放）
- ✅ 学生 / 研究者免费授权计划
- ✅ tinia-cli 开发节点的脚手架完善
- ✅ 至少 3 个开源示范节点（放 store 引导生态）

### 商业化基础设施

- ✅ Pro 试用注册 / License Server
- ✅ EV Code Signing（Windows / macOS 安装包签名）—— 进行中
- ✅ 客户成功文档 / 教程视频（B 站 + YouTube）
- ✅ 销售物料：产品白皮书、对标传统工具的功能矩阵、客户案例

### Won't Have（2026 H2 明确不做）

- ❌ **EMA / 模态分析** —— 重型功能，2027 Q1 评估
- ❌ **TPA（传递路径分析）** —— 同上
- ❌ **建筑声学（RT60 / STI）** —— 不是核心客户群
- ❌ **Beamforming / 声学相机** —— 需要硬件配合
- ❌ **Tinia Production 在线版** —— 2027 Q1 启动设计，H2 不碰

### Should Have if Time（视进度补）

- 🟡 Modulation Spectrum + Fluctuation Strength
- 🟡 Wavelet Transform
- 🟡 单机 + 云端双模部署（用 tinia-engine 的解耦架构，边际成本低）

---

## 2027 战略目标 —— 三主线

### 主线 1：Tinia Production（在线产线版）

**为什么 2027 做**：

- 2026 H2 SmartQuality 客户已经在用 BPMN 流程引擎处理产线数据
- Tinia 离线版 2026 H2 跑通后，把流程引擎"在线化"是水到渠成
- 架构上 `tinia-engine` 与 `tinia-runtime` 的分离就是为此准备的

**关键能力**：

- 实时流处理（秒级延迟）
- 多机协同（一条产线 5-20 个工位同时跑）
- 数据库 / MES 集成
- 异常告警 + 仪表盘
- 模型 OTA + 流程热更新

**目标客户**：

- 现有 SmartQuality 客户的升级（从 NG 检测 → NVH 在线监测）
- 风电场远程 CMS（Cloud + Edge）
- 工业泵 / 电机厂 PdM 联合解决方案

### 主线 2：Tinia AI Agent（Félag 与 Tinia 融合）

**为什么 2027 做**：

- Félag 是 Bestfunc 的虚拟员工平台，Tinia 是节点流程平台
- 2026 H2 开始内部融合实验：Félag 给 Tinia 编排流程，Tinia 给 Félag 提供分析能力
- 2027 推出 "Tinia AI Engineer" 形态：**用户描述问题 → AI 自动搭流程 → AI 解读结果 → AI 写报告**

**为什么这是核心壁垒**：

- 传统 NVH 工具是 30 年单机软件，架构上没有 Agent 接口
- Tinia 的 MCP-native + 节点 DAG + tinia-plugins（Claude Code 插件）架构是为此设计的
- 这一条是 Tinia 的核心叙事 —— **"工程师工具" → "AI 工程师"**

**目标场景**：

- 工程师上传一段录音，说"帮我看看这个发动机异常"，AI 自动给出诊断
- 测试报告自动撰写，工程师只做审核

### 主线 3：节点生态与 Store 商业化

**为什么 2027 做**：

- 2026 H2 把 tinia-store 内部跑通
- 2027 对外开放
- 学术界（高校 NVH 实验室）是天然的节点贡献者
- 商业节点采用 30% 平台分成模式（Apple App Store 路径）

**关键里程碑**：

- 2027 Q2：第一批外部节点开发者
- 2027 Q4：Store 上节点数量持续增长 + 付费节点稳定供给
- 形成"研究者贡献算法 → Pro 用户订阅 → 反哺生态"的正循环

---

## 路线图风险与对冲

| 风险 | 影响 | 对冲 |
|---|---|---|
| 客户太忠诚 HEAD，迁移失败 | H2 交付目标落空 | 主推"补充工具"而非"替代"，让 Tinia 跑在 HEAD 旁边 |
| 算法覆盖度短期不及对手 | 对比时承压 | 销售物料强调"AI + 流程"差异化，给客户"我们做核心 80% + 商店补长尾"叙事 |
| Order Tracking 技术难度超预期 | Q3 延期 | 把 v1.25 拆 2-3 个 sprint，先做 Tach 输入 + Campbell 这两个最直观的；算法精度问题后续迭代 |
| Tinia Production 上线远超预期 | 2027 主线 1 落空 | 借力 SmartQuality 现有架构，不重写；v1.0 Production 就是 Tinia Pro + 实时流处理层 |
| 节点商店冷启动困难 | 2027 主线 3 落空 | 先内部用，2026 H2 自己孵化若干节点示范；再开放给高校 / 独立开发者 |

---

## 长期愿景（2028+）

不在当前 commit 范围，仅作方向参考：

- **行业模型库**：积累各行业的标准声学异常模式
- **跨语言 SDK**：除 Python 外支持 Rust / Node.js 等
- **行业垂类 plugin pack**：汽车 NVH pack / 风电 pack / 工业泵 pack 等开箱即用包
- **企业版 Federated Learning**：跨企业训练共享模型，数据不出域
- **国际化深度**：英、日、德、法等语言完整支持，本地化销售渠道

---

## 下一步

- 当前已实现的具体功能 → `04-key-concepts.md`、`12-node-ecosystem.md`
- 技术架构怎么支持未来路线 → `02-architecture.md`
- 商业模式细节 → `07-pricing-business-model.md`
