---
name: pack-and-publish
display_name: 打包导出 .tar.gz（辅助路径）
description: 把开发完的插件导出成 .tar.gz —— 用于离线传输 / 自托管代码仓库。**主流发布请用 /publish-plugin** 走商店审批。
user-invocable: true
allowed-tools: mcp__tinia__dev_get_project,mcp__tinia__dev_bump_version,mcp__tinia__dev_export
---

# 打包导出 .tar.gz（辅助路径）

> ⚠ 这是**老路径**。绝大多数情况下你应该用 [`publish-plugin`](../publish-plugin/SKILL.md) 把插件提交到组织商店或 Tinia 在线商店审批 —— 用户体验更好（不用手动下载/上传），还有审批留痕。

只在以下情况用本 skill：
- 用户**离线**（没法连商店），需要 .tar.gz 传 USB 走
- 想把代码托管到自己的 Git 仓库（不走 Tinia 分发）
- 想给特定客户私下发包（不上商店）

---

## 流程

### 1. 升版本（如需）

```
dev_bump_version(project_id)
```

规则：
- 从 1.0 起步
- 每次 +0.1（1.0 → 1.1 → ... → 1.9 → 2.0）
- 用户不能直接改 version

升版本时**当前 workdir 会自动存档为可恢复的快照**（详见 v1.21+ 版本快照机制），日后切回安全。

### 2. 导出

```
dev_export(project_id)
```

返回 base64 编码的 tar.gz（小项目）；> 20 MB 会报错，让用户去 Web UI 点导出按钮下载。

### 3. 让用户下载

用户在 Dev Studio 项目卡片菜单 → "导出 .tar.gz"，浏览器直接下载文件。

### 4. 离线安装

把 tar.gz 给目标 Tinia 实例的管理员，他们在「节点仓库」页面 → 离线安装。

---

## 跟商店发布的区别

| 维度 | 商店发布（推荐） | 离线 .tar.gz |
|---|---|---|
| 入口 | `/publish-plugin` skill | `/pack-and-publish` skill |
| 流程 | dev publish → 审批 → 自动装载到订阅者 | 手动下 tar.gz → 手动 upload 安装 |
| 审批 | 有（组织 / 商店运营） | 无 |
| 跟踪状态 | 「商店 → 我的提交」可看 | 无 |
| 跨实例同步 | 订阅者自动跟最新版本 | 每个实例手动 reinstall |

---

## 相关 Skill

- **主流路径** → `publish-plugin`
- 命名空间 → `tinia-repo-yaml`
- 审批人视角 → `review-plugin`
