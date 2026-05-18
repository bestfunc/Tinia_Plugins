# 开发者高频问题 FAQ

> 节点开发者 / Tinia 主仓贡献者 / 集成开发者最常问的问题。给开发同事、新员工、外部开发者用。

---

## 入门

### Q：我怎么开始开发 Tinia 节点？

**A**：三种方式：

1. **AI 主导（最快）**：
   - 装 Tinia Desktop（Community 免费版）
   - 在 Claude Code 装 `tinia-desktop` plugin（在 Tinia_Plugins marketplace 里）
   - 跟 Claude Code 说 `/quickstart` → 跟着步骤走，半天能跑通第一个节点

2. **DevStudio（浏览器内）**：
   - 装 Tinia → 打开 DevStudio 页面
   - 在 monaco editor 里写代码 + 热加载
   - 适合不用 IDE 的工程师

3. **CLI + 自己 IDE**：
   - 装 `tinia-cli` 二进制
   - `tinia init my-project` scaffold
   - 在 VSCode / 任意 IDE 写
   - 适合习惯本地开发的工程师

详见 `Tinia_Cli/README.md` 和 `quickstart` skill。

### Q：节点开发要会什么语言？

**A**：

- **必需**：Python（算法本体写 run.py）
- **推荐**：TypeScript + React（写 ParamsForm.tsx + Viewer.tsx，可选但推荐）
- **不需要**：Go（Tinia engine 是 Go 写的，但节点开发者不接触 engine 代码）

如果只想做"算法节点不带 UI"，纯 Python 就够，前端用默认 Viewer。

### Q：第一个节点要多久写完？

**A**：

- **简单节点**（FFT / 滤波等单算法）：AI 辅助 1-2 小时
- **中等节点**（带 ParamsForm + Viewer）：AI 辅助 3-5 小时
- **复杂节点**（多通道 + 自定义可视化 + 测试套件）：1-2 天

---

## 节点结构

### Q：节点的目录结构是什么？

**A**：

```
nodes/my_node/
├── node.yaml           ← 元数据声明
├── runtime/
│   ├── run.py          ← 算法本体
│   ├── requirements.txt ← Python 依赖
│   └── .venv/          ← 自动创建（不入 git）
└── ui/
    ├── ParamsForm.tsx  ← 参数配置 UI
    ├── Viewer.tsx      ← 结果可视化
    └── ViewerLoader.tsx ← Viewer 懒加载入口
```

详见 `quickstart` skill 和 `Tinia/docs/plugin-design-spec.md`。

### Q：node.yaml 要写什么？

**A**：核心字段：

```yaml
key: my_node
display_name: 我的节点
description: 这个节点做 XX
category: analysis

inputs:
  audio:
    type: AudioData
    required: true

outputs:
  result:
    type: IndicatorData

params:
  threshold:
    type: number
    default: 0.5

runtime:
  type: python
  entrypoint: runtime/run.py

channels_mode: per_channel  # 多通道处理策略
```

详见 `Tinia/docs/plugin-design-spec.md` 完整字段表。

### Q：run.py 要写什么？

**A**：基本模板：

```python
from tinia_runtime import Runtime

def main():
    rt = Runtime.from_stdin()

    # 拿输入
    audio_handle = rt.task["inputs"]["audio"]
    audio_bytes = rt.fetch_blob(audio_handle)

    # 算法核心（你的代码）
    result = your_algorithm(audio_bytes, rt.task["params"])

    # 上传输出
    output_handle = rt.upload_blob(result_bytes, node_type="IndicatorData")
    rt.emit_output("result", output_handle)

    rt.emit_done()

if __name__ == "__main__":
    main()
```

详见 `sdk-reference` skill。

---

## SDK / API

### Q：tinia_runtime SDK 有哪些方法？

**A**：核心方法：

| 方法 | 用途 |
|---|---|
| `Runtime.from_stdin()` | 读 daemon 喂的 task JSON |
| `rt.fetch_blob(handle)` | 拉 input blob 内容（bytes）|
| `rt.upload_blob(bytes, node_type, content_type)` | 上传输出，返回 handle |
| `rt.emit_output(port, handle)` | 声明某 port 的输出 |
| `rt.emit_progress(value, message)` | 报进度 |
| `rt.emit_log(level, message)` | 写日志 |
| `rt.emit_error(message)` | 报错（非 fatal）|
| `rt.emit_done()` | 完成 |
| `rt.get_datasource()` | 拿 datasource 凭据（自动注入）|

详见 `Tinia_nodes/sdk/python/tinia_runtime/`。

### Q：节点能调外部 HTTP API 吗？

**A**：技术上能，但要谨慎：

- 商店审批会检查"是否泄露用户数据到外部"
- 私有节点（org_only）随便调
- 公开节点最好不要调外部 API，影响用户信任

如果需要外部 API（比如调云端 ML 服务），应该：

1. 在 node.yaml 声明
2. 让用户在 ParamsForm 里配 API key / URL
3. 文档说明数据流向

---

## 类型系统

### Q：input/output 有哪些类型？

**A**：核心类型：

| 类型 | 用途 |
|---|---|
| `AudioData` | 音频超类型（含子类型 MaterializedDataset / ProcessedDataset）|
| `IndicatorData` | 分析指标 |
| `AnnotationLayer` | 段落标注 |
| `FileBlob` | 通用文件 |
| `Any` | 透传（少用）|

详见 `types-reference` skill 和 `Tinia/docs/data-analysis-pipeline.md`。

### Q：怎么处理多通道？

**A**：声明 `channels_mode`：

| 模式 | 说明 |
|---|---|
| `per_channel`（默认）| daemon 把每通道分别喂给节点，节点处理单通道|
| `aggregated` | daemon 把全部通道一起喂，节点自己处理多通道 |

大多数算法节点用 `per_channel`（如 FFT / 滤波）；少数（如通道间相干性）用 `aggregated`。

详见 `Tinia/docs/multichannel-architecture.md`。

---

## 前端 / UI

### Q：ParamsForm.tsx 怎么写？

**A**：

```tsx
import { useState } from 'react'

interface Params {
  threshold: number
}

export default function ParamsForm({ params, onChange }: {
  params: Params
  onChange: (p: Partial<Params>) => void
}) {
  return (
    <div>
      <label>阈值</label>
      <input
        type="number"
        value={params.threshold}
        onChange={(e) => onChange({ threshold: Number(e.target.value) })}
      />
    </div>
  )
}
```

参考官方节点的 ParamsForm 学风格 —— 用 `nodes_read_source` 工具读。

详见 `params-form` skill。

### Q：Viewer.tsx 怎么写？

**A**：

```tsx
import { useEffect, useState } from 'react'

export default function Viewer({ runId, nodeId, outputs }: {
  runId: string
  nodeId: string
  outputs: any[]
}) {
  const [data, setData] = useState(null)

  useEffect(() => {
    // 从 outputs 拉数据 + 渲染
    fetch(outputs[0].uri).then(r => r.json()).then(setData)
  }, [outputs])

  return <div>{/* 用 uPlot / ECharts 画图 */}</div>
}
```

详见 `result-view` skill。

### Q：用什么图表库画频谱 / 时序？

**A**：

| 场景 | 推荐 |
|---|---|
| 时序 / 频谱（百万点）| **uPlot**（性能最好）|
| 复杂仪表盘 / 3D | **ECharts** |
| 简单条形图 | 任意 React 图表库 |
| 拖拽布局 | **react-grid-layout** |

参考 `nodes/spectrum_viewer/` 看官方 uPlot 使用。

---

## 测试 / 调试

### Q：怎么本地调试节点？

**A**：

1. `dev_reload` 让 Tinia 装载最新代码（不用重启 daemon）
2. 用 `debug-node` skill 跑测试流程
3. 看 daemon log：
   - Desktop：`<data_dir>/desktop-daemon.log`
   - Server：`/var/log/tinia/`
4. 看节点 stderr：`flow_node_stderr` MCP 工具

### Q：节点 pip install 失败怎么办？

**A**：

- 检查 `requirements.txt` 版本（不要写 `>=`，最好锁版本）
- 检查 platform 兼容（windows-amd64 / linux-amd64 / mac-arm64 wheel 是否存在）
- 用 `--prefer-binary --only-binary=numpy,scipy,...` 强制装 wheel 不源码 build
- 看 daemon log 里的 pip stderr

详见 `Tinia/docs/dev-python-nodes.md`。

### Q：节点跑很慢怎么优化？

**A**：

- **CPU**：用 numpy / scipy 向量化操作，不要 for 循环
- **内存**：大数据用 blob 句柄传，不要 inline JSON
- **IO**：避免反复 fetch_blob 同一个 handle（自己缓存）
- **多通道**：考虑 `channels_mode: aggregated` 一次处理（vs per_channel 多次起 subprocess）

---

## 发布 / 商店

### Q：怎么发布节点到商店？

**A**：

1. 用 `pack-and-publish` skill：
   - `dev_bump_version` 升版本号
   - `dev_export` 打包
2. 用 `publish-plugin` skill：
   - `plugin_publish_submit` 提交到商店
   - 选 scope: `public` / `org_only`
3. 等商店审批
4. 通过 → 上架

详见 `publish-plugin` skill 和 `Tinia/docs/plugin-publish-design.md`。

### Q：审批要多久？

**A**：

- **简单节点**（修复 bug / 小改动）：1-2 工作日
- **新节点**：3-5 工作日（含算法 review）
- **复杂节点 / 商业节点**：5-10 工作日（含安全审查）

### Q：审批不通过怎么办？

**A**：

- 商店会给详细反馈意见（哪里不符合规范）
- 改完重新提交
- 常见拒绝原因：依赖外部 API 没声明 / Python 代码风险 / UI 卡顿 / 缺测试数据

详见 `review-plugin` skill。

### Q：发布商业节点怎么定价？

**A**：

- 一次性买断：建议参考同类节点价格
- 月订阅：适合持续更新的节点
- 年订阅：折扣价
- 免费 + 付费高级版：同一节点出两版

商店分成：30% 平台拿，70% 开发者拿。月度结算。

---

## Tinia 主仓贡献

### Q：我想给 Tinia 主仓提 PR 怎么开始？

**A**：

1. 看 `Tinia/CLAUDE.md` 了解项目结构
2. 看 `Tinia/docs/` 找相关设计文档
3. 跑 `./start-dev.sh` 本地起开发环境
4. 改代码 → 测试 → 提 PR 到 main 分支
5. core team review

**注意**：主仓是闭源核心 + 部分开源（engine）的混合。开源部分欢迎 PR；闭源部分需要先联系 core team 讨论。

### Q：主仓代码结构？

**A**：

```
Tinia/
├── server/                    # Go 后端
│   ├── cmd/server/main.go     # 入口
│   ├── internal/              # 业务逻辑
│   │   ├── api/               # HTTP 路由
│   │   ├── graph/             # DAG 执行
│   │   ├── nodes/             # 节点 runtime
│   │   ├── mcp/               # MCP server
│   │   ├── activation/        # 激活
│   │   └── ...
│   └── migrations/            # SQL migrations
├── client/                    # React 前端
│   └── src/
│       ├── pages/             # 路由页面
│       ├── components/        # 通用组件
│       ├── stores/            # zustand
│       └── lib/               # 工具
├── desktop/                   # Wails 桌面包装
├── docs/                      # 设计文档
└── build-desktop.sh           # 桌面打包脚本
```

### Q：本地开发环境怎么起？

**A**：

```bash
cd Tinia
./start-dev.sh
# 自动启动：
# - PostgreSQL (docker)
# - server (Go，端口 18721)
# - client (Vite，端口 18722)
```

详见 `Tinia/CLAUDE.md` 的"开发"章节。

---

## 跟 AI 工具协作

### Q：AI 写的代码我怎么 review？

**A**：

- AI 会调用 MCP 工具改代码，你能在 DevStudio 看 diff
- 对关键算法（不是 UI），review 数学正确性 + 测试用例覆盖
- 测试通过 + 你审过 → reload + 跑测试流程

### Q：AI 总是写错怎么办？

**A**：

- 把"参考节点"明确告诉 AI："参考 nodes/level_meter 的风格"
- 给具体错误反馈："Viewer 的 X 轴标签应该是 '频率 (Hz)' 不是 'X'"
- 用 skill：`quickstart` / `create-node` 已经把常见错误模式写进 prompt

### Q：AI 能完全代替开发者吗？

**A**：不能。当前 AI 的角色：

- **能做**：脚手架 / 重复代码 / 标准算法 / 简单 UI / 测试
- **需要你**：算法决策 / 创新部分 / 商业判断 / 性能调优

定位：**AI 是工程师助手，不是工程师**。

---

## 没在这里的问题

- 看 `Tinia/docs/` 的设计文档
- 看其他 skill：`quickstart` / `create-node` / `debug-node` / `pack-and-publish` 等
- 跟 Claude Code 直接问 —— 它能读全部代码 + 设计文档

---

## 下一步

- 完整开发流程 → 用 `quickstart` skill
- 节点商店 / 发布细节 → `../reference/12-node-ecosystem.md`
- MCP 工具列表 → `Tinia/docs/mcp-tool-reference.md`
- 客户问题 → `01-customer-questions.md`
