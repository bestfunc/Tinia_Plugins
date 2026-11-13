---
name: shared-python-environments
display_name: 共享 Python 环境
description: 帮管理员查看/创建/调整 Tinia 共享 Python 环境：完整依赖解析与分组预览、目标锁定版本优先的强制合并、节点移出到独立执行、容量 N、后台构建发布与失败重试。阶段 2 起已启动固定 N 共享 worker；本 skill 覆盖运行态查看与 CPU worker 重启，不含模型预热与 GPU 重启。
user-invocable: true
allowed-tools: mcp__tinia__shared_env_list,mcp__tinia__shared_env_get,mcp__tinia__shared_env_unassigned_nodes,mcp__tinia__shared_env_preview_create,mcp__tinia__shared_env_create,mcp__tinia__shared_env_preview_move,mcp__tinia__shared_env_move_nodes,mcp__tinia__shared_env_preview_remove,mcp__tinia__shared_env_remove_nodes,mcp__tinia__shared_env_update,mcp__tinia__shared_env_delete,mcp__tinia__shared_env_job_get,mcp__tinia__shared_env_job_cancel,mcp__tinia__shared_env_job_retry,mcp__tinia__shared_env_runtime_status,mcp__tinia__shared_env_restart
metadata:
  version: "1.6.0"
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

阶段 2 起：

- 会做：依赖解析、分组/合并/移除预览、不可变修订、成员归属、后台构建验证与原子发布；**每环境按容量 N 启动固定数量的共享 worker**，同一 worker 按可信 node key 串行分发多节点；未绑定节点仍走 one-shot。
- 容量 N 是严格上限：满载排队、不回落 one-shot、不临时 N+1。空成员环境 0 worker。
- 不会做：模型预热/流程预热（P3）、GPU sidecar 重启（P3）、管理 UI（P4）。`shared_env_restart` 只重启 CPU worker，返回里会说明范围。
- 验证结果分：依赖（安装 + pip check + 锁核对）、导入（按 metadata 的 top_level 真实导入）、语法（`py_compile`）、共享入口快照/预检。**语法/依赖通过不等于算法兼容已验证**。
- 同名纯 Python 私有 helper 由每节点唯一前缀命名空间隔离，可同环境共存；同名私有原生扩展、反射裸 `sys.modules`、外部文件加载等明确不支持共享执行，会在构建/发布前预检阶段说明并阻止。

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
   → status 与归属分开读：status=succeeded 只表示本次运行态步骤成功；
     status=failed/cancelled/stale 也不等于已回滚。detail_json.attribution 可能
     不存在（尤其构建/验证早期失败）；缺失不等于已证实原归属未变，必须按实际
     status、binding/current revision 与返回字段判断。只有显式
     attribution="unchanged" 才表示原归属未变；attribution="published" 时 DB
     已发布、归属以新修订为准、环境可能 degraded，不能说成已回滚。
     可提议 shared_env_job_retry，但不要自动重试。

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
  → 发布时同一事务把 binding 从原环境迁到目标；失败/取消先读 detail_json：
     只有显式 attribution="unchanged" 才说明原归属保留；字段缺失时不要认作
     unchanged，需读 binding/current revision 判断。attribution="published" 时
     归属以新目标修订为准、环境可能 degraded，不能说成已回滚。

shared_env_preview_remove {env_id, nodes}
  → 返回剩余成员计划 + 每个被移除节点自己的独立环境计划（independent）。
  → independent 里有 resolved=false 就不能确认。

shared_env_remove_nodes {env_id, preview_id, preview_fingerprint?, approve_target_changes?}
  → 剩余成员与被移除节点的独立环境都构建/验证成功才发布。
  → 失败/取消先读 detail_json，与 move 同一规则：只有显式 attribution="unchanged"
    才说明原 binding 保留；字段缺失时需读 binding/current revision 判断。
    attribution="published" 时归属以新修订为准、环境可能 degraded，不能说成已回滚。
```

## 其它操作

- 改名/容量：`shared_env_update {env_id, name?, capacity?}`，capacity 1..1024；不改成员、不生成修订。
- 删除：`shared_env_delete {env_id}`，**只能删空环境**（无成员、无活动 job）。有成员时先 move/remove。
- 取消：`shared_env_job_cancel {op_id}`。已成功的 job 会被拒绝，不能把"已发布"说成"已取消"。
- 重试：`shared_env_job_retry {op_id}`，只对 failed/cancelled/stale；会新建修订和新构建目录，并重新校验节点快照与原归属。


## 阶段 2：运行态与重启

```
shared_env_runtime_status {env_id?}
  → 每环境 state、capacity、actual_slots、ready/busy/starting/dying/backoff/degraded、
    queue_depth、oldest_wait_ms、reserved_env/reserved_g、active_op、last_error、
    gpu_scope="excluded"、gpu_kernel_unpinned。
  → 只有全部槽位 ready 且无缺槽/漂移才显示 ready；否则是 degraded/过渡态。

shared_env_restart {env_id?} 或 {env_ids:[...]}   // 空 = 全部 CPU 共享环境
  → 返回 op_id；串行、每环境独立结果。
  → 用 shared_env_job_get 读 env_ids / results：
     succeeded / partial / failed / cancelled，以及每环境 skipped_busy 等原因。
     restart 不改变节点归属，job detail 不含 attribution；归属按 results 与
     shared_env_runtime_status 回读，不要承诺读取不存在的 detail_json.attribution。
  → 有进行中的构建/切换 op 的环境逐项记 skipped_busy，其余环境继续；
    总体为 partial（单环境停旧前取消可返回 cancelled），不会伪装成全部成功。

重启范围
  - 只重启 CPU 共享 worker；GPU sidecar/宿主不在本阶段，返回中明确 excluded。
  - 取消：停旧之前可撤销；已开始停旧后必须完成恢复，只有环境真正 ready 才显示 cancelled，
    否则 failed。不要把"取消"说成已回滚。
```

## 常见运行态解释

| 读数 | 对用户说 |
|---|---|
| `degraded` + `backoff_slots>0` | 某 worker 启动/补位失败，正在退避；队列会等，不会回落 one-shot |
| `gpu_scope="excluded"` | 本阶段 restart 只覆盖 CPU worker；GPU sidecar 未重启 |
| `gpu_kernel_unpinned=true` | 该环境有 GPU kernel 节点，但 kernel 仍由 sidecar 按原路径加载，P3 才钉住 |
| `dying_slots>0` | 旧进程已 kill、尚未回收，仍计入 N；此时不会起第 N+1 个 |
| `queue_depth>0` | 环境满载，任务在排队；可取消对应流程（`oldest_wait_ms` 是最老等待） |
| `reserved_env/reserved_g>0` | 有流式 run 预占容量；切换/restart 前需要先结束或取消该流程 |
| `scaling_down=true` | 正在缩容，actual 可能暂时大于目标 N；不要按目标 N 谎报已就绪 |

## 不能做的事

- 不要自己拼 requirements、改节点源码或改节点原依赖声明；平台只记录有效覆盖。
- 不要跳过 preview 直接 confirm；confirm 只接受服务端 `preview_id`。
- 不要把 `preview` 的候选 `id` 或 `fingerprint` 当作可以手改的字段。
- 不要在阶段 2 声称模型已预热或 GPU 已重启（P3）；只描述 worker 是否 ready。
- 不要把 restart-all 的 partial 说成全部成功；逐环境读 detail_json。
- 遇到 stale（409）就重新 preview，并把变化点告诉用户，不要反复重试同一个 preview_id。

## 常见错误怎么说

| 返回 | 对用户说 |
|---|---|
| 409 计划已过期 | 节点版本/依赖/原归属或目标修订变了，需要重新预览 |
| 409 必须显式批准 | 目标环境锁定版本会变，列出受影响成员，等用户明确同意 |
| 404 | 环境或 job 不存在，刷新列表 |
| 403 | 缺 scope 或不是超管，请管理员授权 |
| job failed | 先看 status/detail_json：构建/验证失败时归属未变；若 `attribution:"published"` 则是 DB 已发布但 runtime 提交/恢复失败，归属以新修订为准、环境 degraded，不要说成已回滚 |
| pip < 22.2 | 解析需要带 pip>=22.2 的目标解释器；不要改系统 pip |
