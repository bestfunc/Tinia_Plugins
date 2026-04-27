---
name: quickstart
display_name: 快速上手 Tinia 插件开发
description: 从零开始建项目、加节点、写 run.py、测试并发布的完整流程示例
user-invocable: true
allowed-tools: mcp__tinia__dev_list_projects,mcp__tinia__dev_create_project,mcp__tinia__dev_create_node,mcp__tinia__dev_read_file,mcp__tinia__dev_grep_files,mcp__tinia__dev_glob_files,mcp__tinia__dev_write_file,mcp__tinia__dev_edit_file,mcp__tinia__dev_reload,mcp__tinia__dev_bump_version,mcp__tinia__dev_export,mcp__tinia__nodes_list_types,mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_read_source
---

# 快速上手 Tinia 插件开发

你将带用户完成从零到一的 Tinia 节点开发。

## 典型对话示例

**用户**：我想做一个节点，输入一段音频，计算它的声级（dBA）。

**你应该这么做**：

### 1. 了解现状
先调 `dev_list_projects`，看用户是否已有项目：
- 有项目 → 问"用哪个项目？"
- 没项目 → 建议新建

### 2. 确认技术决策

在写代码前要确认：
- **项目名**：英文，不含空格 / 特殊字符，如 `acoustic-tools`
- **节点 key**：英文小写+下划线，如 `level_meter`
- **输入输出类型**：调 `nodes_list_types` 看可选值，别瞎编

### 2.5. 【必做】先读官方相似节点的源码

> ⚠ 跳过这步，写出来的 ParamsForm / Viewer **几乎一定**风格不一致，用户大概率会让重写。

```
nodes_list({namespace: "bestfunc"})              # 找类型相似的官方节点
nodes_describe(选中的 key, fields=["meta","ui"])  # 仅 source_files 清单
nodes_read_source(key, "ui/ParamsForm.tsx")      # 抄风格
nodes_read_source(key, "ui/Viewer.tsx") 或 ViewerLoader.tsx
nodes_read_source(key, "node.yaml")              # 学官方 node.yaml 怎么组织
nodes_read_source(key, "runtime/run.py")         # 学 SDK 调用习惯（不抄业务逻辑）
```

详见「create-node」skill 的步骤 0。

### 3. 创建 & 生成骨架

- **`dev_create_project(template_type="empty")`** —— **默认用 empty**，AI 自己 scaffold 节点干净；
  basic_node / analysis_node 模板会带 example 文件残留，做完用户需要手动删，体验差
- `dev_create_node` 单独 scaffold 节点骨架（生成 node.yaml + runtime/run.py + schemas/params.schema.json）

只有当用户**明确说要看 example / 不熟悉结构想从样例改**时，才用 basic_node / analysis_node。详见 「pick-template」skill。

### 4. 改 node.yaml（如果骨架字段不符）

默认骨架是 `inputs.data: AudioData` / `outputs.result: IndicatorData`，对"声级计"场景是合适的，通常不用改。但你应该：
- `dev_read_file` 读 `nodes/<key>/node.yaml`
- 局部修改用 **`dev_edit_file(path, old_string, new_string)`** —— 改一两行别用 dev_write_file 全量重传

参数 schema 在 `schemas/params.schema.json`，如果需要"加权类型"（A/C/Z）这种参数，加到这里。

### 5. 写 run.py 逻辑

- `dev_read_file` 读 `nodes/<key>/runtime/run.py`
- 骨架里有 TODO 注释，按用户需求改（用 `dev_edit_file` 替换 TODO 段，不要全文重写）
- 注意：节点运行时通过 stdin 接收 `{inputs, params}`，用 `rt.fetch_blob(handle)` 拿数据，用 `rt.emit_output("result", handle)` 发输出
- 需要新依赖（如 `librosa / scipy`）要加到 `runtime/requirements.txt`
- 详细 SDK 用法见 「sdk-reference」skill

### 6. 测试

`dev_reload` 把节点注册到用户的个人命名空间（进程内，重启失效，这是正常的）。
然后告诉用户："已注册，你可以打开 Tinia Web UI 的流程编辑器，节点面板里应该能看到带 DEV 标签的 `{namespace}/{key}`，拖到画布上连线测试。如果 reload 失败，我们再调试。"

### 7. 如果 reload 失败

跳转到 「debug-node」 skill。

### 8. 满意后发布

- `dev_bump_version`（1.0 → 1.1）
- `dev_export` 拿到 tar.gz 的 base64 内容
- 告诉用户"请下载后在 Tinia 的"节点仓库"页面点"离线安装"上传"

## 关键原则

1. **不要盲写代码**。先确认端口类型 → 读骨架 → 改 TODO 区域 → 测试。每步告知用户。
2. **每次调工具前告诉用户在做什么**，一句话即可。
3. **类型体系不熟就查**：用 `nodes_list_types` 和 `nodes_describe` 而不是瞎猜。
4. **节点 key 和目录名是绑定的**，改 key 等于删重建，提醒用户。
5. **测试节点只在自己的个人命名空间**，不影响其他同事。
