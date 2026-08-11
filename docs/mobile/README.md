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

- **结构**:RN 应用 + 自研 TurboModule 桥接层(iOS 包 NMSSH / Android 包 mwiede-jsch)+ 终端用**原生组件**(iOS SwiftTerm / Android Termux terminal-emulator)。
- **复用**:domain 纯逻辑、hooks 编排、AI API 层、云同步客户端;UI 组件需按 RN 重写(radix/tailwind 不可用)。
- **优点**:长期质量与商店体验最优,生态主流,业界路线验证(Termius/Blink 模式)。
- **缺点**:UI 全部重写,工作量最大;原生桥接层(SSH/SFTP/转发/终端)需自研且双端各一份。

### 方案 B:Capacitor 打包现有 React UI + 自研原生 SSH 插件(PoC 验证工具)

- **结构**:现有 React 渲染层几乎原样进 Capacitor WebView;SSH/SFTP/终端走 JS → Capacitor 插件 → 原生(NMSSH/JSch);终端仍在 WebView 内跑 xterm.js(与桌面完全一致)。
- **复用**:UI 复用最大化(桌面 web 与移动 web 共享一套组件);domain/hooks/云同步/AI 全部复用。
- **优点**:MVP 最快,交互与信息架构验证成本最低,桌面行为对齐度最高。
- **缺点**:原生 SSH 桥接层仍需自研(与 A 的原生工作量接近);WebView 终端在移动端的触摸/软键盘/大文件 IPC 是已知老大难;长期体验上限低于 A。
- **风险提示**:Capacitor 生态**没有现成 SSH 插件**,此方案成败取决于自研插件质量,不应假设能"开箱即用"。

### 方案 C:远程网关/代理(薄客户端)

- SSH 经自建网关中转。**不满足"本地直连"预期**,且引入服务器依赖与安全面。
- **定位**:仅作长期可选/受限网络场景补充,不作为主线。

### 体验对比:Capacitor(WebView)vs RN(原生)

| 维度 | Capacitor(WebView + xterm.js) | RN + 原生终端(SwiftTerm / Termux emulator) |
|---|---|---|
| 大输出滚动性能 | DOM 渲染,输出刷屏明显卡顿 | Metal / 原生渲染,碾压级差距 |
| 软键盘 / IME | WebView 焦点管理是移动端老大难(桌面 xterm 按物理键盘设计) | 原生组件精确控制软键盘、补全、IME 组合 |
| 触摸交互 | 无 Ctrl/Alt/滚轮/右键,只能屏幕按钮模拟 | 原生触控条、手势、专业键盘行(Blink 为标杆) |
| 高级终端协议 | xterm addon-webgl 在 WebView 受限 | SwiftTerm 原生支持 Sixel / iTerm2 / Kitty 图像协议 |
| UI 层交互 | 桌面"鼠标/键盘/悬停"思维,响应式适配手感打折 | 从交互模型即移动原生:手势、导航、触觉反馈 |
| 桌面行为一致性 | 与桌面完全一致(也是优点) | 需重新定义移动端交互规范 |

**结论**:两者在"WebView 包 xterm"时终端体验同级;真正的分水岭是能否换掉 WebView 终端。体验优先 → 原生终端路线。

## 5. 推荐路线

**体验是首要目标 → 主推方案 A(RN 重写 + 原生终端)**,Capacitor 仅作 PoC 验证工具,不作为产品形态:

1. **首版直接走 RN(方案 A),工程形态 Expo**:UI 按移动原生交互模型重写;SSH 桥(iOS NMSSH / Android mwiede-jsch)为共同必建底座;终端层**首版即上原生组件**(iOS SwiftTerm / Android Termux terminal-emulator 库,已核实 Apache-2.0),不接受 WebView xterm 作为长期形态。Expo 下自研 TurboModule 用 development build + config plugin 承载,密钥/本地文件用 expo-secure-store / expo-file-system,OTA 更新用 Expo Updates。
2. **首端策略:Android 为主**:iOS 开发者账号/签名/审核流程有时间和金钱成本,Android 先行验证产品,iOS 后置或仅 IPA 打包验证;原生桥按双端共享接口设计,JS 层全复用。
3. **首版范围(最小核心)**:SSH 终端(密码/密钥/host key 验证)+ SFTP 浏览与传输 + 端口转发 + 密钥管理 + vault(主机/分组/snippets)。其余能力(云同步、AI、系统管理器、插件/脚本/MCP、telnet/mosh/et/serial、x11)全部后置。
4. **Capacitor 仅用于 PoC**:正式投入 RN 重写前,用 Capacitor + 现有 UI 快速验证交互与信息架构(≤2 周),产出移动端界面设计依据;验证完即弃,代码不进入产品线。

**理由**:SSH 终端是重度键盘 + 高频刷屏场景,恰是 WebView 最弱的两个点;业界标杆(Blink Shell / JuiceSSH / Termius 移动端)均验证"原生终端 + 原生 SSH"路线。体验优先前提下,WebView 方案的开发速度优势不构成取舍理由。

## 6. 桌面能力 → 移动端映射表

> **首版(最小核心)范围**:SSH 终端、SFTP/传输、端口转发、密钥管理、vault;下表其余能力后置。

| 桌面能力 | 移动端方案 | 复用度 | 备注 |
|---|---|---|---|
| SSH 终端 | 原生 SSH 库(NMSSH / mwiede-jsch)+ 原生终端(Termux terminal-emulator / SwiftTerm) | 协议重写;渲染原生 | 移动端键盘/IME 是最大体验风险 |
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
| Phase 0 | PoC:Capacitor 现有 UI 验证交互/信息架构 + RN 骨架 spike(SSH 桥 + 原生终端渲染可行性) | 产出移动端界面设计依据;RN 技术风险验证完 | S |
| Phase 1 | **Android**:RN(Expo)骨架 + 认证体系(密码/密钥/host key)+ vault 迁移 + Termux terminal-emulator 终端 | 真机可日常使用的基本 SSH 客户端 | M |
| Phase 2 | SFTP 浏览 + 上传下载/传输管理 | 文件操作可用 | M |
| Phase 3 | 端口转发 + 密钥管理器 + known_hosts | 对齐桌面核心(最小核心完成) | M |
| Phase 4 | iOS 适配(SwiftTerm + 签名/审核流程)或仅 IPA 打包验证 | iOS 可运行 | M |
| Phase 5 | 扩展能力:云同步 / AI(API 直连)/ snippets / 系统管理器 + 商店发布 | 商店上架 | M-L |

## 8. 主要风险

1. **终端维护成本**:原生终端组件双端各一份(iOS SwiftTerm / Android Termux terminal-emulator),需跟进上游与许可;WebView 方案已因体验原因排除。
2. **原生 SSH 桥自研成本**:SSH/SFTP/转发/认证都要自研,且 iOS/Android 双实现,维护持续投入。
3. **host key 验证缺失风险**:选用现成 RN 库必须自补验证,否则 MITM。
4. **范围预期管理**:首版仅最小核心,云同步/AI/系统管理器等后置,需与用户预期对齐,避免"全功能对齐"误解。
5. **商店合规**:私钥/known_hosts 落 Keychain/Keystore 的审核要求、iOS 本地网络权限弹窗文案。

## 9. 决策记录(2026-08)

1. **首版路线**:体验优先 → RN 重写 + 原生终端,Capacitor 仅作 PoC。✅
2. **对齐边界**:最小核心优先 —— 首版仅 SSH 终端 / SFTP / 端口转发 / 密钥管理 / vault;云同步、AI、系统管理器、插件/脚本/MCP、telnet/mosh/et/serial、x11 全部后置。✅
3. **终端渲染**:iOS = SwiftTerm;Android = Termux terminal-emulator 库(已核实 Apache-2.0,商用无传染风险)。✅
4. **首端策略**:Android 为主(iOS 签名/账号/审核成本高),iOS 后置或仅 IPA 打包验证;原生桥双端共享接口、JS 层复用。✅
5. **RN 形态**:Expo(development build + config plugin 承载自研 TurboModule)。✅
