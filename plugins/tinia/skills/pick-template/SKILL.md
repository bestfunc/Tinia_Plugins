---
name: pick-template
display_name: 从模板开始
description: Tinia Developer Studio 的 4 个项目模板各自生成什么、适合什么场景、怎么选
user-invocable: false
---

# 从模板开始

Tinia 新建项目有 4 个模板，用户要先选一个。

## `basic_node` — 最简 Python 节点

**生成**：
```
project/
├── tinia-repo.yaml
├── sdk/python/README.md
└── nodes/example/
    ├── node.yaml         # 一个 data:AudioData → result:IndicatorData 的分析节点
    ├── runtime/
    │   ├── run.py        # 带 TODO 注释的骨架
    │   └── requirements.txt
    └── schemas/params.schema.json   # 一个 threshold 参数
```

**适合**：
- 第一次用 Tinia 的新手
- 只想加一个独立分析节点
- 不需要自定义结果视图

**典型例子**：声级计、FFT 频谱、相关系数计算…

## `analysis_node` — 带结果视图的节点

在 `basic_node` 基础上**多生成**：
```
nodes/example/
├── ...（同 basic）
└── ui/Viewer.tsx         # 自定义结果视图（React 组件）
```

`node.yaml` 里多一段：
```yaml
ui:
  result_view: ui/Viewer.tsx
```

**适合**：
- 节点输出不是简单几个数字，需要图表 / 表格 / 音频播放器
- 希望用户点"查看结果"看到定制的界面

**典型例子**：频谱查看器、Z-score 异常详情、聚类散点图…

**和 basic 对比**：什么时候选它？输出不止"几个指标数字" —— 比如要画图、要让用户和结果交互。

## `datasource_plugin` — 数据源插件

**生成**：完全不同于以上 —— 没有 `nodes/` 目录，而是：
```
project/
├── tinia-repo.yaml    # 声明 modules: credentials / datasources / menu_items / migrations / permissions
├── sdk/python/README.md
├── handlers/
│   ├── test_connection.py
│   └── fetch.py
├── migrations/
│   └── 001_init.up.sql
└── ui/
    └── DatasourceManager.tsx
```

**适合**：
- 接入一个外部数据系统（标注平台、SCADA、数据湖、API…）
- 需要让组织里多人共享凭证 / 数据源配置
- 有自己的管理界面

**典型例子**：Diffgram 标注系统接入（`Tinia_nodes_diffgram`）、LIMS 系统接入、阿里云 OSS 桶接入…

详细见 `datasource-plugin` skill。

## `empty` — 完全自定义

**生成**：
```
project/
├── tinia-repo.yaml
└── sdk/python/README.md
```

仅此而已。

**适合**：
- 只想打包一组模板（tinpack）分发
- 特殊结构的插件（如只有 SDK、只有菜单页）
- 知道自己在干什么，不想要默认骨架

## 选型决策树

```
AI 自动开发（默认场景）
│
├─ 数据源插件 → datasource_plugin（这个模板的 example 是必需的脚手架，不算冗余）
│
└─ 其它所有节点（分析 / FFT / 声级 / 视图等）
   → empty（AI 用 dev_create_node scaffold 节点骨架，不带 example）
```

## 为什么 AI 默认用 empty

basic_node / analysis_node 模板会**自带一个 example 节点**（默认 `bestfunc/level_meter` 风格的样例代码）。AI 一旦用了这种模板，就要花精力**判断哪些是 example 残留 → 删除**。常见的失败模式：
- example 节点的 schemas/params.schema.json / runtime/run.py 留在项目里没删，用户安装后看到莫名其妙的"示例节点"
- example 的 ui/Viewer.tsx 还在但功能跟用户要的节点完全无关

**正确做法**：
1. `dev_create_project(template_type="empty")` —— 项目里只有干净的 `tinia-repo.yaml`
2. `dev_create_node(key="<用户要的节点名>")` —— scaffold 一个**专属**节点骨架
3. 按需求改 node.yaml / run.py / Viewer.tsx

## 何时才用模板（用户明确指定时）

- **用户说"建个示例项目让我看看结构"** → basic_node
- **用户说"我要做数据源插件"** → datasource_plugin（这个 example 内容是脚手架本身，必需）
- **用户说"我要带视图的分析节点示例"** → analysis_node

否则一律 empty。
