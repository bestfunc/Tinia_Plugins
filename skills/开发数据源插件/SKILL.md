---
name: 开发数据源插件
description: 用 datasource_plugin 模板开发一个接入外部数据源的 Tinia 插件（凭证 + 数据源 + 迁移 + UI）
user-invocable: true
allowed-tools: mcp__tinia__dev_create_project, mcp__tinia__dev_read_file, mcp__tinia__dev_write_file, mcp__tinia__dev_reload, mcp__tinia__dev_list_projects
---

# 开发数据源插件

数据源插件是一种特殊插件 —— 它**不提供分析节点**，提供的是"数据从哪里来"：

- 凭证类型（如 API Token / OAuth / 账号密码）
- 数据源类型（用凭证拉数据的配置 + handler 脚本）
- 数据库迁移（插件自己的表，前缀隔离）
- 可选：UI 管理页（让管理员在 Tinia Web 里维护凭证 / 数据源实例）

已有样板：**`Tinia_nodes_diffgram`** —— Diffgram 标注系统接入。建议先看它的 `tinia-repo.yaml` 和 `ui/CredentialManager.tsx`（如果用户能访问 GitHub 仓库）。

## 流程

### 1. 建项目

```
dev_create_project(name="acme-data", template_type="datasource_plugin")
```

自动生成：
```
acme-data/
├── tinia-repo.yaml          ← 已声明 modules (credentials / datasources / menu_items / migrations / permissions)
├── handlers/
│   ├── test_connection.py   ← 验证凭证
│   └── fetch.py             ← 拉数据列表
├── migrations/
│   └── 001_init.up.sql      ← 插件自用表
└── ui/
    └── DatasourceManager.tsx ← 管理页面
```

### 2. 改 tinia-repo.yaml

和用户对齐业务：

- 凭证类型 id（全局唯一，建议加命名前缀，如 `acme_api`）
- 凭证字段（host / token / region / ...）—— 每个字段声明 `key/label/type/required`
- 数据源类型 id（如 `acme`）
- fetch_handler 路径
- 权限 key（如 `datasource_acme`）
- 菜单入口（ui 页面的路径）

**重要**：`table_prefix` 已自动生成为 `plg_<name>_`，别改，migrations 里的 SQL 要用这个前缀。

### 3. 写迁移 SQL

`migrations/001_init.up.sql` 建凭证 / 数据源两张表（和 diffgram 插件类似的结构）：

```sql
CREATE TABLE IF NOT EXISTS plg_acme_credentials (
    id          SERIAL PRIMARY KEY,
    owner_id    INTEGER NOT NULL,
    org_id      INTEGER,                    -- 用来做组织内共享（推荐）
    name        VARCHAR(255) NOT NULL,
    type        VARCHAR(50) NOT NULL DEFAULT 'acme_api',
    data        TEXT NOT NULL DEFAULT '{}', -- 凭证字段的 JSON
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS plg_acme_datasources (
    id            SERIAL PRIMARY KEY,
    owner_id      INTEGER,
    org_id        INTEGER,
    name          VARCHAR(255) NOT NULL,
    type          VARCHAR(50) NOT NULL DEFAULT 'acme',
    credential_id INTEGER REFERENCES plg_acme_credentials(id) ON DELETE SET NULL,
    config        TEXT NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_plg_acme_cred_org ON plg_acme_credentials(org_id);
CREATE INDEX IF NOT EXISTS idx_plg_acme_ds_org  ON plg_acme_datasources(org_id);
```

建议：
- 两张表都带 `org_id` 列 —— 做**组织内共享**（一个管理员建的凭证，同组织成员能用）
- 用 `owner_id` 记谁建的，作审计

### 4. 写 handler 脚本

#### `handlers/test_connection.py`

stdin: `{"credentials": {...}}`
stdout: `{"success": true}` 或 `{"success": false, "error": "..."}`

```python
import json, sys, urllib.request

req = json.loads(sys.stdin.read())
creds = req.get("credentials", {})

host = creds.get("host")
token = creds.get("token")

if not (host and token):
    print(json.dumps({"success": False, "error": "host 和 token 必填"}))
    sys.exit(0)

# 实际 ping 一下远端
try:
    r = urllib.request.Request(f"{host}/api/ping",
                               headers={"Authorization": f"Bearer {token}"})
    urllib.request.urlopen(r, timeout=10)
    print(json.dumps({"success": True}))
except Exception as e:
    print(json.dumps({"success": False, "error": str(e)}))
```

#### `handlers/fetch.py`

stdin: `{"datasource_config": {...}, "credentials": {...}, "query": {folder, search, page, limit}}`
stdout: `{"items": [...], "total": N}`

`items` 每项至少含 `id / name / media_type / content_url`（或 `blob_uri`）。这些字段会出现在 Tinia 数据集浏览页和后续 `materialize_node` 的下载逻辑里。

```python
import json, sys
req = json.loads(sys.stdin.read())
q = req.get("query", {})
# 调外部 API …
items = [
    {"id": "1", "name": "sample.wav", "media_type": "audio/wav",
     "content_url": "https://acme.example.com/data/1.wav"}
]
print(json.dumps({"items": items, "total": len(items)}))
```

### 5. 写 UI 管理页（可选）

`ui/DatasourceManager.tsx` 是个完整的 React 组件，在 Tinia Web UI 里通过左侧 "数据源-Xxx" 菜单进入。职责：
- 列凭证 / 数据源实例
- 增删改
- 连接测试

**重要：UI 里访问插件自己的表要通过 plugin-db API**：
```tsx
async function dbQuery(sql, params) {
  const res = await fetch('/api/v1/plugin-db/query', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({ sql, params }),
  })
  return res.json()
}
```

**组织内共享模式**（推荐）：按 `WHERE org_id = $1` 过滤，不是 owner_id。plugin-db 响应里会注入 `org_id` / `user_id` / `is_super_admin`，前端直接用。

详细样板参考 **Tinia_nodes_diffgram 的 ui/CredentialManager.tsx / DatasourceManager.tsx**。

### 6. 测试

```
dev_reload(project_id)
```

成功后：
1. Tinia Web UI 刷新，左侧侧栏应该出现你声明的菜单
2. 打开菜单，UI 会加载
3. 在页面里建凭证、建数据源
4. 在流程编辑器里用 `dataset_node` 选这个新数据源

## 关键区别：数据源插件 vs 分析节点插件

| 方面 | 分析节点插件（basic/analysis） | 数据源插件（datasource_plugin） |
|---|---|---|
| 产出 | 节点（nodes/*/）| 模块（handlers + migrations + ui） |
| 运行 | 流程图里拖节点连线 | 外部数据进 Tinia 的入口 |
| 数据存储 | 不自己建表 | 自己建表存凭证 / 数据源配置 |
| UI | 可选节点视图 | 必选管理页（在侧栏菜单） |

## 相关 Skill

- tinia-repo.yaml modules 字段 → 「tinia-repo.yaml 字段速查」
- 选模板 → 「从模板开始」
