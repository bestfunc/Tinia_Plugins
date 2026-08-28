---
name: report-author
display_name: 出分析报告
description: 把流程跑出来的结果做成 Tinia 报告：挑样式模板 → 新建 → 把 run 的节点输出插进去 → 补文字说明 → 交付。也含「另存为报告应用」把一次性文档变成可换数据重跑的模板。
user-invocable: true
allowed-tools: mcp__tinia__report_list,mcp__tinia__report_create,mcp__tinia__report_get,mcp__tinia__report_open,mcp__tinia__report_delete,mcp__tinia__report_add_data,mcp__tinia__report_add_element,mcp__tinia__report_regenerate,mcp__tinia__report_save_as_app,mcp__tinia__report_template_list,mcp__tinia__report_template_get,mcp__tinia__flow_runs_list,mcp__tinia__flow_run_status,mcp__tinia__flow_describe,mcp__tinia__flow_node_output_preview
---

# report-author —— 把分析结果做成报告

## 何时用

- 用户说「把这次结果出个报告」/「做份 NVH 分析报告给客户」
- 流程刚跑完（`flow_run` / `flow_wait_run` 之后），结果需要交付给人看
- 用户要「跟上次一样的报告，换一批数据」→ 这是**报告应用**，见 §5

## 先搞清楚一件事：报告是「快照」不是「链接」

`report_add_data` 会把节点输出**物化**——拷贝一份 blob 进报告自己的命名空间。

这意味着：

- 报告一旦做好，**源流程怎么改、怎么删，报告都不变**。交付出去的东西不会在客户打开时变样。
- 反过来，**流程重跑出了新结果，报告不会自动更新**。要更新得走 `report_regenerate`（仅 app 类型）。

先想清楚用户要的是「这一次的定稿」还是「以后能换数据重跑的模板」，再决定 kind。

## 标准动作链

```
1. flow_runs_list(graph_id)        → 找到要用哪次 run（通常是最近一次成功的）
2. flow_describe(flow_id)          → 看图上有哪些节点、哪个是要展示的输出
3. report_template_list            → 有哪些样式模板；按 ratio 过滤（16:9 演示 / a4-portrait 打印）
4. report_template_get(template_id) → **看母版页结构**再决定往哪儿放（见下）
5. report_create                   → 建报告，返回 id + target_path
6. report_open                     → 让前端跳过去，用户能实时看着你往里填
7. report_add_data × N             → 把每个要展示的节点输出插进去
8. report_add_element × N          → 补标题、结论、说明文字
9. [可选] report_save_as_app       → 要复用就转成应用
```

第 4 步别跳。`report_template_list` 只给 id/name/category/ratio 这些画廊字段，**看不到页面里已经有什么**。不看结构就往里塞元素，塞出来的图多半压在母版自带的标题或页眉上——用户看到的是「AI 排的版是乱的」。

## 选模板

| ratio | 用在哪 |
|---|---|
| `16:9` | 投屏演示、评审会 |
| `a4-portrait` | 打印、发 PDF 给客户存档 |
| `a4-landscape` | 打印但图多、要横着放频谱 |

拿不准就问用户「这份是给会上讲的，还是要打印存档的」——比问「你要 16:9 还是 A4」好懂得多。

## 插数据：`report_add_data` 的四个参数

```
run_id     哪次运行
node_id    哪个节点
port       哪个输出端口
kind       auto（默认）/ chart / table
```

`kind=auto` 按节点输出类型自动判定：视图类出 chart，数据类出 table。**先用 auto**，出来不对再显式指定。

不确定某个端口的输出长什么样，先 `flow_node_output_preview` 看一眼再插——插错了要删元素重来，比先看一眼贵。

## 补文字

`report_add_element` 插 text/shape/icon/image。element 至少要有 `type`，其余能省则省：

- 缺 `id` → 自动生成 uuid
- 缺 `z` → 自动置顶
- 缺 `frame` → 自动按画布居中

**居中默认值只适合单个元素。** 连插三段文字都不给 frame，它们会叠在同一个位置。多元素时按母版页结构（第 4 步拿到的）算好各自的 frame。

写结论要基于实际数据说话——`flow_node_output_preview` 看到的数值是什么就写什么，不要编「显著优于基线」这类没有出处的话。

## §5 报告应用：一次做好，反复换数据

`report_save_as_app` 把 document 转成 app：

- 数据元素的 `frozen` 翻成 `false`——绑定变「活的」
- 绑一条流程（`bound_graph_id`）
- 之后 `report_regenerate(report_id, run_id)` 换一次 run，整份报告重新物化

`bound_graph_id` 可以不传，它会从报告里已有数据元素的 `payload.binding.flow_ref.graph_id` **按众数推断**。但如果报告里混了多条流程的数据，推断结果可能不是你要的——**混流程时显式传**。

典型场景：每周给同一条产线出一份报告。第一次老老实实排版，存成 app；之后每周只要 `report_regenerate` 一次。

## 容易踩的

- **报告列表看不全**：`report_list` 不传 `folder_id` 时不限目录（返回全部可见报告），传 `0` 才是只看根目录。和网页版「进哪个目录看哪个目录」的行为不一样。
- **app 类型不能 export 成 `.report`**：`.report` 是自包含文档包，app 是绑流程的蓝图，导出会 400。app 要打包走 `.tsuite`。
- **删报告不会删源数据**：报告里的是快照拷贝，删报告不影响流程和 run。反过来删流程也不影响已做好的报告。
- **别替用户决定发出去**：做好后给 `target_path` 让用户自己看一眼。分享链接、导出 PDF 都是对外动作，等用户开口。
