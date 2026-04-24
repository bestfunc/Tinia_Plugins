# Tinia Plugin（AI 开发助手）

> 让你在 Claude Code / Codex / Qwen CLI 里用 AI 直接开发 Tinia 声学分析平台的插件节点。

## 这是什么

Tinia 是基于节点图的声学分析平台（[主项目](https://github.com/bestfunc/Tinia)）。
工程师通常要写 Python 节点（`run.py`）扩展分析能力。这个 Plugin 把 Tinia 的插件开发能力通过 MCP（Model Context Protocol）下沉给你的本地 AI 客户端，让 AI 协助你完成：

- **创建插件项目**：选模板、起个名、指定命名空间
- **添加节点**：AI 帮你想清楚输入输出类型、生成骨架代码
- **写逻辑**：AI 写 `run.py`，你审查
- **测试**：一键注册到个人命名空间，进 Tinia Web UI 流程编辑器连线验证
- **打包发布**：版本 +0.1，导出 `.tar.gz` 安装到组织节点仓库

## 安装

### 前置条件

1. 一个 Tinia 账号（公网 `tinia.bestfunc.com` 或私有化部署的 Tinia）
2. 你的账号所在用户组里，超管已经为你开启了 `mcp:dev` 和 `mcp:nodes` 模块（在「系统设置 → 用户组模版」里）

### Claude Code

```bash
# 通过 marketplace 安装
/plugin add bestfunc/Tinia_Plugins

# 或者 clone 后本地安装
git clone https://github.com/bestfunc/Tinia_Plugins ~/.claude/plugins/tinia
```

首次使用时 Claude 会打开浏览器，跳转到 `https://tinia.bestfunc.com/oauth/authorize`，用 Tinia 账号登录并点「允许」即可。

### Codex / Qwen CLI

把 `plugin.json` 里的 `mcpServers.tinia` 条目复制到你的客户端配置文件（每个 CLI 位置略有不同，参考各自文档）。OAuth 流程是标准 MCP 规范，兼容所有支持 MCP 2024-11-05 协议的客户端。

### 私有化部署

如果你的组织自己部署了 Tinia Server，修改 `plugin.json` 里的 `url` 字段指向你的内网地址即可（如 `https://tinia.internal.company.com/api/v1/mcp`）。

## Skills 目录

安装后 AI 客户端自动加载以下 Skill：

| Skill | 用途 |
|---|---|
| 快速上手 Tinia 插件开发 | 从零到一的对话示例 |
| Tinia 类型体系 | AudioData / IndicatorData / ... 语义与兼容 |
| Tinia SDK 参考 | `tinia_runtime.Runtime` 所有方法 |
| node.yaml 字段速查 | 节点清单字段完整说明 |
| tinia-repo.yaml 字段速查 | 插件级 manifest |
| 创建 Tinia 节点 | 实操：添加节点 + 写 run.py |
| 调试节点运行错误 | 测试失败时的排查流程 |
| 修改节点端口与参数 | 端口改动时的兼容性注意 |
| 打包并安装插件 | 版本递进 + 导出 tar.gz |
| 开发数据源插件 | `datasource_plugin` 模板专用 |
| 从模板开始 | 4 个模板的差异与选型 |

## MCP 工具集（供参考）

AI 会自动选择合适的工具，你一般不用关心。完整列表：

**dev 模块**（开发者工具）
- `dev_list_projects` / `dev_create_project` / `dev_get_project` / `dev_delete_project`
- `dev_bump_version`（版本 +0.1）
- `dev_tree` / `dev_read_file` / `dev_write_file` / `dev_rename` / `dev_delete_file`
- `dev_list_nodes` / `dev_create_node` / `dev_delete_node`
- `dev_reload`（测试节点，注册到个人命名空间）
- `dev_tail_logs`（节点运行日志，v1.19 占位）
- `dev_export`（打包导出）

**nodes 模块**（节点元信息，只读）
- `nodes_list`（可见节点列表）
- `nodes_describe`（单节点详情）
- `nodes_list_types`（类型体系）

## 权限 & 审计

- 你在 OAuth 授权页可以看到 AI 客户端申请的每一个权限模块（dev / nodes / ...）
- 授权后，可以在 Tinia「账号设置 → 连接的应用」里随时撤销
- 所有 MCP 调用都会审计记录，超管或组织管理员可在「系统设置 → MCP 审计」查看

## 许可

[MIT](LICENSE)
