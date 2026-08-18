# Netcatty Mobile 技术可行性草案

> 状态: 调研阶段，移动端尚未开始开发
> 基线: `mobile@121ba2cb`
> 产品范围: 见 `PRD.md`
> 桌面能力证据与处置: 见 `ALIGNMENT.md`
> AI 串行执行任务与提示词: 见 `AI_TASK_PROMPTS.md`

## 1. 文档目的

本文用于在开发前比较移动端架构候选、定义 Phase 0 验证和记录尚未解决的技术风险。它不是实现说明，也不证明任何第三方组件已经满足 Netcatty 的安全、协议、性能、许可或商店要求。

“对齐所有功能”是能力处置目标，不等于首版同时交付全部桌面能力。版本范围以 PRD 为准；本文件只回答如何验证可行性以及哪些桌面资产可能复用。

桌面没有可直接移植的 onboarding 流程：它只有空主机状态、首次播种的 3 条 Vault snippets 和 Compose Bar 首次 pin seed。移动端若设计教程、示例数据或权限预热，应作为移动新增能力单独验证，不能记作桌面复用或 parity。

## 2. 可由仓库确认的桌面边界

当前桌面应用是 Electron + React + TypeScript。核心外部边界包括:

| 能力域 | 桌面实现 | 移动端影响 |
|---|---|---|
| SSH、exec、SFTP/SCP | `ssh2`、`ssh2-sftp-client` 与 Electron bridge | 不能假定可直接进入 RN/移动 WebView，需替换或桥接 |
| 交互终端 | `node-pty`/远端 SSH channel + xterm.js | PTY、传输与渲染生命周期需重新设计 |
| 传输中心 | Electron 主进程/utility process、文件系统与持久化状态 | 状态机可参考；I/O、后台和文件权限需重写 |
| 端口转发 | `node:net` + SSH channel | listener、切网、后台与资源清理必须原生验证 |
| 凭据与备份 | Electron `safeStorage`、文件 bridge | 改用平台安全存储；桌面密文不可假定可移植 |
| Profile 与 Windows portable data | packaged app 可把 `userData`/`sessionData` 重定向到 exe/launcher 旁 `data/` | 移动 sandbox 不适用；跨端迁移依赖待决的交换格式与凭据可移植性 Gate；若采用受保护备份变体才可按相应版本迁移，设备绑定凭据必须重新保护或重新输入 |
| Vault 与设置 | React hooks、domain helper、localStorage adapter | 数据模型/纯逻辑可能复用；持久化和 UI 不可原样复用 |
| 云同步 | renderer/domain 编排 + 主进程 OAuth/HTTP bridge | payload/合并逻辑可能复用；实体、规则、设置、AI 和插件 sidecar 需逐字段兼容，OAuth、凭据与网络边界需适配 |
| AI、脚本与插件 | Web UI、Node/Electron runtime、本地 CLI/子进程、实验性插件 contract/host | API 型 AI 可重做；会话/工具/审批契约需逐项复核，本地 CLI、stdio、BrowserWindow/utilityProcess 和 companion executable 不能直接复用 |

桌面的系统 SSH agent 不是“导入一把私钥”的别名。`useSshAgent`/`IdentityAgent` 选择登录 agent，`IdentitiesOnly` 限定候选身份，`AddKeysToAgent` 与 macOS `UseKeychain` 描述本机 key 生命周期；agent forwarding 则是在认证后把 agent 能力暴露给远端。移动 adapter 必须分别声明 supported/degraded/blocked，且不能假定 iOS/Android 存在 OpenSSH socket。若以应用内 agent 替代，应给出 key selector、用户授权、后台存活、远端转发通道与清理的独立 threat model。桌面还会把旧 biometric/FIDO2/passkey/WebAuthn 实验 key 隔离到不可执行归档，而不是迁移成普通 SSH key；移动 importer 要保留可解释的归档/重新录入路径，并在清空 Vault 前显示它也会被删除。

桌面的 ambient 文件 key 又是另一条认证来源：它按固定顺序扫描 `~/.ssh/id_*`，初次只尝试有效且未加密的 key，合资格失败后才收集加密 key 的 passphrase；password-only、显式 key/certificate 与 `IdentitiesOnly` 会排除无关候选。移动平台通常没有可静默遍历的 home 目录，因此 adapter 必须把候选来源、顺序和排除规则建模为协议状态；v1.0 只使用已导入 Keychain 或显式授权文件，未来目录发现也需受 document URI 权限和同一 request/session/boot 生命周期约束。

当前 Catty sidebar 的生成目录有 72 个工具：session 2、attachment 2、terminal 4、SFTP 11、Vault 39、portforward 8、harness 6。移动端可重写执行 adapter，但不能只复用聊天 UI；scope、observer/confirm/auto、取消、统一 `stopAgentTurn()`、ToolOutputStore 截断 handle、重复读取提示及 chat 删除清理属于共享行为契约。审批语义必须准确迁移：Approve once 只覆盖当前工具调用；普通 Catty/MCP 的 Always Allow grant 当前持久化并跨 chat/session 匹配，Codex App Server 的 allow-for-session 则不写入 Netcatty grant；Auto 当前直接通过审批层。审批拒绝/超时只结束当前工具调用，只有停止按钮、`/stop` 等 turn-stop 入口调用 `stopAgentTurn()`；`session_close`/`terminal_stop` 是可绕过 Observer、审批和 chat-cancel 阻断的清理能力。shell command blocklist 仅用于适用的 execute/start，会按协议/device metadata 跳过 Serial 和 SSH network-device CLI。另有 `HostAIPermission` 类型和持久态但尚未接入 Catty 执行，若移动端让 Auto 受 host policy 约束，应作为新的安全增强设计。内部 CLI/RPC/发现文件仍按仓库边界视为 Netcatty 内部集成面，不把它们扩展成移动公共 API 要求。

插件 contract 的 terminal provider 不是单一“装饰”接口：运行时有 completion、decoration、link、hover、matcher、semantic、prompt、background、theme，contract 另含 input/output interceptor，并与 connection/authentication/sync/importer provider 分开。移动插件研究必须逐 kind 给出 supported/degraded/blocked 和权限边界，不能以“插件后置”代替能力处置。

`domain/` 中没有平台副作用的模型与纯函数是最可信的复用候选。React hooks 只有在剥离 `window.netcatty`、DOM、localStorage、Electron IPC 和桌面生命周期依赖后才可复用。Radix/Tailwind 组件、Electron bridges 和 Node runtime 代码属于概念或行为参考，不属于可直接复制的移动源码。

## 3. 必须保持的架构边界

无论选择 RN、Capacitor 还是其他壳层，都应保持以下责任划分:

1. Domain: 主机、身份、known-hosts、会话、传输与转发的纯模型和状态转换。
2. Application: 用例编排、取消、错误归类、恢复和持久化边界。
3. Platform adapters: SSH、终端、文件、网络 listener、安全存储、后台任务、通知和系统分享。
4. UI: 移动导航、触摸、软键盘、无障碍和系统权限状态。

桌面的终端工具侧栏是独立产品能力，不等同于 workspace 分屏：它承载 SFTP、Scripts、History、Theme、System、Notes、AI，并支持排序/隐藏和侧栏内多 pane。移动 UI 可重做为单工具页或平板并排，但 domain/application 必须保留每终端的工具选择、作用域和关闭/切换语义，不能因不复用桌面 pane UI 而丢掉工具入口。

原生桥必须使用稳定的有类型协议，至少包含 request ID、session/connection ID、事件序号、取消确认、终态、错误阶段和资源所有者。App 前后台、JS reload、原生模块重启、切网、会话关闭与 App 退出都必须有幂等清理；不能让 terminal、SFTP 和 port-forwarding 各自建立互不知情的 SSH 生命周期。

连接复用属于共享安全契约而不只是性能优化。复制会话或打开新 shell、SFTP channel、端口转发时，只有 host/profile、认证、跳板/代理、keepalive、SUDO 和 agent-forwarding policy 满足规定的等价关系才可借用已认证 transport；shell 的 agent forwarding 必须精确匹配。无法证明一致、transport 不健康或协议为 Mosh/ET/Telnet/Serial/local 时新建连接，不能为减少 MFA 次数跨安全上下文复用。普通 SFTP 浏览和共享传输可以持有 endpoint lease；重启恢复或显式专用传输必须能请求独立连接，不附着 terminal/parked transport，允许重新认证/MFA，并在关闭交互终端后继续由任务自身管理生命周期。

会话状态模型必须把显示状态与连接/任务所有权分开。顶层标签顺序、活动标签和自定义名属于可恢复元数据；activity dot 是瞬时标签状态，不属于 restore。非当前会话收到经过控制序列过滤的可见输出或 BEL 时，桌面会在标签上显示 activity dot；workspace 聚合子会话活动，进入对应会话/workspace 后清除。移动端 v1.0 必须保留每会话的标签内活动结果；系统通知是独立的移动新增能力，不能用通知替代活动点，也不能把活动点误判为连接状态。桌面 Clear Buffer 只保证 normal-screen xterm 视图清理和可选 scrollback 擦除，后端同步仅适用于本地 Windows ConPTY；移动端应另行保证自身渲染缓存清理后不会在重挂载时回放旧输出。启用自动关闭也只在明确的零码正常退出后删除会话，异常、超时或未知关闭保留诊断现场。自动重连的移动策略另建状态机并记录停止原因，不能直接把桌面固定 5 秒无限循环描述成有限指数退避。

会话日志也不是三个扩展名共享一个序列化器：raw 写入未渲染终端流，txt/html 通过有状态 cursor/erase/ANSI renderer 生成快照，alternate-screen TUI paint 被省略以保留 shell 历史。移动日志 adapter 必须分别定义编码、时间戳、ANSI 和保真边界，并以同一 corpus 做 contract test。

编辑器所有权必须进入同一关闭协调器。桌面手动关闭 SFTP tab 或断开连接会确认 dirty editor，App quit 会阻止退出，但关闭承载 SFTP panel 的终端会随 owner 卸载直接丢弃；移动端在关闭 editor、SFTP、owning terminal/session/workspace、安装重启或退出前统一执行 save/discard/cancel，并让 Cancel 阻止原破坏性动作。这是移动数据保护增强，不应写成桌面所有关闭路径已经等价。

安全规则与桌面一致地集中处理: 认证解析、每跳 host-key 校验、known-hosts 决策、代理/跳板链、凭据读取和日志脱敏必须是共享服务，不能由每个界面各自实现。桌面当前 safeStorage 不可用或加密失败时，credential bridge 会透传并可能持久化明文，同时启动时仅显示可用性警告；移动平台 adapter 必须把不可用视为硬失败，阻断保存、迁移和同步，不得把桌面降级路径当作可复用安全实现。

剪贴板 adapter 也必须区分公开地址、公钥和明文账密。账密复制使用与连接相同的分组/Identity/Telnet 解析结果，并在写入前完成近期设备认证与前台/解锁检查；过期清理只能作为平台能力，不能声称能撤回系统历史、云同步或其他 App 已读取的副本。

## 4. 架构候选

### 4.1 React Native + 原生网络/终端

候选结构是 RN UI，通过原生模块连接 iOS/Android SSH 实现和原生终端视图。

预期优势:

- 可按移动端方式处理导航、触摸、软键盘、无障碍、系统文件和后台状态。
- 网络、终端与平台生命周期可放在原生层，避免大数据持续穿过 JS/UI bridge。

主要风险:

- UI 需要重做，现有 DOM 组件不能直接使用。
- 双端 SSH、SFTP/SCP、转发、终端和安全存储的能力可能不对称。
- TurboModule、Expo development build/config plugin 或纯 RN 工程形态都需用最小原型验证，不能提前锁定。

### 4.2 Capacitor/WebView + 原生网络插件

候选结构是 WebView 承载经移动适配的 React/xterm UI，原生插件承载 SSH、文件、转发和安全存储。

预期优势:

- 可以较早验证现有信息架构与部分 Web UI 的适配成本。
- xterm.js 行为更接近桌面，部分 renderer 代码可能复用。

主要风险:

- “现有 UI 原样复用”不成立：桌面窗口、多窗格、悬停/右键、文件路径和 Electron bridge 都需改造。
- IME、外接键盘、选择/复制、大输出、后台恢复和 bridge 吞吐必须真机量化。
- 即使只作为 PoC，也应先定义要回答的问题和停止条件；不预设两周工期或完成后必然废弃。

### 4.3 远程网关/代理

SSH 经受控网关中转可以降低部分客户端协议实现负担，但会改变本地直连、安全边界、离线能力、隐私和运维成本。除非产品明确接受这些变化，否则只作为受限网络场景的独立候选，不作为默认路线。

### 4.4 候选组件

以下名称只用于组织 PoC，不表示已选型或已验证:

| 层 | iOS 候选 | Android 候选 | 必验问题 |
|---|---|---|---|
| SSH | NMSSH/libssh2 或其他维护中的实现 | mwiede/JSch 或其他维护中的实现 | 算法、证书、kbd-interactive/MFA、host key、代理、跳板、取消 |
| 终端 | SwiftTerm 或其他原生组件 | terminal-emulator 系组件或其他原生组件 | VT corpus、Unicode/IME、键盘、搜索、选择、图片协议、无障碍 |
| Web 终端对照 | xterm.js in WebView | xterm.js in WebView | 输出吞吐、输入延迟、内存、旋转、后台恢复 |
| 安全存储 | Keychain-backed adapter | Keystore-backed adapter | 设备锁定、失效、备份、迁移、不可用时 fail closed |
| 文件 | document picker/provider adapter | Storage Access Framework adapter | URI 权限、目录、4 GB+、重启恢复、分享回传 |

组件版本、活跃度、API 覆盖、许可证和商店兼容性必须在评审时记录上游 commit/tag、测试代码和 notice 结果。仅凭 README、产品案例或许可证名称不能关闭 Gate。

## 5. Phase 0 验证

### 5.1 原型范围

同时建立 Android 和 iOS 的最小垂直切片；平台可以分先后编码，但在共享协议和 v1.x 路线锁定前，两端都要取得证据。每端至少打通:

1. SSH connect -> host-key decision -> password/key/passphrase/kbd-interactive -> shell -> resize -> disconnect；并发连接分别验证 request/session/boot 归属、队列切换、迟到响应、断连/超时清理、错误已保存 passphrase 清除、skip-key 与 cancel-flow，以及 OTP/二次因子不被预填或保存为登录密码。
2. 一条多级跳板/代理链，逐跳错误和取消可定位。
3. 终端软键盘、IME、外接键盘、alternate screen、选择/复制/直接粘贴选区、回滚与持续输出；搜索以桌面固定的 case-insensitive/literal/non-whole-word fixture 验证前后导航、清空高亮和关闭后回焦，新增高级选项另行标记；另以统一 fixture 验证真实词边界选择、触控字号增减/重置、普通 URL/OSC 8/plugin 链接的 `http(s)` 安全门和 OSC 8 确认，不继承桌面 `select-word -> selectAll` 缺陷。大输出切片还需证明实际消费 ACK、有界队列、高/低水位滞回、pause/drain、休眠/重挂载/关闭无丢重乱序，以及 backlog 下 Ctrl+C 紧急输入不被渲染阻塞。
4. SFTP 浏览和流式上传/下载；SCP fallback 作为独立能力验证。桌面普通目录是单遍增量发现并在已发现文件数上显示 soft progress，没有显式最终总量事件；移动端以增强契约验证分别报告已发现量与已传输量，不把发现未完成阶段显示为稳定全目录百分比/ETA，完成发现后才冻结最终总量；压缩上传另行验证全量扫描/压缩/上传/解压阶段及回退。分别量化文件/目录 worker、递归目录 listing gate 和单文件请求 fanout，不能用一个“并发”数字替代三种资源边界。
5. 本地/远程转发的启动、连接、停止、切网和异常清理。
6. 安全存储写入/读取/删除、设备锁定、App 重装/恢复边界。
7. 文件选择、目录权限、App 被杀后的任务记录和可恢复范围。
8. 两个独立远端之间的传输，来源/目标分别完成认证、跳板和 host-key 校验；验证 copy/cut、部分失败与删源失败。
9. 系统文件编辑回传：远端版本同时变化、URI 权限撤销、App 被杀与临时副本清理；据此选择校验、确认或禁用自动回传。
10. 系统 SSH agent/应用内 agent 探针：分别验证登录 selector、`IdentitiesOnly`、agent forwarding、权限撤销和后台清理；平台不支持时产出明确 blocked 结果，而非以私钥导入代替。
11. Quick Connect 与连接复用：用 IPv4/IPv6、有限 `ssh` 参数和忽略项 fixture 验证解析/警告/ephemeral 保存边界；v1.1 只执行 SSH 的直接 password/key，certificate 与已保存 Identity 随 v1.2 Keychain/Identity 能力交付，Mosh/ET/Telnet 字段保留且显示 unsupported；复制 shell、SFTP 和转发分别验证同 endpoint 免重复 MFA、策略变化后拒绝复用、失败回退和资源 lease 清理。
12. SFTP 符号链接与目录恢复：文件链接、目录链接、断链、循环、深链和超大目录 fixture 覆盖 SFTP/SCP；分别验证导航、下载展开、上传/远端复制策略、取消以及重启恢复的确定性与上限。
13. 剪贴板安全：分别验证 hostname、公钥和解析分组/Identity/Telnet 后的账密复制；覆盖无密码、不可解密占位符、认证取消、App 切后台/锁定、写入拒绝、平台过期/清除能力以及无法撤回系统历史的提示。
14. 会话与终端生命周期：重排后冷启动保持顶层 `tabOrder` 和活动标签；非当前会话的可见输出/BEL 置活动指示，切入后清除、不写入 restore 且不误发系统通知；移动 v1.0 不实现 workspace，未来交付标签分组时再验证其活动聚合；区分桌面 normal-screen xterm 清理/ConPTY 光标同步与移动新增的“清除后重挂载不回放本地缓存”契约；只有确认的零码正常退出自动关闭，非零/timeout/error/unknown 保留；移动自动重连覆盖有限退避、错误分类停止和用户停止。
15. TUI 输入冲突：alternate-screen/mouse tracking 下验证触控手势送达远端，同时确保重连、Clear Buffer、关闭和工具入口始终可达且不会重复触发。
16. 专用传输边界：恢复/重启任务不得借用 terminal/parked transport，可重新认证/MFA且不因关闭终端中断；普通浏览/共享任务仍按 endpoint policy 复用并正确释放 lease。

### 5.2 对照实验

至少比较“RN + 原生终端”和“WebView + xterm.js”两个终端切片。使用同一 SSH server、同一输出 corpus、同一低/中/高档设备与 release build，记录:

- input-to-echo P50/P95、掉帧、峰值内存和 CPU；
- 中日韩 IME、组合字符、emoji、宽字符、外接键盘和修饰键正确率；
- 10 MB ANSI 输出、搜索、选择、复制、旋转和前后台恢复；
- bridge 事件批处理、背压、乱序、重复、取消和 JS reload 后清理。

结果决定终端形态；在结果前不写“原生必然更快”或“WebView 与桌面完全一致”。

### 5.3 出口标准

Phase 0 结束时必须产出:

- 两端可复现的原型、测试设备/OS/构建信息、fixture 与原始测量结果；
- SSH/终端/文件/转发/后台能力矩阵，明确 supported/degraded/blocked；
- 主机导入/导出交换格式决策，特别记录 CSV Password/Passphrase 的兼容与安全取舍；
- 每个依赖的固定版本、许可证、notice 和维护风险记录；
- 原生桥协议草案及连接、通道、任务、listener 的所有权/清理状态图；
- Quick Connect 输入契约与 SSH transport 复用等价关系、拒绝原因和审计 fixture；
- RN、Capacitor 或其他路线的决策记录，包括未选方案的证据和回退条件；
- 对 PRD v1.0/v1.1/v1.2 范围、性能阈值与排期的修订。

安全关键项未通过时不得以 UI mock 或单平台结果放行。

## 6. 复用策略

| 资产 | 预期复用方式 | Phase 0 验证 |
|---|---|---|
| domain models/pure helpers | 建立平台无关 package，物理复用候选 | 依赖扫描、TypeScript/运行时兼容、共享 fixtures |
| hooks/application 编排 | 按 use case 拆出纯状态机后选择性复用 | DOM/Electron/localStorage 依赖清单、取消与恢复测试 |
| React UI | 行为、文案和信息架构参考；WebView 路线才可能局部物理复用 | 触摸、导航、安全区、字体缩放和无障碍审计 |
| Electron/Node bridges | 契约、错误模型和 fixtures 参考 | 与移动 adapter 做 contract tests，不复制平台代码 |
| xterm.js | WebView 对照候选 | release 真机基准和协议 corpus |
| sync payload/domain | schema 与合并规则候选 | Vault 实体、端口转发、跨设备设置、AI 配置和插件 sidecar 的字段兼容；v1 snapshot 与实验性 CRDT v2 migration/conflict/downgrade；凭据、known_hosts、OAuth、冲突和回滚测试 |
| AI SDK / HTTP clients | API/协议概念候选 | RN runtime、streaming、TLS/proxy、密钥和后台测试 |
| Catty capability catalog | 复用 capability ID、policy、scope 与事件契约，平台 I/O 重新实现 | 72 项生成目录逐域 contract tests；审批、observer、取消、handle、历史恢复与 chat 清理 |
| i18n 文案 | key 与翻译参考 | 移动导航、短文本、复数和无障碍上下文复核 |

桌面与移动若共享代码，应通过独立 package 和 contract test 共享，不从移动工程深度引用 Electron 目录。共享 schema 必须版本化，迁移要可回滚。

同步复用必须保留桌面的包含/排除语义：云 payload 包含 Vault 实体、端口转发持久态、跨设备设置、部分 AI 配置和非秘密插件 sidecar；`known_hosts` 只进入本地备份，external CLI agent 的 command/args/env 与插件 secret settings 不进入云 payload。“本地 Vault 为空”的恢复判断实际要求 `CLOUD_SYNC_PAYLOAD_ENTITY_KEYS` 中所有同步实体为空，不是只看 hosts；桌面首次启动会播种 3 条 Vault snippets，所以正常新安装通常不会触发该分支。桌面 `useAutoSync` 对真正全空且检测到变化的云端提供 Restore/Keep Empty，Keep Empty 不推进 anchor/base，但这不是统一安全边界：Settings 单 provider Sync 绕过空 payload guard，unchanged remote anchor 不复核本地丢失，清空本地同步状态保留 canonical CRDT replica，多 provider 分叉和 legacy 同 ID 冲突也缺少完整 UI 决策。移动端必须让自动/手动同步、启动恢复、清空和 v1/v2 共用 fail-closed preflight、保护备份与事务边界；普通空库上传拒绝，只有独立确认的 Force Push 可擦除云端。同步 master-key 的 `NO_KEY/LOCKED/UNLOCKED` 只约束同步，不是 App 锁，也不锁本地 Vault 或终端。`ManagedSource` descriptor 也不在当前云、保护备份或 Vault export payload 中，但 Host 可能携带 `managedSourceId`；移动 restore adapter 必须把它识别为需重新授权/去除的悬空平台引用，不能使用桌面 `filePath` 或假称托管仍有效。设备绑定密文不能原样迁移；AI API key 等秘密只有在本机解密成功并由同步主密钥重新保护后才可携带，否则同步应失败并要求修复或重新输入。OAuth/provider connection secret、App 锁材料等未在 payload 中定义的状态不得被假定为已备份。实验性 CRDT v2 还要求 migration preview、字段级冲突、provider 写后验证、v1 compatibility snapshot 和可验证 downgrade；未实现 v2 的移动客户端不得静默改写其 metadata。

## 7. 阶段建议

| 阶段 | 范围 | 出口 |
|---|---|---|
| Phase 0 | 双端垂直切片、终端对照、依赖/许可/商店验证 | 路线 ADR、桥协议、能力矩阵、修订后的计划 |
| Phase 1 / v1.0 | Vault、认证与信任、SSH 终端与 `ssh://` 深链、移动键盘、多会话与后台输出/BEL 活动指示、基础设置；App 锁仍需产品决策 | PRD v1.0 验收与安全测试通过 |
| Phase 2 / v1.1 | SSH Quick Connect（直接 password/key）、SFTP/SCP、传输中心、文件编辑、终端增强 | 文件与传输 Gate 通过 |
| Phase 3 / v1.2 | 跳板/代理、Identity/证书、密钥生成、端口转发、本地备份 | v1.x GA 验收通过 |
| Phase 4 / vNext | Notes、snippets、sync、AI、系统管理器、脚本、插件研究、其他协议 | 每个能力独立设计和 Gate |

Android 可以作为首个日常开发平台，但 iOS Phase 0 不能后置到核心架构完成之后。上架、TestFlight/Play 分发、国内商店和侧载属于待产品决策，不在技术草案中预先承诺。

## 8. 主要风险与未决项

1. 原生 SSH 库可能无法同时覆盖证书、MFA、多级跳板、每跳 host key、SCP 与转发。
2. 两端终端组件的协议、IME、无障碍和渲染能力可能不同，导致共享 UI 契约分叉。
3. 移动后台限制会影响 SSH、传输和 listener；状态必须如实恢复，不能承诺桌面式常驻。
4. 系统文件 URI 与平台安全存储会改变桌面备份、导入、reference key 和外部编辑语义。
5. 云同步、AI、脚本和插件不是“纯 JS 即可复用”；运行时、网络、安全和商店政策均需验证。
6. 产品仍需决定首发平台/最低 OS、交换格式、App 锁策略、私钥导出、性能阈值和分发渠道。
7. Telnet/Mosh/ET/Serial 不能共享一个模糊的“其他协议”结论：Telnet 自动登录/IAC/字符集和明文风险、Mosh SSH bootstrap/UDP 恢复/自定义 server path、ET 自定义端口与外部 binary 替代、Serial 参数矩阵及双端硬件权限都需要独立 Gate。

在 Phase 0 证据和产品决策齐备前，RN、Expo、具体 SSH/终端组件及时间估算都保持候选状态。
