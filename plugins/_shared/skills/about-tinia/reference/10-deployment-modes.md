# 部署模式

> Tinia 的部署形态差异，给销售评估客户场景用。
>
> **先厘清两件事**：代码里的 **edition（部署形态）只有 3 个**：`saas` / `server` / `desktop`。而 Community / Pro / Production 是**商业 SKU（打包概念）**，不是 edition flag —— Community 与 Pro 同属 `desktop` edition，差异在激活与功能档；Production（产线版）目前是**路线图 / 规划中**，代码里没有 `production` edition 常量。详见 `03-edition-comparison.md`。

---

## 速览

| 部署形态 | Edition（代码） | 谁用 | 物理形态 | 联网要求 |
|---|---|---|---|---|
| **SaaS** | `saas` | 个人 / 小团队 / 早期客户 | 公网托管 | 必须联网 |
| **公司私有化** | `server` | 中大型企业 / 安全敏感行业 | 客户内网服务器 | 内网即可，激活/更新时偶尔联网 |
| **桌面单机** | `desktop` | NVH 工程师个人 / 学生 / 独立顾问 | Windows / macOS 桌面 app | 首次激活联网，之后离线可用 |
| **Production（产线版）** | — *（规划中，无独立 edition）* | 工厂在线监测 | 边缘节点 + 云端协调（设想） | 边缘 + 云混合（设想） |

> 表里 SaaS / Server / Desktop 对应真实存在的三档 edition；Production 是**设想中的产品形态**，尚无对应 edition flag，相关描述均为路线图措辞。

---

## 1. SaaS（多组织公网）

### 部署形态

```
   公网域名 tinia-saas.bestfunc.com
            │
   ┌────────┴─────────┐
   │   Tinia daemon   │
   │   edition=saas   │
   │   multi_org=true │
   └────────┬─────────┘
            │
   ┌────────┴─────────┐
   │  PostgreSQL +    │
   │  MinIO（对象存储）│
   └──────────────────┘
```

### 适合谁

- **个人 NVH 工程师**：注册账号即用，无需部署
- **小团队 / 创业公司**：3-10 人共用一个 Org，免运维
- **早期客户**：免费 / Pro 试用，用一段时间再决定是否私有化

### 优势

- ✅ 零部署：注册即用
- ✅ 总是最新版：服务端升级用户自动享受
- ✅ 团队协作天然：Org / Member / Seat 内置
- ✅ 跟商店 / 节点更新无缝

### 劣势

- ❌ 数据需要上传到云：合资 OEM / 国企可能合规不通过
- ❌ 依赖网络：内网 / 出差现场不行
- ❌ 月度成本：托管服务总要钱

### 当前状态

- `tinia-saas.bestfunc.com` 已上线
- Pro 商业版本已交付

---

## 2. 公司私有化（Server）

### 部署形态

```
   客户内网 t.bestfunc.com（举例）
            │
   ┌────────┴─────────┐
   │   Tinia daemon   │
   │   edition=server │
   │   single org     │
   └────────┬─────────┘
            │
   ┌────────┴─────────┐
   │  客户自家         │
   │  PostgreSQL +    │
   │  MinIO（或 S3）   │
   └──────────────────┘
```

### 适合谁

- **合资车企 NVH 团队**：数据安全 + 合规 + 内网部署刚需
- **国企 / 央企**：采购流程 + 数据安全
- **军工 / 医疗 / 航天**：敏感行业
- **中大型 Tier1 测试部门**：团队级流程沉淀

### 优势

- ✅ 数据 100% 在客户内网：完全合规
- ✅ 内网速度快：跟客户其他系统集成（ERP / MES / Diffgram）顺
- ✅ 团队级共享：组织 / 权限 / 流程模板
- ✅ 灵活定制：客户能要求特殊功能

### 劣势

- ❌ 部署成本：客户 IT 团队要参与
- ❌ 升级需要客户配合：不能像 SaaS 一样总是最新
- ❌ 维护：客户 / 我方运维分担

### 部署形态变种

| 形态 | 说明 |
|---|---|
| **Docker Compose 单机** | 中小型客户，单服务器 |
| **K8s 部署** | 大型客户，已有 K8s 平台 |
| **空气隔离环境** | 军工 / 涉密，完全断网，更新通过 USB 拷贝 |

### 部署文档

`Tinia/.claude/skills/deploy-server-176/` 有详细示例（针对一个内网客户）。

### 跟商店的关系

- 公司私有化版**仍然能连商店**（出口防火墙开个白名单到 `tinia.bestfunc.com`）
- 也能配置"私有商店"（公司自己起一个 Tinia_Store 实例，仅内部节点）

---

## 3. 桌面单机（Desktop）

### 部署形态

```
   Windows / macOS 桌面
            │
   ┌────────┴────────────────┐
   │  Tinia.app / Tinia.exe   │
   │  edition=desktop         │
   │  ┌────────────────┐      │
   │  │  Wails 主进程   │      │
   │  │  + WebView2 UI │      │
   │  └───────┬────────┘      │
   │          │ 自举(同一 binary)│
   │  ┌───────┴────────┐      │
   │  │ tinia daemon   │      │
   │  │ (子命令常驻)    │      │
   │  └───────┬────────┘      │
   │          │                │
   │  ┌───────┴────────────┐  │
   │  │ <data_dir>/        │  │
   │  │  ├ config.yaml     │  │
   │  │  ├ postgres-data/  │  │
   │  │  ├ blobs/          │  │
   │  │  ├ python/         │  │
   │  │  └ plugins/        │  │
   │  └────────────────────┘  │
   └─────────────────────────┘
```

> **入口链路**：桌面版是**同一个主 binary**用 `--desktop` 启动后，以 `daemon` 子命令把后端常驻起来 + Wails 窗口接管 UI（`cmd/server/main.go`：runSolo → runDesktopSetup / runDesktopFull）。不是单独的 `tinia-cli.exe` 那个 CLI 二进制。`--desktop` 还支持 `--setup` / `--window` / `--no-window`。

### 适合谁

- **NVH 工程师个人**：本机分析，数据不出本机
- **学生 / 研究者**：免费版（Community），完全离线
- **独立顾问**：跨客户跑，每次现场分析数据
- **出差场景**：飞机上 / 客户现场没网

### 优势

- ✅ 数据 100% 本机：极致合规
- ✅ 离线可用（首次激活除外）
- ✅ 开箱即用：安装包双击装即可
- ✅ 个人订阅模式更轻量

### 劣势

- ❌ 单用户：团队协作需要外加传文件
- ❌ 设备绑定：seat 跨设备数量受限（默认 3 台）
- ❌ 升级要重装：虽然有自动更新但需要用户配合

### 内嵌依赖

桌面版打成单 exe / app bundle，但首次启动会下载：

| 依赖 | 大小 | 何时下载 |
|---|---|---|
| 安装包本体 | 约 30-50MB | 用户从 tinia-release 下载 |
| PostgreSQL binary | ~50MB | 首次 setup wizard |
| Python runtime（PBS） | ~50MB | 首次 setup wizard（如系统无 Python 3.11）|
| 节点 bundle | ~5MB | 安装包内嵌（直接装好）|
| 节点 venv 依赖 | 10-100MB / 节点 | 首次跑节点时 pip install |

### 自动更新

- 启动时 GET `tinia-release/latest.json`
- 有新版 → UI 提示 → 用户点"更新" → 后台下载 + 重启
- 强制更新（安全补丁）：force=true，启动时阻塞升级

### 首次启动 Setup Wizard

4 步：

1. 选 data dir（数据目录）
2. 自动配置（PostgreSQL 启动 + Python runtime 准备 + 节点 bundle 解压）
3. 创建管理员账号
4. 商店激活（OAuth → 选 Org → 消耗 Seat）

详见 `Tinia/docs/desktop-v1.24-design.md`。

### 平台支持

| 平台 | 状态 | 说明 |
|---|---|---|
| Windows 10/11 x64 | ✅ 主力 | NSIS installer + WebView2 bootstrapper |
| macOS 12+ Apple Silicon | ✅ | .app bundle + DMG |
| macOS 12+ Intel | ✅ | 同上 |
| Linux | 🔜 规划 | AppImage 包，路线图（视用户呼声）|

---

## 4. Production（产线版，规划中 / 路线图）

> 以下为**设想中的产品形态**，尚未交付，代码里无 `production` edition。措辞均为路线图。


### 部署形态

```
   ┌────────────────────────────────────────┐
   │         云端协调中心                    │
   │  ┌─────────────────────────────┐       │
   │  │  Tinia Production 中央实例   │       │
   │  │  - 流程定义 / 模型版本       │       │
   │  │  - 历史归档 / 报告           │       │
   │  │  - 仪表盘 / 告警             │       │
   │  └─────────────┬───────────────┘       │
   └────────────────┼───────────────────────┘
                    │ MQTT / gRPC
   ┌────────────────┴───────────────────────┐
   │         工厂现场                         │
   │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐│
   │  │工位 1│  │工位 2│  │工位 3│  │工位 N││
   │  │ edge │  │ edge │  │ edge │  │ edge ││
   │  │engine│  │engine│  │engine│  │engine││
   │  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘│
   │      │         │         │         │   │
   │  传感器        传感器     传感器     传感器  │
   └────────────────────────────────────────┘
```

### 适合谁

- 现有 SmartQuality 客户的升级（NG 检测 → NVH 在线监测）
- 风电场远程 CMS
- 工业泵 / 电机厂 PdM 联合解决方案

### 优势

- ✅ 实时流处理（秒级延迟）
- ✅ 多机协同
- ✅ MES / ERP 集成
- ✅ 模型 OTA + 流程热更新
- ✅ 数据沉淀 → 行业模型库

### 当前状态

**未上线 / 规划中**（changelog v1.24 明确标注 Production 为"规划中"，是目前唯一明确未交付的主线产品形态）。设计与路线图详见 `08-roadmap.md`。

当前类似需求可用 SmartQuality（Bestfunc 现有产品）+ Tinia Pro 桌面版（离线分析调参）过渡。

---

## 同一份代码，3 种 edition

**关键设计**：三种部署用同一份主仓代码。代码里 edition 只有三个有效值：

```go
// internal/config/config.go
const (
    EditionDesktop = "desktop"
    EditionServer  = "server"
    EditionSaas    = "saas"
)
// IsValidEdition 只认这三个；无 EditionProduction / community / pro 常量

switch cfg.Edition {
case "saas":    // 多 Org，组织管理 UI，外部 PG/MinIO
case "server":  // 单 Org，外部 PG/MinIO
case "desktop": // 单 Org，内嵌 PG，setup wizard（Community/Pro 同属此 edition）
}
```

**edition 怎么确定**：`ResolveEdition` 优先级 = env `TINIA_EDITION` > 构建期 ldflags `config.DefaultEdition` > 兜底 `server`。桌面构建用 `-X .../config.DefaultEdition=desktop`；SaaS 靠 `TINIA_EDITION=saas` 环境变量（**没有 `--saas` flag**）。

前端通过 `/api/v1/meta` 拿 edition 字段，按 edition 分叉 UI。

> Production 产线版（规划中）若落地，会在此之上加流处理层；当前它**不是**一个 edition 分支。

详见 `02-architecture.md`、`Tinia/docs/architecture-v2.md`。

---

## 客户选哪个部署形态？

### 决策树

```
客户类型？
├── 个人 / 小团队（<10 人）
│   ├── 想要 0 部署 → SaaS
│   ├── 数据敏感 / 想要本地 → Desktop
│   └── 想要团队协作 + 私密 → SaaS（私有 Org）
│
├── 中大型企业（10-1000 人）
│   ├── 数据可上云 → SaaS
│   ├── 数据需要内网 → 私有化 Server
│   └── 团队成员各自分析 → Desktop × N seat
│
└── 工厂 / 设备厂（在线监测）
    └── 等 Tinia Production（规划中）；当前过渡：SmartQuality + Tinia Pro Desktop 调参
```

### 销售常见问题

**Q：能从 Desktop 升级到 Server 吗？数据怎么迁？**

A：能。流程定义是 JSON 文件，数据源凭据可重新配，blob 通过导出/导入流程迁移。

**Q：能同时用 SaaS + Desktop 吗？**

A：能。同一个 Tinia_Store 账号能激活多个实例（消耗多个 Seat）。流程不会自动同步，需要导出/导入。

**Q：Server 部署需要我们 IT 干啥？**

A：准备 Linux 服务器（一台够）+ PostgreSQL + MinIO（或用 docker-compose 一键启）。我们提供详细部署文档，远程协助 1-2 天可完成。

**Q：Desktop 版的 PostgreSQL 是要联网下载，公司机器没外网怎么办？**

A：Desktop 安装包可以预下载 PG binary 包进去（offline installer 变种），首次启动不需联网。

---

## 下一步

- 各部署的内部架构 → `02-architecture.md`
- 商店激活流程细节 → `04-key-concepts.md` 的 Activation 部分
- MCP 接入跟部署的关系 → `11-mcp-ai-integration.md`
