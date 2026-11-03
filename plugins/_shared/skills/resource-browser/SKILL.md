---
name: resource-browser
display_name: 管理资源目录与复制
description: 在分析流程、报告、制品仓、数据源和数据组的通用树视图中浏览、新建、编辑、移动、完整复制和删除；管理保护目录，区分人工管理与节点免密选择。
user-invocable: true
allowed-tools: mcp__tinia__resource_folder_list,mcp__tinia__resource_folder_manage,mcp__tinia__resource_item_list,mcp__tinia__resource_item_manage,mcp__tinia__artifact_node_select
metadata:
  version: "1.0.0"
  agentport-product: tinia
---

# 资源树操作

用户要求整理目录、移动/复制资源、建制品仓或准备保护目录测试仓时使用。
`resource_type` 只能是 `graph` / `report` / `artifact_repo` / `datasource` / `datagroup`。
不能传表名、SQL、磁盘路径来绕过资源边界。

## 先定位，再操作

1. `resource_folder_list(resource_type, action="list")` 列根目录；指定 `parent_id` 列子目录。`search` 是跨层名称搜索。
2. `resource_item_list(resource_type, folder_id, limit, offset)` 列精简资源信息。`folder_id:null` 只列根目录；省略字段列全部可见内容。默认50条、上限200，用offset翻页。列表不返回完整图定义、报告文档或凭据配置。
3. 按名称和返回ID核对源、目的目录以及当前组织。不得根据猜测的连续ID批量操作。跨组织、非owner、锁住的内容要先解决授权/解锁，不能借复制绕过。
4. 明确是移动还是复制：移动保留资源身份；复制生成独立身份并保留内容。目的地必须是同种资源的目录；`parent_id:null`（目录）或`folder_id:null`（item）表示根目录。
5. 调用成功后重列源和目的地，用返回新ID确认数量、位置和内容。错误结果不是成功；整棵目录复制应原子完成，不能把只出现空目录当作复制成功。

目录工具 `resource_folder_manage` 的action：

| action | 用途/主要字段 |
|---|---|
| create | 新目录：name、parent_id、visibility；默认private |
| update | 改name、visibility、member_ids；不能靠visibility PATCH进出encrypted |
| move | id、parent_id；拒绝自身/后代目的地 |
| copy | id、parent_id、可选name；复制完整普通目录树，副本private，不继承ACL |
| delete | id；只能删除空目录 |
| encrypt / unlock / lock | 将普通目录保护 / 当前MCP身份解锁 / 作废当前身份解锁凭证 |
| password / recovery / decrypt | 改密码 / 换安全码 / 解除保护；沿原owner与密码校验 |

`resource_folder_list(action="members", id=...)` 查看成员；`update`的`member_ids`省略=不改、空数组=清空。共享成员必须属于当前组织。

## Item操作与复制语义

`resource_item_manage(resource_type, action, id, folder_id, ...)` 支持create/update/delete/move/copy。
create无id；其余必须指定id。name、description、code、note、definition、document、config等业务字段形状沿该资源UI API。先看工具schema；不要擅自拼内部模型字段。

最小create/update字段（下表省略共同的resource_type/action/id/folder_id）：

| 类型 | create | update |
|---|---|---|
| graph | `name`；可选`definition`为图JSON对象、`kind`为analysis/daq | name/description可部分更新；如改definition，传完整JSON对象（不能用空定义覆盖已有图） |
| report | `name`；可选`kind:"document"`、`ratio` | 只传要改的name/ratio/document，document为报告JSON对象 |
| artifact_repo | `name`；建议指定唯一`code`，可选note | name/note；code不可修改 |
| datasource | `name,type:"upload",config:{}`；config为JSON对象 | `name,type:"upload",config:{...原配置...}`；沿必填type，不用空对象误覆盖已有配置 |
| datagroup | `name`，可选description | name/description；如改config，它是JSON编码的字符串，例如`config:"{}"`，不能传对象 |

仅准备临时制品仓可使用非秘密参数形状：`resource_item_manage(resource_type="artifact_repo", action="create", name="临时验证仓", code="唯一的临时标识", folder_id=已解锁目录ID)`。密码相关动作不提供含值示例。

| 资源 | 复制/移动含义 |
|---|---|
| graph | 移动整个family的全部版本；复制当前版本为新family、1.0，保留kind与Store派生来源；目录复制每family最新版 |
| report | 复制真实Document、kind、比例、模板与绑定图引用；嵌入的不可变blob可共用 |
| artifact_repo | 新仓ID和唯一code，复制版本、元数据及文件子行；新版本保留版本号但作为未发布候选，不自动切生产active |
| datasource | 新源ID，独立uploads与Dataset定义，保留通道/采集参数并映射上传ID；不可变blob共用，凭据不能通过共享目录转交 |
| datagroup | 新组、item、标签/属性/标注ID，重绑ItemID/ParentID/段落稳定ID；私有导入图也独立复制；属性名称路径保持不变 |

复制不会改源。副本的删除/编辑不得破坏原件；删除测试副本后还要验证原件内容可读。

## 保护目录边界

目录存在可见与目录内容可进入是两道门。owner和管理员也须解锁。管理工具只在当前TokenID、用户和组织组合的服务端内存中保留解锁token，沿原30分钟TTL和密码变更失效规则。重启、换token、主动lock或到期后必须重新解锁。

密码仅在明确授权的管理动作中作为顶层敏感参数输入；不要在回复、示例、日志、测试报告或持久笔记中打印真实密码。不要把MCP opaque access token当作网页登录JWT，不访问数据库改hash绕过密码。

- 新建encrypted目录或encrypt需要password。先保护、再unlock、再在其内建资源。
- password动作使用old_password与new_password；decrypt使用password及目标visibility（默认private）。
- 结果不会返回密码、解锁token、reset_token或recovery_code。出现`recovery_code_exported:false`表示安全码**没有交付或保存给用户**。如需恢复能力，用户可在现有目录UI凭已知密码重新生成并自行保存安全码；不要声称AI已替用户保存。
- 普通管理列表在锁定后排除目录内容；全局搜索即使解锁也不搜索加密内容。加密目录或含加密后代的目录禁止整树复制；item复制仍需有效管理解锁。
- `artifact_node_select`的repos/versions/bundles/files沿既有节点选择器免目录密码规则，仍要求当前组织和data scope。它只返回节点所需精简信息，不能用于管理修改。制品节点runtime同样保留已授权的免目录密码访问与原组织校验。

## 从零准备本机合成制品测试

只在用户批准的本机隔离环境内操作：新建临时目录 → encrypt → unlock → item create建artifact_repo → lock。
然后分别验证普通`resource_item_list`排除保护仓、`artifact_node_select`仍能找到同org仓。用既有流程工具配置制品节点，运行合成写入/读取，验证版本与文件内容，再验证跨org与未解锁管理操作失败。勿用生产库、客户数据或远程服务替代验证。

报告只写资源ID、动作、通过/失败和脱敏证据；不打印任何秘密。失败时保留可定位错误，不虚报完成。

## 技能同源与嵌入

本技能唯一源为插件仓`plugins/_shared/skills/resource-browser/SKILL.md`；四个插件变体通过共享链接使用，不复制到主仓`.claude`或用户全局目录。
主仓`agentport.lock`固定插件ref，构建脚本从该提交生成索引与嵌入包，server通过go:embed加载。已有嵌入包优先，工作区技能变更不会热替换运行实例；由维护者整合提交、pin、重建bundle/server后，再由唯一测试席重启验收。技能版本与工具是否可用应以运行实例为准。
