# Netcatty Mobile AI Task Prompts

> 用途：把移动端路线拆成可串行执行的 AI 任务。
> 当前基线：移动端尚未开始开发；桌面代码基线为 `upstream/main@c4edefdb`；先完成 M0 和 P0，再进入 v1.0。
> 关联文档：[PRD](./PRD.md)、[技术可行性草案](./README.md)、[桌面能力处置审计](./ALIGNMENT.md)

## 使用规则

1. 严格按任务 ID 顺序执行；前置任务未完成时不得开始后续任务。
2. 每个任务开始前读取关联文档和现有实现，结束后运行对应测试并更新任务状态。
3. `Gate` 未通过时标记 `BLOCKED`，不得通过缩小测试范围或删除需求来标记 `DONE`。
4. 任务只实现自己的范围；发现范围外问题时记录为后续任务，不顺手扩展。
5. 移动代码必须保持 `Domain -> Application -> Platform Adapter -> UI` 边界；不直接复制 Electron bridge 或桌面 UI。
6. 密码、私钥、passphrase、token 和 API key 不得进入日志、错误信息、普通导出文件或测试快照。
7. 除非任务明确要求，所有平台能力都必须在 Android 和 iOS 分别给出证据；单平台结果不能关闭共享 Gate。

## 任务进度索引

执行 AI 完成任务并通过验收后，勾选对应项；被 Gate 阻塞时在任务标题后记录 `BLOCKED: 原因`，不要勾选。

- [ ] M0-01
- [ ] M0-02
- [ ] M0-03
- [ ] M0-04
- [ ] M0-05
- [ ] M0-06
- [ ] M0-07
- [ ] P0-01
- [ ] P0-02
- [ ] P0-03
- [ ] P0-04
- [ ] P0-05
- [ ] P0-06
- [ ] P0-07
- [ ] P0-08
- [ ] P0-09
- [ ] P0-10
- [ ] V1-01
- [ ] V1-02
- [ ] V1-03
- [ ] V1-04
- [ ] V1-05
- [ ] V1-06
- [ ] V1-07
- [ ] V1-08
- [ ] V1-09
- [ ] V1-10
- [ ] V1-11
- [ ] V1-12
- [ ] F1-01
- [ ] F1-02
- [ ] F1-03
- [ ] F1-04
- [ ] F1-05
- [ ] F1-06
- [ ] F1-07
- [ ] F1-08
- [ ] F1-09
- [ ] F1-10
- [ ] F1-11
- [ ] N1-01
- [ ] N1-02
- [ ] N1-03
- [ ] N1-04
- [ ] N1-05
- [ ] N1-06
- [ ] N1-07
- [ ] N1-08
- [ ] VN-01
- [ ] VN-02
- [ ] VN-03
- [ ] VN-04
- [ ] VN-05
- [ ] VN-06
- [ ] VN-07
- [ ] VN-08

## 统一提示词尾部

每个任务提示词都隐含以下输出要求。执行 AI 不得省略：

```text
完成后报告：
1. 修改/新增的文件；
2. 已完成的验收项；
3. 测试命令和结果；
4. 未通过的 Gate、风险和 BLOCKED 原因；
5. 下一项允许执行的任务 ID。
没有满足验收条件时，不要标记 DONE。
```

## M0：产品决策

### M0-01 待决策项清单

```text
你现在只执行任务 M0-01：整理移动端所有待产品决策项。
读取 PRD 第 13 节、README 第 8 节和 ALIGNMENT 的证据缺口。
按平台/版本/安全/数据迁移/发布五类整理决策，明确每项的选项、影响、默认建议和阻塞任务。
不得替用户做最终选择；产出 docs/mobile/DECISIONS.md 初稿，并把无法推断的项目标为 BLOCKED。
```

### M0-02 首发平台与最低 OS

```text
你现在只执行任务 M0-02：确定 Android/iOS 首发范围、最低 OS、分发渠道和设备矩阵。
以 M0-01 的清单为输入，结合 Phase 0 的双端要求，记录 Android 先行但不得后置 iOS 验证的约束。
产出可执行的支持矩阵、真机型号/OS 清单和不支持平台的降级规则；不要凭空承诺商店或后台能力。
```

### M0-03 技术路线候选

```text
你现在只执行任务 M0-03：定义 RN 原生终端、WebView+xterm.js 和其他候选路线的比较标准。
不要直接宣布选型；列出 SSH、终端、IME、无障碍、后台、性能、桥协议、维护成本和商店风险的证据要求。
产出 ADR 草稿和 P0 对照实验计划，明确什么结果会淘汰某个方案。
```

### M0-04 数据交换格式

```text
你现在只执行任务 M0-04：确定移动端与桌面之间的数据交换格式策略。
检查现有 Vault/sync/export payload、CSV 的 Password/Passphrase 风险、known_hosts 和 ManagedSource 语义。
比较独立稳定交换包、受保护备份变体和受限 CSV/ssh_config 导入，定义 schemaVersion、预览、冲突、回滚和凭据重新保护规则。
```

### M0-05 安全策略

```text
你现在只执行任务 M0-05：锁定 App 锁、changed host key、私钥导出、剪贴板和后台行为策略。
所有策略必须遵守移动端 fail-closed 底线，并分别说明 iOS/Android 差异。
产出安全决策记录和待验证威胁模型；不得把桌面 safeStorage 降级行为当作移动实现方案。
```

### M0-06 性能与发布标准

```text
你现在只执行任务 M0-06：定义移动端性能、稳定性、包体和发布验收口径。
覆盖冷启动、10 MB 终端输出、输入延迟、1 KB/10 MB/1 GB/4 GB+ 传输、前后台、低内存、崩溃、无障碍和包体。
每个指标必须写清设备、OS、构建类型、数据集、采样次数、起止事件和失败阈值；没有证据的数字标为候选。
```

### M0-07 决策记录冻结

```text
你现在只执行任务 M0-07：汇总 M0-01 至 M0-06，冻结产品决策版本。
检查是否仍有未解决的安全、平台或格式决策；未决项必须阻塞对应版本，不得默认为 v1.0。
更新 docs/mobile/DECISIONS.md，并在 PRD/README 中记录文档版本、决策日期和受影响任务。
```

## P0：双端可行性验证

### P0-01 移动工程骨架

```text
你现在只执行任务 P0-01：建立移动工程骨架和共享代码边界。
先确认 M0 已冻结；建立 Android/iOS 工程、共享 Domain/Application 包、Platform Adapter 接口和移动 UI 入口。
不得实现生产 SSH 功能；先让空壳工程可构建、可运行、可测试，并记录 Electron 代码不可直接复用的依赖。
```

### P0-02 原生桥协议

```text
你现在只执行任务 P0-02：定义连接、通道、任务、listener 的类型化桥协议。
协议至少包含 requestId、sessionId、connectionId、bootEpoch、事件序号、取消确认、唯一终态、错误阶段和资源所有者。
为迟到事件、重复事件、JS reload、App 退出、切网和幂等清理补充状态图与 contract tests；不绑定具体 SSH 库。
```

### P0-03 SSH 垂直切片

```text
你现在只执行任务 P0-03：在 Android 和 iOS 各打通最小 SSH 垂直切片。
流程必须覆盖 connect、host-key decision、password/key/passphrase、keyboard-interactive/MFA、shell、resize、disconnect。
每个平台记录设备、OS、库版本、构建方式和原始结果；暂不把原型代码当作 v1.0 生产实现。
```

### P0-04 认证与信任验证

```text
你现在只执行任务 P0-04：验证认证取消、host key 和敏感数据清理。
覆盖 trusted/unknown/changed、保存/仅本次/拒绝、并发 MFA、skip-key、取消、超时、迟到响应、错误 passphrase 清除和 OTP 不持久化。
Android/iOS 均需有可重复 fixture 和结果；任何 changed-key 绕过都必须回到 M0-05 的产品决策。
```

### P0-05 终端形态对照实验

```text
你现在只执行任务 P0-05：对比 RN 原生终端与 WebView+xterm.js。
使用同一 SSH server、VT/ANSI corpus、设备档位和 release build，测输入回显、掉帧、内存、IME、组合字符、选择复制、10 MB 输出、旋转和前后台；加入 OSC 9/777/99 通知解析、文本清洗、限流和 CAN/SUB 中止 fixture。
还要验证 bridge 背压、乱序、重复、取消和 JS reload 清理；根据数据更新技术路线 ADR，不凭主观偏好选型。
```

### P0-06 文件与转发探针

```text
你现在只执行任务 P0-06：验证 SFTP/SCP、传输、端口转发和文件权限的可行性。
覆盖目录浏览、流式上传下载、SCP fallback、本地/远程转发、停止清理、系统文件 picker、URI 权限和 App 被杀后的任务记录。
明确 supported/degraded/blocked，尤其要区分普通共享 transport 与重启恢复所需的独立连接。
```

### P0-07 安全存储探针

```text
你现在只执行任务 P0-07：验证 Android Keystore 和 iOS Keychain 的写入、读取、删除、锁定、重装和恢复边界。
验证不可用或设备锁定时 fail closed，不能返回明文或继续保存凭据。
产出平台差异表、密钥生命周期图和迁移限制；不要将设备绑定密文标记为可跨设备复用。
```

### P0-08 后台与切网探针

```text
你现在只执行任务 P0-08：验证 Wi-Fi/蜂窝切换、DNS/VPN、前后台、低内存和 App 被杀。
分别记录 SSH 会话、传输任务、listener 和编辑器的实际状态；禁止用“后台保持连接”笼统描述结果。
产出恢复状态机和用户可见状态规则，明确 iOS 不承诺无限后台。
```

### P0-09 依赖与商店审查

```text
你现在只执行任务 P0-09：固定候选依赖版本并完成许可证、notice、维护活跃度和商店政策检查。
逐项检查 SSH、终端、安全存储、文件、后台服务和构建工具；记录上游 commit/tag、许可证文本和风险。
任何未完成的法务或平台问题标为 Gate，不得写成“可用”。
```

### P0-10 Phase 0 出口

```text
你现在只执行任务 P0-10：汇总 P0 证据并决定是否允许进入 v1.0。
产出双端原型、能力矩阵、桥协议、技术路线 ADR、依赖清单、性能原始数据和修订后的 PRD 排期。
安全关键项或任一平台没有证据时，标记 BLOCKED，并给出改库、自研、降级或调整版本的具体选项。
```

## v1.0：SSH MVP

### V1-01 共享数据模型与迁移

```text
你现在只执行任务 V1-01：实现移动端 Host、Group、Key、KnownHost、Session 的版本化 schema 和可回滚迁移。
复用纯 domain 行为但不引用 Electron runtime；保留未实现协议字段，禁止迁移时静默删除。
为旧数据、未知字段、部分失败和迁移回滚补测试，并输出 schema 兼容说明。
```

### V1-02 平台安全存储

```text
你现在只执行任务 V1-02：实现统一的 Keychain/Keystore 安全存储 adapter。
密码、私钥、passphrase 只能通过该 adapter 读写；不可用、锁定、认证失败或删除失败必须返回明确错误并 fail closed。
补充前后台、设备锁、重装和密钥轮换测试，日志中不得出现秘密。
```

### V1-03 Vault 主机与分组

```text
你现在只执行任务 V1-03：实现 Vault Hosts 的移动列表和分组管理。
覆盖主机 CRUD、复制新 ID、搜索 label/hostname/tags、最近、置顶、多级分组、手动上移/下移、移动和删除影响提示。
v1.0 只承诺列表和手动顺序；不得把 grid/tree 或全部排序模式伪装为已实现。
```

### V1-04 Keychain 基础能力

```text
你现在只执行任务 V1-04：实现 SSH key 导入、查看和删除。
承诺格式必须有 fixtures；显示公钥/证书元数据和 Host/Identity 引用；删除前显示影响。
旧 biometric/FIDO2/passkey/WebAuthn 记录必须作为不可执行归档说明，不得静默转换成普通 key。
```

### V1-05 SSH 连接与认证

```text
你现在只执行任务 V1-05：实现生产级 SSH 连接、认证和错误阶段。
支持 password、已导入 key、passphrase、keyboard-interactive/MFA；区分 DNS/TCP/代理/host-key/认证/shell 阶段。
连接取消、超时、并发 request/session/boot 归属、错误 passphrase 清除和失败重试必须有测试。
```

### V1-06 Host key 与 Known Hosts

```text
你现在只执行任务 V1-06：实现 host key 信任流程和最小 Known Hosts 管理。
默认强制校验；unknown 显示主机、端口、类型、SHA256 指纹并支持拒绝/仅本次/保存；changed 默认 fail closed。
提供当前连接信任记录查看和删除；不要实现未经决策的全局跳过校验开关。
```

### V1-07 移动终端与键盘

```text
你现在只执行任务 V1-07：实现移动 SSH 终端和 Compose/控制键盘。
支持 VT/xterm、ANSI 色、UTF-8、宽字符、resize、alternate screen、Esc/Ctrl/Alt/Tab/方向键、IME、长按选择、复制粘贴。
终端渲染方案必须来自 P0-05 的证据；手机键盘和外接键盘要保持一致语义。
```

### V1-08 大输出流控

```text
你现在只执行任务 V1-08：实现终端输出的有界队列、ACK、pause/drain 和中断优先级。
分别限制原生 transport、桥、JS/UI、renderer 队列；休眠、重挂载、owner handoff、关闭前必须暂停并 drain。
用持续输出和 backlog fixture 证明无丢失/重复/乱序，且 Ctrl+C 不等待完整渲染。
```

### V1-09 多会话生命周期

```text
你现在只执行任务 V1-09：实现移动端多会话、标签元数据和代际隔离。
支持切换、重排、重命名、复制、重连、活动标签、后台可见输出/BEL activity；复制必须生成新 session ID。
bootEpoch/generation 不匹配的旧事件不得污染新连接；activity 不进入 restore，也不等同于系统通知。
```

### V1-10 导航、设置与深链

```text
你现在只执行任务 V1-10：实现 Vault、Sessions、Settings 一级导航和 ssh:// 深链。
深链必须先预览解析结果；含密码 URL 不写历史、日志或持久化主机。
设置只暴露 v1.0 已支持的主题、语言、凭据保护状态、诊断和关于信息，不创建无意义的桌面窗口开关。
```

### V1-11 无障碍与真机适配

```text
你现在只执行任务 V1-11：完成 v1.0 的无障碍、屏幕适配和权限体验。
验证 VoiceOver/TalkBack、200% 字体、横竖屏、平板宽布局、外接键盘、高对比度、reduce motion、软键盘安全区和手势区。
连接状态、host-key 风险、按钮作用不能只靠颜色表达；权限拒绝后其余功能仍可用。
```

### V1-12 v1.0 发布验收

```text
你现在只执行任务 V1-12：执行 v1.0 全量回归和发布 Gate。
覆盖冷启动、SSH 认证、host key、终端 corpus、大输出、会话恢复、切网、敏感日志、崩溃和双端 release build。
只有 Android 和 iOS 都满足 M0-06 的阈值且安全项通过，才标记 v1.0 DONE；否则输出阻塞清单。
```

## v1.1：SFTP 与传输

### F1-01 SFTP 浏览

```text
你现在只执行任务 F1-01：实现单远端 SFTP 浏览和基础目录操作。
支持 home、list、stat、mkdir、rename、delete、chmod，并提供常见归档/单文件压缩的长按解压入口；解压只允许已知格式，复用 v1.0 的 host-key、认证和 session scope。
远端工具探测、路径转义、临时 staging、原子替换、取消、超时和工具缺失必须有明确结果，不得拼接任意 shell 命令。
加载、空目录、权限拒绝、断网和取消状态必须可见；不要依赖桌面双窗格。
```

### F1-02 系统文件选择器

```text
你现在只执行任务 F1-02：实现 Android/iOS 文件 picker 和 document provider adapter。
覆盖单选、多选、目录选择、URI 权限、权限撤销、分享回传和 App 重启后的授权边界。
平台不支持的目录能力必须显示降级结果，不得把临时路径当作永久文件权限。
```

### F1-03 传输状态机

```text
你现在只执行任务 F1-03：实现上传/下载/远端任务队列。
状态至少包含 pending、queued、transferring、pausing、paused、attention、interrupted、completed、failed、cancelled。
分别显示已发现文件、已传输字节、速度和发现阶段；目录未扫描完成前不得伪造稳定百分比或 ETA。
```

### F1-04 冲突与恢复

```text
你现在只执行任务 F1-04：实现传输冲突、暂停恢复、取消、重试和 App 重启恢复。
支持 stop/skip/replace/duplicate/merge，apply-all 范围可见；源/目标变化时进入 attention，不盲目续传。
恢复任务必须使用独立 owner，不能因为关闭终端而停止，也不能附着不等价的 parked transport。
```

### F1-05 远端到远端传输

```text
你现在只执行任务 F1-05：实现两个独立远端之间的 copy/cut/paste。
来源和目标分别完成认证、host-key 和跳板校验；移动单窗格通过来源路径/目标路径选择实现，不依赖同时显示双 pane。
cut 只有目标完成且源删除成功后才清除来源；部分失败保留可重试记录。
```

### F1-06 符号链接与目录限制

```text
你现在只执行任务 F1-06：实现 SFTP/SCP 符号链接、断链、循环和超大目录策略。
明确下载展开、上传和远端复制对链接的不同默认行为；使用 canonical path、深度/条目/目录上限防止无限递归。
重启恢复必须沿用相同排序和 manifest 语义，超过上限的结果要明确 failed/attention/skip。
```

### F1-07 文本编辑器

```text
你现在只执行任务 F1-07：实现远端文本编辑和关闭协调。
默认 10 MB 上限；支持 dirty、word-wrap、language、搜索、保存错误和远端版本冲突。
关闭编辑器、SFTP、连接、owning session、App 重启或退出前统一提供 Save/Discard/Cancel，Cancel 必须阻止破坏性动作。
```

### F1-08 终端增强

```text
你现在只执行任务 F1-08：实现 v1.1 终端编码、搜索、链接和选区发送。
编码运行时切换不能破坏原始日志；搜索保持大小写不敏感、非正则、非整词的桌面基线。
普通 URL/OSC 8 仅允许 http(s)，OSC 8 展示完整目标并二次确认；终端选区直接发送不得经过系统剪贴板。
```

### F1-09 SSH Quick Connect

```text
你现在只执行任务 F1-09：实现 SSH Quick Connect。
支持 user@host:port、括号/裸 IPv6、有限 ssh 参数和 PuTTY 风格命令行参数；无法应用的参数逐项警告。
v1.1 只执行直接 password/key；未保存连接保持 ephemeral，不进入 Vault 或 session restore，不实现未经决策的跳板编辑。
```

### F1-10 大文件 Gate

```text
你现在只执行任务 F1-10：在 Android/iOS 真机验证 4 GB+ 流式传输。
覆盖整数宽度、全程流式、hash、一致性、断网、App 被杀、磁盘不足、目标变化、后台限制和恢复范围。
测试报告必须写清设备、OS、网络、文件系统和实际字节数；任一平台不通过则不能承诺 v1.1 大文件能力。
```

### F1-11 v1.1 发布验收

```text
你现在只执行任务 F1-11：执行 v1.1 文件与传输全量回归。
验证 SFTP/SCP、目录链接、队列、冲突、远端到远端、编辑器、Quick Connect、后台恢复、权限撤销和敏感文件分享。
只有文件与传输 Gate 在 Android/iOS 均通过才标记 DONE，并更新 PRD 版本状态。
```

## v1.2：Network GA

### N1-01 多级跳板

```text
你现在只执行任务 N1-01：实现多级 ProxyJump/跳板链。
每跳独立认证、超时、host key、错误、取消和资源清理；保存时阻止循环和缺失引用。
验证终端、SFTP、传输和转发共用同一 host-chain 解析，不允许某个入口绕过安全校验。
```

### N1-02 HTTP/SOCKS 代理

```text
你现在只执行任务 N1-02：实现主机级和跳板级 HTTP/SOCKS 代理。
代理连接失败必须与目标 SSH 失败分开显示；支持的认证字段进入安全存储。
ProxyCommand 只保留数据并显示 unsupported，不执行本地命令或把它静默改写成 HTTP/SOCKS。
```

### N1-03 Identity 与证书

```text
你现在只执行任务 N1-03：实现 Identity 和 SSH 用户证书。
支持 username、password、key、certificate 的复用；证书与私钥配对校验，引用失效时阻断并提供修复。
分别验证 RSA/ECDSA/ED25519 证书和每跳 host key；不把系统 agent 登录冒充为 Identity。
```

### N1-04 密钥生成

```text
你现在只执行任务 N1-04：实现 Keychain 密钥生成。
支持 ED25519、ECDSA P-256/384/521、RSA 1024/2048/4096；弱选项必须警告。
私钥只进入平台安全存储；生成失败、取消、删除和引用关系均有明确结果，不实现未经安全评审的私钥导出。
```

### N1-05 端口转发

```text
你现在只执行任务 N1-05：实现本地 -L 和远程 -R 端口转发。
支持 CRUD、复制、排序、启停和 connecting/active/error/inactive 状态；隧道 SSH 主机和每级跳板必须校验 host key。
编辑、删除、停止、会话关闭、网络断开和 App 退出后必须释放 listener、forward 和 accepted sockets。
```

### N1-06 本地保护备份

```text
你现在只执行任务 N1-06：实现本地保护备份、预览和恢复。
覆盖保留数量、覆盖前快照、恢复失败回滚、known_hosts、本地实体和 ManagedSource 悬空引用处理。
凭据不可解密时 fail closed；设备绑定密文不得直接跨设备复用；恢复前后显示受影响实体和下一步动作。
```

### N1-07 网络恢复策略

```text
你现在只执行任务 N1-07：实现 v1.x 网络恢复和自动重连策略。
采用有限指数退避，显示下次尝试、次数、停止入口和最终状态；认证失败、changed key、用户拒绝和主动断开必须停止。
分别处理 SSH、SFTP/传输和端口转发，不得复用桌面固定 5 秒无限重试行为。
```

### N1-08 v1.2 GA 验收

```text
你现在只执行任务 N1-08：执行 v1.2 Network GA 验收。
覆盖跳板/代理、证书/Identity、密钥生成、端口转发、备份恢复、切网、后台、资源清理、权限和商店政策。
任一安全关键项或任一平台不通过，必须保留 BLOCKED 状态并提出版本降级或技术替代方案。
```

## vNext：独立能力任务

### VN-01 Notes 与 Snippets

```text
你现在只执行任务 VN-01：为 Notes、Snippets 和 Compose Bar 编写独立 mini-PRD 并实现第一版。
Notes 必须与 Host.notes 分离；Snippets 保留分类、变量、目标主机、pin、排序、导入导出和删除引用清理。
所有触摸替代、外接键盘、无障碍和危险运行操作都要有单独验收，不得把 vNext 能力混入 v1.x。
```

### VN-02 云同步

```text
你现在只执行任务 VN-02：设计并实现移动云同步的安全 preflight、冲突和恢复。
保留桌面 payload 的实体包含/排除语义；known_hosts、本地密文、OAuth secret、插件 secret 和 ManagedSource 必须逐项处理。
普通空库上传拒绝；上传、恢复、清空和 v1/v2 migration 共用保护备份、事务边界、写后验证和可回滚 downgrade。
```

### VN-03 API 型 AI

```text
你现在只执行任务 VN-03：设计移动 API 型 AI，不复用本地 CLI/stdio runtime。
逐域处置 session、attachment、terminal、SFTP、Vault、portforward、harness 工具；保留 scope、Confirm、取消、stopAgentTurn、handle 和历史恢复契约。
API key 使用安全存储，工具写操作默认 Confirm；任何能力未经隐私、后台和权限 Gate 不进入发布范围。
```

### VN-04 移动脚本

```text
你现在只执行任务 VN-04：设计移动脚本语言、API 和执行隔离。
不得把桌面 Python 标签或 Node vm/child_process 直接标成移动支持；定义 screen/wait/send/read、session/log、dialog、progress、超时和资源上限。
写操作审批、触发去重、后台行为、取消和商店政策必须先于执行器实现。
```

### VN-05 System Manager

```text
你现在只执行任务 VN-05：为 System Manager 建立能力探测和危险操作模型。
逐项处置进程、端口、systemd、tmux、Docker、GPU 和网络设备降级；危险操作需要 Confirm 和清晰影响范围。
能力不可用时显示 unsupported/degraded，不伪造桌面 shell 结果；页面顺序和隐藏状态必须可恢复。
```

### VN-06 其他协议

```text
你现在只执行任务 VN-06：分别评估 Telnet、Mosh、ET、Serial，禁止用“其他协议”合并结论。
每种协议单独验证认证、host key、字符集、resize、断网恢复、权限、明文风险、原生库和双端硬件条件。
无法满足维护、安全或商店条件时明确 BLOCKED 或“不适用”，不得复用桌面外部 binary 路径。
```

### VN-07 插件与 providers

```text
你现在只执行任务 VN-07：研究移动插件沙箱和 providers，不直接移植 BrowserWindow/utilityProcess。
逐 kind 评估 settings、commands、views、terminal provider、connection/authentication/sync/importer provider 的 supported/degraded/blocked。
先产出权限、配额、secret、离线、安装回滚和商店合规设计，再决定是否实现 runtime。
```

### VN-08 移动平台新增能力

```text
你现在只执行任务 VN-08：逐项评估健康探测、扫码、系统分享、后台断开通知、App Shortcuts 和 Widget。
每项单独定义隐私、锁屏、敏感信息、权限拒绝、过期对象、App 锁和平台 Gate；不得因为它们是移动特性就自动进入路线图。
通过产品决策和真机验证后，分别建立实现任务，不合格项保持 BLOCKED。
```

## 任务状态格式

建议在执行副本中维护以下状态：

```text
TODO       尚未开始
IN_PROGRESS 正在执行
BLOCKED    被产品、平台、安全或依赖 Gate 阻塞
DONE       验收标准和测试均已通过
```

推荐首个执行命令：`M0-01`。完成 `M0-07` 和 `P0-10` 前，不允许 AI 直接进入 v1.0 页面开发。
