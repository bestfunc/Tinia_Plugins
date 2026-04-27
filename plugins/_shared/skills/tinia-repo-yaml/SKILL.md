---
name: tinia-repo-yaml
display_name: tinia-repo.yaml 字段速查
description: Tinia 插件级 manifest（tinia-repo.yaml）的所有字段 —— 命名空间、modules、migrations、模版
user-invocable: false
---

# `tinia-repo.yaml` 字段速查

插件项目根目录下的 manifest 文件，声明整个插件（可能含多个节点 + 共享模块）的元数据。

## 最小有效例

```yaml
name: "我的插件"
description: ""
author: ""
version: "1.0"
min_tinia_version: "1.18"
namespace: bestfunc
sdk:
  python: sdk/python
```

## 字段详解

### 基础信息

```yaml
name: "声学工具集"
description: "一组常用声学分析节点"
author: "bestfunc"
url: https://github.com/bestfunc/my-plugin
version: "1.2"                    # 插件版本；用 dev_bump_version 自动递增
min_tinia_version: "1.18"         # 最低 Tinia 版本要求
```

### `namespace`（v1.18+，推荐显式声明）

```yaml
namespace: bestfunc               # 官方插件
# namespace: acme-corp            # 组织插件
# namespace: xugf                 # 个人插件
```

**规则**：
- 所有节点注册时会以此为前缀，最终 full_key = `{namespace}/{key}`
- 没声明时 Tinia 会 fallback 到 `bestfunc`（官方），**可能和已有节点 key 冲突**
- 组织 / 个人插件强烈建议声明，避免和官方冲突

### `sdk`（Python SDK 路径）

```yaml
sdk:
  python: sdk/python              # 插件自带的 Python SDK 目录
```

一般插件不需要自带 SDK（用主 Tinia 的即可），但 scaffold 模板里默认留一个空 README。

### `table_prefix`（数据源插件用）

```yaml
table_prefix: plg_diffgram_       # 数据库表前缀；migrations 里的 CREATE TABLE 用这个
```

仅 `datasource_plugin` 模板需要 —— 插件可以创建自己的数据表（凭证、数据源配置），但表名必须带这个前缀避免冲突。

### `migrations`（string[]）

```yaml
migrations:
  - migrations/001_init.up.sql
  - migrations/002_add_indexes.up.sql
```

Tinia 启动时自动执行（按文件名顺序），已执行的会跳过。文件路径相对插件根目录。

### `modules`（object，复合）

声明插件提供的"非节点"能力。

```yaml
modules:
  # 侧栏菜单（插件自定义页面入口）
  menu_items:
    - route: /plugin/diffgram/credentials
      label: "凭证-标记系统v1"
      icon: key                     # lucide 图标名
      permission: credentials_diffgram   # 权限 key，空则所有登录用户可见
      page_type: custom
      ui: ui/CredentialManager.tsx       # 相对插件根，vite 扫描并加载

  # 凭证类型声明
  credentials:
    - id: diffgram_basic
      name: "标记系统v1 Basic Auth"
      fields:
        - key: host
          label: "服务器地址"
          type: url                      # text / url / secret / number / textarea
          required: true
          placeholder: "https://…"
      test_connection: handlers/test_connection.py   # 连接测试脚本

  # 数据源类型声明
  datasources:
    - id: diffgram
      name: "标记系统v1"
      icon: database
      credential_type: diffgram_basic    # 引用上面的 credential id
      fetch_handler: handlers/fetch.py   # 拉取数据的脚本

  # 权限声明
  permissions:
    - key: credentials_diffgram
      label: "凭证 - 标记系统v1"
      group: "标记系统v1"

  # 流程模版打包分发（可选）
  templates:
    category: "数据标记辅助"
    files:
      - templates/随机抽样.tinpack
      - templates/辅助筛选OK_NG数据.tinpack
```

### Handler 脚本协议

#### `fetch_handler`（拉数据）

stdin 传 JSON `{datasource_config, credentials, query}`，stdout 返回 `{items, total}`：

```python
import json, sys
req = json.loads(sys.stdin.read())
# req.datasource_config, req.credentials, req.query.folder, req.query.search, req.query.page, req.query.limit
items = [{"id": "x", "name": "xxx.wav", "media_type": "audio/wav", "content_url": "..."}]
print(json.dumps({"items": items, "total": len(items)}))
```

#### `test_connection`（测试凭证）

stdin 传 `{credentials}`，stdout 返回 `{success: bool, error?: str}`：

```python
import json, sys
req = json.loads(sys.stdin.read())
creds = req.get("credentials", {})
if not creds.get("host"):
    print(json.dumps({"success": False, "error": "地址必填"}))
else:
    print(json.dumps({"success": True}))
```

## 常见坑

- `namespace` 强烈建议显式声明；缺省时和官方节点冲突装不进去
- `table_prefix` 用在 migrations 里的表名 —— 必须手写进 `CREATE TABLE <prefix>items (...)`，Tinia **不会自动替换**
- `modules.menu_items[].ui` 路径相对插件根，不是相对 `ui/` 目录
- `modules.credentials[].fields[].type: secret` 会让前端脱敏显示 + 编辑时不回填
- `migrations` 文件执行幂等性靠 Tinia 自带的版本跟踪表（`schema_migrations` 类似），插件不用自己管
