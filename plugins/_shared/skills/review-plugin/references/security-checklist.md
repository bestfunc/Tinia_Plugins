<!-- 从主仓 docs/plugin-design-spec.md §9 摘出。
     原先 skill 里写的是 ../../../docs/plugin-design-spec.md —— 那个相对路径落在
     plugins/docs/，本仓没有这个目录，插件装到用户机器上更是够不着主仓。
     审批人必读的东西不能依赖一个够不着的路径，所以内联到这里。
     主仓改了规范要同步这份。 -->

## 9. 安全准则（审批人必读）

商店发布的插件会运行在 **服务器进程内** —— 拉 blob、调 Tinia API、写日志。审批前必须做以下安全检查：

### 9.1 危险 import（一票拒绝）

| 模式 | 风险 | 说明 |
|---|---|---|
| `import os; os.system(...)` / `os.popen(...)` | 任意命令执行 | 没有合法用途 |
| `import subprocess; subprocess.run/Popen/...` | 任意命令执行 | 同上 |
| `eval(...)` / `exec(...)` | 动态代码执行 | 唯一可能合法用途是 sandbox 计算器，否则拒 |
| `import pickle; pickle.loads(...)` | 反序列化 RCE | 用 JSON 替代 |
| `__import__(...)` 动态 import 字符串 | 隐藏 import 试图绕检查 | 拒 |
| `compile(...)` + `exec(...)` | 同 eval | 拒 |
| `import ctypes` | 调 native 函数绕安全 | 极少有合法用途，拒 |

### 9.2 网络访问（要看用途）

| 模式 | 准则 |
|---|---|
| `import urllib.request; urllib.request.urlopen(...)` | 任意 URL 访问 |
| `import requests; requests.get/post/...` | 任意 URL 访问 |
| `import socket` | 原始套接字 |

✅ **允许的网络**：
- 调 `rt.task["server_url"]` + `rt.task["api_token"]`（Tinia API）
- 调 `rt.fetch_content_url(...)`（dataset item 内置）

❌ **拒绝的网络**：
- 任意外部 URL（除非声明在 README 里 + 有明确数据来源说明，如官方 NIST 数据集）
- 上传数据到 plugin 作者控制的服务器
- DNS 探测

### 9.3 文件系统

| 模式 | 准则 |
|---|---|
| `open("/etc/...")` / `open("/proc/...")` | 读敏感系统文件，拒 |
| `open("../...")` 路径穿越 | 拒 |
| `os.environ` 读环境变量 | 拒（除非明确说要读 TZ 之类） |
| `tempfile.mkdtemp()` | OK，但要清理 |

✅ **允许**：
- `rt.fetch_blob(...)` / `rt.upload_blob(...)`（走 SDK）
- 临时目录 + 自清理

### 9.4 资源消耗

- 无限循环（while True 没有 break）→ 拒
- 一次性读超大文件到内存（`f.read()` without size limit）→ 警告
- fork / multiprocessing 数百进程 → 拒
- `time.sleep(huge)` → 警告

### 9.5 SDK 边界

- **不允许自己实例化** `Runtime()` 或读自己 stdin —— 必须 `Runtime.from_stdin()`
- **不允许覆盖** `rt.task` / `rt.api_token` / `rt.server_url`
- **不允许**直接调 `requests.post(rt.server_url, ...)` 绕 SDK —— 必须用 `rt.fetch_blob` / `rt.upload_blob`

### 9.6 混淆 / 隐藏

| 模式 | 处理 |
|---|---|
| 大段 base64 字符串 + decode 后 exec | 拒（隐藏代码） |
| 大段 hex/zlib 数据 + decompress 后 exec | 拒 |
| 单字符变量名 + 看不懂的逻辑（明显刻意） | 拒（除非作者说明是数学公式） |
| 特别长的字符串 / 二进制 blob 嵌在源码 | 警告（应该用 schemas/data 文件） |

