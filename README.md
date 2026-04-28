# Tinia_Plugins

Tinia 声学数据分析平台的 AI 开发助手 plugin 市场，支持 Claude Code / Qwen Code 等 CLI。

一条命令接入 **16 个 AI skill + OAuth 自动授权的 MCP connector**，覆盖插件项目创建、节点脚手架生成、源码搜索、`run.py` 编写、分析流程搭建（事务批操作）、节点测试调试、版本管理、打包发布、审批等场景。MCP 认证走 OAuth，首次使用自动弹出 Tinia 浏览器授权页，无需手动配 token。

**三个部署变体，按场景选一个**：

| Plugin | 适用场景 | MCP 地址 |
|---|---|---|
| **`tinia`** | SaaS 版（公网账号） | `https://tinia.bestfunc.com/api/v1/mcp` |
| **`tinia-onprem`** | 公司私有化部署 | `https://t.bestfunc.com/api/v1/mcp` |
| **`tinia-local`** | 本地开发（`./start-dev.sh` 跑着） | `http://localhost:18722/api/v1/mcp` |

> 三个变体共享同一组 16 个 skill（git symlink 到 `_shared`），仅 MCP server 地址不同。

## 这是什么

Tinia 是基于节点图的声学分析平台（[主项目](https://github.com/bestfunc/Tinia)）。
工程师通常要写 Python 节点（`run.py`）扩展分析能力。这个 marketplace 把 Tinia 的插件开发能力通过 MCP（Model Context Protocol）下沉给本地 AI 客户端，让 AI 协助你完成：

- **创建插件项目**：选模板、起个名、指定命名空间
- **添加节点**：AI 帮你想清楚输入输出类型、生成骨架代码
- **写逻辑**：AI 写 `run.py`，你审查
- **测试**：一键注册到个人命名空间，进 Tinia Web UI 流程编辑器连线验证
- **搭分析流程**：AI 自动建流程、连线、跑、看产物、回头修代码（开发-测试闭环）
- **打包发布**：版本递进、商店提交审批 / 离线 `.tar.gz` 双路

## 目录结构

```
Tinia_Plugins/
├── .claude-plugin/
│   └── marketplace.json              ← Marketplace 定义（列三个变体）
├── plugins/
│   ├── _shared/skills/               ← 16 个 Skill（共享源，三个变体软链到这里）
│   │   ├── quickstart/
│   │   ├── types-reference/
│   │   ├── sdk-reference/
│   │   ├── node-yaml/
│   │   ├── tinia-repo-yaml/
│   │   ├── create-node/
│   │   ├── debug-node/
│   │   ├── modify-ports/
│   │   ├── params-form/
│   │   ├── result-view/
│   │   ├── flow-author/
│   │   ├── publish-plugin/
│   │   ├── review-plugin/
│   │   ├── pack-and-publish/
│   │   ├── datasource-plugin/
│   │   └── pick-template/
│   ├── tinia/                        ← SaaS 版（tinia.bestfunc.com）
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills → ../_shared/skills
│   ├── tinia-onprem/                 ← 公司私有化版（t.bestfunc.com）
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills → ../_shared/skills
│   └── tinia-local/                  ← 本地开发版（localhost:18722）
│       ├── .claude-plugin/plugin.json
│       └── skills → ../_shared/skills
├── README.md
└── CHANGELOG.md
```

## 快速开始

### 前置条件

- 一个能访问的 Tinia 实例（公网 SaaS / 公司私有化 / 本地 dev 至少一种）
- 你的账号所在用户组开启了 `mcp:dev` / `mcp:nodes` / `mcp:flow` / `mcp:plugins` 模块（系统设置 → 用户组管理，超管或组织管理员可改）

### Claude Code（原生支持）

```bash
# 1. 添加 marketplace（在 Claude Code 会话里输入）
/plugin marketplace add bestfunc/Tinia_Plugins

# 2. 安装对应变体（三选一，按你的部署场景）
/plugin install tinia@tinia-plugins         # SaaS
/plugin install tinia-onprem@tinia-plugins  # 公司私有化
/plugin install tinia-local@tinia-plugins   # 本地开发

# 3. 查看 MCP 连接状态
/mcp
# 可以看到 tinia 条目首次显示需要认证

# 4. 触发 OAuth 授权
# 直接让 AI 调一个 MCP 工具会自动弹浏览器；
# 也可以 /mcp authenticate tinia 主动触发
```

> 同时装多个变体（如 `tinia` 主用 + `tinia-local` 开发）完全 OK，两个 connector 在 Claude Code 里各自有独立 OAuth token，互不干扰。

### Qwen Code（自动转换格式）

```bash
# 1. 安装扩展（marketplace-url:plugin-name 格式，三选一）
qwen extensions install bestfunc/Tinia_Plugins:tinia
qwen extensions install bestfunc/Tinia_Plugins:tinia-onprem
qwen extensions install bestfunc/Tinia_Plugins:tinia-local

# 2. 重启 Qwen Code 让 MCP 配置生效
qwen

# 3. 查看 MCP 连接状态
/mcp
# tinia 条目首次会显示 ✗ 已断开 / Needs authentication

# 4. 触发 OAuth 授权
/mcp auth tinia
# 或直接让 AI 调一个 MCP 工具；浏览器打开 Tinia 授权页，
# 登录 + 同意后自动完成，回到 /mcp 即可看到 ✓ Connected
```

Qwen Code 会自动把 Claude plugin 格式转成 Qwen extensions 格式并写入 `~/.qwen/extensions/<name>/qwen-extension.json`。

### 其他支持 MCP 的客户端（Cursor / Zed / Cline 等）

本仓库 skill 是纯 Markdown，按客户端各自的规范复制到对应目录即可；MCP connector 单独按客户端 UI 手动配，URL 填对应变体的地址（见上面表格），留空 OAuth Client ID/Secret 走自动注册（OAuth 2.1 + PKCE + RFC 7591 DCR）。

### OAuth 授权流程

首次使用浏览器会跳转到对应 Tinia 实例的 `/oauth/authorize` 页面，登录 Tinia 账号并同意授权后，access_token 默认 30 天有效，到期会自动静默刷新。随时可以在 **Tinia「账号设置 → 已授权应用」** 里撤销。

授权页会列出 AI 客户端申请的每个权限模块（dev / nodes / flow / plugins / data 等），同意前可以核对范围。

### Windows 用户注意

三个 plugin 目录的 `skills` 是 git symlink。Windows 下需要 git 配置 `core.symlinks=true`（开发者模式或 admin shell 下 `git clone` 默认会启用），否则 clone 出来 symlink 会变成普通文本文件。Claude Code 客户端不受影响，只是源码仓库 clone 后看着不对。

## 更新

```bash
# Claude Code
/plugin marketplace update tinia-plugins

# Qwen Code
qwen extensions update tinia            # 或 tinia-onprem / tinia-local
```

## Skills 概览

| Skill（目录名）| 中文名 | 用途 |
|---|---|---|
| `quickstart` | 快速上手 Tinia 插件开发 | 从零到一的对话示例 |
| `types-reference` | Tinia 类型体系 | AudioData / IndicatorData / ... 语义与兼容 |
| `sdk-reference` | Tinia SDK 参考 | `tinia_runtime.Runtime` 所有方法 |
| `node-yaml` | node.yaml 字段速查 | 节点清单字段完整说明 |
| `tinia-repo-yaml` | tinia-repo.yaml 字段速查 | 插件级 manifest |
| `pick-template` | 从模板开始 | 4 个模板的差异与选型 |
| `create-node` | 创建 Tinia 节点 | 实操：添加节点 + 写 run.py |
| `params-form` | 写 ParamsForm.tsx | 节点参数表单的 React 组件规范 |
| `result-view` | 写 Viewer.tsx / ViewerLoader.tsx | 节点结果视图的 React 组件规范 |
| `modify-ports` | 修改节点端口与参数 | 端口改动时的兼容性注意 |
| `debug-node` | 调试节点运行错误 | 测试失败时的排查流程 |
| `flow-author` | 用 AI 搭分析流程 | flow_batch_edit / 跑流程 / 看节点产物 |
| `datasource-plugin` | 开发数据源插件 | `datasource_plugin` 模板专用 |
| `pack-and-publish` | 打包安装插件（离线） | 版本递进 + 导出 tar.gz 本地安装 |
| `publish-plugin` | 发布插件到商店（在线） | OAuth 绑定 + draft → 用户提交审批 |
| `review-plugin` | 审批插件 | 审核员视角：grep 源码 + 安全审查 SOP |

## MCP 工具集（供参考）

AI 会自动选择合适的工具，你一般不用关心。完整列表：

**dev 模块**（开发者工具）
- 项目：`dev_list_projects` / `dev_create_project` / `dev_get_project` / `dev_delete_project` / `dev_open_project` / `dev_bump_version`
- 文件：`dev_tree` / `dev_read_file`（offset/limit）/ `dev_write_file` / `dev_edit_file`（局部三元组替换）/ `dev_rename` / `dev_delete_file`
- 搜索：`dev_grep_files` / `dev_glob_files`（跨 dev / 官方源码）
- 节点：`dev_list_nodes` / `dev_create_node` / `dev_delete_node`
- 测试：`dev_reload`（注册到个人命名空间）/ `dev_tail_logs` / `dev_export`

**nodes 模块**（节点元信息，只读）
- `nodes_list`（可见节点列表）
- `nodes_describe`（fields=meta/ports/params/ui/docs 按需）
- `nodes_list_types`（类型体系）
- `nodes_read_source`（读官方节点源码学风格）

**flow 模块**（分析流程编排）
- 元信息：`flow_list` / `flow_describe`（fields/filter）/ `flow_runs_list` / `flow_node_logs`
- 编辑：`flow_create` / `flow_batch_edit`（事务批操作）/ `flow_replace_node` / `flow_set_node_params`（批量）/ `flow_auto_layout`
- 运行：`flow_run` / `flow_wait_run` / `flow_node_output_preview`

**plugins 模块**（在线发布与审批）
- 发布：`plugin_publish_draft`（merge 模式默认）/ `plugin_diff_last_published`（fields/filter）
- 审批：`plugin_review_grep_files`（搜审批包源码）+ 通过 / 拒绝工具

**datasource 模块**（数据源管理，给 flow 用）
- `datasource_list` / `datasource_describe` / `datasource_search_files`

## 权限 & 审计

- 你在 OAuth 授权页可以看到 AI 客户端申请的每一个权限模块（dev / nodes / flow / plugins / ...）
- 授权后，可以在 Tinia「账号设置 → 已授权应用」里随时撤销
- 所有 MCP 调用都会审计记录，超管或组织管理员可在「系统设置 → MCP 审计」查看

## 问题反馈

- 问题提交：https://github.com/bestfunc/Tinia_Plugins/issues
- 商务合作：Great@bestfunc.com

## License

Apache-2.0
