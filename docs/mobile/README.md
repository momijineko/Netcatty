# Netcatty 移动版可行性方案(草案)

> 分支:`mobile` · 状态:技术调研阶段 · 目标:全功能对齐桌面版
> 本文档是方案草案,关键技术选型见文末「未决问题」,需评审确认后再进入 PoC。

## 1. 背景与目标

Netcatty 是 Electron + React 19 + TypeScript 的桌面 SSH 管理器,能力覆盖 SSH 终端、SFTP/SCP、端口转发、密钥管理、云同步、AI 助手、插件系统等。现规划移动版(iOS + Android),目标**全功能对齐桌面版**。

本阶段任务是技术调研与方案设计,不动代码。

## 2. 桌面版现状(能力面摘要)

### 2.1 主进程能力(electron/bridges/,20+ bridge)

| 域 | 模块 | 核心技术 |
|---|---|---|
| SSH 会话/exec | sshBridge.cjs | `ssh2`(Node),禁用 cpu-features 原生加速 |
| 交互终端 | terminalBridge.cjs + et/mosh/telnet/serial | `node-pty` + xterm.js,utilityProcess worker |
| SFTP/SCP | sftpBridge.cjs(+ scpBackend) | `ssh2` SFTP + 自研 SCP 协议 |
| 全局传输 | transferBridge.cjs(289KB,最大) | utilityProcess 并发/断点续传引擎 |
| 端口转发 | portForwardingBridge.cjs | `node:net` + ssh2 direct-tcpip |
| AI 助手 | aiBridge.cjs + mcpServerBridge.cjs | AI SDK + MCP + codex/claude 等 agent SDK(经 child_process 调 CLI) |
| 云同步 | cloudSyncBridge.cjs + OAuth | WebDAV/S3/GDrive/OneDrive/GitHub |
| 插件系统 | electron/plugins/ | node:sqlite + utilityProcess + BrowserWindow |
| 脚本自动化 | scriptBridge.cjs + scriptRuntime.cjs | child_process 跑 JS 运行时 |
| 系统管理器 | systemManagerBridge.cjs | 经 SSH 会话 exec 远程 Docker/GPU/进程 |
| 其他 | localFs/credential/tempDir/sessionLogs/update/tray/window/deepLink/x11Forwarding | safeStorage、Tray、全局快捷键、深链协议 |

### 2.2 渲染层

- 纯 React hooks 状态管理(无 zustand),组件大量使用 radix-ui + tailwindcss 4。
- 终端用 `@xterm/xterm` 6.1.0-beta + addons(webgl/fit/search/image 等)。
- 桥访问统一走 `window.netcatty`(preload 暴露 200+ 扁平方法)。
- 持久化:localStorage(经 safeStorage 加密敏感字段)+ 主进程文件/sqlite。

## 3. 核心约束(决定架构的事实)

1. **`ssh2` 是 Node 专用库,不支持浏览器/移动端** → 移动端必须换原生 SSH 库。
2. **`node-pty`/`serialport`/`utilityProcess`/`child_process`/`node:net`/`safeStorage` 在移动端不可用** → 主进程"本地终端 + 原生桥"层整体无法复用,需全新移动端桥接架构。
3. **xterm.js 只能在浏览器 DOM 跑,不能在 RN 原生组件直接渲染** → 终端渲染只有两条路:WebView 包 xterm.js,或原生终端组件(SwiftTerm / Termux terminal-emulator)。
4. **RN 生态没有成熟 SSH 库**:唯一活跃 fork 是 `@dylankenneally/react-native-ssh-sftp`(iOS=NMSSH/libssh2,Android=mwiede/JSch fork),但**不做 host key 验证(MITM 风险)、无端口转发 API**。
5. **业界验证**:Blink Shell(纯 Swift + libssh2 + SwiftTerm)、JuiceSSH(Java + JSch)、Termius 移动端(Flutter 壳 + 原生 SSH)——移动 SSH 的**网络层全部落在原生**,跨端框架只承载 UI。
6. **可复用资产**:`domain/` 纯 TS 逻辑、React hooks 的大部分编排逻辑、AI SDK 层(`ai` 包为纯 JS,可移动端直连 API)、云同步 HTTP 客户端、vault/密钥的 UI 编排、i18n 文案、storage key 设计。

## 4. 候选技术方案

### 方案 A:React Native 重写(网络层原生,UI 层重写)

- **结构**:RN 应用 + 自研 TurboModule 桥接层(iOS 包 NMSSH / Android 包 mwiede-jsch)+ 终端用 WebView 包 xterm.js(起步)或 SwiftTerm(进阶)。
- **复用**:domain 纯逻辑、hooks 编排、AI API 层、云同步客户端;UI 组件需按 RN 重写(radix/tailwind 不可用)。
- **优点**:长期质量与商店体验最优,生态主流,业界路线验证(Termius/Blink 模式)。
- **缺点**:UI 全部重写,工作量最大;原生桥接层(SSH/SFTP/转发/终端)需自研且双端各一份。

### 方案 B:Capacitor 打包现有 React UI + 自研原生 SSH 插件(推荐 Phase 1)

- **结构**:现有 React 渲染层几乎原样进 Capacitor WebView;SSH/SFTP/终端走 JS → Capacitor 插件 → 原生(NMSSH/JSch);终端仍在 WebView 内跑 xterm.js(与桌面完全一致)。
- **复用**:UI 复用最大化(桌面 web 与移动 web 共享一套组件);domain/hooks/云同步/AI 全部复用。
- **优点**:MVP 最快,交互与信息架构验证成本最低,桌面行为对齐度最高。
- **缺点**:原生 SSH 桥接层仍需自研(与 A 的原生工作量接近);WebView 终端在移动端的触摸/软键盘/大文件 IPC 是已知老大难;长期体验上限低于 A。
- **风险提示**:Capacitor 生态**没有现成 SSH 插件**,此方案成败取决于自研插件质量,不应假设能"开箱即用"。

### 方案 C:远程网关/代理(薄客户端)

- SSH 经自建网关中转。**不满足"本地直连"预期**,且引入服务器依赖与安全面。
- **定位**:仅作长期可选/受限网络场景补充,不作为主线。

## 5. 推荐路线

**推荐 B 起步、A 为长期演进目标,即"Capacitor 验证 + 原生 SSH 桥 + WebView 终端"双轨**:

1. **Phase 1 用方案 B**:把现有 UI 搬进 Capacitor,自研原生 SSH 桥(认证 + 交互终端 + SFTP 只读浏览),最快验证"移动版全功能对齐"的交互可行性。此阶段**两个方案的原生 SSH 工作量是共享的**(都要包 NMSSH/JSch),区别只在 UI 层。
2. **Phase 2 按体验决策**:若 WebView 终端性能/键盘体验不达标,将终端渲染切换为原生(SwiftTerm / Termux emulator),SSH 桥保持;若 UI 需要深度移动化(手势、性能),再评估整体迁移到 RN(方案 A)。

**理由**:原生 SSH 桥是两条路线共同的必建底座,先建它;UI 层复用优先,用真实移动端体验数据决定是否重写,避免一开始就押注全量重写。

## 6. 桌面能力 → 移动端映射表

| 桌面能力 | 移动端方案 | 复用度 | 备注 |
|---|---|---|---|
| SSH 终端 | 原生 SSH 库(NMSSH/JSch)+ WebView xterm 或 SwiftTerm | 渲染复用(WebView 时);协议重写 | 移动端键盘/IME 是最大体验风险 |
| host key 验证 | 自研(桌面逻辑可参考,库不提供) | 逻辑复用 | **安全必做**,现有 RN 库默认不做 |
| SFTP/SCP | 原生 SSH 库 SFTP;SCP 协议逻辑可移植 | 协议层 JS 可复用 | 大文件/并发需重写传输引擎 |
| 全局传输引擎 | 重写(移动端并发模型不同) | 低 | transferBridge 289KB,依赖 utilityProcess |
| 端口转发 | 自研原生 direct-tcpip + 本地监听 | 低 | iOS 需 NSLocalNetworkUsageDescription |
| 密钥管理 | Keychain(iOS)/Keystore+EncryptedSharedPreferences(Android) | UI 编排复用;存储层重写 | 桌面 safeStorage 不可用 |
| known_hosts | 移动端安全存储 | 逻辑复用 | — |
| vault/分组/snippets | localStorage → AsyncStorage/SQLite | 高 | 桌面就在 localStorage,迁移容易 |
| 云同步(WebDAV/S3/GDrive/OneDrive/GitHub) | HTTP 客户端基本可复用;OAuth 换 Custom Tabs/ASWebAuthenticationSession | 高 | — |
| AI 助手(API 直连) | `ai` SDK 纯 JS 可复用 | 高 | 桌面主进程跑 AI 是为 child_process;移动端直连 API 更合适 |
| AI Agent SDK(codex/claude/copilot CLI) | **不可用**(依赖本地 CLI) | 无 | 需降级为 API 直连或移除 |
| MCP 服务器 | 客户端侧可复用(@modelcontextprotocol/sdk 纯 JS);stdio 服务器不可用 | 中 | — |
| 插件系统 | **不可用**(依赖 BrowserWindow/utilityProcess/sqlite 主进程) | 无 | 需明确降级策略 |
| 脚本自动化 | **不可用**(依赖 child_process) | 无 | 移除或受限 |
| 系统管理器(Docker/GPU/进程) | exec 通道 + UI 均可复用 | 高 | 走 SSH 会话,移动端无额外障碍 |
| 本地文件浏览 | react-native-fs / expo-file-system | 中 | 桌面 localFsBridge 逻辑参考 |
| telnet/serial/mosh/et | telnet 可原生实现;serial 需 OTG 评估;mosh 有现成客户端可集成;et 二进制不可用 | 低 | 建议首版降级 |
| 全局快捷键/托盘/多窗口 | **不可用**(桌面专属) | 无 | 移除 |
| 自动更新 | 商店 OTA(TestFlight/Play)或自建分发 | 低 | 桌面 electron-updater 不可用 |
| 深链 ssh:// / telnet:// | URL scheme 可行 | 中 | 协议注册逻辑参考 |
| x11 转发 | **不可用** | 无 | 移除 |

## 7. 分阶段路线图(建议)

| 阶段 | 内容 | 出口标准 | 相对规模 |
|---|---|---|---|
| Phase 0 | PoC:Capacitor/RN 骨架 + NMSSH/JSch 桥(spike)+ WebView xterm 连接一台真实主机 | 真机连上并跑通交互终端 | S |
| Phase 1 | 认证体系(密码/密钥/host key)+ vault 迁移 + 终端完善 | 可日常使用的基本 SSH 客户端 | M |
| Phase 2 | SFTP 浏览 + 上传下载/传输管理 | 文件操作可用 | M |
| Phase 3 | 端口转发 + 密钥管理器 + known_hosts | 对齐桌面核心 | M |
| Phase 4 | 云同步 + AI(API 直连)+ snippets + 系统管理器 | 覆盖桌面大部分高频能力 | M-L |
| Phase 5 | 降级项定案(插件/MCP/脚本/telnet/mosh)+ 商店发布 | 商店上架 | M |

## 8. 主要风险

1. **终端体验**:WebView xterm 在移动端的滚动、软键盘、IME、性能是行业老大难;若不可接受,需原生终端组件,双端各写一份。
2. **原生 SSH 桥自研成本**:SSH/SFTP/转发/认证都要自研,且 iOS/Android 双实现,维护持续投入。
3. **host key 验证缺失风险**:选用现成 RN 库必须自补验证,否则 MITM。
4. **降级项决策**:插件/MCP/脚本/AI Agent CLI 在移动端不可用,"全功能对齐"的边界需要产品侧明确。
5. **商店合规**:私钥/known_hosts 落 Keychain/Keystore 的审核要求、iOS 本地网络权限弹窗文案。

## 9. 未决问题(需评审确认)

1. **首版路线**:确认 Phase 1 走 Capacitor(方案 B)还是直接 RN(方案 A)?影响后续所有投入。
2. **"全功能对齐"边界**:插件系统、脚本自动化、AI Agent CLI、serial/x11 转发在移动端不可用,是接受降级还是无限期推迟这些能力?
3. **终端渲染**:接受 WebView xterm 起步,还是首版就投入原生终端(SwiftTerm)?
4. **团队与时间**:当前无移动端原生(iOS/Android)人力,原生桥自研部分是否外协或自建?
