---
name: 从模板开始
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

详细见「开发数据源插件」skill。

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
你要做什么？
│
├─ 添加一个分析节点（做 FFT / 声级 / 异常检测 等）
│  ├─ 结果是几个数字或简单列表 → basic_node
│  └─ 结果需要可视化（图表 / 播放器） → analysis_node
│
├─ 接入一个外部数据系统 → datasource_plugin
│
└─ 其他（模板分发 / 非常规） → empty
```

## 建议

- **第一次做**：用 basic_node。跑通全流程后再考虑高级需求
- **想做数据源**：先看 Tinia_nodes_diffgram 的源码，模仿它的结构
- **不确定**：先建 empty 再手动加文件，不如直接 basic_node 改造
