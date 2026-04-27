---
name: result-view
display_name: 节点结果视图（Viewer.tsx / ViewerLoader.tsx）
description: 节点跑完后用户点"查看数据"看到的视图怎么写 —— 按数据类型给完整模板，强制要求先读官方源码学风格
user-invocable: false
allowed-tools: mcp__tinia__nodes_list,mcp__tinia__nodes_describe,mcp__tinia__nodes_read_source,mcp__tinia__dev_read_file,mcp__tinia__dev_write_file,mcp__tinia__dev_edit_file
---

# 节点结果视图：按数据类型选模板 + 强制看官方实现

## 🚨 第一条铁律：写之前必照抄官方实现

**Tinia 主应用对 Viewer 视觉有强约定**（左右布局 / 工具条 / item 列表 / 多视图切换 / 颜色 token 等），文档说不清楚，**直接读 1-2 个官方节点的源码 60 秒就懂**。

```
nodes_list({namespace: "bestfunc"})              # 找一个跟你输出数据类型相同的官方节点
nodes_describe(选中的 key, fields=["meta","ui"])  # 仅 source_files 清单
nodes_read_source(key, "ui/ViewerLoader.tsx")    # 看官方加载层（fetch / loading / 错误）
nodes_read_source(key, "ui/Viewer.tsx")          # 看官方主视图（布局 / 控件 / 图表）
```

**违反后果**（不是建议，是事实）：
- ❌ 自创 `<Section>` `<Field>` `<Card>` 包装组件 → 风格不一致 → **用户必让重写**
- ❌ 顶部写长篇说明文字 → 占空间 + 跟其他节点不一致 → **用户必让重写**
- ❌ 自己手画柱状图（div + width%）→ 跟主应用的 uplot/echarts 风格脱节 → **用户必让重写**
- ❌ 用自己的颜色 hex（如 `#3b82f6`）→ 不跟主题切换 → **用户必让重写**

**正确做法 = 抄官方实现 + 改业务字段**。

---

## 按节点输出类型选官方参考

| 你的节点输出 type | 推荐先读这个官方节点的 Viewer | 关键技术 |
|---|---|---|
| `IndicatorData`（单值/时序指标，如声级、能量频带） | **bestfunc/indicator_viewer** | `uplot`（时序图）+ Tailwind 柱状条（频谱）+ 左右布局 + 多视图切换 |
| `FeatureMatrix` / 高维特征 | **bestfunc/cluster_explore** | `echarts` 散点图 / 热力图 |
| `AnomalyResult` | **bestfunc/zscore_anomaly** | Tailwind 表格 + 高亮异常行 + 阈值控件 |
| `MaterializedDataset` / `AudioData` | **bestfunc/spectrum_viewer** | `AudioPlayer` 组件（在 spectrum_viewer/ui/AudioPlayer.tsx）+ uplot 频谱 |
| `Table` / 表格类 | **bestfunc/matrix_view** | `<table>` + Tailwind |
| `AnnotationLayer` | **bestfunc/active_segment** | 段落叠加层 + 时间线 |
| 简单 Json 总结 | preview-first 模板（见下） | 不需要图表，直接 `out.preview` 渲染 |

---

## Viewer.tsx vs ViewerLoader.tsx —— 写哪个

| 文件 | 何时用 | Props |
|---|---|---|
| `ui/Viewer.tsx` | **节点输出一种视图就够**（柱状图 / 表格 / 文本） | `({ runId, nodeId, outputs })` |
| `ui/ViewerLoader.tsx` | 节点要**多视图切换**（如指标查看器：列表 / 时序 / 频谱 / 散点 N 选 1）| `({ runId, nodeId, outputs, nodeParams })` —— 自己 lazy load 子 viewer |

**只写一个**。两个都存在时前端优先 ViewerLoader，Viewer 永远不被加载。简单节点只写 `Viewer.tsx`，前端会自动 fallback。

---

## 模板 A：preview-first（最简，输出小数据 < 64KB）

适用：节点输出几个数字 / 一个简单 list / JSON 总结，**不需要图表**。

```tsx
import { useEffect, useState } from 'react'

interface ResultPayload {
  // 你的节点输出 JSON 形状
}

export default function Viewer({ outputs }: { runId: string; nodeId: string; outputs: any[] }) {
  const [data, setData] = useState<ResultPayload | null>(null)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const out = outputs.find((o: any) => o.port_key === 'result')
    if (!out) { setError('未找到 result 端口输出'); return }
    if (out.preview && !out.truncated) { setData(out.preview); return }
    fetch(out.url).then((r) => r.json()).then(setData).catch((e) => setError(String(e)))
  }, [outputs])

  if (error) return <div className="p-4 text-sm text-danger">加载错误：{error}</div>
  if (!data) return <div className="p-4 text-sm text-text-muted">加载中...</div>

  return (
    <div className="p-4 space-y-3 text-sm">
      {/* 用主应用 token，不要自定义颜色 */}
      <div className="text-text-secondary">字段 1：<span className="text-text-primary font-mono">{data.field1}</span></div>
      ...
    </div>
  )
}
```

---

## 模板 B：左右布局 + 多 item 切换（最常用）

适用：节点输出**多个 item**（每个 item 一组指标），用户要逐个看。

```tsx
import { useEffect, useMemo, useState } from 'react'
import { cn } from '@/lib/utils'
import { api } from '@/api/client'

interface Item { item_id: string; name?: string; /* ... */ }
interface ViewData { items: Item[]; meta?: any }

export default function Viewer({ runId, nodeId }: { runId: string; nodeId: string; outputs: any[] }) {
  const [data, setData] = useState<ViewData | null>(null)
  const [activeId, setActiveId] = useState<string | null>(null)

  useEffect(() => {
    fetch(`/api/v1/graph-runs/${runId}/nodes/${nodeId}/blob/result`, {
      headers: { Authorization: `Bearer ${api.getToken()}` },
    }).then((r) => r.json()).then(setData).catch(() => {})
  }, [runId, nodeId])

  useEffect(() => {
    if (data?.items.length && !activeId) setActiveId(data.items[0].item_id)
  }, [data, activeId])

  if (!data) return <div className="p-4 text-sm text-text-muted">加载中...</div>

  const active = data.items.find((it) => it.item_id === activeId)

  return (
    <div className="h-full flex bg-bg text-text-primary">
      {/* 左侧 item 列表（固定宽度，可滚动） */}
      <aside className="w-56 border-r border-border overflow-auto p-2 space-y-1 shrink-0">
        {data.items.map((it) => (
          <button
            key={it.item_id}
            onClick={() => setActiveId(it.item_id)}
            className={cn(
              'w-full text-left px-2 py-1.5 rounded text-xs transition-colors',
              activeId === it.item_id
                ? 'bg-primary/15 text-primary border border-primary/30'
                : 'text-text-secondary hover:bg-card-hover',
            )}
          >
            <div className="truncate">{it.name || it.item_id}</div>
          </button>
        ))}
      </aside>

      {/* 右侧主视图 */}
      <main className="flex-1 overflow-auto p-4">
        {active ? <ItemDetail item={active} /> : <div className="text-text-muted text-sm">请从左侧选择</div>}
      </main>
    </div>
  )
}

function ItemDetail({ item }: { item: Item }) {
  // 此处放主图表 / 表格 / 控件
  return <div>...</div>
}
```

---

## 模板 C：多视图切换（层叠 / 平铺 / 竖铺 / 卡片 / 3D）

适用：同一份数据有多种展示方式（如指标列表 / 时序图 / 频谱图 / 散点图）。

**用 ViewerLoader.tsx**（不是 Viewer.tsx），在顶部 tab 切换 + 子组件 lazy 加载：

```tsx
import { useState, lazy, Suspense } from 'react'
import { Layers, LayoutGrid, Rows3 } from 'lucide-react'
import { cn } from '@/lib/utils'

const StackedView = lazy(() => import('./views/StackedView'))
const GridView    = lazy(() => import('./views/GridView'))
const TimelineView = lazy(() => import('./views/TimelineView'))

type ViewMode = 'stacked' | 'grid' | 'timeline'

export default function ViewerLoader({ runId, nodeId, outputs, nodeParams }: {
  runId: string; nodeId: string; outputs: any[]; nodeParams?: any
}) {
  const [mode, setMode] = useState<ViewMode>('stacked')

  return (
    <div className="h-full flex flex-col bg-bg">
      {/* 顶部视图切换 tab */}
      <div className="flex items-center gap-1 px-3 py-2 border-b border-border shrink-0">
        <ViewBtn icon={Layers}     active={mode === 'stacked'}  onClick={() => setMode('stacked')}  label="层叠" />
        <ViewBtn icon={LayoutGrid} active={mode === 'grid'}     onClick={() => setMode('grid')}     label="平铺" />
        <ViewBtn icon={Rows3}      active={mode === 'timeline'} onClick={() => setMode('timeline')} label="时序" />
      </div>

      <div className="flex-1 overflow-hidden">
        <Suspense fallback={<div className="p-4 text-text-muted text-sm">加载视图...</div>}>
          {mode === 'stacked'  && <StackedView  runId={runId} nodeId={nodeId} outputs={outputs} />}
          {mode === 'grid'     && <GridView     runId={runId} nodeId={nodeId} outputs={outputs} />}
          {mode === 'timeline' && <TimelineView runId={runId} nodeId={nodeId} outputs={outputs} />}
        </Suspense>
      </div>
    </div>
  )
}

function ViewBtn({ icon: Icon, active, onClick, label }: any) {
  return (
    <button
      onClick={onClick}
      className={cn(
        'inline-flex items-center gap-1 px-2 h-7 rounded text-[11px] transition-colors',
        active
          ? 'bg-primary/15 text-primary'
          : 'text-text-muted hover:bg-card-hover hover:text-text-secondary',
      )}
    >
      <Icon className="w-3 h-3" />
      <span>{label}</span>
    </button>
  )
}
```

---

## 数据类型 ↔ 推荐图表库

| 数据形态 | 用什么 | 关键代码 |
|---|---|---|
| 时序数据（一条曲线/多条曲线） | **uplot** | `import uPlot from 'uplot'; new uPlot(opts, data, container)` |
| 柱状图（< 30 条） | **Tailwind div 条** | `<div className="h-3 bg-primary/40" style={{width: \`${pct}%\`}} />` |
| 柱状图（> 30 条 / 需要交互） | **echarts** | `import * as echarts from 'echarts'; chart.setOption({type:'bar',...})` |
| 散点图 / 热力图 / 多系列 | **echarts** | 同上 |
| 表格 | 原生 `<table>` + Tailwind | `<table className="w-full text-xs"><thead className="bg-card-hover">...</thead></table>` |
| 音频波形 / 频谱 + 播放 | **复用 spectrum_viewer/AudioPlayer** | `import AudioPlayer from '../../spectrum_viewer/ui/AudioPlayer'`（dev 沙箱里需要 dev_read 拿过来本地放） |

**这些库主应用都打包了**，dev 沙箱 require shim 直接 `import` 即可（参见 `params-form` skill 的"可用依赖（白名单）"）。

---

## 颜色 / 视觉 token 速查

**绝对禁止**自定义 hex。全部走主应用 Tailwind token：

| 用途 | 类名 |
|---|---|
| 主背景 | `bg-bg` |
| 卡片背景 | `bg-card` / `bg-card-hover` |
| 输入框背景 | `bg-input` |
| 边框 | `border-border` / `border-border/30`（更淡） |
| 主文字 | `text-text-primary` |
| 次文字 | `text-text-secondary` |
| 提示文字 | `text-text-muted` |
| 主色（按钮/选中态） | `text-primary` / `bg-primary/15` / `border-primary/30` |
| 成功 | `text-success` |
| 警告 | `text-warning` / `bg-warning/10` |
| 错误 | `text-danger` / `bg-danger/10` |

---

## 常见反模式（看到就 stop，重写）

| 反模式 | 为什么错 | 替代 |
|---|---|---|
| 自己写 `<Section>` `<Field>` `<Card>` 包装组件 | 跟其他节点不一致 | 直接 `<div className="space-y-3">` + 原生 label / div |
| 顶部塞节点说明 / 设计原理长文 | 占空间，节点说明走流程编辑器 ⓘ 帮助按钮 | 直接进数据展示 |
| 用自定义 hex 颜色（`#3b82f6` 等） | 主题切换时不跟随 | 全部用主应用 Tailwind token |
| 手画柱状图 / 散点（div + 计算） | 风格不一致，无交互 | uplot / echarts / Tailwind 简单 div 条 |
| 内嵌大段 markdown 说明 | 跟其他节点不一致 | 帮助按钮（点击弹模态） |
| 把所有指标平铺一长串 | 用户找不到关键信息 | 左右布局，左侧列表 + 右侧主视图 |

---

## outputs[] 字段（必看）

跟 `params-form` 的 outputs 字段表一致 —— **优先用 `out.preview`**（小数据已解析），大数据走 `fetch(out.url)`（自带 ?token= query）。详见 `create-node` skill 的 outputs 字段表。

---

## 写完检查 checklist

- [ ] 写之前调过至少 1 次 `nodes_read_source` 看了官方相似节点的 Viewer
- [ ] 没有自创 `Section`/`Field`/`Card` 包装组件
- [ ] 颜色全部用主应用 Tailwind token，没有任何 hex
- [ ] 多 item 时用了左右布局（左列表 + 右主图）
- [ ] 图表用 uplot / echarts，不是手画 div
- [ ] 没有顶部长篇说明文字
- [ ] outputs 优先用 `preview`，大数据才 `fetch(out.url)`
