---
name: params-form
display_name: 节点参数表单（下拉/开关/滑块/条件字段）
description: 节点在图编辑器右侧参数面板怎么渲染 —— 默认只有输入框，想要下拉/开关/条件显示必须写 ui/ParamsForm.tsx
user-invocable: false
---

# 节点参数表单：默认渲染 + 自定义 UI

## 开门第一条必知

**Tinia 流程编辑器的默认参数面板只会渲染 `<input type="text">`。**

即使你在 `schemas/params.schema.json` 里写了 `"enum"`、`"type": "boolean"`、`"minimum" / "maximum"`、`"format": "color"` —— **默认渲染器全部忽略**，只按 `type === "number" | "integer"` 做个字符串→数字的转换。

源码证据（`client/src/pages/GraphEditor.tsx:1109-1128`）：

```tsx
return (
  <div className="space-y-3">
    {Object.entries(def?.params_schema?.properties || {}).map(([k, schema]) => (
      <div key={k}>
        <label>{schema.title || k}</label>
        <input
          type="text"
          value={params?.[k] ?? schema.default ?? ''}
          onChange={(e) => updateParams(k, schema.type === 'integer' || schema.type === 'number'
            ? Number(e.target.value)
            : e.target.value)}
        />
      </div>
    ))}
  </div>
)
```

没有 `<select>`、没有 `<input type="checkbox">`、没有 `<input type="range">` —— 想要这些必须自己写 `ui/ParamsForm.tsx`。

---

## ⚠ 写之前必读：官方风格参考

**ParamsForm 的视觉风格在 Tinia 主应用是有强约定的**。AI 自己凭空写很容易"画风跑偏" —— 比如把节点说明写成长篇大论的顶部说明区、堆叠多层折叠面板、用彩色方块做按钮组，这些都**不符合**主应用风格。

**正确做法**：写 `ui/ParamsForm.tsx` 之前，先用 `nodes_read_source` 读一个**类型相似的官方节点**的实现做参考：

```
nodes_list({namespace: "bestfunc"})              → 找相似节点
nodes_describe(key)                               → 看 source_files 列表
nodes_read_source(key, "ui/ParamsForm.tsx")       → 抄风格、抄结构
```

**官方风格关键约束**：
- **不写"顶部长说明"** —— 节点描述走 ⓘ 帮助按钮（点击弹模态弹窗），不在表单里堆文字
- **不堆叠折叠面板** —— 简洁的字段列表 + 输入控件即可
- **复用主应用 Tailwind token**：`bg-card / border-border / text-text-primary / text-text-secondary / text-text-muted` 等，**不要自定义 hex 颜色**
- 控件统一用本 skill"常用控件速查"里的写法

---

## 两层分工：schema 是协议，ParamsForm 是 UI

| 层 | 文件 | 作用 | 谁读它 |
|---|---|---|---|
| 参数协议 | `schemas/params.schema.json` | 声明参数名、类型、默认值、范围 | Python `run.py`（通过 `rt.task["params"]`）、Dev Studio 保存/校验、打包发布时打入 node manifest |
| 表单 UI | `ui/ParamsForm.tsx` | 渲染图编辑器右侧参数面板 | 前端（若无 ParamsForm，前端 fallback 到默认文本框渲染） |

**两者都要写**：schema 是硬协议（Python 靠它读参数），ParamsForm 是用户体验（让用户能选下拉而不是敲字）。

---

## 决策树：你的节点需要 ParamsForm 吗？

| 情况 | 建议 |
|---|---|
| 所有参数都是自由文本 / 数字（没有枚举、没有布尔、没有条件显示）| 只写 schema，不写 ParamsForm |
| 有任何枚举（只能从固定值里选）| **必写 ParamsForm**（否则用户能输入任意文字） |
| 有布尔开关 | **必写 ParamsForm**（否则用户输入 "true"/"false" 字符串） |
| 数值有合理范围且希望滑块 | **必写 ParamsForm** |
| 参数之间有依赖（选 A 才显示 B）| **必写 ParamsForm**（schema 不支持条件显示） |
| 要让用户选凭证 / 数据源 / 外部资源 | **必写 ParamsForm**（平台无内置选择器，要自己 fetch 后端 API） |

---

## Dev Studio 期间的重要限制

**⚠️ dev 项目期间写的 `ui/ParamsForm.tsx` 不会加载到流程编辑器。**

前端只 glob 扫描 `server/data/plugins/*/nodes/*/ui/ParamsForm.tsx`（见 `client/src/components/nodes/registry.ts:32`）。dev 项目在 `server/data/dev-workspace/` 下，路径不匹配。

所以：
- `dev_reload` 让节点能被图编辑器使用 ✅
- 但参数面板还是默认文本框 ❌
- 要看到 ParamsForm 真实效果：**pack-and-publish → 用户在 Web UI 安装该插件 → 刷新前端页面**

这是客观限制，没法绕过。建议开发流程：

1. 先用 schema + 默认文本框跑通 Python 逻辑
2. 再写 `ui/ParamsForm.tsx`，肉眼审 JSX
3. 打包发布，安装后再验证 UI

写 ParamsForm 时不要反复 `dev_reload` 找视觉反馈 —— 看不到的。

---

## `ui/ParamsForm.tsx` 完整规范

### 目录位置

```
nodes/my_node/
├── node.yaml
├── schemas/
│   └── params.schema.json    # 参数协议（必写）
├── runtime/
│   └── run.py
└── ui/
    └── ParamsForm.tsx        # 表单 UI（可选但强烈建议）
```

### Props 签名（无例外）

```tsx
interface Props {
  params: any                              // 当前参数 dict（可能为空）
  onChange: (next: any) => void            // 整体替换参数 dict
}
```

**没有其他 props**。拿不到：
- 当前项目 ID
- 其他节点的输出
- 用户凭证列表
- 流程上下文

如果需要这些，在组件内用 `useEffect` + `fetch('/api/v1/...')` 自己拉。

### 更新参数的习惯写法

```tsx
const set = (k: string, v: any) => onChange({ ...params, [k]: v })
```

永远整体替换（spread 保留其他字段），不要原地改 params 对象。

### 可用依赖（白名单）

dev 项目的 tsx 只允许 import 以下模块（其它会被 sandbox reject）：

| 依赖 | 用途 |
|---|---|
| `react` | `useState` / `useEffect` / `useMemo` / `useRef` 等 |
| `react-dom` | 仅在需要 portal 时；ParamsForm 通常用不到 |
| `lucide-react` | 全部图标，按需 named import |
| `zustand` | 需要复杂状态管理（少见） |
| `echarts` / `echarts/core` / `echarts/charts` / `echarts/components` / `echarts/renderers` / `echarts-gl` | 复杂图表（散点 / 折线 / 3D） |
| `uplot` | 时序图（声学指标常用） |
| `@/lib/utils` | `cn`（Tailwind class 合并） |
| `@/api/client` | `api.get` / `api.post` 等调 Tinia 主 API |

任何 Tailwind class 都可用（主应用已扫描所有 dev 项目的 ui/ 目录）。

**项目内相对 import**：允许 `./` `../`，但解析后必须仍在当前 dev 项目目录内（不能 `../../别的项目`）。

**不能 import**：
- `axios` / `dayjs` / `lodash` 等任何主应用没打包的第三方包 → 用 `fetch` / 原生 Date / 自己写
- `@/components/*`（主应用业务组件）→ 高耦合，主应用一改插件就崩，自己用原生 `<select>` `<input>` 配 Tailwind
- `@/stores/*`（主应用全局状态）→ 容易污染，要拿数据用 `@/api/client`
- 任何 node 模块（`fs` / `child_process` 等）→ 浏览器代码本来也用不到

要扩白名单：联系平台维护者（编辑 `server/internal/dev/build_sandbox.go` + `client/src/lib/devComponentLoader.ts`，两处必须对称改）。

---

## 完整示例：含条件字段 + 下拉 + 数字 + 开关 + 帮助弹窗

以下是 Tinia 内置节点 `sample_node` 的 ParamsForm（抽样节点），**可直接当模板用**。

```tsx
import { useState } from 'react'
import { HelpCircle, X } from 'lucide-react'

interface Props {
  params: any
  onChange: (next: any) => void
}

const STRATEGIES = [
  { value: 'random', label: '随机抽样', desc: '随机抽取固定数量或按比例抽取' },
  { value: 'first',  label: '取前 N 条', desc: '按原始顺序取前 N 条' },
  { value: 'top',    label: '按字段排序取前 N', desc: '指定字段排序取 Top N' },
]

export default function SampleNodeParams({ params, onChange }: Props) {
  const [showHelp, setShowHelp] = useState(false)
  const strategy = params?.strategy || 'random'
  const set = (k: string, v: any) => onChange({ ...params, [k]: v })

  const inputCls = 'w-full h-8 bg-input border border-border rounded px-2 text-sm'

  return (
    <div className="space-y-3">
      {/* 帮助入口 */}
      <button onClick={() => setShowHelp(true)}
        className="flex items-center gap-1 text-[10px] text-primary hover:opacity-80">
        <HelpCircle className="w-3.5 h-3.5" /> 说明
      </button>

      {/* 下拉 - 策略 */}
      <div>
        <label className="block text-xs text-text-muted mb-1">抽样策略</label>
        <select className={inputCls} value={strategy}
          onChange={(e) => set('strategy', e.target.value)}>
          {STRATEGIES.map((s) => (
            <option key={s.value} value={s.value}>{s.label}</option>
          ))}
        </select>
        <div className="text-[10px] text-text-muted mt-1">
          {STRATEGIES.find((s) => s.value === strategy)?.desc}
        </div>
      </div>

      {/* 数字输入 */}
      <div>
        <label className="block text-xs text-text-muted mb-1">数量</label>
        <input type="number" className={inputCls}
          value={params?.count ?? 100}
          min={1}
          onChange={(e) => set('count', Number(e.target.value) || 1)} />
      </div>

      {/* 条件字段：strategy === 'random' 时显示 */}
      {strategy === 'random' && (
        <div>
          <label className="block text-xs text-text-muted mb-1">随机种子</label>
          <input type="number" className={inputCls}
            value={params?.seed ?? 0}
            onChange={(e) => set('seed', Number(e.target.value) || 0)} />
          <div className="text-[10px] text-text-muted mt-1">0 = 每次不同，&gt;0 = 可复现</div>
        </div>
      )}

      {/* 条件字段 + 开关：strategy === 'top' */}
      {strategy === 'top' && (
        <>
          <div>
            <label className="block text-xs text-text-muted mb-1">排序字段</label>
            <input type="text" className={inputCls}
              value={params?.sort_field ?? ''}
              placeholder="如 severity"
              onChange={(e) => set('sort_field', e.target.value)} />
          </div>
          <label className="flex items-center gap-2 text-xs text-text-muted">
            <input type="checkbox"
              checked={params?.sort_desc !== false}
              onChange={(e) => set('sort_desc', e.target.checked)} />
            降序
          </label>
        </>
      )}
    </div>
  )
}
```

---

## 常用控件速查

所有控件都是原生 HTML，Tailwind 风格统一用：

```
const inputCls = 'w-full h-8 bg-input border border-border rounded px-2 text-sm'
```

### 下拉（枚举单选）

```tsx
<select className={inputCls} value={params?.method ?? 'sfm'}
  onChange={(e) => set('method', e.target.value)}>
  <option value="sfm">谱平坦度</option>
  <option value="crest">峰值因子</option>
</select>
```

### 数字输入（带范围）

```tsx
<input type="number" className={inputCls}
  value={params?.threshold ?? 3.0}
  min={1} max={10} step={0.1}
  onChange={(e) => set('threshold', Number(e.target.value) || 0)} />
```

### 滑块（视觉范围调节）

```tsx
<input type="range" className="w-full"
  min={0} max={100} step={1}
  value={params?.alpha ?? 50}
  onChange={(e) => set('alpha', Number(e.target.value))} />
<div className="text-[10px] text-text-muted">当前：{params?.alpha ?? 50}</div>
```

### 开关（布尔）

```tsx
<label className="flex items-center gap-2 text-xs text-text-muted">
  <input type="checkbox"
    checked={!!params?.use_time}
    onChange={(e) => set('use_time', e.target.checked)} />
  启用时间维度
</label>
```

### Chips 多选

```tsx
{['low', 'mid', 'high'].map((k) => {
  const active = (params?.bands ?? []).includes(k)
  return (
    <button key={k}
      onClick={() => set('bands', active
        ? params.bands.filter((x) => x !== k)
        : [...(params?.bands ?? []), k])}
      className={`px-2 py-0.5 text-[11px] rounded border ${
        active ? 'border-primary text-primary bg-primary/10' : 'border-border text-text-muted'
      }`}>
      {k}
    </button>
  )
})}
```

### 颜色

```tsx
<input type="color" className="w-10 h-8 rounded border border-border"
  value={params?.color ?? '#5DCAA5'}
  onChange={(e) => set('color', e.target.value)} />
```

### 多行文本

```tsx
<textarea rows={4}
  className="w-full bg-input border border-border rounded p-2 text-sm font-mono"
  value={params?.note ?? ''}
  onChange={(e) => set('note', e.target.value)} />
```

### 枚举按钮组（代替下拉，视觉更强）

```tsx
<div className="grid grid-cols-3 gap-1">
  {['a', 'b', 'c'].map((v) => (
    <button key={v}
      onClick={() => set('mode', v)}
      className={`h-8 rounded text-xs border ${
        params?.mode === v
          ? 'border-primary text-primary bg-primary/10'
          : 'border-border text-text-muted hover:text-text-primary'
      }`}>
      {v}
    </button>
  ))}
</div>
```

---

## Tailwind 主题令牌

用这些 class 和 Tinia 视觉一致（别自己编色号）：

| 用途 | class |
|---|---|
| 输入背景 | `bg-input` |
| 边框 | `border-border` |
| 主色（强调）| `text-primary` / `bg-primary` |
| 卡片背景 | `bg-card` |
| 主文字 | `text-text-primary` |
| 次文字 | `text-text-secondary` |
| 说明文字 | `text-text-muted` |
| hover 背景 | `hover:bg-card-hover` |
| 错误色 | `text-danger` |

尺寸约定：
- 整体容器用 `space-y-3` 统一间距
- 标签 `text-xs text-text-muted mb-1`
- 控件高度 `h-8`（对齐面板密度）
- 说明文字 `text-[10px] text-text-muted mt-1`

---

## schema + ParamsForm 怎么配合

**schema 是 Python 的协议**，无论是否写 ParamsForm 都要写。推荐最小 schema（只声明 key + type + default，title/description 可选）：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "strategy": { "type": "string", "default": "random" },
    "count":    { "type": "integer", "default": 100 },
    "seed":     { "type": "integer", "default": 0 },
    "sort_field": { "type": "string" },
    "sort_desc":  { "type": "boolean", "default": true }
  },
  "required": ["strategy", "count"]
}
```

Python 侧永远用 `get` + 默认值兜底：

```python
p = rt.task.get("params") or {}
strategy = p.get("strategy", "random")
count = int(p.get("count", 100))
seed = int(p.get("seed", 0))
sort_desc = bool(p.get("sort_desc", True))
```

---

## 业务专用选择器（凭证 / 数据源）

**平台没有内置凭证选择器或数据源选择器组件**。但可以在 ParamsForm 里自己 fetch：

```tsx
import { useState, useEffect } from 'react'

export default function MyParams({ params, onChange }: Props) {
  const [credentials, setCredentials] = useState<Array<{id: number, name: string}>>([])

  useEffect(() => {
    fetch('/api/v1/credentials', {
      headers: { Authorization: `Bearer ${localStorage.getItem('token') ?? ''}` }
    })
      .then((r) => r.json())
      .then((d) => setCredentials(d.credentials ?? []))
      .catch(() => {})
  }, [])

  return (
    <select value={params?.credential_id ?? ''}
      onChange={(e) => onChange({ ...params, credential_id: Number(e.target.value) })}>
      <option value="">选择凭证...</option>
      {credentials.map((c) => (
        <option key={c.id} value={c.id}>{c.name}</option>
      ))}
    </select>
  )
}
```

Python 侧通过 `rt.get_datasource(credential_id)` 或类似 API 读凭证 —— 见 sdk-reference skill。

---

## 动态端口数参数 `_port_count`

合并型节点（如 concat / merge）用 `_port_count` 让用户调节动态输入口数量。这是**平台特殊参数**，**不要写进 schema**（会重复），Tinia 看到 `node.yaml` 里 `dynamic_inputs.enabled: true` 自动在参数面板底部添加 +/- 控件。

```yaml
# node.yaml
dynamic_inputs:
  enabled: true
  prefix: in
  min_ports: 2
  max_ports: 16
  label: "输入"
```

Python 读法：

```python
port_count = int(p.get("_port_count", 2))
for i in range(1, port_count + 1):
    data = rt.inputs.get(f"in_{i}")
```

---

## 常见坑

### 1. schema 里写了 enum，期望变下拉

**不会**。默认渲染器不看 enum。要下拉必须写 ParamsForm。

### 2. ParamsForm 写完了但看不到效果

dev 项目的 ParamsForm.tsx 不会被加载。打包发布 + 安装后才生效（见上文限制）。

### 3. 用 camelCase 命名参数

用 snake_case。全仓库约定，Python 侧可直接用变量名对应 schema key：`trim_z`、`max_iter`、`use_time`，不要 `trimZ`。

### 4. 不设默认值

非必填字段若不设 `default`，Python 侧 `p.get("key")` 可能返回 `None`。要么在 schema 里设 `default`，要么在 `p.get("key", fallback)` 里设 —— 推荐两处都设，明确声明。

### 5. onChange 忘了 spread

```tsx
// ❌ 错：其他参数被清空
onChange({ [k]: v })

// ✅ 对
onChange({ ...params, [k]: v })
```

### 6. 修改现有节点参数后，已有流程不兼容

删参数 → 老图 params 字典里残留 key，但 Python 会忽略（用 `p.get`）。
改参数默认值 → 老图会继续用旧值（已保存的值优先于 default）。
换参数名 → 老图视为"该参数不存在"，走默认值 —— **避免换参数名**，如必须改，发 bump-version 时在 run.py 加兼容代码：

```python
# 兼容老参数名
strategy = p.get("strategy") or p.get("method") or "random"
```

---

## 一句话总结

**schema 声明协议，ParamsForm 做 UI**。任何超出"字符串/数字文本框"的需求（下拉、开关、条件、滑块、凭证选择）都要写 `ui/ParamsForm.tsx`。默认渲染器只是"最低保障"，不是交付体验。
