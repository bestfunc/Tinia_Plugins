---
name: 打包并安装插件
description: 开发完成后打包发布 —— 版本 +0.1、导出 .tar.gz、引导用户在 Web UI 里安装
user-invocable: true
allowed-tools: mcp__tinia__dev_get_project, mcp__tinia__dev_bump_version, mcp__tinia__dev_export
---

# 打包并安装插件

开发 → 测试满意 → 发布。

## 流程

### 1. 确认可以发布

问用户：
- 节点都在本地测过了吗（`dev_reload` 成功 + Web UI 流程里跑通）
- 有没有未保存的改动

### 2. 升版本

```
dev_bump_version(project_id)
```

规则：
- 从 1.0 起步
- 每次 +0.1 小数位（1.0 → 1.1 → 1.2 ... → 1.9 → 2.0 进位）
- 用户不能直接改版本号，只能通过这个 tool 递增

告诉用户："新版本 vX.Y"，问他是否确认继续导出（已经递增就不可逆，不回滚，但可以再 bump）。

### 3. 导出

```
dev_export(project_id)
```

返回：
```json
{
  "filename": "my-plugin-v1.2.tar.gz",
  "size_bytes": 12345,
  "encoding": "base64",
  "content": "H4sIAA..."   // base64 编码的 tar.gz 字节
}
```

**如果超过 20 MB 会报错** —— 告诉用户"项目太大，请在 Tinia Web UI 的 Developer Studio 里点导出按钮下载（不受 MCP 限制）"。

### 4. 把文件保存到本地

你有文件系统工具的话，把 base64 解码后写到本地：
```python
import base64
with open("my-plugin-v1.2.tar.gz", "wb") as f:
    f.write(base64.b64decode(content))
```

或者告诉用户具体的 base64 字符串，让用户自己下载（不推荐）。

### 5. 指导用户安装

**方式 A — 自用/本地测试**：
> 1. 打开 Tinia Web UI
> 2. 进"节点仓库"页面
> 3. 点"离线安装"，选择刚下载的 `my-plugin-v1.2.tar.gz`
> 4. 安装时如果没声明 namespace，会被放到 bestfunc —— 可能和官方节点 key 冲突，**建议事先在 tinia-repo.yaml 里写 namespace: <你的组织>**

**方式 B — 发给团队**：
> 1. 上传 tar.gz 到团队共享位置（如公司内网文件服务器 / Google Drive / 企业 OA）
> 2. 团队里的组织管理员按方式 A 安装到公司的 Tinia
> 3. 安装后所有组织成员都能在节点面板看到（按 namespace 可见性规则）

**方式 C — 发布到线上商店**（如果有的话）：
> Tinia 线上商店（Tinia_Store 仓库）支持插件上架，当前 v1.19 还需手动走审核。

## 注意事项

- 导出前最好 `dev_get_project` 确认当前 version / namespace 和预期一致
- 导出包里**不包含** `.venv`、`__pycache__`、`node_modules` 等（自动过滤）
- 测试节点时 `dev_reload` 注册的是用户个人命名空间，**和 tar.gz 里声明的 namespace 不同** —— 正式安装后节点会归入 tinia-repo.yaml 声明的命名空间

## 相关 Skill

- 命名空间讲解 → 见 docs/node-namespace-design.md
- tinia-repo.yaml 的 namespace/table_prefix 字段 → 「tinia-repo.yaml 字段速查」
