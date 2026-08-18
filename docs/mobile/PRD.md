# Netcatty Mobile PRD（产品需求文档）

> 范围: iOS / Android 移动客户端
> 状态: v0.3 草案，移动端尚未开始开发
> 移动文档基线: `mobile@21a3d084`；桌面代码基线: `upstream/main@c4edefdb`；桌面能力处置见 `ALIGNMENT.md`
> 原则: 全量能力处置、分阶段交付；“已列入”不等于“已实现”或“技术已验证”

## 1. 产品定义

### 1.1 定位

Netcatty Mobile 是桌面 Netcatty 的移动配套客户端，服务于离开桌面后的应急连接、日志查看、轻量运维和文件处理。移动端保留同一 vault 和安全心智，但不照搬桌面的窗口、鼠标和多窗格交互。

### 1.2 对齐目标

“对齐所有功能”在本 PRD 中指:

1. 当前桌面能力都必须在 `ALIGNMENT.md` 中有“等价、移动替代、后置、不适用或待验证”处置。
2. v1.0 只交付可独立使用的 SSH MVP；v1.x GA 补齐 SFTP、传输、转发、跳板，以及导入/生成、passphrase、证书和 Identity 等 Keychain 核心能力；reference key、公钥部署、私钥导出和集合布局仍按 vNext 单列。
3. 云同步、AI、系统管理器、Notes、snippets、脚本等进入后续路线图，不因首版后置而从总范围消失。
4. 桌面专属能力必须说明不适用原因和移动替代，不能用“全功能对齐”掩盖差异。

### 1.3 里程碑术语

| 术语 | 定义 |
|---|---|
| v1.0 MVP | 可安全配置主机并完成稳定 SSH 终端操作的最小发布 |
| v1.x GA | v1.0 + SFTP/传输 + 跳板/代理 + 端口转发 + v1.2 范围内的 Keychain 核心能力 |
| vNext | 已纳入最终能力处置、但未承诺具体版本的后续功能 |
| Gate | 未通过前不得写入承诺版本的真机、许可、安全或平台验证 |

优先级只表达产品重要性，不代替版本:

- P0: v1.0 必须满足；显式标为“候选/待决”的条目只有在产品决策纳入 v1.0 后才转为承诺。
- P1: v1.x GA 必须满足。
- P2: vNext 处置项或体验增强。
- Gate: 技术或合规未证实；通过后才能落入 P0/P1 的交付承诺。

### 1.4 核心用户与场景

| ID | 场景 | 可观察结果 |
|---|---|---|
| S1 | 收到告警后连接生产机 | 已有 vault 用户从启动到出现可输入 shell 的中位时间满足发布性能基线；失败能定位网络/信任/认证阶段 |
| S2 | `tail -f`、搜索和复制日志 | 连续输出可暂停、搜索、选择和复制；软键盘不遮挡当前输入行 |
| S3 | 使用 vi/nano 或 Compose Bar 修改配置 | Esc/Ctrl/Alt/Tab/方向键、组合键、IME 和多行输入可达，不丢字符或重复发送 |
| S4 | 上传、下载、编辑和分享文件 | 进度、速度、状态和取消结果可见；冲突、重试、断网和恢复有明确结果 |
| S5 | 临时建立本地/远程转发 | 规则状态和错误可见；停止后监听端口与 SSH 资源被释放 |
| S6 | 新设备迁移（候选/Gate） | 仅在第 13 节确定交换格式且凭据可移植性 Gate 通过后，才要求导入前预览、冲突决策和失败回滚；不预先绑定版本 |
| S7 | 多会话应急操作 | 可重连、重命名、复制、批量关闭和快速切换，不依赖桌面多窗口 |

### 1.5 明确不适用与移动替代

| 桌面能力 | 移动处置 |
|---|---|
| 多窗口、tab detach、复制到新窗口 | 不适用；多标签、最近会话和标签分组替代 |
| workspace 水平/垂直 split、焦点 workspace | 不适用；单会话主视图 + 快速会话切换替代 |
| 托盘、quake window、全局快捷键 | 通知、App Shortcuts、Widget 等平台入口替代；既有协议深链按第 10.4 节单独处置 |
| Explorer/Finder 右键、本地终端、窗口透明度、自定义 CSS | 不适用；不进入移动产品 |
| SFTP 双窗格和完整本地文件管理 | 单远端窗格 + 系统文件选择器/document provider/分享替代 |
| 鼠标右键、中键、拖拽 | 长按菜单、触摸选择、系统文件选择/分享替代 |
| ProxyCommand、本地 CLI agents、stdio MCP server | 依赖本地进程，移动端不适用；HTTP/SOCKS、API 型 AI 或受限客户端替代 |
| X11 转发 | 不适用，除非未来出现独立且通过安全评审的移动 X server 方案 |

telnet、mosh、et、serial、插件、脚本、AI、云同步和系统管理器不是永久删除项，分别在 vNext 处置；其中物理或平台不可行部分需在设计阶段再转为“不适用”。

## 2. 信息架构与通用状态

### 2.1 一级导航

v1.0 至少包含 Vault、Sessions、Settings；v1.1 增加 Files/Transfers；v1.2 增加 Port Forwarding 和 Keychain 核心能力。vNext 可增加 Notes、Snippets、AI、System Manager 和 Sync。桌面 Vault 的八个 section 必须有以下移动入口；“后置”不允许实现时仍没有导航路径:

| 桌面 section | 移动页面/入口 | 可达版本 | 入口结果 |
|---|---|---|---|
| `hosts` | Vault -> Hosts（Vault 默认页） | v1.0 | 主机/分组浏览、搜索、新建和连接 |
| `knownhosts` | Vault -> Hosts -> Trust / Known Hosts | v1.0 最小；v1.2 完整 | 当前连接的信任记录可查看/删除；完整搜索、导入、扫描和转主机后置到 v1.2 |
| `keys` | Vault -> Keychain | v1.0 基础；v1.2 核心 | v1.0 导入/查看/删除 keys；v1.2 增加生成、证书和 Identity 管理；reference key、公钥部署、私钥导出及完整集合布局仍按第 8 节后置 |
| `port` | Network -> Port Forwarding；主机详情提供关联入口 | v1.2 | 规则列表、创建、状态、启停和错误 |
| `proxies` | Vault -> Proxy Profiles；主机/分组代理选择器可跳转 | vNext | 管理复用代理档案；v1.2 的内联 HTTP/SOCKS 配置不依赖此页 |
| `snippets` | Vault -> Snippets；终端 Compose Bar 提供上下文入口 | vNext | 管理、搜索、插入或运行 snippets |
| `notes` | Vault -> Notes；Host/终端提供关联入口 | vNext | 独立 Vault Notes，不与 `Host.notes` 混用 |
| `logs` | Sessions -> Connection Logs；会话详情可进入对应记录 | vNext | 浏览连接记录并在有快照时打开只读终端回放 |

每个能力必须登记页面、入口、版本和以下状态，不能只交付 happy path:

| 状态 | 通用要求 |
|---|---|
| 初始/空 | 解释当前没有数据，并提供唯一明确的下一步操作 |
| 加载 | 保持布局稳定；长操作显示阶段和取消入口 |
| 错误 | 显示失败对象、阶段、可重试性和不泄密的诊断信息 |
| 离线/切网 | 状态不冒充成功；恢复后按能力选择重连、继续或要求用户确认 |
| 权限拒绝 | 说明受影响功能；其余功能可继续使用；可跳转系统设置时提供入口 |
| 后台/恢复 | 明确哪些连接仍存活、哪些需重连、哪些任务可续传 |
| 冲突 | 显示双方信息、默认安全选项、apply-all 范围和撤销/回滚能力 |
| 删除/清空 | 明确级联影响和不可恢复内容；危险操作二次确认 |

桌面当前没有独立的引导式 onboarding：首次进入只显示空主机状态，Vault 会播种 3 条默认 snippets，Compose Bar 也会在从未保存 pin 时写入一组默认快捷项。移动端若增加教程、示例主机、权限预热或登录引导，属于移动新增能力，必须另行决定跳过/重入、无障碍、示例数据删除和真实凭据隔离，不能把它计入桌面功能对齐。

### 2.2 适配与无障碍

- 支持手机竖屏/横屏；平板使用更宽布局，但不以桌面双窗格作为 v1.x 验收前提。
- 支持系统字体缩放、VoiceOver/TalkBack、外接键盘、动态颜色/高对比度和 reduce motion。
- 屏幕阅读器可读出连接状态、传输状态、host-key 风险和按钮作用；不能只靠颜色表达。
- 触控目标、软键盘安全区、底部手势区和分屏/浮窗系统模式纳入真机测试矩阵。

## 3. Vault、主机与数据迁移

### 3.1 主机与分组

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 主机 CRUD、复制、删除影响提示 | P0/v1.0 | 支持 hostname、port、username、认证、标签和分组；复制生成新 ID；删除前列出活动会话/规则/身份引用影响 |
| 搜索、最近、置顶 | P0/v1.0 | 搜索覆盖 label/hostname/tags；桌面近期区按 `lastConnectedAt` 排序、排除置顶项并截取 6 条；移动数量上限由信息架构/性能测试决定且可隐藏该区 |
| 多级分组、手动顺序和移动 | P0/v1.0 | 新建/重命名/删除/移动后路径与子级一致；v1.0 提供触摸可取消的手动上移/下移或编辑顺序，删除组可选择移动或处理其中主机；组路径变更必须原子更新主机、托管源和 snippets/script targets，失败时回滚；A-Z、Z-A、newest、oldest、group 等非手动排序模式与其他视图按下表后置，不把“保留手动顺序”写成“已交付所有排序” |
| 主机列表/网格/树与点击行为 | P2/vNext | 桌面对齐三种视图；移动 v1.0 先列表，其他视图不得伪装成已有能力 |
| 批量连接和批量选择 | P2/vNext | 错峰启动；可取消未开始项；单个失败不阻断其他项 |
| 快速连接 | P1/v1.1（certificate/Identity v1.2；其他协议执行 vNext） | 对齐桌面 `user@host:port`、括号/裸 IPv6、有限 `ssh` 命令参数和 PuTTY 风格命令行参数解析，无法应用的参数逐项警告；v1.1 支持 SSH 的直接 password/key、host key、连接错误与显式保存，certificate 和已保存 Identity 随 v1.2 Keychain/Identity 能力交付。桌面直接自定义认证 UI 也只有 password/key，certificate 只能经已保存 Identity 间接使用。Mosh/ET/Telnet 的选择和协议字段可被解析/保留并显示 unsupported，但连接执行随各协议在 vNext 交付。桌面此入口没有跳板/代理表单，移动若增加属于新增；未保存对象保持 ephemeral，不进入 vault/session restore，显式保存才生成持久主机 |
| 快速切换器 | P2/vNext | 桌面基线搜索 Hosts、Vault/SFTP tab、workspace/session、本地 shell 和插件 command/view，并可连接/编辑主机或新建 workspace；移动保留主机、会话和核心目的地搜索，workspace/本地 shell 不适用，插件项随插件后置。设置搜索是独立能力，若合并进移动全局搜索须标为移动新增 |
| 多协议主机与连接选择 | P2/vNext（SSH 数据兼容 v1.0） | 保留默认协议、附加协议、协议专属端口/主题/凭据和未知 plugin provider 配置；v1.0 只执行 SSH，其他协议显示 unsupported/后置但不得被编辑或迁移静默删除。桌面当前只在附加 Telnet 且默认非 Telnet 时强制弹 picker，不假称每个 SSH+Mosh/ET 主机都会询问 |
| 复制主机地址与账密 | 地址 P0/v1.0；账密 P1/v1.2 | 地址复制只写可连接 hostname，不把 plugin provider label 冒充地址。账密复制先应用分组默认值，再按主协议解析 SSH/Telnet、Identity、IPv6 和非默认端口，输出明确的 host/username/password；无可读密码、凭据占位符、系统剪贴板拒绝或 App 已锁定/进入后台时不得留下半截或旧内容冒充成功。明文密码复制前要求近期设备认证/再次认证，显示系统剪贴板历史与跨设备同步风险，并在平台允许时提供短时过期/清除；系统不允许可靠清除时如实提示，不能承诺撤回其他 App 已读取的内容 |
| 连接日志 | P2/vNext | 与持续会话输出日志分开；按时间浏览，支持收藏/取消收藏、删除，并在存在 terminal snapshot 时打开只读回放。桌面当前没有搜索或从日志直接重连，移动若增加须标为新增 |
| 主机图标/发行版 | P2/vNext | 基础标签与置顶已进入 v1.0；图标和发行版字段后置。自动检测先从当前连接的 SSH banner 识别网络设备，再在同一已认证 transport 上做 OS probe；已识别或手工标记的网络设备不执行 POSIX shell probe，迟到结果不得覆盖新连接。`manualDistro`/自定义图标只覆盖展示，不反向改变探测结果或安全能力分类 |
| 主机内联备注 | P2/vNext | `Host.notes` 为单主机 Markdown 文本，不与 Vault Notes 混用 |
| 启动命令 | P2/vNext | 支持多行 `lineDelay`/`paste`；连接成功且 shell ready 后执行；失败不隐藏终端。全局发送延迟默认 600 ms、可配置 0-10000 ms，既用于首次发送前等待，也用于 `lineDelay` 的逐行间隔；迁移时保留该设置，移动端若采用不同默认值须记录为平台调整 |
| 主机级终端覆盖 | P2/vNext | 主题、字体、字号、字重覆盖全局值；可一键恢复继承 |
| Vault 显示与激活偏好 | P2/vNext | 逐项处置“最近主机”“先选中再连接”“根目录只显示未分组主机”“显示独立 SFTP 入口”“显示主机树侧栏”；手机将鼠标单击语义替换为明确的点击连接/选择模式，将桌面顶栏和侧栏开关替换为移动导航入口设置，关闭入口不能让 SFTP 或主机搜索不可达 |
| 主机健康状态 | 移动新增/P2 | 未经产品确认不进入承诺版本；必须定义探测方式、频率、流量和误报语义 |
| 扫码配置 | 移动新增/P2 | schema 版本化；预览字段；禁止二维码静默携带明文私钥/密码 |

桌面的 Vault 集合并非只有 CRUD。移动端即使采用统一列表，也要逐项保留下列可观察结果；鼠标拖拽可改成触摸编辑模式、移动菜单或无障碍上移/下移，不得因此丢掉用户顺序:

| 集合 | 桌面显示与顺序基线 | 移动处置 |
|---|---|---|
| Hosts | grid/list/tree、A-Z/Z-A/newest/oldest/group 排序、手动顺序、分组内移动 | v1.0 列表、分组和手动顺序/移动先行；grid/tree 与非手动排序模式后置，但导入或编辑不能破坏已有顺序 |
| Keychain keys / Identities | grid/list、手动顺序 | v1.x 读取、迁移并保留已有实体顺序；grid/list 偏好和用户重排 UI 后置到第 8 节的 vNext 项，不得按名称静默重写 |
| Proxy profiles | grid/list、手动顺序 | 随代理档案在 vNext 保留顺序；不得按名称静默重写 |
| Snippets / packages | grid/list、多种排序、手动顺序和包内/包间移动 | vNext 等价结果，触摸交互替代拖拽 |
| Known Hosts | grid/list、多种排序、手动顺序 | 完整管理版本保留排序与顺序；P0 最小删除入口可先不暴露全部布局 |

### 3.2 分组默认设置

分组可继承 username、password/savePassword、port/protocol、认证方式、Identity/key、环境变量、代理档案/代理、跳板链、启动命令及运行模式、设备类型、agent forwarding、legacy/算法覆盖、字符集、Mosh enabled/server path、ET enabled/port、Telnet enabled/port/Identity/manual username/password，以及主题、字体、字号、字重和退格行为。主机显式值优先；显式空值/禁用与“未设置”必须按字段规则区分。v1.0 尚不能执行的 agent forwarding、Mosh/ET/Telnet 等字段也要保留并显示 unsupported，按 C-07/C-08/X-10 后置，不能在编辑 SSH 字段时清除。删除被引用的 key、Identity、proxy profile 或跳板主机时必须给出影响和修复入口。

### 3.3 导入、交换格式与回滚

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 桌面格式导入 | 候选 P0/版本待决 | 支持范围由第 13 节产品决策锁定；承诺的每种格式都要有样例、字段映射、预览和部分失败报告；可选择保留源分组、导入既有分组或新建分组，CSV 提供与当前 parser 字段一致的模板 |
| 主机 CSV 导出 | P2/vNext + 安全决策 | 字段、导出数和跳过数可见；serial/plugin 等不可表示的主机不得静默丢失；系统分享/文件保存失败可重试且不会遗留临时副本 |
| 托管 ssh_config | P2/vNext | 一次性导入与托管源分开；写回前显示 diff；外部修改、删除和冲突有决策流程 |
| 稳定交换包 | 候选 P0/格式 Gate | 是否独立于本地备份由第 13 节决定；必须定义 `schemaVersion`、应用版本、实体 ID、敏感字段封装和完整性校验 |
| 冲突与回滚 | P0/随导入版本 | 导入前预览新增/修改/跳过；支持保留本地/采用导入/复制为新项；失败不留下半完成 vault 并可恢复导入前状态 |
| 凭据可移植性 | Gate | 系统绑定密文不得直接跨设备复用；必须重加密、重新输入或使用受保护交换密钥 |
| known_hosts 本地策略 | P0/v1.0 | 信任记录本地独立维护，云同步默认排除；是否纳入交换包另由第 13 节决定且须显式选择并显示风险 |

桌面代码已有应用内部的 JSON 同步/备份 payload 构建和导入能力，但当前面向主机的导出 UI 主要是 CSV。该 CSV 包含 `Password`、`KeyPath` 和可逆编码的 `Passphrase` 字段，并会报告无法读取的 passphrase 与不可表示主机；这与第 10.2 节的移动安全底线存在产品冲突。移动端必须在第 13 节选择：禁用/脱敏 CSV 凭据，或把含凭据导出作为高摩擦、短时有效、可审计且可清理的显式例外。内部 payload 不等于稳定的跨平台公开格式；在交换格式决定前不能承诺“无缝带到手机”。

### 3.4 Vault Notes 与备份

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 独立 Vault Notes | P2/vNext | 标题、Markdown 内容、层级分组、标签、关联多个主机、搜索、排序和手动重排；编辑/预览模式均不得丢内容 |
| Notes 导入导出 | P2/vNext | 导入 `.md/.markdown/.txt`；单篇导出 Markdown；批量/分组导出 ZIP，文件名安全且不覆盖 |
| Markdown 编辑语义 | P2/vNext | 富文本操作持续导出为 Markdown；粘贴结构化 Markdown 保持当前插入位置；GFM task checkbox 在预览中可切换且不修改代码块/注释中的伪任务 |
| Markdown 图片/链接 | P2/vNext | 图片持久化与删除、远端图片 lazy load、可设置宽高/紧凑尺寸、外链安全打开、无效链接和离线状态有明确表现 |
| 本地保护备份 | P1/v1.2 | 手动/自动、预览、恢复、保留数量；包含本地 trust records；凭据不可解密时 fail closed。桌面 `ManagedSource` 描述符不在当前 backup/export payload 中但 Host 可能保留 `managedSourceId`；移动格式必须选择携带不泄露平台路径且需重新授权的源描述符，或在预览/恢复时去除并报告这些引用，不能产生悬空托管状态 |
| 云同步 | P2/vNext | WebDAV/S3/GDrive/OneDrive/GitHub 与插件 provider、端到端加密、冲突、修订、恢复、force push 另行 PRD；移动端先锁定与桌面 payload 的兼容版本和字段处置，不把“能解密文件”当作“所有字段均可直接迁移”。恢复判定的“本地为空”指 `CLOUD_SYNC_PAYLOAD_ENTITY_KEYS` 中所有同步实体均为空，不等于仅无 hosts；桌面首次启动播种的 3 条 Vault snippets 会使正常新安装通常非空。桌面 `useAutoSync` 只在远端被判为有变化时触发不可绕过的 Restore/Keep Empty，Keep Empty 不推进 anchor/base；Settings 单 provider Sync、旧 anchor 的 unchanged fast path、默认 snippets 和清空后残留 CRDT replica 都会留下保护缺口。移动端必须把所有自动/手动上传、启动恢复、清空与 v1/v2 路径纳入统一 preflight：检查实体丢失和异常收缩，先建立可验证保护点，显示所有受影响实体；普通空库上传一律拒绝，只有独立高摩擦确认的 Force Push 可擦除云端 |

桌面云同步 payload 不只是 Vault 主实体。当前代码还包含 proxy profiles、snippet packages、Notes/分组、group configs、去除运行态字段的端口转发规则、跨设备设置、部分 AI 配置，以及插件的非秘密 `sync:true` 设置和 account/CRDT baseline sidecar。移动设计必须逐字段保持或显式迁移这些内容，不能只写“同步主机/密钥/snippets”。

排除和凭据语义同样属于协议契约:

- `known_hosts` 是设备本地信任记录，只进入本地保护备份，默认不进入云同步；是否进入交换包另行决定。
- 外部 CLI agent 的 command/args/env 是操作系统绑定配置，不同步；移动端不得收到无法执行的桌面命令路径。
- 插件 secret settings 不进入 sidecar；只有声明为非秘密且 `sync:true` 的设置及规定的 baseline 可进入加密 payload。
- 设备绑定的 `enc:v1` 密文不能跨设备直接复用。Vault 凭据和 AI API key 只有在本机可解密、转换为可移植值并置于云同步主密钥加密包后才能迁移；转换失败必须停止上传或要求重新输入，不能上传不可解密占位符，也不能在应用日志或中间文件中落明文。
- 云提供商 OAuth token、provider connection secret、App 锁/生物识别材料和系统 Keychain 元数据是否迁移必须逐项定义；没有进入 `SyncPayload` 的本地状态不得因“云同步已开启”被暗示为已备份。
- 托管 ssh_config 的 `ManagedSource`（含桌面 `filePath`、group、hash）当前独立于云/本地备份和 Vault export payload；云同步继续排除系统路径。收到带 `managedSourceId` 但无对应 descriptor 的 Host 时，导入/恢复预览必须列出并选择去除引用、重新选择文件并建立新 descriptor，或取消，不能自动写回未知路径。

桌面另有实验性的 convergent multi-device sync（CRDT v2），不是普通 `smartMerge` 的同义词。vNext 设计必须单列启停、v1 -> v2 migration preview、provider schema/可用性阻断、离线并发修改与删除、字段级候选冲突（秘密只显示“已设置”）、写后验证、保留 v1 compatibility snapshot，以及向所有已连接 provider downgrade 并移除 CRDT metadata 的高风险流程。移动端未实现这些状态前不得开启或改写 v2 payload；只能按兼容规则读取 v1 snapshot，或明确阻断并提示升级路径。

同步入口和冲突可见性也属于移动端的强制修正项。桌面 Settings 的单 provider Sync 直接进入 `syncToProvider`，没有经过 `useAutoSync` 的空 payload guard；“清空本地数据”直接清 Vault/转发并重置版本，没有保护备份、跨窗口 barrier 或 apply sentinel，且不会清除 canonical CRDT replica。多 provider base 分叉只发出诊断事件，当前设置 UI 不消费；legacy smart merge 在双方新增或修改同一 ID 时静默选择 local，只记录冲突计数。移动端不得沿用这些成功提示：所有入口使用同一事务/取消/崩溃恢复边界，provider 分叉和逐实体冲突可审查，清空后的预期 remote/local 状态必须可证明并可回滚。

## 4. SSH 连接、认证与信任

### 4.1 认证

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| `auto/password/key/certificate` | P0/v1.0（certificate v1.2） | 模式含义明确；terminal、SFTP、跳板、exec/公钥部署和转发使用同一解析规则 |
| 密码和保存策略 | P0/v1.0 | 可临时输入或保存；保存开关关闭时不持久化；日志/崩溃报告不含凭据 |
| 密钥与 passphrase | P0/v1.0 | ED25519/RSA/ECDSA 导入；passphrase 可单次输入或显式选择安全保存，默认不记住；错误可区分格式/口令/算法。多个连接请求按 `requestId` 和 session/boot 归属，迟到或所属连接已断开的提示自动取消；超时关闭对应提示并可重试，错误的已保存口令被清除；“跳过当前候选 key，继续尝试其他 key”与“取消本次认证流程”是两个独立动作 |
| 默认/ambient 私钥发现与回退 | P0/v1.0 认证边界；P2/vNext 文件发现 Gate | 桌面扫描 `~/.ssh` 中有效的 `id_*` 私钥并排除 `.pub`，顺序为 `id_ed25519`、`id_ecdsa`、`id_rsa` 后接其余名称排序；初始 fallback 跳过加密 key，合资格的认证失败后才按 request/session/boot 请求口令并重试。password-only、显式选择 key/certificate 和 `IdentitiesOnly` 不得尝试无关默认 key。移动 v1.0 只使用已导入 Keychain 或用户显式授权的文件，不静默扫描系统目录；若 vNext 提供授权目录发现，保持同一校验、顺序、排除、取消与重试语义 |
| keyboard-interactive/MFA | P0/v1.0 | 支持多轮、多字段和多个会话并发请求；每项显示目标主机/所属会话，按 `requestId` 排队或可明确切换，响应绝不串到其他连接。`requiresMfa` 时优先顺序与桌面一致；断连、重连 supersede、后端取消或用户取消只收敛所属认证。保存登录密码只允许明确识别的 password prompt，OTP/token/verification code/二次密码等因子不得预填、保存或覆盖主机登录密码 |
| Identity 预设 | P1/v1.2 | 用户名 + 密码/密钥/证书可复用；引用失效时阻断并提供修复 |
| 系统 SSH agent 登录 | P2/vNext + Gate | 分别处置 `useSshAgent`、`IdentityAgent`/socket、`IdentitiesOnly`、`AddKeysToAgent` 和 macOS `UseKeychain`；移动平台没有等价系统 agent 时显式标为 degraded/blocked，不得假装使用已导入私钥就等价 |
| Agent forwarding | P2/vNext + Gate | 与“使用 agent 登录”分开授权；按主机/会话显示暴露给远端的风险、agent 不可用状态和取消/关闭清理，不因登录方式为 password/certificate 就静默改变转发选择 |

桌面的自动认证中，未显式关闭的 ambient agent 可作为候选；`IdentityAgent none` 则明确禁止 agent 登录。显式 key 模式只有存在公钥/identity file selector 时才可约束 agent key，`IdentitiesOnly` 控制可提交的身份集合；`AddKeysToAgent` 与 `UseKeychain` 描述本地 agent/key 生命周期，不等同于认证成功。移动端若设计内置 agent，必须为这些语义定义迁移结果；若只提供平台私钥存储，则将不支持字段保留在数据中并在编辑时解释，不能静默删除。Agent forwarding 是认证后的独立通道权限，即使复用同一 agent adapter，也不能与 agent 登录合并为一个开关。

### 4.2 host key 与 known_hosts

默认启用校验且不可由普通连接流程静默绕过。桌面虽有全局 `verifyHostKeys=false` 开关并会让终端、SFTP、mosh/et 和转发绕过校验，移动端基线不提供等价的全局关闭开关；若企业策略确需例外，必须另行安全设计、限域且清楚显示未验证状态:

| 状态 | 行为 |
|---|---|
| trusted | host/port 对应指纹匹配，透明继续 |
| unknown | 显示 hostname、port、key type、SHA256 指纹；允许拒绝、仅本次继续、保存并继续 |
| changed | 默认停止认证并显示新旧指纹；v1.0 确定承诺拒绝和进入独立信任记录修复流程。“仅本次继续”或连接内替换并继续仅在第 13.5 节产品决策保留 changed override 后提供，并需显式高摩擦确认；若选择硬阻断则连接流程不得暴露这些绕过动作 |

P0 必须提供本次连接产生的信任记录查看和删除入口；P1 完整管理再提供搜索、批量导入、平台文件扫描和 known-host 条目转主机。桌面同时支持打开 Vault 时静默导入系统 OpenSSH `known_hosts` 和手动“扫描系统”；移动端只有在平台持续文件授权可靠且来源可解释时才提供自动导入，否则以显式选择文件/扫描替代，不能把手动导入写成自动等价。删除、替换以及产品决策后启用的 changed override 写入本地安全审计事件，但不得记录密码或私钥。

同一策略适用于目标主机、每级跳板、SFTP/SCP 连接和端口转发所用 SSH 隧道主机。普通 TCP 转发目标没有 SSH host key，不做伪校验。

### 4.3 连接高级项

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 两类超时 | P0/v1.0 | TCP connect 与 auth-ready 分开配置和报错；默认值由实现规范统一，不在 PRD 假写单一 10s |
| 取消和错误定位 | P0/v1.0 | 连接可取消；区分 DNS、TCP、代理、跳板层级、host key、认证、shell/channel 阶段 |
| keepalive | P1/v1.x | 全局 interval/count + 主机 `keepaliveOverride`；关闭后不继续发送探测，修改继承/覆盖时不混淆“未设置”与显式禁用 |
| SSH 自动重连 | P1/v1.x | 桌面基线只有全局开关：曾成功连接过的普通 SSH 意外断开后固定每 5 秒重试，直到成功、关闭标签或关闭设置；不适用于 local/serial/telnet/Mosh/ET，且没有主机级覆盖或按错误类型停止。移动端采用有限指数退避并对认证失败、changed host key、用户拒绝/主动断开停止，属于移动安全增强，必须显示下一次尝试、次数、停止入口和最终状态 |
| 环境变量与字符集 | P2/vNext | 连接注入失败有提示；终端字符集与 SFTP 文件名编码分开 |
| 网络设备模式 | P2/vNext | 不做 shell 包装，命令原样发送；AI/脚本执行也遵守该模式 |
| legacy 与算法覆盖 | P2/vNext | 支持 legacy、skip ECDSA 和 kex/cipher/hmac/serverHostKey/compress 列表；危险配置警告且可恢复默认 |

### 4.4 跳板与代理

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 多级跳板 | P1/v1.2 | 按顺序连接；每跳独立认证、超时和 host key；循环/缺失引用在保存时阻止 |
| HTTP/SOCKS | P1/v1.2 | 支持主机/跳板级配置；代理连接错误与目标 SSH 错误分开 |
| 代理档案 | P2/vNext | 对齐 HTTP/SOCKS5/ProxyCommand 数据、Identity 或手工 username/password、创建/查看/编辑/复制/删除、label/host/command/type 搜索、空结果、引用计数、grid/list 与手动顺序；删除前列出并修复 Host/分组引用 |
| ProxyCommand | 执行不适用、数据兼容 v1.0 | 依赖本地命令执行，移动端不执行；导入、同步或编辑其他字段时保留 command 档案并显示 unsupported，不静默改写为 HTTP/SOCKS |

## 5. SSH 终端与会话

### 5.1 核心终端

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 交互终端 | P0/v1.0 + Gate | 渲染形态由 Phase 0 对照实验决定；支持常用 VT/xterm 控制序列、ANSI 色、UTF-8、宽字符、resize 和 alternate screen；协议兼容清单由自动化 corpus 给出 |
| 字符编码 | P1/v1.1 | UTF-8/GB18030 等运行时切换；切换不破坏已接收原始日志 |
| 回滚与搜索 | P1/v1.1 | 可配置 scrollback；桌面搜索固定为不区分大小写、非正则、非整词，支持上/下一个；清空或关闭搜索移除高亮，关闭后恢复终端输入焦点。移动若增加大小写/正则/整词开关须标为新增，并为触控与外接键盘保持同一结果 |
| URL 与终端超链接 | P1/v1.1 | 普通 URL 与 OSC 8 链接均只允许 `http(s)`；危险或未知 scheme 拒绝打开。OSC 8 在跳转前显示完整目标并二次确认；插件贡献的终端链接经过同一 scheme、用户手势和外部打开安全门。手机用点按/长按，外接键盘可保留可配置修饰键，取消或打开失败后终端焦点与内容不变 |
| 大输出与流控 | P0/v1.0 | 测试设备与数据集固定；UI 不假死。原生 transport、bridge、JS/UI 与渲染器分别有有界队列；输出只在实际消费/明确丢弃后 ACK，使用高/低水位滞回而不因每个 chunk 抖动。休眠、前后台重挂载、渲染器替换、snapshot/owner handoff 和关闭捕获前先 pause 并 drain；drain 失败时不保存过期快照、不解除到无人消费的 route。恢复后 byte 序列无丢失、重复和乱序；阈值、批量与 timeout 由两端真机 Gate 锁定，不复制 Electron 数值 |
| 中断与信号 | P0/v1.0 | Ctrl+C/D/Z、Esc、Tab、方向键等可达；外接键盘与屏幕键盘结果一致。持续洪水输出或 renderer backlog 下，Ctrl+C 的发送不得等待可见输出完全渲染；必须优先送达正确 session，并在中断前后保持输出顺序、ACK 账本和诊断 trace 一致，失败时有可重试的明确结果 |
| 复制/粘贴 | P0/v1.0 | 长按选择、全选、复制、粘贴；软换行/填充空格规范化可配置；敏感剪贴板给出风险提示 |
| 粘贴终端选区 | P1/v1.1 | 将当前终端选区直接作为用户输入发送，不经过系统剪贴板；应用与 bracketed-paste、复制规范化和 broadcast 的既有策略，空选区不发送。触屏用选区操作菜单，外接键盘保留可配置快捷键；不能以“先复制到系统剪贴板再粘贴”冒充等价 |

### 5.2 移动键盘与 Compose Bar

P0 键盘栏至少提供 Esc、Ctrl、Alt、Tab、方向键和可配置符号页；修饰键支持一次、锁定和取消三种状态并有明确视觉反馈。IME 组合文本只能发送已提交内容；中日韩输入、候选栏、语音输入、外接键盘、硬件 Ctrl/Alt/Meta 和横竖屏都进入真机矩阵。

P1 Compose Bar 支持多行编辑、可调高度、从终端选择内容填入、发送/仅粘贴、搜索和 pin/unpin snippets、固定项顺序与失效引用清理，以及广播目标提示。首次使用可从 Vault snippets 或内建集合播种，之后用户 pins 持久化；删除 snippet 后不能留下不可执行入口。密码提示场景不能把密码留在可恢复草稿或普通剪贴板。桌面当前还支持按 host/group path 动态解析 snippet/script targets；移动端后置实现必须保留路径规范化、重命名迁移和删除组后的安全收缩语义。

### 5.3 会话管理

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 多标签切换、重排、重连、重命名、复制 | P0/v1.0 + 复用 Gate | 连接中不可重复重连；复制使用新 session ID；名称可恢复默认动态标题。顺序可通过长按拖动或编辑排序调整，并提供外接键盘/无障碍上移下移；重排不改变 session ID、连接或当前活动标签。复制/新 shell 在安全上下文完全相同时复用已认证 SSH transport 以避免重复 MFA；host/profile、认证、跳板/代理、keepalive、SUDO 与 agent-forwarding policy 不匹配时必须新连，复用失败可回退且不能跨主机/凭据 |
| 重连代际与迟到事件隔离 | P0/v1.0 | 同一逻辑 session 的每次连接使用单调 `bootEpoch`/generation 或等价的 opaque owner。旧启动、host-key/MFA 响应、输出、exit、发行版/CWD probe 和 cleanup 不得更新、污染或关闭新连接；旧代际 close 对新 owner 为 no-op，代际不匹配可诊断但不泄露凭据 |
| 后台输出与 BEL 活动指示 | P0/v1.0 | 非当前会话收到经过控制序列过滤的可见输出或终端 BEL 时显示活动指示，进入对应会话后清除；该状态是瞬时 UI 状态，不进入 session restore，重启后不恢复。桌面 workspace 会聚合子会话活动；移动 v1.0 不实现 workspace，未来若交付标签分组再提供等价聚合。上游桌面现已解析 OSC 9/777/99 并按 `off/unfocused/always`、标题/正文清洗、长度上限和会话级限流发送系统通知；移动端仍不得把通知替代 activity dot，需另行定义权限、后台、锁屏脱敏和去重规则 |
| 编辑来源 Vault 主机 | P1/v1.x | 已保存主机发起的会话提供编辑主机入口并保持会话连接不变；ephemeral、local 和没有可解析 Vault 来源的会话不显示该动作，不能把会话快照误保存回 Vault |
| 关闭当前/其他/右侧/全部 | P1/v1.x | 批量关闭先检查忙碌会话；取消不关闭任何尚未处理项；活动标签选择可预测 |
| 忙碌确认 | P2/vNext | 能识别已知忙碌进程/前台任务；无法判断时不假称安全 |
| 广播输入 | P2/vNext | 明确目标会话；密码输入和 host-key 对话期间自动暂停广播 |
| 会话恢复 | P1/v1.1 | 恢复标签、自定义名称、活动标签和用户调整后的顶层标签顺序，不恢复输出/scrollback/进程；不持久化凭据或 ephemeral/password deep-link host；重连策略与 CWD 恢复分别配置，CWD 命令至多尝试一次 |
| 后台恢复 | P1/v1.x + Gate | 区分仍连接、系统挂起、连接丢失；回前台不重复发送启动命令 |
| 后台断开通知 | 移动新增/P2/vNext + 平台 Gate | 仅在用户授权且会话确实由 connected 转为 disconnected 时发送系统通知；内容默认不含主机、用户名、命令或输出，点击进入对应会话，重连成功、用户主动断开和重复事件不误报 |

多窗口、detach 和 workspace split 不进入移动端；可在 vNext 评估“标签分组”，但不得沿用桌面 workspace 的分屏承诺。

桌面另有与 workspace 不同的终端工具侧栏：SFTP、Scripts、History、Theme、System、Notes、AI 可排序、隐藏或收进溢出菜单；当前 7 种工具可在同一侧栏内水平/垂直拆成最多 7 个唯一 pane，domain 的 8-pane 硬上限为未来工具预留。移动端不能把它随 workspace split 一并判为“不适用”。vNext 需保留所有工具的可达入口和会话作用域；手机以单工具 sheet/全屏页 + 快速切换替代多 pane，平板是否支持并排工具由尺寸与触控 Gate 决定。关闭工具、切换会话和恢复后台时不能把某会话的工具状态错误复用到另一会话。

桌面的终端动作工具栏和主机树工具栏还支持 item order 与 `show/collapse/hide`，窄宽度时把部分 show 项临时收进 overflow；这与“工具侧栏 tab 排序”也是两套偏好。移动端可用显式“编辑工具栏”页面替代右键定制，但隐藏项必须能从设置恢复，锁定的最低可达动作不能全部隐藏，协议/会话不支持的动作要区别于用户隐藏。System Manager 的子 tab 使用相同的 order/placement 结果，在第 9.3 节单独处置。

### 5.4 终端设置与增强

| 需求簇 | 优先级/版本 | 范围 |
|---|---|---|
| 外观 | P1/v1.1 | 主题、字体/字号、常规/粗体字重、行距、fallback font、ligatures；迁移保留 `drawBoldInBrightColors`（默认 true，会改变 ANSI 粗体颜色且进入桌面同步 payload），即使移动首版不提供单独 UI 也不得静默反转或删除；桌面 WebKit font smoothing 与 `auto/webgl/dom` renderer 属平台实现项，移动端不暴露无效同名设置 |
| 仿真与光标 | P1/v1.1 | TERM `xterm-256color/xterm-16color/xterm`、光标形状/闪烁/当前行高亮、最低对比度 |
| 粘贴与剪贴板 | P1/v1.1 | bracketed paste、OSC 52 off/write-only/read-write/prompt、shell `clear` 是否擦除 scrollback；粘贴远端会话中的系统剪贴板图片可通过 SFTP 上传到已确认的远端 CWD 并回填路径，读取/上传失败不改发其他剪贴板内容 |
| Clear Buffer 动作 | P1/v1.1 | 长按菜单或固定可达入口提供应用级清缓冲。桌面基线仅在 xterm normal screen 清当前视图，可按 `clearWipesScrollback` 擦 scrollback；后端 `clearPty` 只为本地 Windows ConPTY 同步光标，SSH/Unix PTY 是 no-op。移动端新增更强契约：清理移动渲染器中会在重挂载时回放的本地缓存，且不得向远端发送 `clear` 命令冒充完成；是否保留 scrollback 由移动设置明确 |
| 滚动与选择 | P1/v1.1 | input/output/key/paste 滚动、smooth scrolling、copy on select（外接键盘/触控适配）、复制规范化、输入时保留选区、可配置词分隔符和真正的词边界选择。桌面设置虽把右键动作标为 `select-word`，当前 callback 实际调用 `selectAll()`；这是桌面现有缺陷，移动端不得把全选冒充选词，触控双击/长按与外接指针均以同一 word-boundary fixture 验收 |
| 字号临时缩放 | P1/v1.1 | 提供触控可达的增大、减小和恢复默认入口；是否支持双指缩放由终端组件 Gate 决定，不能只依赖桌面 Ctrl/Meta+滚轮。外接键盘快捷键与触控结果一致；迁移 `disableTerminalZoom` 时禁用所有临时缩放入口，但不阻止在设置中修改持久字号 |
| 标签与退出 | P1/v1.1 | 动态标题按全局 `off/agent/all` 模式处理。`Host.disableDynamicTabTitle` 当前仅存在于桌面模型、schema 和 importer，运行时未消费；迁移必须保留该兼容字段，但在产品决定把它实现为移动新增的主机级覆盖前，不得声称桌面或移动已支持“主机禁用动态标题”。开启自动关闭时仅 `reason=exited && exitCode=0` 的确认正常退出关闭标签；非零/未知退出、timeout、transport error 和 channel close 保留标签、输出与诊断并标记 disconnected；关闭设置后所有退出均保留 |
| TUI 手势优先级 | P1/v1.1 + 真机 Gate | alternate-screen/TUI 开启 mouse tracking 时，远端触控/鼠标手势优先级明确；仍提供始终可达且不依赖终端右键的应用操作入口（重连、Clear Buffer、关闭、工具），避免强制菜单与 TUI 输入互相吞噬 |
| 移动资源回收 | P1/v1.1 + Gate | 后台/隐藏会话可释放渲染资源但不得伪装连接仍存活；恢复不重复输出或输入；替代桌面 renderer/hibernate 调节项 |
| 外接键盘映射 | P1/v1.1 | Alt/Meta、Option+方向键、Shift+Enter、Tab 编号和自定义快捷键按平台处置；与系统保留手势冲突时显式降级 |
| Kitty keyboard | P2/vNext + Gate | 远端 capability 探测、启停、兼容回退 |
| Autocomplete | P2/vNext | ghost text/popup 二选一、debounce/min chars/max、host/global history scope |
| 密码提示辅助 | P2/vNext | off/hint/picker；不自动提交，凭据选择可取消 |
| Server Stats/主机信息 | P2/vNext | 展示 hostname、OS、kernel、uptime、load average、总 CPU/核心数/每核 CPU、内存 used/free/buffers/cache、swap、内存前十进程、所有挂载磁盘、总计/各网卡 Rx/Tx 和 SSH endpoint TCP latency。Linux/macOS 支持，网络设备不探测；刷新 5-300 秒、默认 5 秒，连续 3 次硬失败后停止轮询，Mosh handshake `pending` 不计失败，手动刷新/重新启用可重试；连接代际变化后丢弃旧响应 |
| 显示增强 | P2/vNext | 全局与主机级关键词正则、label/color、启停、增删改、清空、无效正则反馈及合并/覆盖规则；主机级规则可从活动终端直接编辑，并持久化回对应 Vault 主机；另含行时间戳和 force prompt new line |
| Inline images | P2/vNext + Gate | Kitty/SIXEL/iTerm IIP 独立开关、存储 MB、单图 MP、序列 MB 限制 |
| ZMODEM | P2/vNext + Gate | 终端协议检测、发送/接收、进度/速率/文件序号/最终确认和取消可观察；桌面拖放上传由移动文件选择/分享替代，远端 `rz -y` 缺失时的 SFTP fallback 明确提示。远端同名文件提供 overwrite/skip/cancel 与 apply-to-rest，不与普通 SFTP 队列、inline image 或 YMODEM 生命周期混用 |
| YMODEM | P2/vNext + Gate | 仅 Serial，会话内单任务互斥；显式选择发送文件或接收目录，覆盖 CRC、ACK/NAK、超时/retry、远端 cancel、进度和二进制完整性。接收文件名清洗并生成不覆盖的本地唯一目标名，不复用 ZMODEM 的远端 overwrite/skip/apply-to-rest 交互 |

### 5.5 历史和日志

- 远端命令历史: 按 host/global 作用域搜索、运行/粘贴、删除、存为 snippet。
- 连接日志: 记录连接目标、协议、开始/结束时间等元数据，并可带连接结束时捕获的完整 scrollback 供只读回放；支持收藏/取消收藏、删除和打开有快照的回放。桌面保留最多 500 条未收藏记录及全部收藏记录，最近 50 条未收藏回放使用独立 side store，收藏回放持续保留；移动端可另定经容量测试得出的上限，但必须显示保留策略和“无回放可用”状态。
- 会话日志: 自动或手动记录，格式 `txt/raw/html`，可选时间戳；支持查看、导出和清理。raw 保存未经有状态渲染的终端流，txt/html 维护 cursor、erase 与 ANSI 状态后写出可读快照；进入 alternate screen 的 TUI paint 在渲染日志中省略以保留 shell 历史。三种格式必须分别说明输入单位、编码、时间戳插入和可回放/可读边界，不把 rendered log 宣称为 raw 字节等价物。
- 三者存储、保留、隐私和清空行为分别定义，不能合并为“历史”。

## 6. SFTP、SCP 与传输中心

### 6.1 浏览协议与目录

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 协议 `auto/sftp/scp` | P1/v1.1 + Gate | `auto` 先 SFTP，子系统不可用时降级 SCP；用户可强制协议；结果显示实际协议 |
| SCP 能力差异 | P1/v1.1 | 明确远端需 shell/scp 命令；不支持项禁用并解释，不静默按 SFTP 语义执行 |
| list | P1/v1.1 | 桌面 SFTP 的列表视图；移动默认列表，支持排序、过滤、隐藏文件和稳定的可见列；不要与 Vault 的 grid/list/tree 混淆 |
| tree | P2/vNext | 桌面 SFTP 的树视图；移动 v1.1 不得把列表伪装成树，后续实现需保留目录展开、链接身份和路径结果 |
| 浏览操作 | P1/v1.1 | 排序、刷新、过滤、隐藏文件、新建文件/目录、重命名、删除、chmod、复制路径 |
| 符号链接 | P1/v1.1（list）；P2/vNext（tree） | v1.1 列表区分普通文件、目录、文件链接、目录链接与断链；目录链接可导航但持续显示链接身份，目标不可达时不冒充空目录。vNext tree 在展开节点中保留同样的链接身份与路径结果。打开、删除、重命名、chmod、冲突和目标路径提示明确作用于链接本身还是目标，SFTP 与 SCP fallback 结果一致或显式降级 |
| 编码 | P1/v1.1 | `auto/utf-8/gb18030`；编码错误不损坏原文件名 |
| 书签 | P2/vNext | 主机级和全局；无权限/不存在路径有恢复入口 |
| SUDO | P2/vNext | 明确支持矩阵；密码处理与 sudo prompt assist 一致 |
| 多标签与浏览偏好 | P2/vNext | SFTP 标签新建/复制/关闭/排序、每主机路径/视图记忆、目录优先、`name/modified/size/type` 可见列与列宽、键盘 typeahead；名称列始终可见；桌面的跨左右 pane 移动在单窗格移动端不适用，改为标签排序和显式选择来源/目标标签 |
| 文件工具栏布局 | P2/vNext | bookmark、终端 CWD 双向定位/跟随、复制路径、视图、过滤、编码、新建、隐藏文件、刷新等动作保留结果；移动以编辑模式实现 order 与 show/collapse/hide，窄屏 overflow 不改写持久偏好，刷新等锁定动作保持可达 |
| 归档解压 | P1/v1.1 + Gate | 上游桌面支持对常见 `.tar`/`.tar.gz`/`.tar.bz2`/`.tar.xz`/`.tar.zst`/`.zip` 及单文件压缩格式执行安全解压，并按远端工具可用性回退；移动端以长按菜单替代右键，必须限制格式、正确转义路径、使用临时目标/原子替换、显示阶段/取消/失败结果，不得把归档名拼接成任意 shell 命令 |
| 触屏打开/传输与自动打开文件工具 | P1/v1.1 | 桌面的双击“打开/传输”在触屏上改为单击默认动作 + 长按菜单；连接后自动打开文件工具和队列并发属于同一版本验收，桌面“自动打开侧栏”改称自动打开文件工具，不能因没有侧栏而删除结果 |
| 浏览与入口偏好持久化 | P1/v1.1（list、隐藏文件、跟随终端 CWD）；P2/vNext（tree 默认视图及其余未交付偏好） | 逐项迁移并显示当前实际生效值；移动端未交付的 tree 或偏好不得伪装成已支持，窄屏 overflow 不得静默改写持久偏好 |
| 传输调优偏好 | P1/v1.1 + Gate | 压缩上传保留自动回退；“跳过未变化文件”必须说明比较依据和误判风险；共享 SSH transport 的 idle TTL `1/5/15/30 min/never` 属桌面连接池调优，移动端由原生连接所有权/电量 Gate 决定是否暴露等价选项，否则使用有文档的固定策略 |

### 6.2 上传、下载与编辑

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 文件/目录上传下载（移动进度契约） | P1/v1.1 | 系统 picker 支持单/多选并替代桌面本地 pane、拖拽和 OS 文件剪贴板入口；目录能力依平台权限降级。普通目录不做全树预扫描，目录发现与文件传输交错；桌面当前只提供已发现文件数上的 soft progress，并没有显式 discovery-complete/final-total freeze。移动端验收必须分别显示已发现文件/字节、已传输字节、速度和发现阶段，遍历完成前百分比与全目录 ETA 保持不确定或明确标为仅基于已发现量，不能伪装成稳定总进度；发现完成后才固定最终总量并计算确定 ETA |
| 目录链接传输策略 | P1/v1.1 | 方向分别定义：远端目录下载可展开目录链接并复制目标内容；上传和远端复制/移动默认不得静默递归展开链接。遍历以 canonical path 检测当前分支循环，并设置可测试的链接深度、目录数和条目数上限；达到上限、断链或目标变化产生明确 failed/attention/skip 结果，不无限递归。重启恢复使用同一排序和 manifest 语义，不能因重新扫描改变已完成项或跨过安全上限 |
| 压缩上传 | P1/v1.1 | 可开关；这是与普通增量目录传输分开的全量扫描路径，扫描/压缩/上传/解压阶段可见；远端缺少工具时回退普通传输并切换到未知总量语义 |
| 暂停/恢复/取消 | P1/v1.1 | 状态机包含 pausing/paused；取消后清理临时目标；不可暂停时说明原因 |
| 断点续传 | P1/v1.1 + Gate | App 内与进程重启恢复范围分别标明；源/目标指纹变化转 attention，不盲续 |
| 远端复制/移动 | P1/v1.1 | 文件和目录；跨文件系统失败可回退复制+删除且显示风险 |
| 文本编辑 | P1/v1.1 | 内置编辑器默认上限 10 MB；语言模式、自动换行、搜索、内容统计和保存错误可见；超限转只读/系统应用或拒绝；保存冲突显示远端变化 |
| 多编辑器与未保存状态 | P1/v1.1 | 可把远端文件提升为独立编辑器标签并在多个编辑器间切换；每项独立保留 content/baseline、dirty、word-wrap、language 和 view state。移动端在关闭编辑器/SFTP tab、断开或替换 SFTP、关闭 owning terminal/session/workspace、关闭所有、安装重启或退出前统一提供 Save/Discard/Cancel；Cancel 不继续破坏性动作，保存失败保留 dirty 内容和重试入口。这是对桌面路径差异的保护增强：桌面手动关闭 SFTP tab/断开会逐项确认，App quit 只阻止退出，而关闭承载 SFTP panel 的终端会随 owner 卸载强制丢弃 dirty editor |
| 系统打开/关联 | P2/vNext | default opener 与扩展名关联；document provider 临时文件安全回传并可清理 |
| 外部编辑自动同步 | P2/vNext + Gate | 桌面基线是本地临时副本变化后单向上传覆盖远端，没有远端版本/冲突检测；移动端必须选择“保存前远端版本校验并提示”“显式确认回传”或“禁用自动回传”，并定义 URI 权限、App 被杀和临时副本清理边界 |
| 跟随终端 CWD | P1/v1.1 | 可自动开文件面板并定位；无 OSC 7/无权限时回退 home；主机覆盖全局设置。提供“配置目录追踪”动作：预览将写入的远端 shell 配置文件和命令，确认后为当前远端用户安装/升级带版本标记的 OSC 7 hook；覆盖 bash/zsh/fish、sudo/su 用户、已配置、部分 marker、所有权/权限、unsupported shell、失败和取消，不能静默改写 rc 文件 |
| 反向在终端打开 | P2/vNext | 正确 shell quoting；网络设备模式禁用或明确处理 |

### 6.3 冲突与队列

冲突动作完整集合为 `stop/skip/replace/duplicate/merge`。目录只有目录对目录时可 merge；不安全类型组合禁用 replace；同类冲突可 apply-all，范围和剩余数量可见。

传输中心至少支持:

- pending、queued、transferring、pausing、paused、attention、interrupted、completed、failed、cancelled；
- 单项暂停/恢复/取消/重试/优先处理/dismiss；
- 全部暂停/全部恢复；
- 文件、目录、远端到远端和后台任务；
- 过滤 active/queued/paused/attention/completed；
- App 重启后的任务恢复、adopt/reconnectRequired 和源变化处理；
- 定位目标文件/目录，失败或已删除时给出提示。

SFTP 内部文件剪贴板与系统文件剪贴板是两条路径。内部路径支持文件/目录 copy、cut、paste，来源包含连接、目录和桌面 pane；可跨 pane、跨连接形成远端到远端传输。移动单窗格必须提供等价的来源标签/路径与目标标签/路径选择，不能依赖同时可见的左右 pane。cut 仅在目标完成且源删除成功后移除对应项；部分失败项保留在内部剪贴板并可重试。系统剪贴板中的文件仍走上传预览和系统 URI 权限，不与内部 cut 状态混用。

并发概念分开:

1. 文件/目录 worker: 桌面默认 2，可配 1-16；移动端根据设备制定默认值，但配置范围和资源上限要有测试。
2. 递归目录发现 listing gate: 桌面当前固定默认 4，helper 接受 1-8，但实际调用没有读取用户设置；它限制并行 OPENDIR/READDIR，不等于同时传输文件数。移动是否固定或暴露设置由目录规模、内存和共享 SFTP session 真机 Gate 决定。
3. 单文件 SFTP 请求 fanout: 桌面上传/下载为 64 个 32 KB 请求窗口；这是底层调优，不作为 UI 队列并发。

### 6.4 大文件 Gate

4 GB+ 传输是产品目标，不是本地代码可证明的移动能力。v1.1 承诺前必须在 Android/iOS 真机验证:

- 文件 API 和整数宽度不截断；
- 全程流式，不整文件进 JS/原生内存；
- 断网、杀进程、磁盘不足、目标变化后行为可预测；
- 上传/下载 hash 一致；
- 后台限制和系统文件权限有明确降级。

## 7. 端口转发

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 本地 `-L`、远程 `-R` | P1/v1.2 + Gate | bind address/port/remote target 校验；状态 connecting/active/error/inactive 可恢复 |
| 动态 `-D` SOCKS5 | P2/vNext + Gate | 定义 IPv4/IPv6/域名/认证范围；不沿用未经验证的桌面限制描述 |
| CRUD、复制、排序、搜索 | P1/v1.2 | 编辑连接字段时先安全停止旧 tunnel；复制不继承 runtime status/error；支持 manual/A-Z/Z-A/newest/oldest，手动移动后回到 manual 且顺序持久化 |
| 显示与新建流程偏好 | P1/v1.2 | grid/list 选择可持久化；移动可只提供响应式列表但必须给出明确替代并保留已有偏好数据；wizard 与完整表单可互相切换，草稿字段不丢失，用户选择决定下次默认流程 |
| 多规则 | P1/v1.2 | 不假定桌面硬上限；移动资源上限由真机负载测试决定并向用户显示 |
| 跳板与代理 | P1/v1.2 | 使用主机完整认证/代理/host-chain 解析 |
| 自动启动与网络恢复 | P1/v1.2 | 仅配置完整且认证可用的规则启动；网络恢复重试有次数上限；用户停止取消重试 |
| 生命周期清理 | P1/v1.2 | 停止/删除/会话关闭/App 退出后释放 listener、forward 和 accepted sockets |
| host key | P1/v1.2 | 校验隧道 SSH 主机和每级跳板；不声称验证普通 TCP remoteHost |
| 流量统计 | P2/vNext | bytes/速率口径和采样周期固定；不因统计阻塞转发 |

## 8. Keychain 与身份

| 需求 | 优先级/版本 | 验收 |
|---|---|---|
| 导入、查看、删除 | P0/v1.0 | PEM/OpenSSH 等承诺格式有 fixtures；显示公钥/证书元数据；删除前列出 Host/Identity 引用。旧 biometric/FIDO2/passkey/WebAuthn 实验记录作为不可执行归档单列并解释需重新录入，不得静默转换为普通 SSH key；清空 Vault 前将归档纳入删除预览 |
| 生成 | P1/v1.2 | ED25519、ECDSA P-256/384/521、RSA 1024/2048/4096；默认采用安全强度，弱选项有警告 |
| passphrase | P0/v1.0 | 可不保存/安全保存；系统认证失败时不降级明文存储 |
| reference key/identity file | P2/vNext | 文件引用失效可修复；系统 document URI 生命周期明确 |
| 证书 | P1/v1.2 + Gate | 公钥证书与私钥配对校验；RSA/ECDSA/ED25519 双端支持分别通过真机测试 |
| Identity | P1/v1.2 | 创建、查看、编辑、删除 username + password/key/certificate；按 label/username 搜索，空集合与无匹配分别显示；读取并保留已有手动顺序，但不把 vNext 的重排 UI 伪装成 v1.2 能力。可被主机/分组/代理/快速连接复用，删除前列出引用并要求替换、清除或取消，不留下静默失效项 |
| 公钥部署 | P2/vNext | 选择目标主机，通过 SSH exec 写入默认 `~/.ssh/authorized_keys` 或自定义位置；成功后可关联 Host/Identity |
| 本地公钥导出 | P1/v1.x | 可复制/分享完整公钥并反馈成功、拒绝或失败；不得夹带私钥、passphrase 或内部存储标识，不与远端部署混名 |
| 私钥导出 | P2/vNext + 安全评审 | 默认关闭或高摩擦确认；目的地、分享目标和剪贴板风险可见，不写入普通日志 |
| 集合显示与顺序 | P2/vNext | keys 和 Identities 提供 grid/list 偏好、用户手动重排、取消/回滚及外接键盘/无障碍上移下移；核心版本只需保留已有顺序和未知偏好字段 |

## 9. Snippets、脚本及后续功能

### 9.1 Snippets

vNext 对齐分类/包、tags、目标主机/全部主机、变量替换、shortkey、`noAutoRun`、多行 `lineDelay/paste`、终端插入、onConnect、导入导出和分享。保留 grid/list、manual/A-Z/Z-A/newest/oldest、snippet 与 package 的手动顺序及包内/包间移动；移动端用触摸编辑模式实现。快捷触发用 Compose Bar pinned snippets、外接键盘和系统 Shortcut 设计，不照搬桌面全局键位；pin/unpin、搜索和删除后的引用清理由第 5.2 节验收。

### 9.2 脚本自动化

脚本创建/编辑、录制、目标主机/全部主机、manual/onConnect/onOutput（正则）触发、每主机有序 onConnect 队列、运行列表/日志、进度，以及运行的停止/暂停/恢复进入独立设计。桌面 `language: python` 当前只是 UI/数据标签，实际执行器仅运行包装后的异步 JavaScript `nct` API；移动端不得把 Python 标成已支持，也不得直接假设可使用 Node `vm`/`child_process`。需定义移动脚本语言和版本化 API（screen wait/send/read/clear、session/log、dialog、progress）、写操作审批、触发去重、超时/取消、资源限制、后台策略和商店政策 Gate；Observer 模式不得执行写脚本。

### 9.3 AI、MCP、插件、系统管理器和协议

| 能力 | 处置 |
|---|---|
| Catty API 型 AI | vNext；保留 provider/model 管理、终端/会话作用域、附件与终端选区、会话历史/删除/恢复、停止与支持时的 turn steering、工具活动/产物、token/compaction 状态、对话导出和错误恢复；API key 存储、工具权限/审批、后台和隐私需独立设计 |
| Catty 72 项工具目录 | vNext；当前生成目录按 capability 域为 session 2、attachment 2、terminal 4、SFTP 11、Vault 39、portforward 8、harness 6。移动端按下表逐域处置，数量是审计基线而非永久 API 承诺 |
| AI 工具与安全设置 | vNext；observer/confirm/auto、permission grants、当前仅持久化/同步而未接入 Catty 执行的 `HostAIPermission`、command blocklist/timeout/max iterations、web search、quick messages、user skills/MCP 模式和模型/agent 映射逐项处置；移动默认使用 Confirm，停止仍必须走统一 turn lifecycle；若让 Auto 受 host policy 约束，须标为移动安全增强而非桌面现状 |
| Codex/Claude/Copilot/Cursor/CodeBuddy/OpenCode/Grok 等本地 CLI/SDK agents | 桌面本地进程、binary discovery、app-server/ACP/stdio runtime 不适用；若未来有供应商远程 API，作为新的 API agent 单独验证，不承诺复用桌面 thread/session identity |
| MCP | 远程/HTTP client 可评估；本地 stdio server、CLI discovery 和 Netcatty launcher 集成不适用 |
| AI/MCP 静默会话 | vNext；桌面静默会话不进入 tab、Quick Switcher 或 session restore，但可从托盘发现并以 popup 附着同一 live PTY，且受空闲超时清理。移动端不复刻 BrowserWindow，也必须明确 supported/degraded/blocked；若支持，需定义创建者 scope、发现/观察/关闭、空闲终止、App 被杀和不得泄露为普通恢复会话的规则 |
| 插件管理与沙箱 | vNext 研究；安装、启停、重启、卸载、版本快照、权限/secret/配额、离线 view 沙箱和高级 utility process 均需移动商店与运行时 Gate，不能直接复用 BrowserWindow/utilityProcess/companion executable |
| 插件贡献与 providers | vNext 研究；settings/commands/keybindings/menus/views，以及 terminal completion/decoration/link/hover/matcher/semantic/prompt/background/theme、input/output interceptor、connection/authentication/sync/importer provider 必须按 kind 分别声明 supported/degraded/blocked；importer 保留格式检测、选择 token 生命周期、流式解析进度/取消、有界安全预览、字段规范化，以及带 journal 的事务提交和崩溃回滚；移动端不能仅写“插件 host 后置”便默认这些用户能力已覆盖 |
| 系统管理器 | vNext；概览、进程、监听端口、systemd 服务、tmux、Docker containers/images、NVIDIA/Ascend GPU 与对应刷新、搜索、能力探测、危险操作确认逐项处置；子 tab 的 order 与 show/collapse/hide 可恢复，Overview 等锁定入口不能全部隐藏；网络设备等不支持 shell 工具的主机显式降级 |
| telnet | vNext + 安全警告/Gate；支持独立 port/Identity 或 username/password、prompt 驱动自动登录、IAC negotiation/escaped IAC、NAWS resize、远端 echo 与字符集；明确所有流量含凭据为明文，警告不可被普通默认值掩盖 |
| mosh | vNext + 原生 UDP/协议 Gate；验证 SSH bootstrap 的认证与 host key、解析 `MOSH CONNECT` 后切换 UDP、切网/休眠恢复、resize、UTF-8 locale、自定义 `mosh-server` path 和 companion stats；不能复用桌面外部 `mosh-client` binary 路径 |
| et | vNext 研究；保留自定义 ET server port、认证/host-key 与连接统计结果；桌面外部 binary 方案不可直接复用，移动若无可维护原生协议栈则明确 blocked |
| serial | vNext + Android USB OTG/iOS 外设 Gate；逐项处置 port、任意正 baud rate、5/6/7/8 data bits、1/1.5/2 stop bits、none/even/odd/mark/space parity、none/XON-XOFF/RTS-CTS flow control、local echo、line mode、backspace 与 charset；拔线、权限撤销和前后台清理有终态 |

Catty 工具域的用户结果与契约基线如下。这里审查的是 in-app sidebar catalog；按仓库 Review Boundaries，不把内部 CLI、discovery、TCP bridge 或 RPC dispatch 当作面向第三方的公共 API:

| 工具域 | 数量 | 必须处置的结果 |
|---|---:|---|
| session | 2 | 列出当前 chat scope 可见的环境/会话；`session_close` 只能关闭该 scope 内会话，但作为清理能力可绕过 Observer、Confirm 审批和 chat-cancel 阻断，仍须遵守所有权与忙碌状态 |
| attachment | 2 | 只列出和读取用户附加到当前 chat 的文件；路径、大小、敏感读取和会话删除后的生命周期受限 |
| terminal | 4 | execute/start/poll/stop；长任务有稳定 ID、增量输出、超时、取消确认和唯一终态。`terminal_stop` 作为清理能力可绕过 Observer、Confirm 审批和 chat-cancel 阻断；它只停止指定后台 job，不等于停止 agent turn |
| SFTP | 11 | list/read/write/download/upload/stat/home/mkdir/delete/rename/chmod；复用 host-key/认证/scope，写入/传输按策略审批，大输出走 handle |
| Vault | 39 | Hosts、Host notes、Vault Notes、Identities、proxy profiles、groups、snippets、scripts 与 connect-script bindings 的读写；永不向模型返回 password/private key，引用/删除/导入语义与对应产品功能一致 |
| portforward | 8 | rules/tunnels list、create/update/duplicate/delete/start/stop；start 和配置写入受审批，host scope、host key 与 listener 生命周期不降级 |
| harness | 6 | 大输出 handle read、workspace/session info、terminal context、web search、URL fetch；web 未配置时不暴露，网络读取遵守 provider/隐私限制 |

所有域共同保留以下行为契约:

- scope 在服务端/adapter 校验，不能只靠提示词；chat、terminal、workspace 与 host ID 不得越界复用。
- 桌面 Catty 的 Observer 拒绝非豁免写操作；Confirm 对需审批的写能力请求用户确认。“Approve once”只批准当前调用；普通 Catty/MCP 的“Always Allow”会持久化 grant，当前 matcher 按 capability/command/args 全局匹配并忽略 `sessionPattern`，不是 chat/session 级授权。Codex App Server 的“allow for session”是其原生运行时会话决策，不写入 Netcatty 持久 grant。Auto 当前不调用审批或 grant 匹配，写能力会直接通过审批层，但仍受 capability/scope、适用的 command blocklist、取消和资源上限等执行约束。`HostAIPermission` 当前只有类型、存储和同步，未传入 Catty turn，不能当成已生效的桌面安全边界。移动默认保持 Confirm；若产品要求 Auto 也受 host allow/deny policy，必须作为移动安全增强实现并验证 fail closed、迁移和策略冲突。
- 审批拒绝或超时只把当前工具调用记为 denied，模型 turn 可继续；`session_close` 和 `terminal_stop` 也分别属于会话/job 工具生命周期。只有停止按钮、`/stop` 等明确 turn-stop 入口调用统一 `stopAgentTurn()`，清理 pending approvals、取消当前 turn 并形成 aborted 终态。工具取消与 turn 取消都须幂等，迟到结果不能落入下一 turn。
- command blocklist 是 shell-like `terminal_execute`/`terminal_start` 的纵深防护；桌面明确对 Serial 和 SSH network-device CLI 跳过，因为相同文本具有不同语义且 raw PTY 不按 shell 解释。移动端要保留这一区分，以协议/device metadata 决策，并用 scope、审批和能力限制覆盖不适用 blocklist 的会话，不能宣称所有命令都由同一 shell 正则保护。
- `ToolOutputStore`/`tool_output_read` 的截断 handle 在 chat session 内持续、删除 chat 时清理；重复读取提示和敏感输出策略保留，不把截断文本冒充完整结果。
- activity、artifact、usage、performance、step/turn end 与 compaction 事件可恢复到历史；retryable warning 不冒充成功，terminal lifecycle 以唯一终态收敛。

## 10. 设置、平台与安全

### 10.1 设置范围

v1.0 提供主题（明/暗/系统）、语言、凭据保护状态、诊断基础、关于/版本/隐私/反馈；App 锁仍是 P0 候选，只有第 13 节产品决策纳入后才进入 v1.0 设置范围。桌面当前明确支持 `en`、`es`、`zh-CN`、`zh-TW`、`ru` 五种 locale；移动 v1.0 保持这五种语言并定义系统 locale fallback，任何暂缺翻译都回退到英文而不是显示 key。桌面的云同步 master-key `NO_KEY/LOCKED/UNLOCKED` 只控制同步操作和 auto-sync，不锁 Vault、终端或整个应用；移动 App 锁必须使用独立状态与生命周期，不能把“同步已解锁”冒充“App 已解锁”。v1.x 增加终端、SFTP、传输、恢复和平台更新设置；vNext 增加设置搜索、自定义 UI/终端主题、iTerm2 导入、文件关联、应用图标变体、诊断导出和清空 vault。应用图标变体在移动端属于平台能力，需要确认 Android launcher alias/iOS alternate icons、系统确认 UI、失败回滚和商店素材，而不是直接复用桌面图标文件。

`domain/settingsSearchCatalog.ts` 当前有 116 个可搜索设置入口，它只是稳定的反向审计种子，不是全部设置总数；实际页面还有未被搜索索引的子设置和高级调优项。移动设置 IA 可以合并页面，但这些用户结果必须逐项处置，并继续反查各设置页的 `SettingRow`/嵌套表单:

| 设置簇 | 移动处置 |
|---|---|
| Application | 检查更新、报告问题、社区、GitHub、What's New 保持可达；更新入口使用商店/允许的 OTA 流程，外链失败和离线有反馈 |
| Appearance / Vault | UI 字体、明暗/系统主题、主题色/强调色、应用图标按平台等价；窗口透明度和 custom CSS 不适用；最近主机、激活方式、根目录过滤、SFTP 入口、主机树入口及系统 known_hosts 自动导入按第 3/4 节替代或后置 |
| Terminal | 主题跟随、字体/CJK fallback、字号/字重/行距、粗体亮色兼容字段、TERM、光标、对比度、键盘、复制粘贴、scrollback、自动关闭/重连、keepalive interval/count、工具默认 pane、关键词、host info/server stats 与刷新周期、renderer/隐藏 renderer 回收调优、inline image 协议与容量限制、补全模式/scope 和密码提示分别落入 T-01 至 T-12；System Manager 各刷新周期落入 X-04。右键/中键、本地 shell、X11、workspace focus UI 和 Web renderer/WASM 实现名不适用或平台替代，不保留无效同名开关 |
| Shortcuts | 桌面预设 keymap、自定义 binding、禁用终端字体缩放、仅 shell 标签参与数字快捷键、显示标签编号徽标都进入外接键盘设计；系统保留组合键不可覆盖，手机无键盘时不显示无效控制，App Shortcuts 另属系统入口而非 keymap |
| File associations / SFTP | 双击行为改为触摸默认动作；list/tree、隐藏文件、外部编辑自动回传、压缩上传、跳过未变化、跟随 CWD、自动打开文件工具、传输并发、transport idle TTL、默认 opener 和按扩展名 opener 规则按第 6 节逐项等价/替代/Gate |
| AI | provider 与 endpoint/model/API key、各 agent（含未进入搜索目录的 OpenCode/Grok）、默认 agent、选区动作、工具访问、MCP mode/idle/focus/silent session/launcher/discovery、skills/quick messages、web search provider/key/host/max results，以及 permission/timeout/max iterations/blocklist/grants 按第 9 节逐项后置；本地进程型 agent 明确不适用 |
| Sync | 本地保护备份随核心数据保护在 v1.2 交付；provider、自动同步、`smartMerge/preferCloud/preferLocal`、实验性 CRDT v2 migration/conflict/downgrade 和“清空本地同步状态”均为 vNext。清空前必须创建并验证可恢复备份，以事务方式清实体、base/anchor、provider baseline 和 canonical CRDT replica；完成后明确本地/远端状态及下一次动作，不能无条件承诺“下次从云端下载”，也不能与“永久删除云端”混用 |
| System / Plugins | 更新、HTTP 代理、凭据状态、临时目录、诊断、会话/CWD 恢复、会话日志、深链按本节和第 5/9 节处置；Explorer 菜单、全局 hotkey、close-to-tray 不适用；插件根入口随插件研究后置 |

桌面还有作用于云同步、AI provider 及部分外部进程环境的应用级 HTTP(S) 网络代理，模式为 system/direct/custom，custom 含 URL 与 bypass。移动端随首个需要出站 HTTP 的能力交付，最迟 vNext 云同步/AI 阶段逐项处置：明确它与 SSH 主机代理/ProxyJump 是两套设置；验证系统 VPN/代理、认证代理、PAC 不支持时的提示、TLS 例外、bypass 语义和秘密存储。不能写成“移动系统代理自动生效”后静默丢掉 direct/custom 结果。

更新机制必须区分商店二进制更新与允许范围内的 JS/资源更新，不直接照搬 Electron updater。验收状态至少包含 idle/checking/up-to-date/available/downloading/ready/error，显示版本与下载进度；不支持自动安装时回退到可信商店/发布页；失败可重试；安装或重启前检查未保存编辑器和未完成传输，不能静默丢失内容。

桌面 Windows packaged app 在 exe 或 portable launcher 旁存在 `data/` 目录时，会把 `userData` 和 `sessionData` 重定向到该目录。该能力依赖可执行文件旁可写 profile，在 iOS/Android sandbox 中不适用，移动设置不提供同名无效开关。跨设备迁移不承诺固定版本，必须先通过第 13 节稳定交换格式与凭据可移植性 Gate；只有产品选择 v1.2 受保护备份 payload 变体时才可由该版本承接，否则保持 blocked/待决。直接复制桌面 portable profile 不是支持的导入契约，Windows 用户绑定的密码/私钥密文也不能冒充可迁移凭据，必须在来源端解密后重新保护或在移动端重新输入。

移动诊断至少分为 crash、SSH 协议、会话输出和性能/中断四类。诊断包导出前必须预览类别、时间范围和大小，默认脱敏主机标识、用户名、命令参数、路径和文件内容，需用户明确同意；定义保留期、存储上限、打开/分享失败、逐类清理和全部清理。会话日志继续支持 `txt/raw/html` 与可选时间戳，但不能因“诊断导出”自动包含在包内。

桌面设置中的右键/中键、窗口透明度、close-to-tray、全局 hotkey、Explorer 菜单、custom CSS 明确不适用；不能为了字段对齐在移动 UI 暴露无意义开关。

### 10.2 安全底线

- 密码、私钥、passphrase、同步密钥和 API key 只进入平台安全存储或受保护加密包；移动端不可用时必须 fail closed。桌面当前 `safeStorage` 不可用或 `encryptString` 失败时，凭据 bridge 会原样返回输入值并可能持久化明文，启动时只给出凭据保护不可用警告；这不是移动端可接受的降级，也不能在迁移说明中写成桌面与移动安全存储等价。
- CSV 等明文交换格式默认不得携带密码、私钥或 passphrase；若产品决定保留桌面兼容例外，必须在导出前逐项揭示字段与风险、再次认证、限制分享目标、记录不含秘密的审计事件，并清理 App 控制范围内的临时文件。
- 复制 hostname、公钥和账密采用不同风险级别。复制明文账密必须先解析当前有效继承/Identity，要求近期设备认证或再次认证，写入前确认 App 仍在前台且未锁定；平台允许时设置短时过期或清除，无法控制系统剪贴板历史、云同步或其他 App 已读取副本时必须明确告知。无可读密码或写入失败不得保留旧剪贴板内容并显示成功。
- App 锁是移动新增而非桌面 parity：支持系统生物识别失败后的受控 fallback、超时/前后台触发和最近任务遮挡；锁定期间终端输入、剪贴板、文件预览、通知动作、深链和 Widget 不得绕过。活动会话、listener 和传输是继续、暂停还是断开的默认行为由安全评审决定，并在解锁后如实显示实际状态。
- 截屏/最近任务预览、通知内容、剪贴板和分享目标有敏感信息保护策略。
- host key、弱算法、私钥导出和不安全代理/TLS 例外均为显式用户决策。
- 不提供可让所有 SSH/SFTP/转发连接静默跳过 host-key 校验的普通全局开关；受管例外必须限主机/策略、持续显示风险并进入不含秘密的安全审计。
- 日志、analytics、crash report 和错误 UI 不包含明文凭据、私钥、完整命令敏感参数或文件内容。
- 本地备份和导入在覆盖前创建可恢复快照；清空 vault 明确清理范围和系统 Keychain 残留策略。

### 10.3 网络与后台

- Wi-Fi/蜂窝切换、DNS 变化、VPN、代理和 captive portal 分别测试。
- Android 前台服务只有在活动连接/转发/传输需要时启用，通知说明活动并提供允许的操作。
- iOS 不承诺无限后台 SSH；进入后台、被挂起和恢复后的状态必须如实显示。
- 自动重连采用有限指数退避；认证失败、changed host key、用户拒绝或主动断开不自动重试。该策略是移动安全增强；桌面事实是对曾连接成功的普通 SSH 固定每 5 秒持续重试，只有全局开关而无主机覆盖，不能把两者混写为桌面 parity。

### 10.4 深链、分享和权限

- 桌面对齐的 `ssh://` 深链为 P0/v1.0：先预览解析结果再连接；含密码的 URL 不写入历史、日志或持久化主机。
- `telnet://` 随 telnet 能力后置：解析 hostname/port/username/password 后先预览；URL 密码只作为一次性临时凭据，`savePassword=false`，不得匹配并污染已保存主机或进入历史/日志。
- `jms://` 保持后置：桌面 payload 是 base64/base64url JSON，包含 JumpServer 一次性 token、endpoint、asset 和 `ssh/sftp/telnet` 意图；移动端只能创建不保存 token 的 ephemeral 会话。`sftp` payload 连接 SSH gateway 后自动打开对应文件入口；过期、重放、日志/剪贴板泄露和不支持协议必须 fail closed。
- 本地网络、通知、文件、相机/二维码、生物识别权限按使用时申请；拒绝后其余功能可用。
- 下载文件系统分享是移动新增/P2/vNext：分享前确认文件已经完整落地且 MIME/type 正确；取消、目标 App 失败和临时 URI 授权回收不改变下载完成状态。私钥和敏感日志使用更严格确认。
- App Shortcuts 是对桌面全局快捷键/托盘入口的移动替代/P2/vNext：只暴露不含秘密的固定动作或最近对象 ID，失效对象回到安全页面，不在快捷项标题中泄露敏感主机信息。
- Widget 是移动新增替代/P2/vNext + 平台 Gate：只展示经用户选择的非敏感状态/快速连接入口；锁屏、App 锁、过期快照、删除主机和未授权状态均有降级，Widget 不直接持有密码、私钥或一次性 token。

## 11. 非功能验收与 Gate

所有数值在实施前写入版本化测试规范，至少包含设备、OS、构建类型、网络、数据集、采样方法、次数和失败阈值。以下是目标，不是桌面事实:

| 域 | v1.x 验收框架 |
|---|---|
| 冷启动 | 在约定低/中/高档真机各测至少 10 次，报告 P50/P95；“<2s”只有在基准设备和起止事件定义后才生效 |
| 终端吞吐 | 固定 10 MB ANSI/UTF-8 corpus、持续 `yes`/日志场景和交互延迟；报告 dropped frames、input-to-echo、峰值内存 |
| 传输 | 1 KB、10 MB、1 GB、4 GB+，局域网/高延迟/丢包；校验 hash、速度、终端并行交互和恢复 |
| 包体 | 分 Android ABI/AAB 与 iOS archive 报告；“60 MB”需先定义下载/安装/解压口径 |
| 稳定性 | 前后台、旋转、低内存、权限撤销、断网/切网、服务端重启、App 被杀、磁盘不足 |
| 无障碍 | VoiceOver/TalkBack、200% 字体、外接键盘、高对比度和 reduce motion |

### 11.1 Phase 0 强制 Gate

Android 与 iOS 真机分别验证:

1. trusted/unknown/changed host key；unknown 覆盖保存/仅本次/拒绝，changed 覆盖 fail-closed 初始停止与第 13.5 节最终选定的修复分支，未决的 override 不能作为 v1.0 既定验收项。
2. password/key/passphrase/certificate/keyboard-interactive/MFA。
3. 多级 ProxyJump、HTTP/SOCKS、逐跳错误和取消。
4. 候选终端 VT corpus、IME、外接键盘、大输出、回滚、选择/复制及 RN 原生/WebView 对照。
5. SFTP 与 SCP fallback、目录操作、暂停/恢复/取消、4 GB+ 流式传输；普通浏览/共享传输验证 endpoint 复用，重启恢复或显式专用传输验证独立重新认证/MFA、关闭交互终端不误停任务，以及专用连接不反向附着 parked transport。
6. 两个独立远端之间的 copy/cut/paste：来源和目标分别认证、逐跳 host key、冲突、部分失败、源删除失败与恢复；单窗格不依赖同时显示双 pane。
7. 系统应用编辑临时副本后的回传、远端同时变化、URI 权限撤销、App 被杀和清理。
8. 本地/远程转发、切网、后台/恢复和资源清理。
9. Keychain/Keystore 不可用、锁定、备份/恢复和跨设备迁移。
10. 许可证 notices、商店政策、iOS 本地网络和 Android 前台服务权限。

任一安全关键 Gate 不通过时，对应功能不得进入承诺版本；必须改库、自研、降级或调整范围。

## 12. 版本规划

下表是目标范围。任何标有 Gate 的能力只有在对应真机、安全、许可或平台验证通过后才转为发布承诺；失败时必须改库、自研、明确降级或调整版本，不能保留“已承诺”措辞。

| 版本 | 目标范围（Gate 通过后转为承诺） |
|---|---|
| v1.0 MVP | Vault 本地 schema/迁移、主机与分组、密码/密钥/kbd-interactive、host key + 最小 known_hosts 管理、SSH 终端与 `ssh://` 深链、移动键盘、多会话与后台输出/BEL 活动指示、基础设置；App 锁为 P0 候选，桌面导入/交换包范围待产品决策 |
| v1.1 Files | SFTP/SCP 浏览、传输中心、冲突/重试/恢复、文本编辑、跟随 CWD、终端增强和 SSH Quick Connect；Mosh/ET/Telnet 只保留数据并显示 unsupported |
| v1.2 Network GA | 跳板/HTTP/SOCKS、证书与 Identity、密钥生成、本地/远程转发、自动启动/网络恢复、本地保护备份 |
| vNext | 动态转发、完整 Notes/snippets/scripts、云同步、AI、系统管理器、终端图片/补全、zmodem/ymodem、其他协议、插件研究，以及待决的断开通知、系统分享、App Shortcuts、Widget、健康探测和扫码等移动新增/替代项 |

平台节奏可 Android 先行，但 iOS 不能整体塞进 v2.0 后才做基础可行性；双端 Phase 0 Gate 必须在锁定共享架构和 v1.x 承诺前完成。

## 附录 A：上游桌面基线更新

本轮同步到 `upstream/main@c4edefdb` 后，以下桌面能力发生变化，移动端以本 PRD 的版本和 Gate 处置为准，不把桌面新增自动升级为移动承诺:

| 上游变化 | 移动端影响 |
|---|---|
| 新增 `en/es/ru/zh-CN/zh-TW` 五种 UI locale | 移动语言验收矩阵增加 `es`；缺失翻译仍回退英文 |
| 终端支持 OSC 9/777/99 通知，含 `off/unfocused/always`、文本清洗、长度上限和会话级限流 | 移动系统通知仍是独立新增能力；必须重新定义权限、后台、锁屏脱敏、点击回会话和重复通知抑制 |
| SFTP 支持常见归档和单文件压缩解压，并有远端工具回退/临时文件保护 | v1.1 长按菜单可纳入，但必须通过路径转义、工具能力、取消、原子替换和安全 Gate |
| 桌面深链可解析 PuTTY 风格 SSH/Telnet 参数 | Quick Connect/深链 fixture 增加 PuTTY 参数、忽略项警告和 ephemeral 凭据边界 |
| 分组路径、动态脚本/snippet targets 和组变更提交更加原子化 | 移动迁移、重命名和删除组必须同步更新 host、managed source、script/snippet targets，失败可回滚 |
| 终端补全、搜索/选择、连接复用和 SFTP 重连安全持续修正 | Phase 0 corpus 与 contract test 使用上游最新行为，不复制已修复的桌面选择或复用缺陷 |

## 13. 待产品决策

1. 首发平台和最低 OS 版本、商店/侧载渠道。
2. 稳定交换包是独立格式还是受保护本地备份 payload 的可移植版本。
3. v1.0 是否必须包含桌面格式全量导入，还是只承诺 CSV + ssh_config + 交换包。
4. App 锁是否纳入 v1.0、默认值，以及锁定时会话/转发/传输的行为。
5. changed host key 的“仅本次继续”是否保留，或移动端采用更严格硬阻断。
6. 私钥导出是否允许；允许时的风险级别和平台分享限制。
7. Android 前台保活时长与通知操作；iOS 恢复预期。
8. 产品性能基准设备、包体口径和发布阈值。
9. 移动新增/替代的健康探测、扫码、后台断开通知、下载系统分享、App Shortcuts、Widget 是否进入正式路线图；各自允许显示哪些锁屏信息。
10. 品牌名、隐私政策、analytics/crash reporting 范围。
11. CSV 导出是否携带 Password/Passphrase；若不携带，如何与桌面格式兼容；若携带，采用何种再次认证、风险确认、分享限制和清理策略。
12. `known_hosts` 是否纳入稳定交换包；默认保持设备本地时，导入预览、冲突和风险提示采用什么规则。
