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

> Tinia 官方节点（`bestfunc/*` namespace），约 33 个。覆盖核心声学 / 振动算法 + 数据源 + 可视化。

### 数据源 / 输入

| 节点 | 说明 |
|---|---|
| `dataset_node` | 通用数据集加载（接 datasource）|
| `signal_generator` | 合成信号（白噪声 / 正弦 / 扫频，多通道）|
| `materialize_node` | 把 lazy 数据集固化到 blob |

### 信号处理

| 节点 | 说明 |
|---|---|
| `audio_segment_split` | 按时间 / 段长切割音频 |
| `active_segment` | 自动检测有效段（活动段提取）|
| `convergent_trim` | 收敛剔除（去除异常段）|
| `weighting_filter` | A/B/C 计权滤波 |
| `fir_filter` | FIR 数字滤波器 |
| `channel_split` | 多通道拆单通道 |
| `channel_select` | 选择部分通道 |

### 频域分析

| 节点 | 说明 |
|---|---|
| `fft_spectrum` | FFT 频谱 |
| `spectrum_smooth` | 频谱平滑 |
| `octave_analysis` | 倍频程 / 三分之一倍频程 |
| `spectrum_viewer` | 频谱图可视化 |

### 时域 / 时频

| 节点 | 说明 |
|---|---|
| `fbank_extract` | Filter bank 特征 |
| `st_features` | 短时统计特征 |

### 心理声学

| 节点 | 说明 |
|---|---|
| `roughness` | 心声 roughness（基于 mosqito）|
| 计划：`loudness`、`sharpness`（Order Tracking 战役配套）|

### 指标 / 可视化

| 节点 | 说明 |
|---|---|
| `level_meter` | 声级表（dBA / dBZ 等）|
| `indicator_viewer` | 指标表格 / 趋势图 |
| `indicator_merge` | 多源指标合并 |
| `matrix_view` | 矩阵可视化 |

### 高级分析

| 节点 | 说明 |
|---|---|
| `cluster_explore` | 聚类探索（PCA / UMAP 降维 + 可视化）|
| `zscore_anomaly` | Z-score 异常检测 |
| `tnr` | TNR（Tone-to-Noise Ratio）|

### 仪表盘

| 节点 | 说明 |
|---|---|
| `dashboard_view` 输出 | 多 Viewer 组合给 Dashboard 用 |

### 节点开发节奏

- 节点列表持续扩充，每月新增 3-10 个
- 2026 H2 三波交付清单详见 `08-roadmap.md`
- 跟 HEAD 134 项笛卡尔积清单对比 → 详见 `06-competitive-landscape.md`

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
