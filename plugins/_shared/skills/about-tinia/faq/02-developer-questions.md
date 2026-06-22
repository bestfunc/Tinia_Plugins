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
    ├── ParamsForm.tsx  ← 参数配置 UI（≥4 参数或有 enum 必配 + 预设）
    ├── Viewer.tsx      ← 结果可视化
    ├── ViewerLoader.tsx ← Viewer 懒加载入口
    └── Help.tsx        ← 节点说明 + 参数表 + 算法 + 更新履历（vite glob 自动注册，节点版本号旁渲染 HelpCircle 图标）
```

> Help.tsx 是现役约定：所有官方节点都内嵌它，含「SDK 说明」标签页（示例代码）。新节点建议一并补。

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

> **SDK 源在哪**：节点 Python SDK（`tinia_runtime` / `tinia_audio` / `tinia_audio_input` / `tinia_features`）**已统一到主仓 `Tinia/server/sdk/python/`**，通过 go:embed 嵌进 server 二进制，节点 fork 时 server 注入 `PYTHONPATH` 指向嵌入版。节点本身不携带物理 SDK 目录。要 IDE 类型提示可 `tinia sdk install --target ./sdk/python/` 拷一份本地副本（gitignore，不进发布）。早前各节点仓自带 SDK 副本导致版本漂移，现已唯一源。

### Q：节点能调外部 HTTP API 吗？

**A**：技术上能，但要谨慎：

- 商店审批会检查"是否泄露用户数据到外部"
- 私有节点（org_only）随便调
- 公开节点最好不要调外部 API，影响用户信任

如果需要外部 API（比如调云端 ML 服务），应该：

1. 在 node.yaml 声明
2. 让用户在 ParamsForm 里配 API key / URL
3. 文档说明数据流向

### Q：外部程序怎么调用我在平台上调好的节点 / 流程？（SDK 通路）

**A**：这是 `tinia_sdk`（**调用方**的 SDK，区别于节点开发的 `tinia_runtime` 运行时）：

- 超管在「SDK 管理」输入名称生成可下载 SDK 包，**凭据 + 服务器地址内置，零配置**（鉴权走 license：`license.json` 里的 `license_id` + `secret`，请求头 `Bearer <license_id>.<secret>`；server 端 `sdkapi` 中间件校验 `sdk_licenses` 表）
- 调用方式：`tinia_sdk.connect(...)` 然后传节点类型 + 参数 / 传「复制参数」串 / 引用平台流程里调好的节点；整条流程可放「API 输入」+「API 输出」节点整体调用
- 传数据：直接传文件路径（wav/csv/npz/tdms）或内存数组自动上传；单输入节点连端口名都不用写
- 同机调用自动走 **Unix domain socket 本地直连 + 路径直传**，大文件显著更快
- **算法仍在 server 进程内执行**——SDK 是纯通路，没有"第二个引擎"，平台改参调用方零改动
- 实时场景用**流式会话**（JWT session_token + push / recv / close / keepalive），跳过缓存走内存中转
- 给节点补「SDK 说明」：在 Help.tsx 加 SDK 示例代码标签页（官方 40 个节点已全补齐），让别人知道怎么调你的节点

相关源：`Tinia/server/sdk/client-python/tinia_sdk/`、`Tinia/server/internal/sdkapi/{middleware,handler,stream}.go`、`Tinia/docs/sdk-design.md`。

> 注意：`Tinia/docs/sdk-design.md` 里若出现 `connect(url, api_key=...)` 是早期设计草稿——**实际落地是 license，不是 api_key**，以 `tinia_sdk/client.py` 的 `connect(server_url, license_path, socket_path, use_uds)` 为准。

---

## 类型系统

### Q：input/output 有哪些类型？

**A**：核心类型：

| 类型 | 用途 |
|---|---|
| `AudioData` | 音频超类型（含子类型 MaterializedDataset / ProcessedDataset）|
| `IndicatorData` | 分析指标 |
| `FeatureMatrix` | 特征矩阵（特征工程枢纽，feature_merge 等输出）|
| `AnnotationLayer` | 段落标注 |
| `AttributeTable` | 属性表（attribute_extract 输出，可与 features join）|
| `Any` | 透传（少用）|

> 通道 / 物理量语义：NPZ v2 + `value_kind` / `quantity` 通道语义在持续演进——声压 / 加速度 / 速度等 11 种物理量 + 传感器灵敏度自动换算，整链路传递。心理声学节点有 quantity 契约。写跨物理量节点时注意保真。

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
5. UI（ParamsForm / Viewer / Help）实时编译热加载，带编译指示灯 + 错误波浪线（DevStudio 内）

> 调试坑：① 如果节点被常驻执行预热了，改完 Python 代码要确保 reload 后旧热进程被回收，否则可能仍跑旧代码——重测前先确认进程刷新；② 节点输出缓存可能让你"以为改生效了其实读的是缓存"——调试期把该节点缓存关掉，或换输入触发重算。

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
- **起进程开销**：高频调用场景下，进程 fork + import 库的固定开销（约 530ms）可被**常驻执行池**消化——把节点加进预热白名单，第二次起复用热进程，只剩纯计算时间
- **缓存命中**：节点输出按「输入指纹」自动缓存，相同输入直接返回缓存；不想缓存可单节点关（会显示「不可缓存」图标）
- **GPU**：重算法可走共享 GPU sidecar（torch IPC 复用），未 provision 时自动 numpy 回退（参考 scale_space_spectrum）

### Q：常驻执行（HotPool）是什么？开发节点要注意什么？

**A**：

- 常驻执行池让分析节点进程常驻待命（只加载一次库），SDK 高频 / 实时调用直接复用热进程，省掉每次 fork + import 的固定开销
- 超管「常驻执行」页可看运行状态、调进程上限 / 空闲回收、勾选预热白名单
- 进程按 node fullKey 分桶（per-node venv 隔离），所以**不同节点不会串 venv**
- 开发注意：常驻意味着进程级状态会跨调用保留——**模块级全局变量 / 单例不要假设每次都是新进程**；该清的状态在 run 入口清，避免上一次调用的残留污染下一次

### Q：怎么写「流式 / 实时」节点（边算边推）？

**A**：

- 基础：SDK 的 **ChunkRuntime（SV2）** 给节点提供分块流式 endpoints + `upstream_total` 进度，是流式调度基座
- 跨窗无缝：声明 `_stream_continuous` 标志并**维护跨窗状态**（如滤波器 `zi`、STFT carry、滚动平均 / 滚动 percentile），让逐窗结果在窗边界不断裂；批量行为由该标志门控，零回归
- 参考实现：声级计（A/C 计权 `zi` 跨窗 + 滚动 Leq）、倍频程（每频带 `zi` + 滚动谱）、FFT 频谱（STFT carry + 线性功率滚动平均），emit 新增 `rolling_*` 字段
- 会话生命周期由 server 侧「SDK 流式会话」管，节点作者不用自己管——只声明跨窗算法
- 黑盒依赖（如 mosqito 的响度）做不到无缝时，退回"准实时逐窗"即可

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

**注意**：主仓平台核心保留闭源（执行引擎 / 多租户 / 商店 / AI MCP 集成）；节点 SDK（`server/sdk/python/`）随程序内嵌可见，节点生态欢迎贡献。改主仓核心前先联系 core team 讨论。

> 历史说明：早期曾设想把 engine / runtime 抽成独立开源仓（`Tinia_Engine` / `Tinia_Runtime`），该方案已归档——执行只发生在 tinia-server 进程内，节点 SDK 唯一源并入主仓 go:embed，不再有独立 engine/runtime 仓库。

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
