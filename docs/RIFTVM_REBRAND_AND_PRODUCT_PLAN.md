# RiftVM 全面改名与产品演进方案

状态：历史方案，已由 [RiftVM 统一产品计划](RIFTVM_UNIFIED_PRODUCT_PLAN.md) 取代。本文的双应用、双 Bundle ID、双发行包要求不再执行。

日期：2026-09-06  
改名前基线：`main` / `b0e2b80`  
主域名：`riftvm.com`

最新决策：按全新产品发布，不考虑 EZVM 旧版本兼容。RiftVM 与 RiftVM Omarchy 的首个正式版本均为 **1.0.0**。以下条款取代上一版方案中的旧数据迁移、兼容别名与新旧 Host/Agent 组合要求。

## 1. 决策摘要

EZVM 全面更名为 **RiftVM**，产品定义同步从“macOS 上的 VM Manager”升级为：

> **RiftVM brings another world to your Mac.**
>
> **Isolated underneath. Seamless on top.**

RiftVM 的目标不是隐藏虚拟化提供的隔离，而是消除用户跨越 macOS 与 Guest 工作空间时不必要的摩擦。Omarchy 是第一个、也是体验最完整的第二工作空间；通用 macOS 与 ARM64 Linux VM 能力继续保留。

最终产品命名：

| 当前名称 | 新名称 | 定位 |
|---|---|---|
| EZVM | **RiftVM** | 通用 macOS / ARM64 Linux 虚拟机产品与共享核心 |
| EZVM Omarchy | **RiftVM Omarchy** | 打开即进入 Omarchy 的旗舰独立应用 |
| EZVM Guest Agent | **Rift Agent** | Host 与 Guest 之间的受认证能力桥 |
| EZVM Session Agent | **Rift Session Agent** | 普通用户级 Wayland/桌面会话集成服务 |
| `ezvm` CLI | **`riftvm`** | Host CLI，统一采用新名称 |

内部产品标准：

> **If the user has to think about the VM boundary, we still have work to do.**

## 2. 四条产品原则

1. **Native first**：先问 Mac 用户期望怎样工作，再决定虚拟化实现。
2. **Host owns the machine**：macOS 保有系统级快捷键、Spaces、权限与安全策略的最终控制权。
3. **Share explicitly**：Host 资源通过明确 capability 授权，默认不暴露整个 Home。
4. **Hide virtualization details**：Virtio、NAT、Guest IP、mount、SSH key 和 Agent 生命周期不应成为日常操作负担。

## 3. 当前仓库影响面

初步盘点：约 1,836 处 `EZVM`、739 处 `ezvm`。不能一次性盲目替换，必须按语义分类。

### 3.1 用户可见品牌

- 两个 macOS App 的名称、窗口标题、菜单、About、权限说明和诊断文案。
- README、Omarchy README、规划文档、TODO、截图、Alt 文本与第三方说明。
- App 图标、网站 Logo、Open Graph 图片、页面标题和 SEO 元数据。
- ZIP、诊断包、Factory、Guest Overlay 等发布制品名称。

这些内容应完整替换为 RiftVM，不保留 EZVM 作为并列品牌。

### 3.2 工程与源码名称

- `EZVM/`、`EZVMOmarchy/`、Xcode project、scheme、target、entitlements。
- Swift Package：`EZVMCore`、`EZVMCLIKit`、`ezvm` executable。
- Swift 类型与测试：`EZVM*`、`EZVMOmarchy*`。
- VirGL 原型中的模块名、日志 subsystem 与测试 fixture。
- `.github/workflows/ezvm-omarchy.yml`、脚本名与临时路径前缀。

目标名称：

- 目录：`RiftVM/`、`RiftVMOmarchy/`
- 模块：`RiftVMCore`、`RiftVMCLIKit`
- targets/schemes：`RiftVM`、`RiftVM Omarchy`
- Swift 类型：`RiftVM*`
- workflow：`riftvm-omarchy.yml`
- 命令：`riftvm`

### 3.3 全新数据与格式

以下旧标识的代码定义全部切换为新标识，不提供旧数据自动发现、导入适配或迁移：

- `~/EZVM Virtual Machines`
- `.ezvm` machine bundle 扩展名
- `~/Library/Application Support/EZVM*`
- UserDefaults keys/suite
- Keychain service/account
- Guest Agent enrollment records与机器身份
- saved-state、snapshot、thumbnail、运行锁和诊断记录
- `io.github.everettjf.ezvm.preinstalled-image` manifest kind

新默认值：

- `~/RiftVM Virtual Machines`
- 新建 bundle 使用 `.riftvm`
- `~/Library/Application Support/RiftVM*`
- 新 manifest kind：`com.riftvm.preinstalled-image`（最终值在实现前锁定）

实施原则：

- machine bundle、导出格式、UTI、manifest 校验与文件选择器统一使用 RiftVM 格式。
- Application Support、UserDefaults、Keychain、enrollment 与运行锁统一使用新命名空间，无旧值回退。
- 使用全新 VM 和重新构建的 Omarchy 镜像验证首装流程。
- 不考虑旧版本兼容不等于删除旧数据；现有磁盘、目录、Keychain 项目保持原样，清理另行处理。

### 3.4 Bundle ID 与 macOS 权限

建议正式改为：

- `com.everettjf.riftvm`
- `com.everettjf.riftvm.omarchy`
- 对应 test Bundle ID

这是一次新的 macOS 应用身份，会影响 Accessibility、Notifications、Microphone 等 TCC 权限。应用无法代替用户静默迁移这些授权。

发布体验必须做到：

- 首次启动显示 RiftVM 的全新产品引导。
- 在需要功能时逐项请求权限，不在首次启动一次索要全部权限。
- 权限页面显示明确状态和“Open System Settings”。
- Accessibility event tap 未授权时功能降级但应用仍可运行。
- 签名、Notarization 与 designated requirement 在发布前固化并记录。

使用新 Bundle ID，按全新应用请求授权；不实现旧 Bundle ID 或旧权限迁移。

### 3.5 Host ↔ Guest 协议

Host、Agent、Guest Overlay 和 factory 镜像作为同一套 RiftVM 发行链同步切换。

- 只支持新 Host 与 Rift Agent，不保留旧 Agent 协议适配。
- 保留有效的认证与能力协商设计；审计现有字段后确定协议版本，不因改名重复添加字段。
- systemd/OpenRC 服务直接使用 `rift-agent` 与 `rift-session-agent`，无旧服务 alias。
- 运行目录改为 `/run/rift-agent`，共享挂载改为 `/mnt/riftvm-shared`，VirtioFS tag 改为 `riftvm_shared`，同步更新双方代码和测试。
- Omarchy factory/Overlay 重新构建、校验和签名；不直接修改已签名 manifest 内容。
- vsock 端口等无品牌参数可保留既有技术值；协议版本与应用版本独立管理。

建议协议层永远使用中性或版本化 wire identifiers，避免下一次品牌变化再次破坏兼容。

### 3.6 环境变量与自动化接口

新变量统一采用 `RIFTVM_*`，例如：

- `RIFTVM_SIGNING_IDENTITY`
- `RIFTVM_OMARCHY_FACTORY_PUBLIC_KEY_BASE64`
- `RIFTVM_SOURCE_REVISION`
- `RIFTVM_VIRGL_*`

切换规则：

1. 只读取 `RIFTVM_*`，不提供 `EZVM_*` fallback。
2. CI、发布脚本、构建设置与文档同步更新。
3. secret 的名称可改，密钥材料不因品牌变化而随意轮换。

测试、fixture 与临时路径统一采用 `/tmp/riftvm-*`。脚本清理自己创建且已不用的临时产物。

### 3.7 版本基线

- RiftVM：`1.0.0`；RiftVM Omarchy：`1.0.0`。
- 首个发布 build number 从 `1` 开始，后续候选重建递增。
- 同步更新 Xcode、XcodeGen、CLI 版本来源、Cask、发布脚本、README 示例与验收预期。
- 首发 tags：`riftvm-v1.0.0`、`riftvm-omarchy-v1.0.0`。
- 首发制品：`RiftVM-1.0.0.zip`、`RiftVM-Omarchy-1.0.0.zip`。
- 不沿用 EZVM 的版本序列；Omarchy 上游系统版本和协议/schema 版本各自独立。
- 首页下载按钮继续不显示版本号。

## 4. 产品架构目标

```text
macOS
├── RiftVM / RiftVM Omarchy
│   ├── Workspace & Space Controller
│   ├── Permission Broker
│   ├── Keyboard / Trackpad Router
│   ├── Clipboard & File Bridge
│   ├── URL / Notification Bridge
│   ├── Port Discovery
│   └── VM lifecycle + Virtualization.framework
│
│        authenticated virtio socket
│
└── Guest
    ├── Rift Agent (system service)
    │   ├── authentication & capability negotiation
    │   ├── power, status, files and device plumbing
    │   └── controlled local IPC
    └── Rift Session Agent (user service)
        ├── Wayland clipboard
        ├── Hyprland/workspace state
        ├── notifications
        ├── open URL/file
        └── Omarchy adapter
```

Root Agent 不直接操作用户 Wayland session；桌面语义由普通用户 Session Agent 承担。

## 5. 分阶段实施

### Phase 0 — 基线封存

- 确认 `main`、tag、Release、Homebrew 与 Pages 当前状态。
- 创建改名前 tag，例如 `ezvm-final-baseline-2026-09-06`。
- 保存当前签名、Bundle ID、版本、Agent 协议和 factory manifest 样本。
- 准备全新 RiftVM/Omarchy 测试工作区，不制作旧版本升级 fixture。

验收：源码基线已推送并可追溯；全新安装测试条件就绪。

### Phase 1 — 锁定新身份与版本

- 集中定义产品名称、Bundle ID、路径、manifest kind、CLI 与环境变量。
- 两款 App 的正式版本统一设为 `1.0.0`。
- 锁定 `.riftvm` 文件格式与新 Host/Agent 协议契约。
- 盘点品牌、API、持久化与 wire 标识的同步修改位置。

验收：新身份映射明确，测试预期使用同一套新标识。

### Phase 2 — 工程内部改名

- 使用 `git mv` 重命名目录、project、scheme、target、模块、源码和测试文件。
- 将 Swift API 从 `EZVM*` 改为 `RiftVM*`。
- 更新 XcodeGen、SwiftPM、脚本与 workflow 路径。
- 生成工程后确认无 stale project diff。
- 单独提交机械改名，不混入功能变化，便于 review 和 bisect。

验收：`swift build --build-tests`、Swift tests、两个 Xcode projects 的 build-for-testing、Guest Agent tests 全部通过。

### Phase 3 — macOS 产品身份与全新安装

- App 名称改为 RiftVM / RiftVM Omarchy。
- Bundle ID、entitlements、日志 subsystem、Notification identifiers 更新。
- Application Support、UserDefaults、Keychain、VM root 与 enrollment 使用新命名空间。
- 创建、打开、导入导出统一使用 RiftVM 格式。
- 完成新产品首次启动与权限引导。
- 在新建 VM 上验证 snapshot、saved state、thumbnail、USB/network 配置和 Guest credentials。

验收：干净环境中首装、创建、启动与重新打开正常，不依赖任何 EZVM 数据。

### Phase 4 — Rift Agent 与新镜像

- binary：`ezvm-agent` → `rift-agent`。
- user service：`ezvm-session-agent` → `rift-session-agent`。
- package、systemd/OpenRC、journal 文案与 Overlay 改名。
- service、socket、挂载路径、VirtioFS tag 与 Host 调用同步改名。
- 增加 Agent 自升级/重启时的断线恢复与 key-up 清理。
- 文档与诊断统一为 Rift Agent；重新构建匹配的新 factory 和 Overlay。

验收：两款新 Host 与新 Agent 的认证、桌面会话、文件通道及断线恢复正常。

### Phase 5 — CLI、制品、CI 与发行链

- 只发布 `riftvm` CLI，无 `ezvm` shim。
- 发布全新 `riftvm` Homebrew cask，不实现旧 cask 自动升级迁移。
- Release tag 改为 `riftvm-v*`、`riftvm-omarchy-v*`。
- ZIP/App/diagnostics/factory/overlay 制品全部改名。
- CI workflow、cache、artifact、environment 与 secret 名切换到 RiftVM。
- GitHub 仓库计划改名为 `everettjf/riftvm`；依靠 GitHub redirect 过渡，并显式更新所有 checkout、raw、release URL。
- 外部 `omarchy-aarch64-image` 仓库只改关联 manifest/下载 URL，不把 Omarchy 名称替换掉。

验收：干净机器通过 Homebrew 和独立 ZIP 两条路径安装；签名、Notarization、Stapler、Gatekeeper 和 CI 全绿。

### Phase 6 — `riftvm.com` 与品牌发布

- DNS：apex 与 `www` 均启用 HTTPS，`www` 统一跳转 apex。
- GitHub Pages 或后续托管配置自定义域名与 CNAME。
- `xnu.app/ezvm` 保留长期 301 跳转，不制造断链。
- 官网首页延续裂缝、火/冰、拉力车视觉，但品牌文字全面替换为 RiftVM。
- 首页主叙事从“Virtual machines made easy”升级为“another native workspace on your Mac”。
- 产品入口：RiftVM Omarchy 为旗舰；RiftVM 为通用 macOS/Linux 能力。
- 更新 canonical URL、Open Graph、favicon、下载链接、隐私说明与 Support URL。
- 发布 RiftVM 1.0.0 公告，说明产品定位与安装方式，不承诺 EZVM 数据兼容。

验收：桌面/移动端、SEO metadata、下载、HTTPS、重定向和 analytics 均在线验证。

### Phase 7 — 第二工作空间体验

品牌迁移完成后，按用户价值推进产品能力：

#### P0：进入 Omarchy 就像进入一个 Mac Space

- `Open Omarchy` 直接恢复/启动并进入全屏 Space。
- 默认保存运行状态，冷启动、恢复和错误状态都不暴露 VM Manager 流程。
- 三/四指 macOS Spaces 和 Mission Control 手势由 Host 优先。
- Window mode 保留为 secondary mode。

#### P0：输入模型

- 默认 Mac-first：`⌘Tab`、`⌘Space`、Mission Control 与 Spaces 留在 Host。
- Guest GUI 支持语义映射：`⌘C/V/X/A/Z/F` → 对应 Linux GUI 行为。
- Terminal 中 `⌘C` 是复制，物理 `Control-C` 保留 SIGINT。
- 提供 Mac / PC / Raw 三种模式和清晰 Host escape 行为。
- 所有焦点丢失、暂停、Agent 重启路径补齐 key-up。

这与当前“Command → Super 全部送给 Omarchy”的策略不同，必须作为独立产品决策和可用性测试处理，不能在纯改名提交中顺手改变。

#### P0：文本剪贴板与共享目录

- 文本双向、Unicode、大文本和 loop suppression。
- 默认建议 `~/Projects`、`~/Downloads`，不共享整个 `$HOME`。
- 每个目录具备 No Access / Read Only / Read & Write。
- 主动阻止或高风险提示 `.ssh`、`.gnupg`、Keychains、Mail、Messages。
- Guest 统一呈现为 `~/Mac/...`，底层可继续使用 VirtioFS。
- 单一文件系统是真相来源，不做双份后台同步。

#### P1：图片、文件与打开语义

- 双向图片剪贴板。
- Finder → Open in Omarchy。
- Guest → `rift open` / `rift reveal`。
- 双向 Drag & Drop；共享目录内优先传 path reference，非共享文件走受认证传输。

#### P1：通知与 URL

- Guest 通知映射到 macOS Notification Center。
- 点击通知进入对应 Omarchy Space/应用。
- Guest URL 可按策略交给 Mac 默认浏览器。
- 所有 Host 动作都受 capability 和用户策略控制，不提供任意 macOS shell 接口。

#### P1/P2：网络

- Guest listening-port discovery。
- 明确授权的端口可从 Mac 使用 `localhost:<port>` 访问。
- 提示 “Omarchy is serving on port 8000 — Open in Safari”。
- 后续评估稳定本地域名；避免隐式暴露所有 Guest 服务。

#### P2：硬件与长期可靠性

- Retina/HiDPI、动态分辨率、60/120Hz、GPU、无 capture 感 pointer。
- Audio、Microphone、Camera 与 USB 的 Ask / Allow / Deny 策略。
- 合盖/唤醒、网络切换、Hyprland restart、Agent restart 自动恢复。
- 连续数日 soak、升级前快照、失败回滚与版本兼容矩阵。

## 6. 明确不做的事情

- 不把改名与输入策略重设计混在同一个提交。
- 不实现 EZVM 数据、格式、CLI、环境变量与旧 Agent 兼容；不自动删除用户已有数据。
- 不默认共享整个 Home，不默认开放 Guest 服务端口。
- 不给 Guest 任意执行 Host shell 的接口。
- Host、Agent、镜像和发布脚本同步切换新标识。
- 不把旧版本升级测试列为发布前提。

## 7. 测试矩阵与发布门槛

### 构建与静态检查

- SwiftPM build/tests。
- 两个 Xcode app targets 与 test bundles。
- Go Guest Agent tests + ARM64 static cross-build。
- XcodeGen generated project freshness。
- shell syntax、artifact naming、entitlements、Info.plist、codesign。

### 全新安装测试

- RiftVM 1.0.0 与 RiftVM Omarchy 1.0.0 的独立首装。
- 新 `.riftvm` 位于默认目录、自定义目录和外置磁盘。
- 新 Keychain credential、enrollment、saved state 与 snapshots。
- 安装/下载中断后重试，应用重启后正常恢复。
- 确认未读取、修改或删除 EZVM 旧数据。

### Host/Guest 验证矩阵

| Host | Guest Agent | 预期 |
|---|---|---|
| RiftVM 1.0.0 | Rift Agent | 通用 Linux 集成能力 |
| RiftVM Omarchy 1.0.0 | 新 factory 内 Rift Agent | 完整 Omarchy 集成能力 |
| 任一新 Host | Agent 未就绪或协议不支持 | 明确状态与受控错误 |

### 发布门槛

- 签名、Notarization、Stapler 与 Gatekeeper 全通过。
- Accessibility、Notifications、Microphone 权限均在新 Bundle ID 实机验证。
- Command/Control、焦点切换、暂停恢复不产生 stuck modifiers。
- Clipboard、动态分辨率、通知和生命周期 acceptance evidence 重新生成。
- Pages/`riftvm.com` 桌面与移动端实际访问验证。
- CI 不允许依赖“测试返回成功但运行时测试未执行”的模糊状态；受 runner 限制的测试必须在原生 acceptance host 补证据。

## 8. 提交与合并策略

建议拆成可审查、可回滚的提交/PR：

1. `Define RiftVM identity and 1.0.0 release baseline`
2. `Rename shared Swift modules and projects to RiftVM`
3. `Set up fresh RiftVM application identity and storage`
4. `Rename Rift Agent and rebuild guest assets`
5. `Rename CLI, CI and release artifacts`
6. `Launch riftvm.com for the 1.0.0 release`
7. 后续独立功能提交：Spaces、输入语义、共享能力、通知与端口发现。

机械重命名与行为变化必须分开；每一步都保持可构建，并在合入 `main` 前通过对应测试。

## 9. 回滚策略

- 改名前 tag 和最后一个 EZVM Release 长期保留。
- 不实施旧数据迁移；源码基线用于开发回退，不代表支持旧版本读取 RiftVM 数据。
- 发布前使用独立测试工作区验证新数据；失败候选不替换已验证制品。
- Website/DNS 可立即回滚到上一版本；旧 URL 保留重定向。
- 新 Agent Overlay 发布失败时阻止发布，修复后重新验证匹配的 Host/Agent/factory 组合。
- 每个发布制品附 source revision、tree state、协议版本和数据 schema version。

## 10. 推荐执行顺序

立即开始时，严格按以下顺序：

1. 锁定 Bundle ID 与 reverse-DNS namespace。
2. 创建改名前 tag 和全新安装 fixture。
3. 集中定义新身份，将两款产品版本设为 `1.0.0`。
4. 再做工程级重命名。
5. 然后切换产品身份、Agent、CLI 和发行链。
6. 最后切换 `riftvm.com` 与公开品牌。
7. 品牌稳定后进入“第二原生工作空间”功能路线。

首发完成标准：全新安装的 RiftVM 与 RiftVM Omarchy 均显示 1.0.0，新命名贯通 App、CLI、Agent、镜像、CI、制品和官网，首装工作流通过实机验收。
