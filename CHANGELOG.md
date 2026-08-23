# Changelog

## v1.4.0 — 2026-08-11

跟进 Tinia v1.45「判别函数从 AutoML 剥离 + 质检落地三件套」，同步 Skill 文档。

**修掉一个一直是错的字段名**

- `node-yaml`：AutoML 角色声明里写的是 `labelable`，实际字段是 **`data_source`**。写错会被 yaml 解析静默忽略 —— 节点不报错，但在 AutoML 的「标签来源」下拉里根本不出现。同时补上漏掉的 `item_split` 和 `eval_columns`（限值直评）。

**新增能力的文档**

- `node-yaml`：新增 `skip_side_effects_in_trial` 字段说明 —— 写库 / 发消息类插件节点声明后，AutoML 搜参时会收到 `rt.skip_side_effects`，跳过副作用但**照常输出透传**（整个跳过节点会让下游断流）。
- `flow-author`：「AutoML 评分预测部署」整节重写为**训练流程 + 检测流程两张图**，含 `model_artifact` 制品库读写、`flow_record` 检测记录，以及三个最容易踩的坑（忘接 dataset 端口 / 不填 positive_label 导致分数方向反转且不报错 / 特征池接错位置）。

**跟着删掉的节点走**

- `baseline_stats` / `model_assemble` 已删除，`zscore_anomaly` v2.0.0 改为 fit/apply 双模式（接不接 `baseline` 输入就是开关）。`node-test` 的测试模板、`types-reference` 的类型来源与链路图同步改写。
- `score_predictor` 更名「评分器」v2.0.0：三模式 + 单类算法 + 模型走端口而非粘 JSON。旧的「判别函数 tab / 复制 JSON / → 创建评分节点」入口已从平台移除，文档相应删除。
- `params-form` / `result-view` 的参考实现改指 `modulation_spectrum`（标杆）—— 原先指的 `score_predictor` / `chart_viewer` 是**内置 Go 节点**，表单在主仓，插件仓里根本读不到。
- `about-tinia` 百科同步：节点清单补 `acoustic_fingerprint` / `model_artifact` / `flow_record` / 特征池，术语表新增「评分器 / 制品库 / 检测记录」词条。

## v1.0.0 — 2026-04-24

首版。配合 Tinia v1.19 的 MCP + OAuth 能力发布。

- `plugin.json` OAuth 2.1 自动授权（Tinia Server 动态客户端注册 + PKCE）
- 11 个 Skill 覆盖插件作者全流程：入门 / 类型 / SDK / yaml 字段 / 节点 CRUD / 调试 / 打包 / 数据源插件 / 模板选择
- MCP Tool 集：`dev` 模块 16 个 + `nodes` 模块 3 个（只读）
