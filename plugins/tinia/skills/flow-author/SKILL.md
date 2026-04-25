---
name: flow-author
display_name: 搭建分析流程
description: 在 Tinia 分析流程模块里搭建测试/分析流程：选数据源 → 加节点 → 连线 → 跑 → 看结果。配合开发者工具用于"开发完插件后立刻搭测试图验证"。
user-invocable: true
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__datasource_list,mcp__tinia__datasource_describe,mcp__tinia__flow_create,mcp__tinia__flow_list,mcp__tinia__flow_describe,mcp__tinia__flow_open,mcp__tinia__flow_add_node,mcp__tinia__flow_remove_node,mcp__tinia__flow_set_node_params,mcp__tinia__flow_connect,mcp__tinia__flow_remove_edge,mcp__tinia__flow_run,mcp__tinia__flow_run_status,mcp__tinia__flow_node_output_preview
---

# flow-author —— 帮用户搭分析流程

## 何时用

- 用户说"帮我建个流程测一下 XX 节点" / "搭一个 XX 分析流程"
- 用户在开发者工具刚开发完一个新节点，要"立刻测一下"
- 用户描述了一个分析需求（"我想看 XX 数据的频谱"），需要 AI 把它翻译成节点图

## 标准动作链（每次都按这个顺序）

```
1. nodes_list                 → 看用户有哪些节点可用
2. nodes_describe(目标节点)   → 看输入输出端口、参数 schema
3. datasource_list            → 列候选数据源
4. [必问]                     → 让用户选用哪个数据源（除非用户已明说）
5. flow_create                → 建空流程
6. flow_open                  → 让前端跳转过去（用户跟随模式自动看到）
7. flow_add_node × N          → 加 source → 主节点 → viewer
8. flow_connect × N           → 连起来（端口类型自动校验）
9. flow_set_node_params × N   → 配参数（datasource_id 必填）
10. flow_run                  → 跑
11. flow_run_status (轮询)    → 等结果
12. 失败 →
    flow_node_output_preview(出错节点) → 拿到具体报错
    切回 dev_* 修代码 → dev_compile → 回到第 10 步
```

## 节点发现：避免一次拉全量

- **不要一上来就 nodes_describe 所有节点** —— `nodes_list` 已经返回 inputs/outputs 简略，足够选型
- 只对**确定要加进流程**的节点调 `nodes_describe`，看完整 params_schema
- 命名空间：`bestfunc/*` 是官方节点；用户自己开发的节点在 `<user_namespace>/*`

## 数据源 elicitation 措辞模板

调 `datasource_list` 拿到候选后，用以下风格反问用户：

> 当前可用数据源：
> 1. **demo_acoustic**（diffgram，2025-03 创建）
> 2. **prod_audio**（diffgram，2025-04 创建）
>
> 用哪个测试这个流程？

如果用户在 prompt 里已明确（如"用 demo 数据测"），直接选名字匹配的，不再反问。

如果一个候选都没有：

> 当前账号下没有可用数据源。要先去"数据源"模块创建一个，
> 还是先让我用流程里其它节点（如 dataset_random）做合成数据测试？

## 端口连接报错怎么解读

`flow_connect` 失败时常见原因：

| 错误信息 | 原因 | 解决 |
|---|---|---|
| `源节点 X 没有输出端口 Y` | 端口名拼错 | 调 `nodes_describe` 看正确端口 key |
| `目标节点 X 没有输入端口 Y` | 端口名拼错 / 该端口是动态的 | 同上；动态端口看 `dynamic_inputs.prefix` |
| `类型不兼容：A.x (KindA) → B.y (KindB)` | 端口类型不匹配 | 调 `nodes_list_types` 看类型体系，可能要在中间加转换节点 |

## "测试新插件"的典型流程模板

用户刚在开发者工具开发完 `level_meter`，想测一下：

1. `nodes_describe(level_meter)` → 假设它要 `audio_in`（AudioData），输出 `level`（IndicatorData）
2. `datasource_list` → 选 demo
3. `flow_create({name: "测试 level_meter"})`
4. `flow_open`
5. `flow_add_node({class_type: "bestfunc/dataset_node"})` → 拿到 n1
6. `flow_set_node_params({node_id: "n1", params: {datasource_id: <选的 id>}})`
7. `flow_add_node({class_type: "bestfunc/materialize_node"})` → n2
8. `flow_connect(n1.out → n2.in)`
9. `flow_add_node({class_type: "<user_ns>/level_meter"})` → n3
10. `flow_connect(n2.out → n3.audio_in)`
11. `flow_add_node({class_type: "bestfunc/indicator_viewer"})` → n4
12. `flow_connect(n3.level → n4.indicator)`
13. `flow_run`
14. 轮询 `flow_run_status`，failed → `flow_node_output_preview(出错节点)`

## 关键约束（绝不违反）

- **数据源 id 必须从 `datasource_list` 里取**，**不能猜数字**。猜了用户跑流程必然失败。
- **节点 class_type 必须用 full_key**（`bestfunc/level_meter` 不是 `level_meter`），用裸 key 后端会 fallback 到 bestfunc 但模糊，AI 应明确。
- **flow_run 是异步**：返回 run_id 后必须轮询 `flow_run_status`，看到 `completed` / `failed` / `cancelled` 才算结束。中途别假设它做完了。
- **节点位置不要纠结**：默认布局 (200 + i*180, 200) 已够用；不要为了"美观"频繁调用 set_position（且目前没这个工具，位置在 add_node 时定）。

## 与 dev_* 协同

AI 在测试中发现节点 bug（比如 viewer 不显示 / params 报错）：

1. `flow_node_output_preview` 看具体错误
2. `flow_open` 跳回开发者工具（实际跳到流程也 OK，让用户决定）
3. 用 `dev_read_file` / `dev_write_file` 修代码
4. `dev_reload` 让沙箱拉新版（自动触发 dev_compile）
5. 回流程模块，**不需要重建流程**，直接 `flow_run`（同一个 graph_id 重跑用新代码）

修改循环：用户大概率不希望 AI 跨多个模块跳来跳去（尤其旁观模式下），尽量一次修复多个错。
