---
name: review-plugin
display_name: 审批组织内插件提交
description: 帮组织管理员审批待批的插件提交 —— 读元数据 / 检查代码安全性 / 据 Tinia 设计规范给意见 → 反问用户通过/拒绝 → 自动写拒绝理由
user-invocable: true
allowed-tools: mcp__tinia__plugin_review_pending,mcp__tinia__plugin_review_detail,mcp__tinia__plugin_review_read_file,mcp__tinia__plugin_review_open,mcp__tinia__plugin_review_approve,mcp__tinia__plugin_review_reject
---

# 审批组织内插件提交

帮拥有 `plugins_publish_review` 权限的人审批插件提交。AI 的核心价值：**逐文件读源码**，按 [plugin-design-spec.md §9 安全准则](../../../docs/plugin-design-spec.md#9-安全准则审批人必读)做代码审计 + 设计规范检查。

> **本 skill 必读**：[../../../docs/plugin-design-spec.md](../../../docs/plugin-design-spec.md) — 安全准则 + checklist 章节

---

## 典型对话

**用户**：看下我有什么待审批的。

**你应该这么做**：

### 1. 列待审

```
plugin_review_pending
```

返回 `{pending: [{approval_id, plugin_name, version, requester_username, changelog_md, ...}]}`。

如果只有 1 个 → 直接 review；多个 → 报给用户让他选一个开审。

### 2. 在前端打开（让用户跟看）

```
plugin_review_open(approval_id=N)
```

这会让前端跳到「商店 → 待审批」+ 自动打开该 approval 的详情对话框。用户能看到 AI 接下来在 review 什么。

### 3. 拿元数据 + 文件树

```
plugin_review_detail(approval_id=N)
```

返回 `{plugin: {...}, version: {...}, submitter: {...}, files: [...]}`。

AI 检查：
- **`display_name` / `description` / `readme_md`** — 有没有写清楚用途？
- **`category` / `namespace`** — 命名空间合理吗（不会和官方冲突）？
- **`changelog_md`** — 升版本时有没有写清改动？
- **`files`** —— 文件结构是否符合规范（tinia-repo.yaml + nodes/*/node.yaml + runtime/run.py）

### 4. 逐个文件读源码

按这个顺序读：

1. **`tinia-repo.yaml`** — 字段齐？namespace 不是 bestfunc？
2. **每个 `nodes/*/node.yaml`** — key 合法？inputs/outputs 类型对（详见 [§4](../../../docs/plugin-design-spec.md#4-类型系统)）？runtime 字段齐？
3. **每个 `nodes/*/runtime/run.py`** — 重点！按 §9 安全准则审计：
   - 危险 import：`os.system` / `subprocess` / `eval` / `exec` / `pickle.loads` / `__import__`(string) / `ctypes` → ❌
   - 网络：除 `rt.fetch_*` / `rt.upload_*` 外的 `urllib` / `requests` / `socket` → ❌（除非合法说明）
   - 文件越界：`open("/etc/...")` / `open("../...")` → ❌
   - 资源失控：`while True` 无 break / `os.fork` 大量 → ❌
   - 混淆：大段 base64 / hex 字符串 + decode 后 exec → ❌
   - SDK 边界：自实例化 `Runtime()` 而不是 `Runtime.from_stdin()` → ❌
4. **`requirements.txt`** — 有没有可疑包名（typosquatting：`requestss` / `urllib4` 这种打错字的）？
5. **`schemas/params.schema.json`**（如有） — JSON Schema 合法？

```
plugin_review_read_file(approval_id=N, path="tinia-repo.yaml")
plugin_review_read_file(approval_id=N, path="nodes/wavelet_transform/node.yaml")
plugin_review_read_file(approval_id=N, path="nodes/wavelet_transform/runtime/run.py")
plugin_review_read_file(approval_id=N, path="nodes/wavelet_transform/runtime/requirements.txt")
```

### 5. 总结审批意见

读完后跟用户报告 review 结果，**结构化输出**：

```
✅ 设计规范检查
   - tinia-repo.yaml: 字段齐 ✓
   - nodes/wavelet_transform/node.yaml: 字段齐，inputs.data: AudioData 合理 ✓
   - run.py: 入口 from_stdin → emit_done 套路正确 ✓
   - 类型选择: outputs.result IndicatorData 符合分析节点惯例 ✓

⚠ 设计建议
   - readme_md 缺少"已知限制"章节（如采样率范围、最大时长）
   - params_schema.json 没给 weighting 默认值，用户体验稍差

🔴 安全问题
   - run.py:23 有 import subprocess → 用于调 ffmpeg。**该用 librosa.load 替代**，否则审批拒。
   - requirements.txt 含 librosa==0.8.* 范围太宽，建议钉版本

判断：当前版本不能通过 —— subprocess 调用是阻塞性问题，需要作者改后重新提交。

要我帮你拒绝并填好理由吗？
```

### 6. 等用户决定 → 通过 / 拒绝

**通过**：

```
plugin_review_approve(approval_id=N, comment="代码 review OK，无安全问题。建议下版本补 README 限制章节。")
```

通过会立即生效（plugin.latest_version_id 指向新版本 + 节点物理装载到 namespace）。

**拒绝**：comment 必填，自动从 review 结论里写：

```
plugin_review_reject(
    approval_id=N,
    comment="""
拒绝理由：

1. 【安全】run.py:23 用 subprocess 调 ffmpeg —— 不允许（详见 docs/plugin-design-spec.md §9.1）。
   建议改用 librosa.load() 或 soundfile.read()。

2. 【建议】readme 缺"已知限制"章节，建议补充支持的采样率范围。

3. 【建议】requirements.txt 中 librosa==0.8.* 版本范围太宽，请钉到 0.8.1。

修复后重新提交即可（同 version 号 OK，被拒后允许覆盖重发）。
"""
)
```

拒绝后告诉用户："已拒绝，理由会反馈给提交者。"

---

## 关键原则

1. **必读源码** —— 不能只看元数据就批，必须 read_file 每个 node 的 run.py + tinia-repo.yaml + node.yaml。
2. **安全 > 设计 > 风格** —— 优先级：安全问题（拒）→ 设计规范偏差（建议）→ 风格小事（一般不影响通过）。
3. **拒绝必须详细** —— comment 字段要列具体行号 + 建议改法（提交者据此改完重发）。
4. **跟用户确认决定** —— 不要 AI 自己决定通过/拒绝。先报 review 结论 → 用户拍板。
5. **可疑就拒** —— 看不懂的混淆代码、大段未解释的二进制 blob、可疑的网络调用 —— 拒绝（"宁可错杀，不可放过"）。
6. **首版本要严** —— 新插件首发更要看清，因为它会在 namespace 占名 + 上架后才发现问题代价高。

---

## 安全审计 checklist（粘给 AI 用）

每次 review 至少跑一遍：

```
□ tinia-repo.yaml 字段齐（name / namespace / version / min_tinia_version）
□ namespace 不是 bestfunc（不冲突官方）
□ 所有 nodes/*/node.yaml 字段齐（key / name / inputs / outputs / runtime）
□ runtime/run.py 入口 from_stdin → emit_done 正确
□ requirements.txt 无 typosquatting 包名 / 来源可疑包
□ 无 os.system / subprocess / pickle.loads / eval / exec / __import__(str) / ctypes
□ 网络访问仅限 rt.fetch_* / rt.upload_* / Tinia API
□ 无 open("/etc...") / open("../...") 路径越界
□ 无硬编码密码 / API key / token
□ README 写明用途 / 输入输出 / 参数 / 限制
□ 无明显混淆（base64 + exec / hex blob 等）
□ 资源使用合理（无无限循环 / 不失控 fork）
```

---

## 相关 Skill

- 设计规范（必读） → `../../../docs/plugin-design-spec.md`（主仓库）
- 类型体系 → `types-reference`
- SDK 接口 → `sdk-reference`
