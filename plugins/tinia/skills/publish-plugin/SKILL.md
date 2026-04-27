---
name: publish-plugin
display_name: 发布插件到商店
description: 帮用户把开发完的插件提交到组织商店或在线商店 —— AI 起草元数据 / 写 README / 据上版本 diff 写 changelog → 推到前端表单让用户 review → 提交审批
user-invocable: true
allowed-tools: mcp__tinia__dev_list_projects,mcp__tinia__dev_get_project,mcp__tinia__dev_read_file,mcp__tinia__dev_tree,mcp__tinia__plugin_publish_self_check,mcp__tinia__plugin_diff_last_published,mcp__tinia__plugin_publish_draft,mcp__tinia__plugin_my_submissions
---

# 发布插件到商店

帮用户把 Dev Studio 里开发好的插件**提交到组织商店或 Tinia 在线商店**审批。AI 不只是调 API，更要：
- 读代码理解插件功能 → 起草 description / readme
- 对比上一发布版本的 diff → 写 changelog
- 字段不明确 → 反问用户
- 推 draft 到前端 PublishDialog 让用户**看到 AI 填好的字段**，可调整后提交

> ⛔ **铁律**：AI 没有"直接提交"工具。你能做的是把字段推到 PublishDialog（`plugin_publish_draft`），由用户在对话框点「提交审批」按钮。**任何时候不要尝试帮用户提交、不要询问"要不要直接帮你提交"** —— 提交是用户的动作，AI 只负责起草和填字段。

> **本 skill 必读**：[../../../docs/plugin-design-spec.md](../../../docs/plugin-design-spec.md) — 插件设计规范 + README 风格章节

---

## 典型对话

**用户**：帮我把刚写完的 wavelet_analysis 发布到组织商店。

**你应该这么做**：

### 1. 找项目 + 自检

```
dev_list_projects                       # 列项目找 wavelet_analysis 的 project_id
plugin_publish_self_check(project_id)   # 拿元数据 / 文件树 / 上一发布版本号 / 商店账号绑定状态
```

self_check 返回里关键字段：
- `last_published_version` —— 空 = 首次发布；非空 = 升版本场景
- `store_account_bound` —— target=store 时检查；未绑定要先让用户去链接（/account → 外部账号）
- `files` —— 后续 dev_read_file 要读哪些

### 2. 读关键代码

按这个顺序读，每读一个跟用户报告"我读了 X，发现 Y"：

1. **`tinia-repo.yaml`** — 拿 name / namespace / sdk 路径
2. **`README.md`**（如果有） — 用户已写的介绍可作为 display_name / description 起点
3. **每个 `nodes/*/node.yaml`** — 看节点 key / inputs / outputs / category
4. **每个 `nodes/*/runtime/run.py`** — 理解节点实际做什么（DSP 算法 / 调的库 / 输出格式）
5. **`schemas/params.schema.json`**（如有） — 看用户能配什么参数

### 3. 升版本场景 → 拉 diff（按需精简）

**两步走，不要一次拉全文 diff（容易爆 token）：**

```
# 步骤 a) 看改动规模
plugin_diff_last_published(project_id, fields=["summary","stats"])
# → {added: [...], removed: [...], modified: [{path, from_bytes, to_bytes, from_lines, to_lines}]}
```

看 modified 文件名清单 + 字节/行变化，决定关注哪几个：

```
# 步骤 b) 拉关注文件的完整 diff
plugin_diff_last_published(project_id, fields=["full"], glob="*.py")  # 仅 Python
plugin_diff_last_published(project_id, fields=["full"], path="nodes/level_meter/")  # 单节点
```

AI 据此为 changelog_md 总结：
- modified 数量大 + 涉及 run.py → 算法升级，重点说
- added 节点 → "新增 X 节点"
- modified 仅 README → "文档更新"
- 跨大版本（1.x → 2.0）→ 提醒用户"是否有破坏性变更要警示"

### 4. 起草元数据

按 [plugin-design-spec.md §8](../../../docs/plugin-design-spec.md#8-readme-风格) 的风格起草：

```yaml
display_name: "声学小波分析"  # 中文友好名
description: "对音频做 DWT 多频带能量/熵/RMS/峰度/偏度统计 — 可配 11 种小波基"
readme_md: |
  # 声学小波分析
  ## 用途
  ...
  ## 输入输出
  ...
  ## 参数
  ...
  ## 示例
  ...
  ## 已知限制
  ...
changelog_md: |
  - 新增对 db8 / sym4 小波基的支持
  - 修复采样率 < 16k 时分解层数推断错误
  - 输出指标加 spectral_entropy
```

### 5. 反问用户不明确字段

不要瞎填这些，问用户：
- **target**：组织内 / 在线商店？
- **category_id**：分类（要列商店的分类让用户选 — 调 admin/template-categories endpoint 拿，或让用户报名字）
- **cover_image_url**：有没有截图/封面？没有就空（可后补）

### 6. 推 draft 给前端 review

```
plugin_publish_draft(
    project_id=42,
    target="org",           # 或 "store"
    metadata={
        "display_name": "...",
        "description": "...",
        "readme_md": "...",
        "changelog_md": "...",
        "category_id": 3,            # 可选
        "cover_image_url": "...",     # 可选
    }
)
```

调用后：
- 前端会跳到 Dev Studio 的该项目页 + 自动打开发布对话框
- 字段已填好，header 显示"AI 已填好字段"emerald 徽章
- 用户能 review 并改字段
- 用户改完点"提交审批" → 走常规流程

**这一步就是终点**。给用户的回答里要明确写：

> "字段已填到发布对话框（Dev Studio 已自动打开它）。请检查后点对话框右下角的「提交审批」按钮 —— 我没有直接提交的入口，必须由你点提交才会进入审批队列。"

不要写"OK 我帮你提交"或"要不要我直接提交"之类的话 —— **AI 没有这个工具**，提交永远是用户动作。

### 7. 跟踪状态

用户点「提交审批」后，调：

```
plugin_my_submissions
```

返回我的提交列表 + 状态（pending / approved / rejected / withdrawn）。告诉用户："已进审批队列。审批人通过后，订阅者就能用了。"

---

## 关键原则

1. **不替用户提交** —— AI 没有 submit 工具。draft 推完就停下，等用户在对话框点提交。任何"我直接帮你提交"的话都是错的。
2. **不替用户想 target** —— 组织内 vs 在线商店是不同审批人 + 不同可见范围，必须问用户。
3. **不瞎写 readme** —— 一定要先读代码（run.py / node.yaml），不能瞎吹功能。
4. **diff 大改要警告** —— 跨大版本（major bump）或 modified 文件超过 5 个，提醒用户"这是破坏性变更吗？要不要在 changelog 里说明？"
5. **首次发布额外检查** —— `last_published_version` 为空时，问用户："这是 X 的首次发布，包名以后不能改，确认 'wavelet_analysis' 这个 slug？"
6. **拒绝复发同版本** —— 如果当前 version 已发过且 status=pending/approved，先问用户要不要 dev_bump_version 升号。

---

## 安全建议

发布前 AI 应该自查：
- run.py 没有 `os.system` / `subprocess` / `eval` / `exec` / `pickle.loads`
- 网络访问限于 `rt.fetch_*` / `rt.upload_*`
- 没有硬编码 API key / 密码

详见 [plugin-design-spec.md §9](../../../docs/plugin-design-spec.md#9-安全准则审批人必读)。

发现问题 → **告诉用户先修，再发布**。商店审批人会卡这些。

---

## 相关 Skill

- 写代码 → `create-node` / `sdk-reference`
- 调试运行错误 → `debug-node`
- 离线安装（不走商店）→ `pack-and-publish`
