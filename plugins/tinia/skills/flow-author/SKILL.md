---
name: flow-author
display_name: 搭建分析流程
description: 在 Tinia 分析流程模块里搭建测试/分析流程：选数据源 → 加节点 → 连线 → 跑 → 看结果。配合开发者工具用于"开发完插件后立刻搭测试图验证"。
user-invocable: true
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_list_types,mcp__tinia__datasource_list,mcp__tinia__datasource_describe,mcp__tinia__flow_create,mcp__tinia__flow_list,mcp__tinia__flow_describe,mcp__tinia__flow_open,mcp__tinia__flow_batch_edit,mcp__tinia__flow_add_node,mcp__tinia__flow_remove_node,mcp__tinia__flow_set_node_params,mcp__tinia__flow_connect,mcp__tinia__flow_remove_edge,mcp__tinia__flow_replace_node,mcp__tinia__flow_auto_layout,mcp__tinia__flow_run,mcp__tinia__flow_wait_run,mcp__tinia__flow_run_status,mcp__tinia__flow_node_output_preview,mcp__tinia__flow_node_logs,mcp__tinia__flow_runs_list
---

# flow-author —— 帮用户搭分析流程

## 何时用

- 用户说"帮我建个流程测一下 XX 节点" / "搭一个 XX 分析流程"
- 用户在开发者工具刚开发完一个新节点，要"立刻测一下"
- 用户描述了一个分析需求（"我想看 XX 数据的频谱"），需要 AI 把它翻译成节点图

## 标准动作链（每次都按这个顺序）

```
1. nodes_list                 → 看用户有哪些节点可用
2. nodes_describe(目标节点, fields=["meta","ports","params"]) → 端口 + schema（不拉 readme/yaml）
3. datasource_list            → 列候选数据源
4. [必问]                     → 让用户选用哪个数据源（除非用户已明说）
5. flow_create                → 建空流程
6. flow_open                  → 让前端跳转过去（用户跟随模式自动看到）
7. **flow_batch_edit({ops})   → 一次提交所有 add_node + connect + set_params**
                                节点用 alias 引用，连边用 alias 不用 node_id
                                batch 末尾自动 auto_layout，无需单独再调
8. flow_run                  → 异步触发，返回 run_id
9. flow_wait_run(run_id)     → **同步阻塞等结果**（默认 60s，最大 300s）
                                优于自己 ScheduleWakeup 轮询：不用猜延时、绝不漏结果
10. 失败 →
    flow_node_logs(run_id, node_id) → status / error / traceback / **stderr 全文**
    flow_node_output_preview(出错节点) → 已成功输出 + 摘要
    切回 dev_* 修代码 → dev_reload → 回到第 8 步
```

> **不要分散调用 add_node × N + connect × N + set_params × N** —— 用 `flow_batch_edit` 一次提交。
> 11 节点 + 10 边的流程从 22 次工具调用变 1 次，减少上下文切换 + 别名引用避免维护 n1/n2 编号。

## 节点发现：避免一次拉全量

- **不要一上来就 nodes_describe 所有节点** —— `nodes_list` 已经返回 inputs/outputs 简略，足够选型
- 只对**确定要加进流程**的节点调 `nodes_describe`，看完整 params_schema
- **`nodes_describe` 默认带 readme/node_yaml 体积大** —— 用 `fields=["meta","ports","params"]` 精简（看代码风格才需要 docs，且建议单独调）
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

1. `nodes_describe(level_meter, fields=["meta","ports","params"])`
2. `datasource_list` → 选 demo
3. `flow_create({name: "测试 level_meter"})`
4. `flow_open`
5. **一次** `flow_batch_edit`（自动 auto_layout）:
   ```js
   {flow_id, ops: [
     {op: "add_node", class_type: "bestfunc/dataset_node",
       alias: "src", params: {datasource_id: <id>}},
     {op: "add_node", class_type: "bestfunc/materialize_node", alias: "mat"},
     {op: "add_node", class_type: "<user_ns>/level_meter", alias: "ana"},
     {op: "add_node", class_type: "bestfunc/indicator_viewer", alias: "viz"},
     {op: "connect", src: "src", src_port: "out",     dst: "mat", dst_port: "in"},
     {op: "connect", src: "mat", src_port: "out",     dst: "ana", dst_port: "audio_in"},
     {op: "connect", src: "ana", src_port: "level",   dst: "viz", dst_port: "indicator"},
   ]}
   ```
6. `flow_run` → `flow_wait_run`
7. failed → `flow_node_logs(run_id, node_id)` 看 stderr / traceback
   再 `flow_node_output_preview(run_id, node_id, port_key)` 看部分 outputs

## 关键约束（绝不违反）

- **数据源 id 必须从 `datasource_list` 里取**，**不能猜数字**。猜了用户跑流程必然失败。
- **节点 class_type 必须用 full_key**（`bestfunc/level_meter` 不是 `level_meter`），用裸 key 后端会 fallback 到 bestfunc 但模糊，AI 应明确。
- **flow_run 是异步，用 flow_wait_run 等结果**：`flow_run` 返回 run_id 后，立刻调 `flow_wait_run(run_id)` 阻塞等到终态。**不要**用 ScheduleWakeup 自己设延时 —— 流程跑得多快猜不准，要么醒早错过要么醒晚浪费上下文。仅当流程预计 > 5 分钟时才考虑 wakeup 模式。
- **节点位置不要传 x/y**：后端会自动避让已有节点找空白格摆放。除非用户明确指定坐标，否则 `flow_add_node` 不要带 x / y。
- **节点 + 边都加完后必须调一次 `flow_auto_layout`** —— `findFreeSpot` 只保证不重叠，**不保证整齐**。auto_layout 按拓扑分层（入度 0 在第 0 列，下游列 = max(上游列)+1）让画布看着整齐，连线少交叉。**漏调 = 用户看到节点堆成一团**。

## 操作节奏（用户视觉跟随）

前端给 AI 操作做了视觉加成，AI 不需要为了"让用户看清"而手动 sleep：

| 你做 | 用户视觉 |
|---|---|
| `flow_add_node` 添加节点 | 画布上先出现一个**蓝色虚线占位框**（"AI 创建中…"），1.5 秒后渐入真节点 |
| 一次性 `flow_add_node` 多个节点 | 多个占位框一起出现 → 一起渐入，看起来像批量加载 |
| `flow_connect` 连线 | 画布立即出线 |
| 任何 flow_* 工具 | 画布顶部出现"AI 正在编辑此流程"绿色提示条 + 全画布光晕脉冲；2 秒无新操作后淡出 |

**建议节奏**：
- 添加节点和连线**可以紧凑批量**，前端动画会让用户觉得"过程感"刚好
- **不要**在两次 `flow_add_node` 之间无意义地等待 / 调闲工具拖时间 —— 前端已经给你"减速"了
- 但**搭完整套流程后，flow_run 之前**适合调一次 `flow_describe` 自己确认结构，让用户也能稍稍消化前面的操作

## 流程运行后

`flow_run` 在分析流程页面**原地触发**，不会跳到独立运行页。用户继续看到画布上节点状态变化（idle → running → completed / failed），跟手动点"运行"按钮一样的体验。

跑挂时：
1. `flow_run_status(run_id)` → 找到 status=failed 的节点；返回里有每个节点的 `error / traceback / message`
2. `flow_node_output_preview(run_id, failed_node_id)` → 拿完整的 `traceback`（Python 堆栈）+ `error` + `message` + 该节点已成功的输出 ports
3. 切回开发者工具修代码 → `dev_reload` → 回流程**直接 flow_run 重跑**（同一 graph_id 不需要重建流程，新代码已生效）

### 诊断查不到日志？

**没有"节点 stdout 日志"接口**（`dev_tail_logs` 是 dev project 模板调试用，不是流程运行用）。流程节点的运行问题完全通过：
- `flow_run_status` 看每个 NodeRun 的 status / error / traceback / message
- `flow_node_output_preview` 看单节点完整诊断 + 已产生的 output

#### `flow_node_output_preview` 两种模式（按需选）

| mode | 用途 | 返回 |
|---|---|---|
| `summary`（默认）| **看格式对不对** | 每个 port 抽结构摘要：`type / size_bytes / preview.summary / preview.items[前3条] / preview._fields` |
| `full` | **看数据对不对** | 小数据完整 inline；blob 也实际拉内容（默认上限 32KB / port，超过截断 + truncated:true 同时给 summary 兜底） |

**典型诊断节奏**：
1. 先 `mode="summary"` 看输出**形状**是不是预期（有没有 indicators 字段、items 多少条、字段列表对不对）
2. 形状对但**数据可疑**时 `mode="full" port_key="result"` 拉具体数据看 value
3. 节点有多 port 时**强烈建议加 `port_key`**，只拉感兴趣的，节省 context

#### 输出为空 / 数据不对

- 让节点临时往 result 里塞 debug 字段（如 `_debug_n_indicators: len(...)` / `_debug_first_item: items[0]`），重跑后 `mode="summary"` 直接看
- **避免反复跑流程做 debug** —— 改一次 run.py，flow_run 一次，从 summary 拿数据，比反复 reload 高效得多

## 与 dev_* 协同

AI 在测试中发现节点 bug（比如 viewer 不显示 / params 报错）：

1. `flow_node_output_preview` 看具体错误
2. `flow_open` 跳回开发者工具（实际跳到流程也 OK，让用户决定）
3. 用 `dev_read_file` / `dev_write_file` 修代码
4. `dev_reload` 让沙箱拉新版（自动触发 dev_compile）
5. 回流程模块，**不需要重建流程**，直接 `flow_run`（同一个 graph_id 重跑用新代码）

修改循环：用户大概率不希望 AI 跨多个模块跳来跳去（尤其旁观模式下），尽量一次修复多个错。
