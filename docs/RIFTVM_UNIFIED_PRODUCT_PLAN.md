# RiftVM 1.0.0 统一产品实施计划

状态：拟执行。本文取代 RIFTVM_REBRAND_AND_PRODUCT_PLAN.md 中的双应用结构；本轮仅制定计划。

## 1. 产品决策

- 只发布一个 **RiftVM 1.0.0**，合并现有 EZVM 与 EZVM Omarchy。
- Omarchy 和 macOS 是两种主打工作空间；其他 ARM64 Linux 放在“更多系统 / 自定义 ISO”。
- 一个 App、Bundle ID、CLI、Homebrew Cask、权限身份和应用发行包。
- Omarchy 专用安装器和集成模块保留，独立 App target 和发行入口取消。
- 不考虑 EZVM 旧数据、格式、命令、环境变量和 Agent 兼容；不自动删除旧数据。

产品定义：**RiftVM brings another world to your Mac.**

工作空间对应一个独立 VM。应用提供一致的创建和切换体验，各系统的功能根据实际支持能力显示，不能把 Linux Agent 能力默认宣称为 macOS Guest 能力。

## 2. 用户路径

首次打开，展示“创建 Omarchy 工作空间”和“创建 macOS 工作空间”；更多系统作为次级入口。

Omarchy：选择名称/目录/资源 → 下载并验证 factory → 创建独立工作磁盘 → 设置账号 → 进入桌面。默认使用 macOS 原生全屏窗口形成 Space，保留窗口模式。

macOS：选择支持的恢复镜像或本地 IPSW → 选择名称/目录/资源 → 安装和系统设置 → 进入桌面。保留快照、窗口和全屏操作。

| 再次打开时的状态 | 行为 |
|---|---|
| 没有工作空间 | 创建页面 |
| 只有一个可用项 | 激活原窗口，或启动/恢复后直接进入 |
| 多个且指定默认项 | 进入默认项 |
| 多个且无默认项 | 简洁的工作空间选择页 |
| 默认项离线或安装失败 | 说明原因，允许重试或切换 |

菜单始终提供“工作空间”和“新建工作空间”。支持多 VM 同时运行；同一个工作空间只能由一个 runner/窗口持有，重复打开激活原窗口。

窗口偏好、默认项和工作空间状态分别保存。关闭窗口、停止 VM、退出 App 分开处理；退出时协调所有实例。仅在后端和设备状态支持时保存运行状态，否则提供明确的关机流程。登录自动启动作为后续可选功能。

安装支持取消、断网重试与空间不足处理。取消只清理本次临时产物，不删除共享缓存或其他工作空间。

## 3. 统一身份

| 对象 | 唯一目标 |
|---|---|
| App / 主 target / scheme | RiftVM |
| Bundle ID | com.everettjf.riftvm |
| 主工程 | RiftVM/RiftVM.xcodeproj |
| 核心 / CLI 模块 | RiftVMCore / RiftVMCLIKit |
| Host CLI / Cask | riftvm |
| 正式版本 | 1.0.0 |
| 首发 tag / ZIP | riftvm-v1.0.0 / RiftVM-1.0.0.zip |
| 新 VM 格式 | .riftvm |
| 默认目录 | ~/RiftVM Virtual Machines |
| 应用数据 | ~/Library/Application Support/RiftVM |
| 环境变量 | RIFTVM_* |
| Linux 服务 | rift-agent / rift-session-agent |
| Guest 运行目录 | /run/rift-agent |
| 共享 tag / 底层挂载 | riftvm_shared / /mnt/riftvm-shared |

build number 从 1 开始，候选重建递增。版本在 App、CLI、脚本、Cask 和验收中保持一致。取消 riftvm-omarchy 应用 tag、ZIP、Bundle ID 和独立发布脚本入口。

factory、Overlay、Agent 保留资源构建流水线；它们不是第二个桌面产品。Omarchy 上游版本和协议/schema 版本独立管理。

## 4. 工程合并

以通用 App 的多机器管理和生命周期为基础，接入独立 Omarchy App 的专用体验。先迁入功能并验证，再移除旧 App 入口。

| 现有位置 | 迁入内容 |
|---|---|
| EZVM/EZVM/Application | App shell、工作空间列表、创建入口、菜单、退出处理 |
| EZVM/EZVM/Core/VMKit | VM 模型、runner、快照、导入导出、机器锁 |
| EZVMOmarchy/Sources | factory 安装、owner setup、显示、输入、剪贴板、通知 |
| EZVMOmarchy/GuestOverlay | 迁入 Guest 资源目录，保留 Overlay 工具 |
| GuestAgent/linux | Rift Agent 系统与用户会话服务 |
| CLI、Tools、scripts | 一个 CLI 和应用发行链，保留资源工具 |

内部职责：

- Workspace Registry：UUID、profile、位置、默认项。
- Workspace Coordinator：创建、打开、窗口路由、生命周期和退出协调。
- Workspace Profile：Omarchy / macOS / Custom Linux 的安装方式、资源建议和能力。
- VM Runtime：复用现有 Virtualization.framework、图形和设备实现。
- Integration Session：按工作空间管理 Agent、输入、剪贴板、通知和授权。

这些是职责划分，不要求额外拆成五个 Package。复用现有实现，避免双份状态管理。

### 必须解决的合并问题

- Omarchy 单例路径/默认 workspace 改为 UUID 隔离，支持创建两个 Omarchy 实例。
- 只读 factory 缓存可复用，可写磁盘、机器身份和运行状态不可共用。
- 通用与 Omarchy 图形路径按 profile 路由，不同时创建两个显示实例。
- 输入只发送到真正聚焦的 VM 显示区域；失焦、暂停、断线释放修饰键。
- 一个 Host 剪贴板协调器选择活动工作空间，后台 VM 不覆盖 Mac 剪贴板，不向其他 Guest 串传内容。
- 通知附 workspace ID，点击定位正确窗口；App 权限统一，镜像策略按 workspace 设置。
- 共享目录权限按 workspace 保存；删除 VM 不删除 Host 源目录。
- macOS Guest 不依赖 Linux Rift Agent，显示经过验证的原生能力。
- 多 VM 同时运行时检查总内存/磁盘需求并允许调整。
- 独立 Omarchy 的有效测试迁入主 App 测试集，删除旧工程不能丢失覆盖。

## 5. 1.0.0 范围

必须交付：

1. 同一 App 内完成 Omarchy/macOS 创建、运行和切换，保留其他 Linux ISO 入口。
2. 默认工作空间、直接启动、多窗口、全屏与统一退出处理。
3. Omarchy 镜像信任校验、安装、账号设置和失败重试。
4. macOS 恢复镜像/本地 IPSW 安装和首次设置。
5. 保留已有快照、存储、网络和图形能力，完成合并回归。
6. 迁入已有 Omarchy 输入、剪贴板、动态分辨率和通知能力，按实测提供开关。
7. 目录共享显式授权，明确不共享/只读/读写，不默认共享 Home。
8. 全新 RiftVM 身份、权限、CLI、Cask、签名发行与官网。

后续演进：

- Mac / PC / Raw 输入模式、终端感知语义快捷键。
- Finder “Open in Omarchy”、Guest “Open on Mac”、文件路径映射。
- 图片/文件剪贴板完善、双向拖放和更多 MIME 类型。
- Guest 端口发现与显式授权 localhost 转发。
- 可选登录启动、合盖恢复、长期 soak。
- 摄像头、外设、触控板缩放、高刷新率增强。

首发范围完成即可发布，后续增强不成为无限延长 1.0.0 的门槛。

### 输入与文件原则

附件的 Mac-first 是目标：系统 App 切换、Spotlight、Mission Control 和 Spaces 默认属于 Host。它与当前 Command → Super 行为不同；合并阶段保持已验证输入策略并正确限定焦点，语义映射独立开发验收。不能将所有 Command-C 机械转成 Control-C，导致终端收到 SIGINT。界面明确当前模式。

共享目录目标呈现为 ~/Mac/...，采用一个共享文件系统，不做两份目录后台同步；验证大小写、锁、symlink、watcher 和权限差异。敏感目录或其父目录授权需明确提示。Guest 不获得任意执行 macOS shell 的接口。

## 6. 实施阶段

| 阶段 | 工作 | 验收 |
|---|---|---|
| A 基线 | 封存源码，盘点入口、runner、全局状态和脚本；准备新 fixtures | 基线可追溯、合并清单完整 |
| B 改名 | 新身份、1.0.0、模块/CLI/目录改名、全新格式 | 可构建，无旧兼容分支 |
| C 合并 | Omarchy 模块迁入，统一 registry/coordinator，移除独立入口 | 一个 App 同时运行 Omarchy 和 macOS |
| D 工作流 | 首装、默认项、直接进入、全屏、切换和退出 | 单/多 workspace 路径完整 |
| E 集成 | Agent/Overlay/factory 重建，输入/剪贴板/通知/共享隔离 | 多实例不串数据，镜像验证通过 |
| F 发行 | 合并 CI、构建、公证、Cask 和证据 | 唯一 RiftVM-1.0.0.zip 通过验证 |
| G 上线 | GitHub、riftvm.com、README、下载、公告 | HTTPS、下载、首装全链路通过 |

实现分支建议：codex/riftvm-unified-1.0。机械改名与行为变化分开提交，每阶段保持可构建。

## 7. CI 与验收

保留共享核心、CLI、Agent 和图形测试；只构建一个主 App。把原 Omarchy GUI readiness、通知、生命周期和安装验收接入统一 App。更新路径过滤和测试路径，删除重复 App 构建。runner 无法执行的检查明确标为仅编译，并由原生测试主机补充运行证据。

| 场景 | 预期 |
|---|---|
| 干净首装 | 两个主创建入口，版本 1.0.0 |
| Omarchy/macOS 安装 | 各自完整安装和进入桌面 |
| 单项/多项/默认项 | 路由正确，不重复启动 |
| Omarchy 与 macOS 同时运行 | 生命周期、焦点和通知互不干扰 |
| 两个 Omarchy | 磁盘、身份、Agent、剪贴板目标独立 |
| 设置页/文件面板/失焦 | 不误吞输入，不粘修饰键 |
| 只读/读写共享 | 权限生效，删除 VM 不动源目录 |
| 断网/空间不足/磁盘离线 | 可解释、可重试，其他 workspace 不受影响 |
| 关闭/退出/恢复 | 全部实例得到协调，无孤儿进程 |
| 快照 | 新 VM 上成功，不覆盖其他磁盘 |
| Agent 未就绪 | 功能降级明确，基础 VM 可运行 |
| 发布包 | 签名、公证、Gatekeeper、版本和镜像信任通过 |

测试使用全新工作区，不测试旧 EZVM 升级。遵循此前暂缓决定，首发不新增强制合盖睡眠测试，也不宣传已全面验证睡眠恢复。

只发布 riftvm Cask 和 CLI；Omarchy factory/Overlay 独立资源脚本继续运行。合并原发布脚本的签名、公证、制品复验和发布前检查，保留有效发布门槛。

## 8. 官网与视觉

- 品牌仅 RiftVM；不再出现两个应用下载选择。
- 左侧 Omarchy、右侧 macOS，保留斜向裂缝和中心赛车视觉。
- 唯一主按钮“Download RiftVM”，不显示版本号。
- 两侧进入各自功能介绍，最终下载同一包；其他 Linux 在次级区域介绍。
- App 图标、网站 Logo、截图及赛车上的 EZ/Z 标记统一检查，必要时换成 R/裂缝；实测 Dock/Finder 图标。
- canonical、Open Graph、Cask homepage、支持链接使用 https://riftvm.com。
- DNS、HTTPS 和托管实际确认后切换；旧站跳转按托管能力实施，不假设静态 Pages 支持任意 301。
- 仓库计划改为 everettjf/riftvm，更新源码与发布引用。

## 9. 提交与回退

建议提交顺序：

1. Define unified RiftVM identity and 1.0.0 baseline
2. Rename projects, modules and CLI to RiftVM
3. Integrate Omarchy into the shared workspace lifecycle
4. Add default workspace launch and unified navigation
5. Scope desktop integration to individual workspaces
6. Rebuild Rift Agent and Omarchy factory assets
7. Consolidate RiftVM build and release pipeline
8. Launch the unified RiftVM website and documentation

源码基线用于开发回退，不承诺旧 App 读取 RiftVM 数据。失败候选不发布，修复后重新验证匹配的 App/Agent/factory。测试产物用任务专属临时目录，完成后清理。用户已有磁盘和密钥不自动删除。

历史双应用方案保留为设计记录，由本文取代。既有 TODO 按首发必需与后续重新归类。

完成标准：用户只下载一个 RiftVM 1.0.0，即可创建、进入和切换 Omarchy/macOS 工作空间，工程、CLI、Agent、镜像、CI 与官网身份一致。
