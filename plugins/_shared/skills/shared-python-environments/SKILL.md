---
name: shared-python-environments
display_name: 共享 Python 环境
description: 帮管理员查看/创建/调整 Tinia 共享 Python 环境：完整依赖解析与分组预览、目标锁定版本优先的强制合并、节点移出到独立执行、容量 N、后台构建发布与失败重试。阶段 1 只管理环境与归属，不启动共享进程、不切换执行池、不执行模型预热。
user-invocable: true
allowed-tools: mcp__tinia__shared_env_list,mcp__tinia__shared_env_get,mcp__tinia__shared_env_unassigned_nodes,mcp__tinia__shared_env_preview_create,mcp__tinia__shared_env_create,mcp__tinia__shared_env_preview_move,mcp__tinia__shared_env_move_nodes,mcp__tinia__shared_env_preview_remove,mcp__tinia__shared_env_remove_nodes,mcp__tinia__shared_env_update,mcp__tinia__shared_env_delete,mcp__tinia__shared_env_job_get,mcp__tinia__shared_env_job_cancel,mcp__tinia__shared_env_job_retry
metadata:
  version: "1.5.0"
  agentport-product: tinia
---

# 共享 Python 环境

## 何时用

- 用户问「哪些节点在共用 Python 环境」「这个环境现在有哪些成员/锁定了什么版本」
- 用户想把若干节点合成一个共享环境，或把节点从一个环境移到另一个环境
- 用户想把节点移出共享环境、回到独立执行
- 用户要改环境名称 / 每环境进程容量 N，或查看/取消/重试后台构建
- 构建失败、计划过期、目标版本要变化时

## 先读边界（必须对用户说清楚）

阶段 1 **只做后端闭环**：

- 会做：依赖完整解析、分组/合并/移除预览、不可变修订、成员归属、容量 N 持久化、后台 venv 构建与验证、成功后原子发布归属。
- **不会做**：不启动共享 worker、不切换执行池、不改 one-shot 执行路径、不执行模型预热。`capacity` 只是保存配置，目前不会因此多起进程。
- 验证结果分三段：依赖（安装 + pip check + 锁核对）、导入（按 metadata 的 top_level 真实导入）、语法（仅 `py_compile`）。**语法通过不等于节点导入可用**，更不能说成"算法兼容已验证"。

## 权限

- 只读工具需要 `mcp:shared_envs`；变更工具需要 `mcp:shared_envs_write`。
- 所有工具还要求调用者是**超级管理员**；被拒时如实告诉用户缺哪个 scope / 权限，不要重试猜测。

## 标准动作链

先核对当前会话的授权范围。用户已明确授权相应创建、移动或配置调整时，完成预览并在该范围内执行，不重复索要确认；未授权的目标版本变化等额外影响，先展示具体预览再请求批准。审批字段必须反映实际授权，不能为绕过校验而填写。

```
1. shared_env_list + shared_env_unassigned_nodes
   → 读现有环境、成员数、未分配节点；确认用户要动的节点 full_key 与当前 version。

2. 创建：shared_env_preview_create {nodes:[{full_key,version}]}
   → 拿 preview_id、candidates。
   → 向用户解释：每个候选的成员、锁定版本、冲突、requires_approval。

3. 用户确认后：shared_env_create
   {preview_id, candidates:[{id,name,capacity,approve_target_changes}]}
   → 每个候选独立建环境 + 修订 + 后台 job。
   → 若 requires_approval=true，必须同时带 preview 返回的 fingerprint 作为
     preview_fingerprint，并设 approve_target_changes=true；否则会被拒绝，
     不要绕过、不要自己改 plan。

4. shared_env_job_get 轮询 op_id 到终态。
   → succeeded 才算发布；failed/cancelled/stale 读 error/log 如实汇报，
     原归属没有改变。可提议 shared_env_job_retry，但不要自动重试。

5. shared_env_get 读回：当前修订、members（权威归属）、checks 三段。
```

移动与移除同构：

```
shared_env_preview_move {env_id, nodes, mode:"target_preferred"}
  → 默认以目标环境现有锁定版本为准。冲突时生成 override，并记录原要求/
    采用版本/操作人/时间；原 requirements 文件不改。
  → 若目标锁定版本会变化，候选 requires_approval=true 且列出 target_changes
    与受影响成员。逐条向用户说明后再确认。

shared_env_move_nodes {env_id, preview_id, preview_fingerprint?, approve_target_changes?}
  → 发布时同一事务把 binding 从原环境迁到目标；失败/取消保留原归属。

shared_env_preview_remove {env_id, nodes}
  → 返回剩余成员计划 + 每个被移除节点自己的独立环境计划（independent）。
  → independent 里有 resolved=false 就不能确认。

shared_env_remove_nodes {env_id, preview_id, preview_fingerprint?, approve_target_changes?}
  → 剩余成员与被移除节点的独立环境都构建/验证成功才发布；否则原 binding 保留。
```

## 其它操作

- 改名/容量：`shared_env_update {env_id, name?, capacity?}`，capacity 1..1024；不改成员、不生成修订。
- 删除：`shared_env_delete {env_id}`，**只能删空环境**（无成员、无活动 job）。有成员时先 move/remove。
- 取消：`shared_env_job_cancel {op_id}`。已成功的 job 会被拒绝，不能把"已发布"说成"已取消"。
- 重试：`shared_env_job_retry {op_id}`，只对 failed/cancelled/stale；会新建修订和新构建目录，并重新校验节点快照与原归属。

## 不能做的事

- 不要自己拼 requirements、改节点源码或改节点原依赖声明；平台只记录有效覆盖。
- 不要跳过 preview 直接 confirm；confirm 只接受服务端 `preview_id`。
- 不要把 `preview` 的候选 `id` 或 `fingerprint` 当作可以手改的字段。
- 不要在阶段 1 声称"环境已经启动/预热完成/执行已经切换"。
- 遇到 stale（409）就重新 preview，并把变化点告诉用户，不要反复重试同一个 preview_id。

## 常见错误怎么说

| 返回 | 对用户说 |
|---|---|
| 409 计划已过期 | 节点版本/依赖/原归属或目标修订变了，需要重新预览 |
| 409 必须显式批准 | 目标环境锁定版本会变，列出受影响成员，等用户明确同意 |
| 404 | 环境或 job 不存在，刷新列表 |
| 403 | 缺 scope 或不是超管，请管理员授权 |
| job failed | 说明失败阶段（依赖/导入/语法）与日志要点；归属未变 |
| pip < 22.2 | 解析需要带 pip>=22.2 的目标解释器；不要改系统 pip |
