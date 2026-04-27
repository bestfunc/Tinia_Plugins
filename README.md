# Tinia_Plugins（AI 开发助手插件集）

> Claude Code Plugin Marketplace：让 AI（Claude Code / Codex / Qwen CLI）直接帮你开发 Tinia 声学分析平台的插件节点。

## 这是什么

Tinia 是基于节点图的声学分析平台（[主项目](https://github.com/bestfunc/Tinia)）。
工程师通常要写 Python 节点（`run.py`）扩展分析能力。这个 marketplace 把 Tinia 的插件开发能力通过 MCP（Model Context Protocol）下沉给本地 AI 客户端，让 AI 协助你完成：

- **创建插件项目**：选模板、起个名、指定命名空间
- **添加节点**：AI 帮你想清楚输入输出类型、生成骨架代码
- **写逻辑**：AI 写 `run.py`，你审查
- **测试**：一键注册到个人命名空间，进 Tinia Web UI 流程编辑器连线验证
- **打包发布**：版本 +0.1，导出 `.tar.gz` 安装到组织节点仓库

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

## 安装

### 1. 前置条件

- 一个能访问的 Tinia 实例（公网 SaaS / 公司私有化 / 本地 dev 至少一种）
- 你的账号所在用户组里，超管已经为你开启了 `mcp:dev` 和 `mcp:nodes` 模块（在「系统设置 → 用户组模版」里）

### 2. 按部署场景选一个 plugin 装

我们提供三个 plugin 变体，**只需选一个安装**，不需要改任何配置文件：

| Plugin | 适用场景 | MCP 地址 |
|---|---|---|
| **`tinia`** | SaaS 版（公网账号） | `https://tinia.bestfunc.com/api/v1/mcp` |
| **`tinia-onprem`** | 公司私有化部署 | `https://t.bestfunc.com/api/v1/mcp` |
| **`tinia-local`** | 本地开发（`./start-dev.sh` 跑着） | `http://localhost:18722/api/v1/mcp` |

> 三个变体共享同一组 Skills（通过软链接），只是 MCP server 地址不同。

### 3. Claude Code

```
/plugin marketplace add bestfunc/Tinia_Plugins

# 三选一，按你的部署场景：
/plugin install tinia@tinia-plugins         # SaaS
/plugin install tinia-onprem@tinia-plugins  # 公司私有化
/plugin install tinia-local@tinia-plugins   # 本地开发
```

首次使用时 Claude 会打开浏览器跳到对应 Tinia 实例的 `/oauth/authorize` 页面，用 Tinia 账号登录并点「允许」即可。

> 如果你既要在 SaaS 上做主力，又要本地调试 Tinia 主仓库，可以同时装 `tinia` 和 `tinia-local`，两个 connector 在 Claude Code 里各自有独立的 OAuth token，互不干扰。

### 4. Codex / Qwen CLI（不通过 marketplace）

把对应变体目录下的 `plugin.json` 里 `mcpServers.tinia` 条目复制到你的客户端配置文件（每个 CLI 位置略有不同，参考各自文档）。OAuth 流程是标准 MCP 规范，兼容所有支持 MCP 2024-11-05 协议的客户端。

### Windows 用户注意

三个 plugin 目录的 `skills` 是 git symlink。Windows 下需要 git 配置 `core.symlinks=true`（开发者模式或 admin shell 下 `git clone` 默认会启用），否则 clone 出来 symlink 会变成普通文本文件。Claude Code 客户端不受影响，只是源码仓库 clone 后看着不对。

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

- 你在 OAuth 授权页可以看到 AI 客户端申请的每一个权限模块（dev / nodes / ...）
- 授权后，可以在 Tinia「账号设置 → 连接的应用」里随时撤销
- 所有 MCP 调用都会审计记录，超管或组织管理员可在「系统设置 → MCP 审计」查看

## 许可

Apache-2.0
