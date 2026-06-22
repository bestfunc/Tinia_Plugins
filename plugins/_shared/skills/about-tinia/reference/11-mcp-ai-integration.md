# MCP-native（AI 协作）

> Tinia 三大差异化护城河之一。给销售讲"AI 工程师助手"的事实底座；给开发同事看 MCP 接入细节。

---

## 一句话定位

**Tinia 主仓直接内置 MCP server**，让 Claude Code / Codex / Qwen CLI 等 AI 客户端通过 OAuth 2.1 一键授权后**直接驱动 Tinia 完成节点开发 / 流程编排 / 数据上传 / 跑流程 / 发布**全流程。

不是"给 Tinia 加了一个 AI 助手按钮"，是"AI 是 first-class 客户"，跟人类用户平级使用 Tinia。

> **MCP 与 SDK 是两条并行通路**：MCP（本页）面向"AI 自动驾驶平台"——AI 客户端读写流程/节点/数据；Python SDK 面向"外部程序调用平台算力"——外部程序传数据、调用平台上已调好的分析拿结果。SDK 详见 `04-key-concepts.md` 的 SDK 通路 / 流式会话条目。

---

## 为什么这是护城河

### 传统 NVH 工具的现状

HEAD ArtemiS / Siemens Testlab 基于桌面单机架构演进多年：

- 进程模型是单机 GUI，没有内置 HTTP server
- 内部 SDK 主要面向"人写脚本"，未原生支持 AI agent 协议
- 加 MCP 接入需要重写认证 / 事件总线 / 工具调用框架
- 工程量预估 3-5 年

### Tinia 是为此设计的

主仓从 v1.0 就是 web 架构（Go HTTP server + React），加 MCP server 只是新增一个 endpoint：

- 复用现成的 OAuth 2.1 + PKCE + DCR 授权
- 复用现成的权限 / 多租户体系
- 工具调用 = HTTP API 调用 + 序列化包装

演进脉络：v1.19 AI 客户端 OAuth 授权接入（覆盖项目操作 / 文件读写 / 节点脚手架 / 编译 / 重载 / 导出）→ v1.20 新增分析流程 + 数据源操作工具，打通"开发-测试"闭环，并上线全局 AI 活动面板 + 跟随/旁观双模式 + 流程编辑器视觉联动 → v1.21 AI 一次操作完成多步（加节点 + 连线 + 改参 + 布局）失败自动回滚、插件审批安全约束（AI 只能起草、用户手动确认）→ v1.24 起 tinia-desktop 免授权直连本机。

---

## 是什么样的体验

### 典型对话（销售可以现场演）

> **工程师**（在 VSCode 里跟 Claude Code 说）：
> 我想做一个心声 roughness 节点，输入 AudioData，输出 IndicatorData，参数包括 fmin、fmax、step。
>
> **Claude Code**：
> 好的，我帮你：
> 1. 创建项目 acoustic-tools（如不存在）
> 2. 用 dev_create_node scaffold 节点骨架
> 3. 写 run.py 使用 mosqito 库
> 4. 写 ParamsForm.tsx
> 5. 跑 dev_reload 让 Tinia 装载
> 6. 创建测试流程跑通
> 7. 提交到商店审批
>
> *（实际调用 20+ 个 MCP 工具完成全部步骤）*
>
> 完成！节点 acoustic-tools/roughness 已上线，测试流程跑通 1 次，提交审批中。

### 跟传统工作流对比

| 步骤 | 传统（HEAD / Testlab） | Tinia + AI |
|---|---|---|
| 决定要做什么算法 | 人 | 人 |
| 查文献 / 找参考实现 | 人手工查 | AI 帮查 + 推荐 |
| 写代码 | 人手工写 | AI 写人审 |
| 配 UI 参数 | 大厂自带模板 / SDK | AI 写 ParamsForm |
| 测试 | 人手工搭测试 | AI 自动搭测试流程 |
| 发布 | 大厂内部流程（不开放）| AI 提交商店审批 |

→ 时间从"几天 / 几周"压缩到"几小时"

---

## 技术实现

### MCP server 入口

```
POST /api/v1/mcp     ← 单一 endpoint
```

- 协议：JSON-RPC 2.0 over HTTP
- 协议版本：`2024-11-05`（Anthropic 早期版本号）
- 形态：Streamable HTTP 的最简退化形态（每个 POST 直接返 JSON，无 SSE upgrade）
- 兼容：Claude Code / Codex / 大部分 MCP client

### 鉴权

OAuth 2.1 完整流程：

```
1. AI 客户端首次启动：动态客户端注册（DCR）
2. 客户端发起 OAuth /authorize → 弹浏览器
3. 用户登录 Tinia → 同意授权
4. 重定向回 client loopback → 拿 code
5. 客户端 PKCE 换 access_token + refresh_token
6. 之后所有 MCP 调用带 Bearer access_token
```

实际用户感知：**第一次用某 AI 客户端时弹一次浏览器登录，之后无感**。

### 工具列表（70+ tools 覆盖 8 个模块）

代码里全 server 共注册约 72 个唯一工具，分属 8 个模块（`MCPAllModules` = `dev / nodes / graphs / data / data_write / templates / assistant / plugins`）：

| 模块 | 工具数 | 代表工具 |
|---|---|---|
| **dev** | 25 | dev_create_project / dev_create_node / dev_write_file / dev_reload / dev_export |
| **nodes** | 10+ | nodes_list / nodes_describe / nodes_read_source / nodes_list_types |
| **graphs** | 20+ | flow_create / flow_add_node / flow_connect / flow_run / flow_get_run_status / flow_node_logs / flow_node_outputs |
| **data** | 5+ | datasource_list / datasource_create_composite / channel_template_* |
| **data_write** | 1 | tinia-file upload_file_to_datasource（走本地 MCP）|
| **templates** | 数个 | 流程模板读写 |
| **assistant** | 数个 | 助手类工具 |
| **plugins** | 10+ | plugin_publish_submit / plugin_review_diff / plugin_review_approve |

另含 SDK 相关工具（`sdk_input` / `sdk_output` / `sdk_license` / `sdk_licenses` / `sdk_local`），用于 AI 协助生成 SDK 调用代码与管理 license。

详见 `Tinia/docs/mcp-tool-reference.md`。

### 权限 / scope

OAuth scope 控制工具可见性，scope = `mcp:<module>`，共 8 个，由 user_group 的 `permissions.mcp.<module>` 决定（超管全开）：

| Scope | 含义 |
|---|---|
| `mcp:dev` | DevStudio 项目读写 |
| `mcp:nodes` | 节点定义读 |
| `mcp:graphs` | 流程读写 |
| `mcp:data` | 数据源读（含通道模板）|
| `mcp:data_write` | 数据源写（含上传）|
| `mcp:templates` | 流程模板读写 |
| `mcp:assistant` | 助手类工具 |
| `mcp:plugins` | 商店发布 / 审批 |

用户授权时勾选 scope，AI 客户端只能用授权范围内的工具。

### 双 MCP 架构（远程主 MCP + 本地文件 MCP）

```
本地 AI 客户端（Claude Code / Codex / Qwen CLI）
  ├─ tinia          → Tinia Server /api/v1/mcp    远程 HTTP，OAuth + Bearer
  │                  70+ 工具：dev / nodes / graphs / data / data_write / templates / assistant / plugins
  │
  └─ tinia-file     → 本机 Node 进程（@bestfunc-com/tinia-file-mcp）
                      1 工具：upload_file_to_datasource
                      用 OAuth access_token 走 multipart POST → /api/v1/mcp_data
```

**为什么拆**：

- 主 MCP 走 JSON-RPC over HTTP，不适合 multipart 大文件上传（远程 base64 编码 + 网络往返耗时）
- tinia-file MCP 跑在本机 Node 进程，能直接走 multipart 流式上传到 daemon
- 鉴权用同一套 OAuth，用户无感

### 免授权直连（桌面本机）

装 tinia-desktop 插件后，AI 客户端连本机 Tinia 无需走 OAuth 即可使用；CLI `tinia login` 在桌面窗口内完成授权。这是桌面单机场景下的简化通路。

---

## 实际部署：4 个 Plugin 变体

`Tinia_Plugins` 仓库（Claude Code marketplace）提供 4 个变体 plugin，对应 4 种 Tinia 部署：

| Plugin 名 | 接 Tinia | 谁用 |
|---|---|---|
| `tinia` | `tinia-saas.bestfunc.com`（SaaS）| 个人开发者，用公网账号 |
| `tinia-onprem` | `t.bestfunc.com`（公司私有化）| 公司内部开发者 |
| `tinia-desktop` | `localhost:18720`（桌面单机）| Desktop 用户自己 |
| `tinia-local` | `localhost:18722`（本地开发）| Tinia 主仓贡献者 |

用户从 Claude Code 装对应 plugin，自动配好 MCP connector。

详见 `Tinia_Plugins/.claude-plugin/marketplace.json`。

---

## Activity Stream（AI 行为可视化）

不是 MCP 协议的一部分，是 Tinia 自家功能：

- AI 通过 MCP 调用工具时，daemon 把"AI 正在做什么"推到 `GET /api/v1/mcp/activity/stream`（SSE）
- 前端右下角浮窗实时显示：`AI 创建了项目 acoustic-tools`、`AI 正在写 run.py`、`AI 跑了流程 #123`
- 同时 UI 显示"流程锁"（AI 在改的流程光晕）、"跟随导航"（AI 切到的页面）

让人类用户**始终知道 AI 在做什么**，避免黑盒焦虑。

详见 `Tinia/docs/ai-activity-architecture.md`。

---

## Skill 体系（封装 MCP 工具组合）

`Tinia_Plugins` 的 plugin 内含 20+ AI skill，每个 skill 把"完成某种工作流"的多个 MCP 工具组合 + prompt 模板封装好：

| Skill | 用途 |
|---|---|
| `quickstart` | 从零开发第一个节点的完整教程 |
| `create-node` | 创建节点（含读官方节点学风格的步骤）|
| `pick-template` | 选合适的节点模板 |
| `flow-author` | 写流程 |
| `debug-node` | 调试节点 |
| `result-view` | 写 Viewer.tsx |
| `params-form` | 写 ParamsForm.tsx |
| `pack-and-publish` | 打包提交到商店 |
| `publish-plugin` | 走完整发布流程 |
| `review-plugin` | 商店审批 |
| `datasource-test` | 测数据源 |
| ... | （共 20+）|

skill 跟 MCP 工具的关系：

- MCP 工具是**机械操作**（"创建节点"、"写文件"）
- skill 是**工作流模板**（"开发一个完整节点"是什么样、按什么顺序、要避免哪些坑）

skill 在用户的 Claude Code 装上 Tinia plugin 后自动可用。用户说 `/quickstart`，Claude Code 就按 quickstart skill 的步骤跑全流程。

---

## Tinia 跟 Claude Code 的边界

| 谁负责 | 内容 |
|---|---|
| **Tinia daemon** | 提供 MCP server + 工具实现 + 数据持久化 + UI |
| **Claude Code** | LLM 推理 + 工具调用编排 + skill 执行 + 跟用户对话 |
| **plugin（marketplace 装的）** | OAuth connector 配置 + skill 模板 + 本地辅助 MCP |
| **用户** | 描述需求 + 看 AI 做事 + 偶尔接管 |

→ Tinia 不做 LLM；Claude Code 不做声学算法；plugin 是连接桥梁。

---

## 安全 / 权限

### Tinia 端

- 所有 MCP 工具都走 mcp.Auth middleware（解析 access_token → 注入 user_id / org_id / scope）
- 工具执行调用现有的权限检查（如修改某个流程必须是 owner / org_admin）
- 全部调用进审计日志（mcp_audit_logs 表）

### Plugin 端

- 用户可在 Tinia 设置页查看"已授权的 AI 客户端"
- 可随时撤销某个客户端的 token
- token 过期 / 撤销后 AI 客户端再调用会失败

---

## 跟客户讲 MCP 的话术

### 不要说

- ❌ "我们对接了 ChatGPT" —— 不够准确，MCP 是协议
- ❌ "我们用了大模型" —— 听起来像调用 OpenAI API
- ❌ "我们有 AI 功能" —— 太弱

### 应该说

- ✅ "Tinia 内置 MCP server，AI 是 first-class 客户" —— 准确 + 突出架构差异
- ✅ "工程师用 Claude Code / Codex 直接驱动 Tinia 完成 90% 工作" —— 给具体场景
- ✅ "我们不是给软件加 AI 按钮，是为 AI 工作流设计的软件" —— 战略叙事

### 给具体场景

不要抽象讲，给场景：

> "您日常做 NVH 分析报告 8 小时？用 Tinia + Claude Code：
> 1. 跟 AI 说'帮我做这周新能源车型的下线测试报告'
> 2. AI 找到模板 → 加载数据源 → 跑分析流程 → 出图
> 3. AI 写解读 → 您审一遍 → 报告完成
>
> 全程 30-60 分钟，您只需要审 + 改关键判断。"

---

## 下一步

- MCP 完整工具列表 → `Tinia/docs/mcp-tool-reference.md`
- MCP 协议细节 → `Tinia/docs/mcp-spec.md`
- AI Activity 可视化架构 → `Tinia/docs/ai-activity-architecture.md`
- Plugin marketplace 结构 → `Tinia_Plugins/.claude-plugin/marketplace.json`
- 各 skill 具体内容 → `Tinia_Plugins/plugins/_shared/skills/*/SKILL.md`
