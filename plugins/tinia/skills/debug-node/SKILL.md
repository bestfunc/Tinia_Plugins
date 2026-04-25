---
name: debug-node
display_name: 调试节点运行错误
description: dev_reload 失败或节点测试出错时的排查流程
user-invocable: true
allowed-tools: mcp__tinia__dev_reload,mcp__tinia__dev_compile,mcp__tinia__dev_build_status,mcp__tinia__dev_build_history,mcp__tinia__dev_tail_logs,mcp__tinia__dev_read_file,mcp__tinia__dev_write_file,mcp__tinia__dev_tree,mcp__tinia__nodes_describe
---

# 调试节点运行错误

## ⚠ 工具职责必须分清（v1.19 引入新工具后）

| 工具 | 干什么 | 耗时 | 何时用 |
|---|---|---|---|
| `dev_reload` | 重装 Python venv（装 requirements.txt）+ 重新注册 node.yaml + 编译 ui/*.tsx | 可能几十秒到几分钟（pip install 慢）| **改了 run.py / node.yaml / requirements.txt / tinia-repo.yaml 必须用** |
| `dev_compile` | 只编译 ui/*.tsx 一个文件几毫秒 | < 1s | 改了 ui/ParamsForm.tsx / Viewer.tsx 等纯前端时；快速验证 UI 编译 |
| `dev_build_status` | 查最近一次编译结果 | 即时 | 想知道 UI 是否编译通过、错在哪一行 |
| `dev_build_history` | 列最近 N 次编译摘要 | 即时 | 回看历史 |

### 三条铁律

1. **dev_compile 不能替代 dev_reload**。如果你改了 Python 代码 / 节点 yaml / 依赖，只调 dev_compile 是不会让节点生效的 —— 用户测试时会发现节点行为没变化。
2. **dev_reload 报错不要"绕开"**。yaml 解析失败、依赖错误、migration 失败 —— 这些是真正需要解决的问题。**先告诉用户报错内容**，让用户决定怎么办，而不是切换到 dev_compile 假装一切正常。
3. **dev_reload 已经包含 dev_compile**。dev_reload 末尾会顺带编译 UI。所以你不需要在 reload 之后再调 compile —— 看 reload 返回体的 `build` 字段就有完整编译结果。

### `dev_reload` "streaming HTTP 错误"的诊断

如果 MCP 客户端报 `streaming HTTP error` / `connection closed` / `request timeout`，**不一定**是服务端失败。reload 是同步任务，server 端可能已经完成但响应没传回客户端（客户端 HTTP/MCP 超时）。

**不要瞎猜原因**（"可能是 pip install 卡住"是常见误判 —— dev_reload 不会触发 pip install，pip install 只在节点真正运行时才发生）。

**正确做法**：服务端是否实际完成，用三个独立信号判断：

| 信号 | 工具 | 判断 |
|---|---|---|
| UI 是否已编译 | `dev_build_status(project_id)` | 返回最近一次 build，status=ok 说明 reload 已经走到末尾的 Compile 步骤，节点注册大概率也成功了 |
| 节点是否注册到 namespace | `nodes_list(namespace=<your_ns>)` | 看新节点是否在列表里 |
| 用户在 Web UI 流程编辑器节点面板能否找到 | （让用户验证）| 最直接证据 |

如果三者都说"完成了"，告诉用户"客户端报错但服务端已成功"。如果服务端确实没完成，再分析后端日志。

### 慢 reload 怎么办

reload 一般 < 2 秒。如果反复超时：
- 项目特别大（几十个节点）→ 正常，等
- 第一次有 migration → 可能跑 SQL，有时较慢
- yaml 解析卡死 → 让用户去 server 终端看日志
- **不要重复 reload**：可能服务端正在跑，重复请求只会让锁等待

### 实战决策流

```
用户改了什么？
├─ 只改 ui/*.tsx → dev_compile
├─ 改了 run.py / node.yaml / requirements.txt / tinia-repo.yaml → dev_reload
└─ 不确定 → dev_reload（保险）

dev_reload 报错？
├─ 网络/超时类（pip 拉不下来包）→ 告诉用户原始错误，让用户决定是否重试 / 换源
├─ 语法错误（yaml/python 语法）→ 修代码再 reload
├─ 依赖缺失（缺包名）→ 检查 requirements.txt，更新后再 reload
└─ 不要切到 dev_compile 假装绕过 ❌
```

---

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
