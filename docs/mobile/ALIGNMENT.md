# Netcatty Mobile 桌面能力处置审计

> 审计日期: 2026-08-18
> 移动文档基线: `mobile@21a3d084`
> 桌面代码基线: `upstream/main@c4edefdb`
> 审计对象: `docs/mobile/PRD.md`、`docs/mobile/README.md` 与当前桌面代码
> 目标: 每个桌面能力都有可追踪的移动端处置，不把“已处置”误写为“已实现”

## 1. 结论与边界

移动端尚未开始开发。本报告只能证明文档对当前桌面能力的**处置覆盖**，不能证明移动端功能等价、技术可行或已通过验收。

上一版报告在 `4e85a6f6` 创建后，PRD 又被 3 个提交修改，仍声称“293 行、完全对齐、无遗漏”。当前审计已确认传输并发、SCP、端口转发 host key、密钥导出、Notes、终端设置等反例，因此撤销该绝对结论。

处置状态统一为:

| 状态 | 含义 |
|---|---|
| 等价 | 保留桌面能力和用户结果，交互可按移动端重做 |
| 替代 | 移动端用不同交互或系统能力达到同一目标 |
| 后置 | 纳入最终路线图，但不进入当前里程碑 |
| 不适用 | 依赖桌面窗口、鼠标、托盘、本地 CLI 等机制，说明原因和替代 |
| 待验证 | 产品要做，但原生库、性能、平台政策或安全方案尚无本地证据，必须先过 PoC gate |

只有“等价”和“替代”且通过对应版本验收后，才能称为功能 parity。“后置、不适用、待验证”只代表没有静默遗漏。

## 2. 可复现基线

以下命令均在仓库根目录运行，不把模糊的“500+ API”或“20+ bridge”当审计证据:

```bash
git rev-parse HEAD
git rev-parse upstream/main
git merge-base mobile upstream/main
git diff --name-only upstream/main...mobile
git log --author=momijineko --format='%H' -- docs/mobile | wc -l
wc -l docs/mobile/*.md
find electron/bridges -maxdepth 1 -type f ! -name '*.test.*' ! -name '*.spec.*' | wc -l
rg -o 'ipcMain\.handle\(' electron --glob '*.cjs' --glob '!**/*.test.cjs' --glob '!**/*.spec.cjs' | wc -l
rg -o 'ipcRenderer\.on\(' electron/preload/api.cjs | wc -l
```

同步后结果: `mobile` 已包含 `upstream/main@c4edefdb`，相对上游只多 `docs/mobile/AI_TASK_PROMPTS.md`、`ALIGNMENT.md`、`PRD.md`、`README.md` 四份移动文档；`momijineko` 有 38 个相关文档提交。同步时四份文档分别为 564、212、570、207 行，随本轮修订继续变化。非测试顶层 bridge 文件为 85 个，`ipcMain.handle(` 文本匹配合计 366，preload 的 `ipcRenderer.on(` 文本匹配为 48。后两项只是可复现文本计数，不等于稳定 API 数量。

## 3. 桌面能力处置矩阵

| ID | 桌面能力簇 | 移动端处置 | 目标版本 | 代码证据 |
|---|---|---|---|---|
| V-01 | 主机 CRUD、复制、标签、置顶、最近、搜索、批量连接、grid/list/tree | 等价；基础标签/置顶进入 v1.0；桌面近期区截取 6 条，移动数量另定；v1.0 列表保留手动顺序，A-Z/Z-A/newest/oldest/group 等非手动排序和 grid/tree 后置 | v1.0/vNext | `components/vault/VaultHostListSection.tsx`; `components/vault/useVaultHostCollections.tsx`; `domain/hostClickBehavior.ts` |
| V-02 | 多级分组、拖拽/手动顺序及其他排序；分组默认 username/password/save、认证/Identity/key、port/protocol/device、agent forwarding、代理/跳板、启动命令、环境变量/字符集、legacy/算法、Mosh/ET/Telnet 专属字段及终端外观 | 等价；触摸移动替代拖拽。v1.0 起保留和编辑手动顺序与完整继承字段，未交付的 agent forwarding/Mosh/ET/Telnet 等显示 unsupported 并按对应能力后置；A-Z/Z-A/newest/oldest/group 等非手动排序随 V-01 的后置视图能力处理，不能因当前只执行 SSH 而删除继承值 | v1.0/v1.2/vNext | `domain/models/connection.ts`; `domain/groupConfig.ts`; `components/GroupSshSettingsSection.tsx` |
| V-03 | 主机内联备注、图标/发行版、启动命令、每主机终端覆盖 | 后置；字段与继承规则必须保留。发行版自动检测先用握手已有的 SSH banner 识别网络设备，再在同一已认证 transport 上执行 OS probe；已识别/手工标记的网络设备不做 POSIX shell probe，迟到结果不得覆盖新连接。`manualDistro`/自定义图标只改变展示，不得反向改变探测和能力安全分类 | vNext | `domain/models/connection.ts`; `components/HostDetailsAdvancedSections.tsx`; `components/HostIconPicker.tsx`; `components/terminal/runtime/terminalDistroDetection.ts`; `domain/systemManager/systemTarget.ts` |
| V-04 | CSV、PuTTY、MobaXterm、SecureCRT、ssh_config 导入；保留源分组/导入既有分组/新建分组、CSV 模板；托管 ssh_config 双向写回 | 导入等价但格式版本范围待产品决定，目标分组和模板属于同一导入契约；托管源后置，移动文件权限需验证 | 待决/vNext | `components/vault/ImportVaultDialog.tsx`; `application/state/vaultImportOptions.ts`; `domain/sshConfigSerializer.ts` |
| V-05 | 独立 Vault Notes：标题、正文、层级分组、标签、关联主机、搜索、MD/TXT 导入、MD/ZIP 导出 | 后置；与 `Host.notes` 分开 | vNext | `domain/models/connection.ts`; `domain/notes.ts`; `components/notes/NotesManager.tsx` |
| V-06 | 本地加密备份、保留、预览、恢复；云同步快照/冲突/修订；桌面 `ManagedSource` 描述符独立持久化且不在当前 backup/sync payload 中，Host 仍可能带 `managedSourceId` | 本地备份进入 v1.2、云同步后置；恢复时必须携带可重新授权的源描述符，或去除/修复悬空 `managedSourceId`，不得把桌面 `filePath` 假定为跨端可用 | v1.2/vNext | `application/syncPayload.ts`; `application/state/useVaultState.ts`; `components/cloud-sync/CloudSyncLocalBackupsPanel.tsx`; `domain/models/history.ts` |
| V-07 | 主机 CSV 导出：字段映射、密码、KeyPath、passphrase、IPv6、跳过不可表示协议与结果提示 | 后置且需安全决策；移动端默认不让明文格式携带凭据，兼容例外需高摩擦确认 | vNext | `components/VaultView.tsx`; `application/vaultCsvExportCredentials.ts`; `domain/vaultImport/csvExport.ts` |
| V-08 | Quick Connect：解析 `user@host:port`、括号/裸 IPv6、有限 `ssh` 命令参数和 PuTTY 风格命令行参数；忽略项警告；SSH/Mosh/ET/Telnet、password/key/certificate/Identity；连接或保存主机 | 等价且纠正范围：桌面直接自定义认证 UI 只有 password/key，certificate 仅能经已保存 Identity 间接使用；移动 v1.1 交付 SSH password/key，certificate/Identity 随 v1.2 Keychain/Identity 能力交付。桌面此入口没有跳板/代理编辑；未保存连接是 ephemeral，不能进入 vault 或 session restore；移动若增加跳板属于新增。PuTTY 参数解析必须保留可应用字段、逐项报告忽略项并禁止 URL/命令行凭据进入历史或 Vault | v1.1/v1.2/vNext | `domain/quickConnect.ts`; `domain/quickConnectHost.ts`; `components/QuickConnectWizard.tsx`; `electron/puttyCommandLine.cjs`; `electron/deepLink.cjs` |
| V-09 | Quick Switcher：搜索 Hosts、Vault/SFTP tab、workspace/session、本地 shell、插件 command/view；主机连接/编辑和新建 workspace | 结果拆分处置：移动全局搜索保留主机、会话与核心目的地并整体后置；workspace、本地 shell 不适用；插件 command/view 随插件后置；设置搜索是另一能力，不能误报为桌面 Quick Switcher 基线 | vNext | `components/QuickSwitcher.tsx`; `application/app/AppHandlers.ts`; `domain/settingsSearchCatalog.ts` |
| V-10 | 复制主机地址与明文账密：Vault/终端可复制 hostname；Vault 账密复制先应用分组继承，再解析 SSH/Telnet、Identity、IPv6 和非默认端口，输出 host/username/password；无可读密码时不写剪贴板 | 地址复制等价；账密复制等价结果但安全强化，复制前再次认证并明确系统剪贴板历史/同步风险，平台允许时限时清除，锁定或后台时不可触发；plugin endpoint、无密码和写入失败均有明确结果 | v1.0/v1.2 | `components/VaultView.tsx`; `domain/host.ts`; `components/host/HostTreeContextMenus.tsx`; `components/terminal/TerminalView.tsx` |
| V-11 | Vault 八个 section：Hosts、Keychain、Proxy Profiles、Snippets、Notes、Port Forwarding、Known Hosts、Connection Logs | 全部登记移动页面、入口和版本；可合并导航但不能让分阶段能力在其承诺版本不可达，具体映射见 PRD 2.1 | v1.0-vNext | `components/VaultView.tsx`; `components/vault/VaultViewLayout.tsx` |
| C-01 | `auto/password/key/certificate`、Identity 继承、MFA/kbd-interactive 顺序；并发认证请求归属；加密 key passphrase 的记住/超时/失败/跳过/取消；桌面 `~/.ssh/id_*` 默认 key 的发现、排序与认证模式排除 | 等价或移动替代；各连接面行为一致。移动交互可不用桌面全局 modal，但每个 `requestId` 必须绑定正确 session/boot/host，多个请求可排队或明确切换；断连/重连/超时清理旧请求，MFA 二次因子不得覆盖登录密码，passphrase 的“跳过当前 key”和“取消整个认证”保持不同结果。桌面先试 `id_ed25519/id_ecdsa/id_rsa`、再按名称试其他有效 `id_*`，初始跳过加密 key，合资格失败后再请求口令重试；password-only、显式 key/certificate 和 `IdentitiesOnly` 不得混入无关默认 key。移动 v1.0 以已导入 Keychain/显式文件授权替代静默 home 扫描，若后续提供目录发现则保持同一顺序与排除语义 | v1.0/v1.2/vNext | `domain/models/connection.ts`; `domain/sshAuth.ts`; `electron/bridges/sshBridge.cjs`; `electron/bridges/sshAuthHelper.cjs`; `application/app/useAppStartupEffects.ts`; `application/app/AppSideEffects.tsx`; `application/app/AppHandlers.ts` |
| C-02 | TCP connect 与 auth-ready 两类超时、错误分类、取消 | 等价 | v1.0 | `domain/sshConnectionTimeouts.ts`; `components/terminal/connectionTimeouts.ts` |
| C-03 | host key `trusted/unknown/changed`；仅本次继续/保存/替换/拒绝；桌面全局校验关闭开关 | trusted/unknown 决策和 changed 的 fail-closed 初始停止进入 v1.0；changed 的“仅本次继续”或连接内替换并继续仍由 PRD 第 13.5 节决定，未决前不能算 v1.0 等价能力。移动端不照搬可跨终端/SFTP/转发静默绕过的普通全局开关 | v1.0/待决 | `electron/bridges/hostKeyVerifier.cjs`; `components/settings/tabs/SettingsTerminalTab.tsx`; `components/terminal/TerminalHostKeyVerification.tsx` |
| C-04 | known_hosts 查看、搜索、导入、扫描系统、删除、转主机 | 最小信任记录管理 v1.0；完整管理后置 | v1.0/v1.2 | `components/KnownHostsManager.tsx`; `domain/knownHosts.ts` |
| C-05 | 多级跳板、每跳认证与 host key、HTTP/SOCKS5、ProxyCommand；代理档案含 Identity/手工账密、CRUD/复制/搜索、引用数、grid/list 与手动顺序 | 跳板和内联 HTTP/SOCKS5 等价；ProxyCommand 执行不适用但配置迁移需保留为 unsupported；完整代理档案后置且删除需修复引用 | v1.2/vNext | `domain/models/connection.ts`; `components/ProxyProfilesManager.tsx`; `electron/bridges/sftpBridge/openConnection.cjs` |
| C-06 | keepalive、环境变量、legacy 算法、5 类算法覆盖、skip ECDSA、网络设备模式 | keepalive interval/count 与主机覆盖在 v1.x 等价；环境变量、legacy/算法覆盖、skip ECDSA 和网络设备模式后置，安全警告与继承规则等价 | v1.x/vNext | `domain/models/connection.ts`; `components/host-details/AlgorithmOverridesPanel.tsx` |
| C-07 | 系统 SSH agent 登录：`useSshAgent`、`IdentityAgent`、`IdentitiesOnly`、`AddKeysToAgent`、macOS `UseKeychain`；登录后的 agent forwarding | 后置/Gate；移动端不得假定存在 OpenSSH agent/socket 或照搬 macOS Keychain 指令；agent 登录与把 agent 暴露给远端是两种权限，分别设计、提示和审计 | vNext | `domain/models/connection.ts`; `domain/sshAuth.ts`; `components/HostDetailsAdvancedSections.tsx` |
| C-08 | 保存主机的默认/附加协议与连接时协议选择：SSH、Telnet、Mosh、ET、plugin provider，各自端口/主题/凭据字段 | 数据等价、可用能力分阶段；v1.0 只能执行 SSH，但必须保留未知/后置协议字段并显示 unsupported，不得静默改写默认协议；当前桌面 picker 实际只在“附加 Telnet 且默认非 Telnet”时强制弹出，不能泛称所有多协议主机都会询问 | v1.0/vNext | `domain/models/connection.ts`; `components/ProtocolSelectDialog.tsx`; `application/app/AppHandlers.ts`; `components/HostDetailsPanel.tsx` |
| C-09 | 已认证 SSH transport/channel 复用：复制/拆分新 shell、SFTP、转发等尽量避免重复 MFA；endpoint 指纹含 host/profile、route、auth、keepalive、SUDO 与 agent-forwarding policy；不匹配时新连 | 等价结果 + 实现 Gate；复用只能在可证明相同安全上下文时发生，shell 的 agent-forwarding 必须精确匹配；Mosh/ET/Telnet/Serial/local 不冒充可复用；普通浏览/共享传输可复用 endpoint，但重启恢复或显式专用传输以 `reuseTransport:false` 独立连接，可能重新认证/MFA，且不能附着 terminal/parked transport；失败回退新认证且不丢操作 | v1.0/v1.x | `application/state/terminalConnectionReuse.ts`; `application/state/sftp/dedicatedTransferResume.ts`; `electron/bridges/sshConnectionPool.cjs`; `electron/bridges/sshBridge/startSession.cjs`; `electron/bridges/sftpBridge/openConnection.cjs` |
| T-01 | VT/xterm 终端、UTF-8/GB18030、宽字符、URL、回滚搜索、Ctrl 信号；大输出 transport/renderer 背压、消费 ACK、pause/drain 和紧急中断优先级 | 等价；搜索桌面基线固定不区分大小写、非正则、非整词，支持前后导航、清除高亮与关闭后回焦；终端引擎与渲染形态待真机对照 Gate。移动 bridge 必须按实际消费确认输出，以高/低水位滞回限制 transport 与渲染队列；休眠、重挂载、snapshot/owner handoff 和关闭前 drain，失败不可保存过期快照；Ctrl+C 等紧急输入不被输出洪水阻塞，恢复后不丢失、重复或乱序输出。阈值由真机 Gate 决定，不照搬 Electron 常量 | v1.0/v1.1 | `components/Terminal.tsx`; `components/terminal/hooks/useTerminalSearch.ts`; `components/terminal/TerminalSearchBar.tsx`; `components/terminal/runtime/terminalOutputPipeline.ts`; `components/terminal/runtime/terminalSessionAttachment.ts`; `electron/bridges/terminalFlowAck.cjs`; `electron/bridges/terminalFlowPauseArbiter.cjs`; `electron/bridges/terminalEncoding.cjs` |
| T-02 | 移动键盘、IME、Esc/Ctrl/Alt/Tab/方向键/符号、多行 Compose Bar | 移动替代，必须做设备级验收 | v1.0/v1.1 | `components/terminal/TerminalComposeBar.tsx`; `components/terminal/TerminalToolbar.tsx` |
| T-03 | 多会话、顶层标签重排、重连、重命名、复制、编辑来源 Vault 主机、关闭当前/其他/右侧/全部、广播、恢复/CWD、后台输出/BEL 活动指示 | 等价；会话核心、重排、重连代际和活动指示在 v1.0，restore/CWD 在 v1.1，编辑来源/批量关闭/后台恢复在 v1.x，busy 检测和广播后置。长按菜单/编辑排序替代鼠标拖动并提供无障碍移动；activity 是瞬时状态，不进入 restore，移动 v1.0 不实现 workspace 聚合，系统通知另属移动新增；`tabOrder`、活动标签和自定义名可恢复，但不恢复输出或凭据；ephemeral/local 不显示编辑主机。同一逻辑 session 每次连接使用单调 generation/opaque owner，旧启动、host-key/MFA、输出、exit、发行版/CWD 探测和 cleanup 不得污染或关闭新连接 | v1.0/v1.1/v1.x/vNext | `components/top-tabs/SessionTabContextMenuContent.tsx`; `components/TopTabs.tsx`; `components/terminalLayer/useTerminalLayerEffects.ts`; `components/TerminalLayer.tsx`; `components/terminal/runtime/createTerminalSessionStarters.ts`; `components/terminal/runtime/terminalDistroDetection.ts`; `electron/bridges/sessionBootEpoch.cjs`; `electron/bridges/terminalWorkerManager.cjs`; `application/state/sessionActivity.ts`; `application/state/sessionActivityStore.ts`; `application/state/useSessionState.ts`; `application/state/sessionRestoreState.ts`; `domain/sessionRestore.ts` |
| T-15 | OSC 9/777/99 终端通知：OSC9 简单正文、OSC777 title/body、OSC99 分片/编码/关闭；焦点模式、文本清洗、长度上限、每 session 限流和 CAN/SUB 中止 | 桌面能力已验证；移动不直接复用桌面系统通知。移动若进入 vNext，需定义权限、后台/锁屏脱敏、点击回 session、session activity 联动、去重/限流和通知关闭后的终态；通知内容不得包含秘密或未经用户授权的命令输出 | vNext + platform Gate | `domain/terminalOscNotifications.ts`; `application/state/oscDesktopNotifications.ts`; `components/Terminal.tsx`; `electron/bridges/systemNotification.cjs` |
| T-04 | 多窗口、detach、workspace 分屏/焦点模式 | 不适用；多标签、最近会话、标签分组替代 | - | `domain/models/workspace.ts`; `application/state/useSessionState.ts` |
| T-05 | scrollback、TERM、字体/粗细/行距/fallback/ligatures、ANSI 粗体亮色、光标、对比度、字号临时缩放、滚动/选择/复制/链接策略 | 等价设置，设备不支持项显式降级；移动为缩放提供触控增减/重置，双指缩放待 Gate，不能只保留桌面快捷键/滚轮；`drawBoldInBrightColors` 当前无设置 UI但影响渲染并进入同步 payload，迁移时保留默认/字段或显式废弃，不能静默反转；WebKit smoothing 与 webgl/dom renderer 不作为跨端产品开关。普通 URL、OSC 8 与插件链接统一限制为 `http(s)`，OSC 8 保留确认；桌面右键 `select-word` 当前错误调用 `selectAll()`，移动按真实词边界实现而不继承缺陷 | v1.1/vNext | `domain/models/terminal.ts`; `application/syncPayload.ts`; `components/settings/tabs/SettingsTerminalTab.tsx`; `components/terminal/runtime/createXTermRuntime.ts`; `components/terminal/runtime/terminalFontZoom.ts`; `components/terminal/runtime/terminalLinkHandler.ts`; `components/terminal/hooks/useTerminalContextActions.ts` |
| T-06 | bracketed paste、OSC 52、直接粘贴当前终端选区、shell `clear` scrollback 策略、独立 Clear Buffer、动态标题、自动关闭、启动命令延迟、剪贴板图片上传并回填路径、Kitty keyboard、隐藏渲染器休眠 | 等价、替代或后置，安全默认值保留；Paste Selection 不经过系统剪贴板。动态标题运行时当前只消费全局 `off/agent/all`；`Host.disableDynamicTabTitle` 只是未接线的兼容字段，移动须保留字段并单独决定是否新增主机级覆盖。桌面 Clear Buffer 只在 xterm normal screen 清视图并可选擦 scrollback，后端 `clearPty` 仅同步本地 Windows ConPTY 光标且对 SSH/Unix PTY no-op；移动“不回放已清本地缓存”是新增验收契约，不能冒充桌面通用 replay-buffer 清理；自动关闭只作用于已确认的零码正常退出，故障退出保留诊断；启动延迟同时控制首次发送与 `lineDelay` 行间隔；图片上传不等于 inline image 渲染；移动资源回收需独立 gate | v1.1/vNext | `domain/models/terminal.ts`; `domain/models/keyBindings.ts`; `domain/models/connection.ts`; `domain/sessionTabTitle.ts`; `application/state/resolveTerminalSessionExitIntent.ts`; `components/settings/tabs/TerminalBehaviorSettings.tsx`; `components/terminal/clearTerminalViewport.ts`; `components/terminal/hooks/useTerminalContextActions.ts`; `components/terminal/runtime/terminalStartupCommands.ts`; `components/terminal/terminalClipboardPaste.ts`; `electron/bridges/terminalBridge.cjs` |
| T-07 | 自动补全 ghost/popup/history scope、sudo 提示辅助、全局与主机级关键词规则及活动终端快捷编辑、行时间戳 | 后置；主机级规则持久化回 Vault Host，并定义与全局规则的合并/覆盖结果 | vNext | `domain/models/terminal.ts`; `domain/models/connection.ts`; `components/terminal/TerminalAutocomplete.tsx`; `components/terminal/HostKeywordHighlightPopover.tsx` |
| T-08 | Kitty/SIXEL/iTerm inline raster output 及存储、单图和序列容量限制 | 待验证/后置；三个图片协议分别 Gate，并与 T-06 的剪贴板图片 SFTP 上传、T-13/T-14 的文件传输协议分开 | vNext | `domain/terminalInlineImages.ts`; `components/terminal/runtime/createXTermRuntime.ts` |
| T-09 | 自动/手动会话日志，txt/raw/html，时间戳、清理；连接日志元数据及可选只读终端回放；远端命令历史 | 后置，三类数据分开；会话日志的 raw 直接写入未渲染终端流，txt/html 以有状态 cursor/erase/ANSI renderer 生成快照并在 alternate screen 期间省略 TUI paint，以保留 shell 历史，三种格式不能宣称同一保真语义。连接日志桌面基线为最多 500 条未收藏记录加全部收藏记录，最近 50 条未收藏回放另存，收藏回放持续保留 | vNext | `domain/models/history.ts`; `domain/connectionLogTerminalData.ts`; `components/ConnectionLogsManager.tsx`; `components/HistorySidePanel.tsx`; `electron/bridges/sessionLogStreamManager.cjs`; `electron/bridges/terminalLogSanitizer.cjs` |
| T-10 | 每终端工具侧栏：SFTP/Scripts/History/Theme/System/Notes/AI，可排序/隐藏/溢出；当前 7 种工具各最多占一个 pane，可水平/垂直 split，layout 域另设 8-pane 防御上限 | 工具结果等价、布局替代；手机单工具页/快速切换，平板多 pane 待尺寸 Gate；不要与 workspace split 一并删除，也不要把未来硬上限误报成当前可打开 8 种工具 | vNext | `application/state/terminalSidePanelTabs.ts`; `domain/sidePanelLayout.ts`; `application/state/useTerminalSidePanelLayoutState.ts`; `components/terminalLayer/TerminalLayerSidePanelSection.tsx` |
| T-11 | alternate-screen/TUI mouse tracking 下的上下文菜单抑制与强制显示设置 | 移动替代；定义远端手势优先级，并提供不依赖终端右键/长按的常驻应用动作入口，避免 TUI 和重连/Clear Buffer/关闭互相不可达 | v1.1 | `components/terminal/TerminalContextMenu.tsx`; `components/settings/tabs/TerminalBehaviorSettings.tsx` |
| T-12 | Server Stats/主机信息：hostname、OS、kernel、uptime、load average、总 CPU/核心数/每核 CPU、内存 used/free/buffers/cache、swap、内存前十进程、所有挂载磁盘、总计/各网卡 Rx/Tx、SSH endpoint TCP latency | 后置；Linux/macOS 等价结果，网络设备不探测。桌面刷新 5-300 秒、默认 5 秒；连续 3 次硬失败后停止轮询，Mosh handshake `pending` 不计失败，手动刷新或重新启用可重试；连接代际变化后丢弃旧响应 | vNext | `application/state/useServerStats.ts`; `components/terminal/TerminalServerStats.tsx`; `components/systemManager/SystemOverviewTab.tsx`; `components/settings/tabs/SettingsTerminalTab.tsx`; `domain/systemManager/systemTarget.ts`; `electron/bridges/sshBridge/sessionOps.cjs` |
| T-13 | ZMODEM 终端协议检测、拖放上传、远端 `rz -y`、缺少 `rz` 时 SFTP fallback、收发进度/速率/文件序号/最终确认、取消、远端同名文件 overwrite/skip/cancel 与 apply-to-rest | 后置/Gate；保持终端字节流与 transfer 生命周期，不与普通 SFTP 队列、inline image 或 YMODEM 合并；移动没有桌面拖放时用文件选择/分享入口替代，但 fallback、冲突和取消结果仍可观察 | vNext | `lib/zmodemDragDrop.ts`; `components/terminal/hooks/useZmodemTransfer.ts`; `components/terminal/ZmodemOverwriteDialog.tsx`; `electron/bridges/zmodemHelper.cjs` |
| T-14 | YMODEM Serial 单会话发送/接收：显式选择发送文件或接收目录、CRC、ACK/NAK、超时/retry/远端 cancel、进度和二进制完整性；接收文件名清洗并在本地选择唯一目标名 | 后置/Gate；仅 Serial，单会话互斥；接收端自动生成不覆盖的本地文件名，不复用 ZMODEM 的远端 overwrite/skip/apply-to-rest 交互 | vNext | `components/Terminal.tsx`; `components/terminal/TerminalToolbar.tsx`; `electron/bridges/terminalBridge.cjs`; `electron/bridges/ymodemTransfer.cjs` |
| F-01 | SFTP `list/tree`、排序/过滤/隐藏、路径、书签、多标签/复制/重排、跨 pane 移动、name/modified/size/type 列、目录优先、typeahead、可定制工具栏、文件操作/chmod；识别文件/目录/符号链接、断链和目录链接导航；终端 CWD 跟随及为当前远端用户配置 OSC 7 shell hook | 等价或后置；v1.1 的 list 保留链接类型、断链和目录链接导航，vNext 的 tree 再保留展开节点中的同一身份/路径结果。移动保留标签顺序、可见列和工具结果，跨 pane 移动改为单窗格来源/目标标签选择；工具栏的 show/collapse/hide/order 改为触控可达设置；没有桌面 SFTP grid。配置 CWD 追踪是会修改远端 rc 文件的显式确认操作，需预览目标/命令并处理 bash/zsh/fish、sudo/su、版本升级、权限和部分 marker | v1.1/vNext | `domain/models/sftp.ts`; `domain/sftpTypeahead.ts`; `application/state/sftp/useSftpTabsState.ts`; `application/state/sftp/columnLayout.ts`; `components/sftp/SftpPaneToolbar.tsx`; `components/sftp/SftpFileRow.tsx`; `components/terminal/osc7Setup.ts`; `components/terminal/TerminalView.tsx` |
| F-02 | `auto/sftp/scp`、SFTP 不可用自动降级 SCP、编码、SUDO | 等价；原生 SCP 能力待 gate | v1.1/vNext | `domain/models/connection.ts`; `electron/bridges/sftpBridge/openConnection.cjs` |
| F-03 | 上传/下载、目录、压缩、暂停/恢复/取消、断点、重试、远端复制/移动；目录链接的方向相关跟随、循环/深度/总量防护与确定性恢复；普通目录边发现边传。桌面 parent `totalBytes` 实际按已发现文件数推进，传输中心可显示 soft progress，未定义 discovery-complete/final-total freeze；压缩上传另走扫描/压缩/上传/解压流程 | 替代；4 GB+ 流式能力待真机 gate；下载目录可按定义展开远端目录链接，上传/远端复制不得静默套用同一策略，越界或循环形成明确失败/跳过结果。移动端把已发现文件/字节、已传输量和发现阶段分开显示，发现未完成时百分比/全目录 ETA 保持不确定，完成发现后才固定最终总量；这是移动进度契约增强，不宣称桌面已有同一 UI 语义。压缩上传保留独立扫描阶段 | v1.1 | `domain/models/sftp.ts`; `domain/sftpDirectoryCheckpoint.ts`; `application/state/sftp/transferDirectoryOps.ts`; `application/state/sftp/dedicatedTransferResume.ts`; `application/state/useSftpState.ts` |
| F-04 | 冲突 `stop/skip/replace/duplicate/merge` 与同类 apply-all | 等价 | v1.1 | `domain/models/sftp.ts`; `components/sftp/SftpConflictDialog.tsx` |
| F-05 | 全局传输中心、优先处理、全部暂停/恢复、attention/interrupted、重启恢复、dismiss | 等价；生命周期必须持久化 | v1.1 | `components/GlobalSftpTransferCenter.tsx`; `application/state/sftpTransferCenterStore.ts` |
| F-06 | 远端到远端、目标定位；文件/目录 worker 默认 2、范围 1-16；递归目录发现的 listing gate 当前固定默认 4，helper 范围 1-8；单文件请求 fanout 64 | 等价，三个并发概念分开；listing gate 当前调用未读取用户设置，不能误报成桌面可配置项。移动默认值和是否暴露设置由真机资源 Gate 决定；单窗格需分别选择来源/目标并各自认证与校验 host key | v1.1/vNext | `application/state/sftp/transferConcurrency.ts`; `application/state/sftp/transferDirectoryOps.ts`; `application/state/sftp/useSftpTransfers.ts`; `electron/bridges/transferLimits.cjs` |
| F-07 | 内置编辑器 10 MB 上限、语言模式/自动换行、多编辑器标签与 dirty 状态；手动关闭 SFTP tab/断开时 save/discard/cancel，App quit 阻止 dirty 退出，但关闭 owning terminal 会随 side panel 卸载强制丢弃其 dirty editor；外部打开/关联、临时文件、watcher 单向自动回传 | 编辑能力等价，关闭 owner/session/workspace/退出前统一 save/discard/cancel 属移动数据保护增强；系统 document provider 替代外部编辑；桌面 watcher 不检测远端版本，移动回传策略待 gate | v1.1/vNext | `components/sftp/sftpEditorFileLimits.ts`; `components/editor/TextEditorPane.tsx`; `components/sftp/hooks/useSftpViewTabs.ts`; `components/SftpSidePanel.tsx`; `application/app/useAppStartupEffects.ts`; `electron/bridges/fileWatcherBridge.cjs` |
| F-08 | SFTP 双窗格、本地文件窗格、拖拽、OS 剪贴板文件上传 | 桌面双窗格、本地 pane 和拖拽机制不适用；v1.1 以系统 picker 完成文件/目录上传。系统分享接收/回传是移动替代候选，随平台 Gate 后置；不能把 v1.1 picker 误写为 vNext | -/v1.1/vNext | `components/SftpView.tsx`; `components/sftp/clipboardUpload.ts` |
| F-09 | SFTP 内部 copy/cut/paste：记录来源连接/路径/pane，跨连接传输，cut 成功后删源，部分失败保留 | 等价但交互替代；单窗格用来源/目标标签和路径选择，不与 OS 文件剪贴板混用 | v1.1 | `application/state/sftp/sftpClipboardStore.ts`; `components/sftp/hooks/useSftpKeyboardShortcuts.ts` |
| F-10 | 常见归档/单文件压缩解压：远端 `tar/unzip` 能力探测和回退、本地解压计划、临时 staging、原子替换、超时与错误 | 移动替代；v1.1 长按菜单可提供，但必须限制已知格式、正确转义远端路径、隔离临时文件、可取消并显示工具缺失/目标冲突/部分结果；不得实现任意远程 shell 解压入口 | v1.1 + Gate | `electron/bridges/sftpBridge/archiveExtract.cjs`; `components/sftp/useSftpPaneTreeContextMenu.tsx`; `components/sftp/SftpPaneView.tsx`; `domain/sftpArchive.ts` |
| P-01 | 本地/远程/动态转发、bind address、多规则、CRUD/复制、状态/错误、流量；grid/list、排序/手动重排、wizard/完整表单偏好 | 本地/远程等价；动态与流量后置；移动可替换布局但保留排序、手动顺序和新建流程偏好结果 | v1.2/vNext | `domain/models/portForwarding.ts`; `application/state/usePortForwardingState.ts`; `components/PortForwardingNew.tsx` |
| P-02 | 跳板、自动启动、网络恢复重连、关闭清理、host key | 等价；校验 SSH 隧道主机/每跳，不校验普通 TCP 目标 | v1.2 | `application/state/usePortForwardingAutoStart.ts`; `electron/bridges/portForwardingBridge.cjs` |
| K-01 | 密钥生成 RSA/ECDSA/ED25519、导入、查看、删除、复制公钥、passphrase 策略、reference key；grid/list 与手动顺序；旧 biometric/FIDO2/passkey/WebAuthn 实验记录隔离 | 等价；系统安全存储；v1.0/v1.2 读取并保留实体顺序，vNext 再提供 grid/list 与触摸重排 UI；公钥复制不得夹带私钥/passphrase。旧实验记录保持不可执行归档，解释需重新录入，不得静默转成 SSH key；清空 Vault 前把归档列入删除影响 | v1.0/v1.2/vNext | `domain/models/connection.ts`; `application/state/useVaultState.ts`; `infrastructure/config/storageKeys.ts`; `components/KeychainManager.tsx`; `components/keychain/GenerateStandardPanel.tsx` |
| K-02 | 证书；Identity（用户名 + 密码/密钥/证书）的创建/查看/编辑/删除、label/username 搜索、空结果、grid/list 与手动顺序 | 等价，iOS/Android 证书支持待 gate；v1.2 提供 CRUD、搜索并保留已有顺序，vNext 再提供 grid/list 与用户重排；删除前显示并处理 Host/分组/代理引用，不能留下不可解释的失效认证 | v1.2/vNext | `components/KeychainManager.tsx`; `components/keychain/IdentityPanel.tsx`; `domain/models/connection.ts` |
| K-03 | 将公钥部署到远端 authorized_keys 并关联主机/Identity | 等价；与本地私钥导出分开 | vNext | `components/KeychainExportPanel.tsx` |
| S-01 | Snippet 分类、变量、目标、快捷键、多行/noAutoRun、导入导出/包、grid/list/排序/手动重排，以及 Compose Bar pinned snippets | 后置；触摸重排和移动快捷入口替代桌面交互 | vNext | `domain/models/connection.ts`; `components/SnippetsManager.tsx`; `application/state/useComposeBarPinnedSnippets.ts` |
| S-02 | 脚本创建/编辑/录制、目标与有序 host queue、manual/onConnect/onOutput、运行日志/进度、停止/暂停/恢复、`nct` screen/session/dialog/progress API；`language: python` 当前仅标签，执行器只跑 JS | 后置；移动沙箱、API/权限和脚本引擎待设计，不把 Python 标签误报为运行时 | vNext | `domain/scriptApiReference.ts`; `domain/hostConnectScripts.ts`; `components/scripts/ScriptAutomationRoot.tsx`; `application/state/scriptAutomationCoordinator.ts`; `application/state/scriptRunsStore.ts` |
| X-01 | 外观/语言/终端主题、iTerm2 导入、应用图标变体、设置搜索、清空数据、反馈 | App 明暗/系统主题、语言和反馈进入 v1.0；内建终端主题与基础终端外观按 T-05 进入 v1.1；自定义 UI/终端主题、iTerm2 导入、图标变体、设置搜索和清空数据后置 vNext。桌面 CSS/窗口外观不适用，移动 alternate icon 待平台 Gate | v1.0/v1.1/vNext | `components/SettingsPage.tsx`; `components/settings/tabs/SettingsAppearanceTab.tsx`; `domain/settingsSearchCatalog.ts` |
| X-02 | 凭据状态；更新检查/可用/下载进度/重启安装/手动回退/未保存守卫；crash/SSH/session/performance/interrupt 诊断；临时目录；会话恢复；Windows packaged app 检测 exe/portable launcher 旁 `data/` 并重定向 `userData`/`sessionData` | 平台替代或等价；v1.0 显示凭据保护状态。桌面当前 safeStorage 不可用或加密失败时字段 bridge 会原样透传并持久化明文、只显示可用性警告；移动端必须 fail closed，不能把该桌面降级当作安全存储等价。portable profile 目录依赖 Windows 可执行文件布局，在 iOS/Android sandbox 中不适用；跨端迁移须先通过第 13 节稳定交换格式与凭据可移植性 Gate，只有明确采用 v1.2 受保护备份 payload 变体时才进入该版本，否则保持 blocked/待决。复制桌面 profile 或设备绑定密文不构成可导入格式，凭据需重新输入或重新保护；另需区分商店二进制与 OTA，并定义诊断包脱敏/保留/同意/清理 | v1.0/v1.x/vNext | `infrastructure/services/updateService.ts`; `infrastructure/persistence/secureFieldAdapter.ts`; `electron/bridges/credentialBridge.cjs`; `electron/portableData.cjs`; `electron/main.cjs`; `application/app/AppSideEffects.tsx`; `application/state/useUpdateCheck.ts`; `application/app/useAppStartupEffects.ts`; `components/settings/tabs/SettingsSystemTab.tsx` |
| X-03 | WebDAV/S3/GDrive/OneDrive/GitHub/插件 provider 云同步、端到端加密、冲突/修订/强推；实验性 CRDT v2 migration/字段冲突/downgrade；payload 含 Vault 实体、端口转发持久态、跨设备设置、部分 AI 配置和非秘密插件 sidecar；本地同步实体全空与有数据云端的恢复决策、收缩和空库上传保护 | 后置，复用仅为候选；v1/v2 兼容和写后验证必须保留；`known_hosts` 仅本地备份，external agent command/args/env 与插件 secret 排除；设备绑定密文须转换后再由同步主密钥保护。“空”按 `CLOUD_SYNC_PAYLOAD_ENTITY_KEYS` 的全部实体而非仅 hosts 判定，默认 Vault snippets 会使正常新安装通常非空。桌面 `useAutoSync` 路径提供 Restore/Keep Empty 和空 payload guard，Keep Empty 不推进 anchor/base；但 Settings 的单 provider Sync 绕过该 guard，保留旧 anchor 且 remote 未变化时启动检查不复核本地丢失，清空本地状态未清 CRDT replica，多 provider 分叉只有事件/日志，legacy 同 ID 冲突会静默偏向 local。移动端必须把恢复、上传、清空、冲突和所有手动/自动入口收敛到同一 fail-closed preflight；只有单独确认的 Force Push 可擦除云端 | vNext | `application/syncPayload.ts`; `application/state/useAutoSync.ts`; `components/CloudSyncSettings.tsx`; `components/cloud-sync/CloudSyncDialogs.tsx`; `domain/sync.ts`; `domain/syncMerge.ts`; `infrastructure/config/defaultData.ts`; `infrastructure/services/cloudSync/syncAllStorageMethods.ts`; `domain/convergentSync/`; `domain/pluginSyncSidecar.ts` |
| X-04 | 系统管理器：概览、进程、监听端口、systemd 服务、tmux、Docker containers/images、NVIDIA/Ascend GPU；能力探测、刷新/搜索、危险操作及 tab show/collapse/hide/order | 后置；SSH exec 可复用仅为架构假设，不支持命令/设备类型显式降级；移动导航保留可见性和顺序结果 | vNext | `components/systemManager/`; `application/state/systemManagerTabLayout.ts`; `electron/bridges/systemManagerBridge.cjs` |
| X-05 | Catty AI provider/model、终端/会话 scope、附件/选区、历史/恢复/删除、停止/steer、activity/artifact/token/compaction、导出；当前 sidebar catalog 72 工具（session 2、attachment 2、terminal 4、SFTP 11、Vault 39、portforward 8、harness 6）；Observer/Confirm/Auto、Confirm permission grants、尚未接入执行的 `HostAIPermission`、web search、skills；外部 agents/MCP；AI/MCP 静默会话 | API 型 AI 逐项后置；72 工具按用户结果逐组处置，保留 scope、审批、取消、统一 stop 和截断输出 handle 契约。准确保留桌面事实：Observer 阻止非豁免写；`session_close`/`terminal_stop` 是清理豁免；Approve once 只覆盖当前调用，Always Allow grant 当前持久且全局匹配；Codex allow-for-session 不持久化；工具拒绝不等于 turn stop；Auto 不匹配 grant 而直接通过审批层；shell command blocklist 跳过 Serial/network-device CLI。桌面静默会话不进 tab/Quick Switcher/restore，可由托盘发现并以 popup attach 同一 live PTY，另有空闲自动关闭。移动不需复刻 BrowserWindow，但必须决定 supported/degraded/blocked，并定义创建者 scope、发现/观察/关闭、空闲终止和不泄露为普通恢复会话。移动若让 Auto 受 host policy 约束，标为安全增强。本地 CLI/SDK/app-server/ACP/stdio 与 discovery 不适用，远程 agent 需新契约 | vNext/- | `components/AIChatSidePanel.tsx`; `application/app/AppHandlers.ts`; `application/state/useSessionState.ts`; `domain/sessionRestore.ts`; `components/TrayPanel.tsx`; `components/settings/tabs/ai/ExternalMcpCard.tsx`; `infrastructure/ai/harness/cattyToolApproval.ts`; `infrastructure/ai/harness/permissionGrants.ts`; `infrastructure/ai/harness/agentStop.ts`; `infrastructure/ai/shared/toolExecutors.ts`; `infrastructure/ai/harness/generated/cattyToolSpecs.json`; `electron/capabilities/policy.cjs` |
| X-06 | 插件安装/启停/重启/卸载、权限/secret/配额/沙箱；settings/commands/keybindings/menus/views；terminal completion/decoration/link/hover/matcher/semantic/prompt/background/theme 与 input/output interceptor，以及 connection/auth/sync/importer providers；importer 检测、流式解析进度、安全预览与事务提交/崩溃回滚 | 插件各贡献面和每种 provider kind 逐项声明 supported/degraded/blocked 并后置/Gate；BrowserWindow/utilityProcess/companion executable 不适用；importer 即使后置也保留检测、取消、边界校验和原子落库结果 | vNext/- | `electron/plugins/`; `electron/plugins/terminalProviderService.cjs`; `packages/plugin-contract/schema/plugin-contract.schema.json`; `domain/pluginImporter.ts`; `application/state/pluginImporterTransaction.ts` |
| X-07 | 托盘、全局快捷键、多窗口、Explorer 菜单、窗口透明度/quake | 不适用；后台断开通知、下载系统分享、App Shortcuts 和 Widget 作为移动新增/替代项后置并分别过隐私/平台 Gate；既有协议深链由 X-08 单独处置 | -/vNext | `components/settings/tabs/SettingsSystemTab.tsx`; `electron/bridges/windowManager.cjs` |
| X-08 | `ssh://`、`telnet://`、`jms://` 深链；一次性 URL 凭据/JumpServer token；JMS SFTP 意图自动打开文件入口 | `ssh://` 在 v1.0 等价交付；telnet/JMS 随对应能力后置到 vNext，全部使用 ephemeral 凭据并禁止持久化/日志泄露 | v1.0/vNext | `electron/deepLink.cjs`; `domain/telnetDeepLink.ts`; `domain/jmsDeepLink.ts`; `application/app/AppSideEffects.tsx` |
| X-09 | 应用级 HTTP(S) 出站代理 system/direct/custom、custom URL 与 bypass，作用于云同步/AI/部分外部进程；区别于 SSH 主机代理 | 随首个出站 HTTP 能力等价或平台替代；认证、系统 VPN/PAC、TLS、bypass 与凭据存储待 Gate | vNext | `domain/httpNetworkProxy.ts`; `components/settings/tabs/SettingsSystemTab.tsx`; `electron/bridges/httpNetworkProxyBridge.cjs`; `electron/bridges/httpNetworkProxyAgent.cjs` |
| X-10 | 本地终端、Telnet 自动登录/IAC/字符集、Mosh SSH bootstrap/UDP 恢复/自定义 server path、ET 自定义端口、Serial baud/data/stop/parity/flow-control/local-echo/line-mode/backspace/charset、X11 | Telnet/Mosh/ET/Serial 逐项后置/Gate；明文风险、双端硬件/API、后台与协议兼容分别验收；桌面外部 `mosh-client`/ET binary、本地 shell 与 X11 机制不适用，不能用“其他协议”一笔删除 | vNext/- | `electron/bridges/telnetAutoLogin.cjs`; `electron/bridges/telnetProtocol.cjs`; `electron/bridges/terminalBridge/moshSession.cjs`; `components/SerialConnectModal.tsx`; `domain/models/connection.ts` |

### 3.1 设置入口反向审计

`domain/settingsSearchCatalog.ts` 是桌面设置搜索入口的稳定清单。当前逐项计数为 116，下面按源文件分组反查；它不是设置总数，实际设置页还有未索引子控件。每组都已落到上方矩阵和 PRD 第 10.1 节，且另以 `SettingRow`/嵌套配置表单反查未索引项；合并移动页面不等于删除设置结果。

| 源分组 | 数量 | 覆盖处置 |
|---|---:|---|
| Application | 5 | X-01/X-02：更新、反馈、社区、GitHub、What's New 等价或商店/外链替代 |
| Appearance | 14 | V-01/C-04/X-01/X-07：语言、字体、主题/强调色、图标和 6 个 Vault 偏好逐项保留/替代；窗口透明度、custom CSS 不适用 |
| Terminal theme/font/cursor/keyboard | 16 | T-01/T-02/T-05/T-06：平台有效设置等价；font smoothing/renderer 实现名降级；Kitty keyboard 与 IME 真机 Gate |
| Terminal behavior | 27 | C-03/C-06/T-03/T-05 至 T-12/X-04/X-06：行为、连接、server stats 和增强功能逐项等价/后置；未索引的 keepalive count、host info/System Manager 刷新周期、renderer 回收、inline image 限额、粗体亮色与补全子项也纳入；鼠标、本地 shell、X11、workspace focus UI 不适用或替代 |
| Shortcuts | 5 | T-02/T-03/T-05/X-07：keymap、自定义 binding、字体缩放和标签编号规则进入外接键盘设计；系统 App Shortcuts 另行处置 |
| File associations / SFTP | 9 | F-01/F-03/F-05/F-07/T-10：双击改触摸动作；视图、隐藏文件、自动回传、CWD、自动入口、并发与 opener 逐项处置；未索引的压缩、skip-unchanged 和 transport idle TTL 同样纳入 |
| AI | 18 | X-05：API 型能力后置，本地 CLI/SDK/stdio runtime 不适用，安全审批不可降级；未索引的 OpenCode/Grok、provider/web-search 细项、MCP lifecycle 和 max-iterations 同样纳入 |
| Sync | 5 | V-06/X-03：本地保护备份进入 v1.2；providers、auto-sync、三种策略和重置本地版本/历史后置到 vNext，均保留危险操作语义；未索引的 CRDT v2 migration/字段冲突/downgrade 同样纳入 vNext |
| System | 16 | T-03/T-09/X-02/X-07/X-08/X-09：更新、代理、凭据、临时文件、诊断、恢复、日志和深链等价/替代；桌面外壳项不适用 |
| Plugins | 1 | X-06：插件根入口与各贡献面一起后置/Gate |
| 搜索入口合计 | 116 | `rg -c '^\\s+id: "' domain/settingsSearchCatalog.ts` 可复现；这不是全部设置数，新增/删除入口或实际设置控件必须同步更新本表、矩阵和 PRD |

### 3.2 移动新增提议来源

以下不是桌面 parity 证据，必须作为移动产品提议单独决策，不能用来提高桌面能力覆盖率:

| ID | 提议 | 当前处置 |
|---|---|---|
| M-01 | App 锁、生物识别解锁、最近任务遮挡 | 移动新增/P0 候选；桌面只有设备绑定凭据加密和云同步 master-key 解锁，没有用户可操作的 App 锁。同步 `NO_KEY/LOCKED/UNLOCKED` 只约束同步操作，不锁 Vault、终端或整个应用；移动 App 锁的会话、转发、传输、通知与后台行为仍待安全决策 |
| M-02 | 主机健康探测、扫码配置 | 移动新增/P2；探测成本/误报与 QR schema/秘密字段分别验收 |
| M-03 | 后台断开通知、下载系统分享、App Shortcuts、Widget | 移动新增或桌面入口替代/P2；每项分别通过隐私、锁屏、URI 权限与平台 Gate |
| M-04 | 引导式 onboarding | 移动新增/待产品决策；桌面没有独立 onboarding，只显示空主机状态、首次播种 3 条 Vault snippets，并为 Compose Bar 首次写入默认 pin。移动若增加教程、示例主机或权限预热，必须单独定义跳过、重入、无障碍和真实数据边界，不能记作桌面 parity |

## 4. 已修正的高风险事实

1. 文件/目录传输 worker 默认并发是 2、范围 1-16；64 是单文件 SFTP READ/WRITE 请求 fanout，不是队列默认并发。
2. 桌面 SFTP 视图只有 `list/tree`；“grid”若保留只能标为移动端新增提议。
3. 桌面已有 `auto/sftp/scp` 与 SFTP 失败时 SCP fallback，不能只在标题里写“SFTP/SCP”。
4. 端口转发 host key 校验对象是用于建立隧道的 SSH 主机及跳板，不是任意 TCP `remoteHost`。
5. `KeychainExportPanel` 的核心行为是把公钥写到远端并关联凭据，不是导出本地私钥文件。
6. 桌面生成密钥包含 ECDSA；不能只列 RSA/ED25519。
7. `Host.notes` 与独立 `VaultNote` 是两种能力。
8. 桌面内置 SFTP 编辑器限制是 10 MB；4 GB+ 是移动传输引擎必须验证的目标，不是桌面已有编辑限制。
9. 桌面主机 CSV 会导出 Password 和可逆编码 Passphrase；它是兼容行为，不符合移动端默认安全底线，必须显式决策。
10. 桌面外部编辑 watcher 是 local temp -> remote 的单向上传，并没有远端版本或冲突检测。
11. 桌面的“粘贴剪贴板图片”是先通过 SFTP 上传再把远端路径写入终端，不是 Kitty/SIXEL/iTerm inline image 渲染。
12. 桌面 `verifyHostKeys=false` 会跨多个连接面关闭校验；移动端为满足 fail-closed 基线不应直接复制该全局开关。
13. 桌面的“复制账密”会解析分组继承、Identity 与 Telnet 专属字段后，把明文 `host/username/password` 写入系统剪贴板；无可读密码时不复制。它与复制 hostname、复制公钥是三种风险不同的能力。
14. 桌面连接日志可带完整终端 scrollback 回放，并非只保存连接结果；未收藏记录最多 500 条，最近 50 条未收藏回放另存，收藏记录的回放持续保留。
15. `ManagedSource` 独立存储，当前同步、本地保护备份和 Vault export payload 都不包含其 descriptor，但 Host 可保留 `managedSourceId`；跨端恢复必须显式修复该非对称关系。
16. 桌面自动重连不是指数退避：只有全局开关，曾成功连接的普通 SSH 意外断开后固定每 5 秒持续尝试；认证失败、changed host key 等不会由当前循环按错误类型自动停止。移动有限退避和安全停止分类是增强，不是桌面现状。
17. 桌面顶层标签可拖动重排，`tabOrder` 与活动标签进入 session restore；它不是纯 UI 临时顺序。
18. 桌面应用级 Clear Buffer 仅在 xterm normal screen 清当前视图，并按 `clearWipesScrollback` 决定是否擦 scrollback；后端 `clearPty` 只为本地 Windows ConPTY 同步光标，对 SSH/Unix PTY no-op。移动端可新增“清理本地可重放缓存、重挂载不回放”的契约，但不能把它写成桌面已有的通用 PTY replay-buffer 行为。
19. 桌面“退出后自动关闭”只关闭确认的 `reason=exited, exitCode=0`；非零/未知退出、timeout、transport error 和 channel close 会保留标签与输出。
20. 恢复/重启的专用 SFTP 传输明确禁用 transport reuse，可能重新认证；普通浏览和共享传输仍可按安全 endpoint 复用，不能概括为“所有传输独立”或“所有传输免 MFA”。
21. 桌面终端搜索没有大小写/正则/整词开关；实际选项固定为 case-insensitive、literal substring、non-whole-word，支持前后导航和关闭后回焦。
22. `drawBoldInBrightColors` 当前没有设置 UI，但默认 true、实际改变 xterm ANSI 渲染且进入同步 payload；它仍是兼容字段，不能因入口不可见就从移动迁移中静默丢弃。
23. Catty Auto 当前不请求审批或匹配 permission grant；grant 只会免除 Confirm 的重复审批。普通 Catty/MCP 的 Always Allow grant 当前持久化并按 capability/command/args 全局匹配，matcher 忽略 `sessionPattern`；只有 Codex App Server 的原生决策支持不写入 Netcatty grant 的 session scope。`HostAIPermission` 虽有类型、存储和同步，但没有进入 Catty turn 执行路径，不能声称桌面 Auto 已受 host policy 限制。
24. 桌面 dirty editor 保护因关闭路径而异：手动 SFTP tab close/disconnect 有 save/discard/cancel，App quit 只阻止退出，关闭承载 SFTP panel 的终端则随 owner 卸载强制丢弃。移动端统一覆盖 owner/session/workspace 关闭是数据保护增强，不是已有桌面全路径等价行为。
25. 审批拒绝/超时只结束当前工具调用，`session_close`/`terminal_stop` 也只是清理工具；停止按钮和 `/stop` 才进入统一 `stopAgentTurn()`。两种清理工具显式绕过 Observer、Confirm 审批和 chat-cancel 阻断，不能被笼统的“Observer 禁止所有写”挡住。
26. command blocklist 只适用于 shell-like execute/start；桌面按协议/device metadata 对 Serial 和 SSH network-device CLI 跳过，不能把 shell 正则描述成所有终端协议的统一安全边界。
27. 非当前终端收到经过过滤的可见输出或 BEL 时，桌面以 session activity dot 标记；workspace 聚合子会话活动，切回对应 session/workspace 后清除。它是标签内活动状态，不等同于系统通知；移动若增加通知必须另行取得权限并脱敏。
28. 桌面凭据 bridge 在 safeStorage 不可用或 `encryptString` 失败时当前会原样返回并可能持久化明文，同时启动时只给出可用性警告；移动端的“不可用即 fail closed”是安全增强，必须阻断保存/同步而不是声称桌面行为已等价。
29. 桌面云同步的恢复判定不是“没有主机”，而是 `CLOUD_SYNC_PAYLOAD_ENTITY_KEYS` 中所有同步实体均为空；由于首次启动会播种 3 条 Vault snippets，正常新安装通常不会进入这条恢复分支。`useAutoSync` 在真正全空、检测到变化的有数据云端时显示不可关闭的 Restore/Keep Empty 决策；Keep Empty 不推进 anchor/base，后续启动可能再次询问。
30. 桌面终端右键设置提供 `select-word`，但实际 context callback 调用 `term.selectAll()`，会选择整个缓冲；`wordSeparator` 传给 xterm 并不能修正这条动作链。移动端若承诺选词，必须以真实 word-boundary fixture 验收，不能把这一桌面缺陷写成已对齐行为。
31. `Host.disableDynamicTabTitle` 当前只在模型、plugin schema 和 importer 中声明，桌面运行时标题更新只消费全局 `dynamicTabTitleMode`。移动迁移须保留未知/兼容字段，但主机级禁用若实现属于新增行为，不能作为桌面现状举证。
32. 桌面启动会把旧 biometric/FIDO2/passkey/WebAuthn 实验 key 从可用 Keychain 列表隔离到 `STORAGE_KEY_LEGACY_KEYS`，不会把它们转换为普通 SSH key；清空 Vault 会连该归档一起删除。移动迁移必须预览、解释和要求重新录入，不能静默转换或丢弃。
33. 云同步 master-key 的 `NO_KEY/LOCKED/UNLOCKED` 状态只控制同步与自动同步，不是桌面 App 锁，也不会锁住本地 Vault 或活动终端。移动生物识别/App 锁不能直接复用这些名称或语义。
34. 桌面的空库/收缩保护不是统一边界：Settings 的单 provider Sync 直接调用 `syncToProvider`，绕过 `useAutoSync` 的空 payload guard；保留旧 remote anchor 且远端未变化时，启动检查会直接认为一致而不核对本地丢失；默认 snippets 还可能掩盖其他实体丢失。移动端必须让所有自动、手动和恢复入口共用同一 preflight，不能把桌面注释里的“both auto and manual”当作已实现事实。
35. 桌面“清空本地数据并重置同步”不创建保护备份或 apply barrier，`resetLocalVersion()` 清 base/anchor 却保留 canonical CRDT replica，因此“下次一定从云端下载”并不成立。多 provider 分叉目前只有事件/诊断，设置页不展示；legacy smart merge 对双方新增/修改的同 ID 冲突偏向 local，只计数而不提供逐项审查。移动同步设计必须显式修正这些缺陷，而不是照搬其成功提示。
36. 桌面目录传输确实边发现边传，但 parent `totalBytes`/全局进度是已发现文件数上的 soft progress，没有显式 discovery-complete/final-total freeze；移动端要求分开显示已发现量、已传输量并在发现完成前保持不确定百分比/ETA，是增强契约而非桌面现状。

## 4.1 上游同步增量审计

本轮将 `upstream/main@c4edefdb` 合并到移动分支。相对上一移动文档基线，新增或变化的桌面事实已重新登记：

| 增量 | 移动端处置 |
|---|---|
| `es` 成为正式 UI locale | 更新语言矩阵；移动缺失文案回退英文，不显示 key |
| OSC 9/777/99 桌面通知 | 新增 T-15；通知与 activity dot 分离，移动权限、锁屏和后台行为另过 Gate |
| SFTP 归档/单文件压缩解压 | 新增 F-10；长按入口、已知格式、临时文件、原子替换和取消必须可观察 |
| PuTTY 风格命令行参数 | 更新 V-08/X-08 的输入 fixture；保留忽略项警告和 ephemeral 凭据边界 |
| 动态 group path、snippet/script targets、原子组变更 | V-02/S-01/S-02 需要重命名迁移、删除收缩和失败回滚 |
| 终端搜索/选择/背压、SSH 复用、SFTP 重连安全修正 | Phase 0 fixture 以当前上游行为为准，不复刻已修复桌面缺陷 |

## 5. 双向审计规则

### 代码到文档

每次桌面能力变更必须更新至少一个矩阵行，并在 PRD 中给出版本和处置。新增 UI 入口不能只按组件名检查，还要覆盖空、加载、错误、取消、权限拒绝、离线、后台、恢复、冲突和删除状态。

### 文档到代码

PRD 中每项必须标记来源之一:

- `桌面对齐`: 有本地代码证据；
- `移动替代`: 桌面目标相同但交互不同；
- `移动新增`: 产品提议，不伪装成桌面事实；
- `待验证`: 依赖外部资料或真机 PoC。

没有来源或可观察验收结果的条目不能进入已承诺版本。

## 6. 证据缺口与准入 gate

本地仓库无法证明以下断言，完成 PoC/法务/产品决策前保持“待验证”:

1. React Native SSH fork 的活跃度、host-key/转发/证书 API 现状及固定版本。
2. Android mwiede/JSch 与 iOS NMSSH/libssh2 对 RSA/ECDSA/ED25519 用户证书、每跳 host key 和取消语义的实际支持。
3. SwiftTerm 与 Android terminal-emulator 对 Kitty keyboard、SIXEL、Kitty graphics、iTerm IIP、IME 和无障碍的覆盖。
4. 上游组件许可证、notice、商用分发、Expo/商店政策兼容性。
5. 4 GB+ 流式传输、后台恢复、切网、内存上限、冷启动、包体和帧率目标。
6. iOS 本地网络权限、后台时限、签名和商店审核路径。
7. 桌面与移动的稳定交换格式版本、凭据可移植性、known_hosts 策略、冲突合并和回滚。

## 7. 维持对齐

- 审计基线必须记录 commit；PRD 或桌面能力变更后重新跑双向审计。
- 数量只能由脚本生成并附口径，不再用“20+”“500+”支撑完整性。
- “无已知遗漏”仅表示本轮矩阵没有未处置项；证据缺口继续保留，不能改写成“已确认可行”。
