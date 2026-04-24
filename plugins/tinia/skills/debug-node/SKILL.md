---
name: debug-node
display_name: 调试节点运行错误
description: dev_reload 失败或节点测试出错时的排查流程
user-invocable: true
allowed-tools: mcp__tinia__dev_reload,mcp__tinia__dev_tail_logs,mcp__tinia__dev_read_file,mcp__tinia__dev_write_file,mcp__tinia__dev_tree,mcp__tinia__nodes_describe
---

# 调试节点运行错误

## 错误分两类

### A. `dev_reload` 返回 `status: error`（注册期错误）

这是 Tinia 启动节点前的静态检查错误，**节点根本没跑起来**。常见原因：

1. **yaml 解析失败** → node.yaml 或 tinia-repo.yaml 格式有误（缩进 / 冒号 / 引号）
2. **依赖装不上** → requirements.txt 里写了不存在的包；或包名拼错；或 pip 拉镜像超时
3. **entrypoint 路径不对** → node.yaml 里 `runtime.entrypoint: runtime/run.py`，但实际文件不在
4. **模块冲突** → tinia-repo.yaml 里声明的 credential/datasource id 和其他插件重名

#### 排查步骤

1. 看 reload 响应的 `error` 字段 —— 通常直接说是哪个节点哪一步失败
2. `dev_tree(project_id)` 确认文件结构完整
3. `dev_read_file` 读 node.yaml 和 tinia-repo.yaml，用 YAML parser 人肉过一遍
4. 读 `runtime/requirements.txt`，检查依赖是否合理
5. 修完再 `dev_reload`

### B. reload 成功但运行时报错（运行期错误）

节点已注册，Tinia Web UI 流程里跑时才失败。这时需要看运行日志：

#### `dev_tail_logs` 占位说明

v1.19 **暂未**实现节点运行日志持久化（tool 返回占位信息）。

**替代方案**：让用户去 **Tinia Web UI → 运行详情页**，看节点卡片底部的错误信息 + stderr 文本；把关键错误粘贴回对话里给你分析。

#### 常见运行期错误

**`KeyError: 'data'`**
```python
data = rt.fetch_blob(rt.task["inputs"]["data"])
```
说明输入端口 `data` 没接。要么节点 yaml 里 data 是 `required`（此时 Tinia 启动前应该已经拦截），要么写成了 `optional` 但 run.py 里没判空。

修法：
```python
data_handle = rt.task["inputs"].get("data")
if not data_handle:
    rt.emit_error("必须连接 data 输入")
    return
data = rt.fetch_blob(data_handle)
```

**`ModuleNotFoundError: No module named 'xxx'`**
加到 `runtime/requirements.txt`，重新 `dev_reload`（reload 会重装 venv）。

**`json.decoder.JSONDecodeError`**
`rt.fetch_blob()` 回来的不是 JSON —— 可能是二进制数据（音频文件），要按字节处理，不要 json.loads。

**节点永远 running 状态**
run.py 可能没调 `rt.emit_done()` 就退出了。加上。

**下游节点说 "上游失败"**
看上游节点的错误；如果上游正常但下游还是拿不到 handle，检查 `rt.emit_output(port_key, ...)` 的 port_key 是否和 node.yaml 的 outputs key 一致。

## 交互风格

1. 用户报错时**先问是 reload 错误还是运行时错误**（影响排查路径）
2. 引导用户用具体的错误消息，别泛泛地"跑不起来"
3. 改代码前**先诊断**，避免瞎改把问题搞复杂
4. 每次改完回 `dev_reload`，报错变了就是进展

## 相关 Skill

- 节点字段不对 → `node-yaml`
- SDK 方法用错 → `sdk-reference`
- 节点端口重新设计 → `modify-ports`
