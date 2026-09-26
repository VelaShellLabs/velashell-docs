# 宿主分层架构

> 这一篇讲**宿主主程序怎么分层、依赖往哪个方向走、每一层放什么**。
> 想知道「为什么这么重构」看 [架构设计.md](架构设计.md)（工程化蓝图）；
> 想知道「插件系统为什么长这样」看 [`../plugins/`](../plugins/)。
> English: [`../../en/host/architecture.md`](../../en/host/architecture.md)

## 1. 工程划分

| 工程 | 职责 | 能依赖谁 |
| --- | --- | --- |
| `VelaShell` | 桌面入口、DI 组合根、XAML 视图、VelaDock 停靠、全局样式与行为 | 所有层 |
| `VelaShell.Presentation` | 跨层 ViewModel、连接 / 隧道工作流服务 | Core、Terminal |
| `VelaShell.Controls` | 复用控件、设计令牌、内置 Cascadia Mono 字体 | Core（仅共享 UI 契约） |
| `VelaShell.Terminal` | VT 引擎、自绘渲染控件、X/Y/ZMODEM 路由 | Core |
| `VelaShell.Core` | 领域模型、服务契约、持久化抽象、协议引擎、本地化 | **谁都不依赖** |
| `VelaShell.Infrastructure` | SSH / SFTP / FTP / 隧道实现、SonnetDB 持久化、代理、Gist 同步、插件管理与能力实现 | Core、Terminal（仅当适配器确实属于这里） |
| `VelaShell.PluginHost` | 隔离插件的宿主进程 | **只依赖 SDK 契约** |

> 桌面入口工程实际名为 `VelaShell`（程序集 `VelaShell`，`OutputType=WinExe`）。
> 老文档里的 `VelaShell.App` 是历史别名。每个工程各自带 `README.md` 说明内部结构与依赖。

```mermaid
graph RL
    App["<b>VelaShell</b><br/>桌面入口 · DI 组合根<br/>视图 · VelaDock"]
    Pres["VelaShell.Presentation<br/>ViewModel · 工作流"]
    Ctrls["VelaShell.Controls<br/>控件 · 令牌 · 字体"]
    Term["VelaShell.Terminal<br/>VT 引擎 · 自绘渲染"]
    Infra["VelaShell.Infrastructure<br/>SSH/SFTP/FTP · 持久化<br/>代理 · 插件运行时"]
    Core["<b>VelaShell.Core</b><br/>领域模型 · 契约 · 协议引擎<br/>（无任何 UI 依赖）"]
    Host["VelaShell.PluginHost<br/>隔离插件宿主进程"]
    Sdk["VelaShell.PluginSdk<br/>（NuGet 契约包）"]

    App --> Pres
    App --> Ctrls
    App --> Term
    App --> Infra
    Pres --> Core
    Pres --> Term
    Ctrls --> Core
    Term --> Core
    Infra --> Core
    Infra -.-> Term
    Infra --> Sdk
    Host --> Sdk

    style Core fill:#2d6a4f,color:#fff,stroke:#1b4332
    style Sdk fill:#5a3e85,color:#fff,stroke:#3c2859
    style App fill:#1d3557,color:#fff,stroke:#0d1b2a
```

箭头指向**被依赖方**。虚线是「仅当适配器确实属于这里」的弱依赖。

**这条依赖方向是硬约束**，两条推论值得写下来：

1. **Core 不依赖任何 UI 框架**，所以领域模型、协议引擎与持久化契约可以脱离 Avalonia 单测 ——
   VT 解析、ZMODEM 编解码、隧道计量这些最需要回归保护的东西全在这一层。
2. **`PluginHost` 与第一方插件只认 SDK 契约**，不依赖宿主任何内部程序集。
   这正是插件能跨进程、跨 ALC 而类型仍然同一的前提。

### 为什么单拆 `VelaShell.Controls`

- 设计稿里大量是**可复用面**而非一次性页面。
- 主题令牌、面板外壳、会话树条目、标签条、传输行、隧道卡片，应当独立于应用引导演进。
- 令牌字典与控件样式住在独立程序集，运行时换主题才好做。

## 2. 终端子系统

终端**不是一个控件，是一个子系统**。数据从 SSH 流一路到屏幕，中间每一层都可以单独替换与单测：

```mermaid
flowchart LR
    SSH["SSH / ConPTY<br/>字节流"] --> Router["TerminalTransferRouter<br/>X/Y/ZMODEM 期间改道"]
    Router --> Sink["Utf8Sink<br/>增量解码 · 可配编码"]
    Sink --> Parser["VtParser<br/>DEC ANSI 状态机"]
    Parser --> Emu["TerminalEmulator<br/>SGR · 模式 · 字符集 · 应答"]
    Emu --> Screen["TerminalScreen<br/>主/备屏 · 滚动区 · scrollback"]
    Screen --> Sem["SemanticMatcher<br/>URL / IP / 错误 / 警告"]
    Sem --> Snap["渲染快照<br/>+ 行号 / 时间侧栏"]
    Snap --> Draw["VelaTerminalControl<br/>自绘 Avalonia Control"]
    Router -.传输会话期间.-> Engine["Core/ZModem<br/>Core/XYModem"]

    style Router fill:#7c4a03,color:#fff
    style Engine fill:#7c4a03,color:#fff
```

这样切分才撑得住：增量流式（逐字符 / 逐行）、ANSI 转义序列、URL / 错误 / 警告高亮、
选区不劫持 `Ctrl+C`、多行粘贴，以及 Linux 进度条那种整行重绘。

渲染器另带**行号 / 时间侧栏**（`Terminal/Rendering/GutterLayout` + `GutterFoldModel`）：
两列独立开关、带折叠标记与空白间隔、接快捷键切换。

**十种终端 profile**（vt52 / 100 / 102 / 220 / 320 / 340 / 420 / 520 / xterm / xterm-256color）
各自的 TERM 名与 Device Attributes 应答在 `TerminalType.cs`，默认 xterm-256color。
TERM 在**连接时**协商，因此改设置只对新连接生效 —— 活动会话热切会造成本地仿真与远端能力档不一致。

## 3. 远程传输栈

### SSH / SFTP

由 **[Tmds.Ssh](https://github.com/tmds/Tmds.Ssh)** 提供（全托管、async-first，2026-07 由 SSH.NET 迁入）。

**库类型绝不离开 `Infrastructure/Ssh/`**：`TmdsSshClientWrapper` / `TmdsSftpClientWrapper` /
`ShellStreamWrapper` 把 `SshClient` / `SftpClient` / `RemoteProcess` 适配成 `Core.Ssh` 的中立接口，
`TmdsSshInterop` 把库异常翻译成 Core 声明的 `VelaSsh*Exception` 族。

> ⚠️ **跨层识别异常一律用类型匹配**（`ex is VelaSshAuthenticationException`），
> **绝不用 `GetType().Name` 字符串** —— 换库或改名时字符串匹配不会产生任何编译错误。
> 这条是踩出来的：曾有一版按 SSH.NET 的旧类型名做字符串匹配，而实际类型早已带 `Vela` 前缀，
> 导致认证失败重试与全部分类错误提示**静默失效**；测试还因为自定义了同名假异常而长期全绿。

**跳板机**走库原生 `SshProxy` 链，由 `InfrastructureServiceCollectionExtensions.BuildProxyChain`
按已保存的跳板配置递归组装（≤5 跳、带环检测、**指纹按各跳逻辑主机分别校验**，绝不按 127.0.0.1 记录）。

### FTP / FTPS

FluentFTP 后端 + 连接池 + `RoutingRemoteFileService` 按会话分派，
上层文件浏览器 / 传输 / 限速零改动。连接池上限会**自己往下调**：
池里已有活连接却开不出新连接（`421 Too many users`）就收到当前连接数并排队复用；
传输被 `450 Transfer busy` 顶回来就收到 1 并重试该传输。取舍见
[FTP客户端可行性调研.md](FTP客户端可行性调研.md)。

### 全局网络代理

**应用级**（不是按会话）。唯一出口是 `Core/Net/IProxyResolver` —— **新功能接网络一律消费它**。

```mermaid
flowchart TD
    Cfg["设置 → 网络代理<br/>直连 / 跟随系统 / HTTP CONNECT / SOCKS5"] --> R["IProxyResolver<br/>唯一代理出口"]
    R --> S1["SSH<br/>LoopbackProxyRelay 环回中继"]
    R --> S2["FTP<br/>FluentFTP 代理子类<br/>代理下强制被动模式"]
    R --> S3["全部 HttpClient<br/>VelaWebProxy.Install 进程级接管<br/>更新 / Gist / Webhook / 头像 / 插件"]
    R -.有意不走.-> S4["ICMP（ping / traceroute）<br/>连接诊断的裸 TCP"]

    style R fill:#2d6a4f,color:#fff
    style S4 fill:#5c1a1a,color:#fff
```

- SSH 走环回中继，是因为 Tmds.Ssh 的 `Proxy` 抽象成员是 `internal`、外部无法派生；
  于是把首个真实 TCP 出站跳（有跳板链时为最内层跳板）改写到 `127.0.0.1` 中继。
  **主机指纹按原始 `ci.Host` 键控，不受影响。**
- **代理配置不完整时抛错拒连，绝不静默直连。**
- **四种模式只决定「VelaShell 自己连远程主机的第一跳 TCP 怎么走」**，作用于**全部出站**
  （SSH / SFTP / FTP 控制与数据 / 更新检查 / Gist / 资讯源 / 插件 HTTP）。入站隧道
  （`-L` / `-R` / `-D`）是另一层，不要塞进这四个选项。

| 模式 | 位置 | 覆盖范围 | 要点 |
| --- | --- | --- | --- |
| **none 无代理** | 直连 | — | **必须真正直连**：不回看系统代理，也不读 `HTTP_PROXY` / `ALL_PROXY` |
| **system 系统代理** | OS 设置（**解析器，不是协议**） | 全部出站 | 读 OS 当前代理并折成 http 或 socks5；系统是 SOCKS 就按 SOCKS5，不硬套 HTTP CONNECT；解析不出（没配 / 命中 bypass / PAC 失败）则直连 |
| **http HTTP 代理** | 应用层（第 7 层） | HTTP / HTTPS，以及用 `CONNECT` 打的裸 TCP 隧道 | 不支持 UDP；FTP 必须被动模式，数据连接走同一条 CONNECT |
| **socks5** | 会话 / 传输层 | TCP（本产品只用 TCP，不做 UDP ASSOCIATE） | 不解析应用内容；SSH / FTP 要挂代理时的首选 |

- **system 变更不需要重启**：数据源是实时的（Windows 上给 Internet Settings 注册了变更通知），
  每次连接前重读；长连接不必中途切换，新会话用新值。
- 设置页「网络代理」下方**按表格列出这四种模式**（模式名对齐成一列 + 说明换行，用下拉框里同一套词）——
  文档与界面同一份口径，用户不必来翻文档。
- **DNS**：经 HTTP CONNECT / SOCKS5 时**默认把主机名交给代理解析**（`ProxyDns` 默认开，
  等价 socks5h，避免本机解析到污染 / 内网 IP 再 CONNECT 那个 IP）；只有 `none` 才本机解析。
- **代理只作用于第一跳**：跳板链上真正发起 TCP 出站的只有最内层那一跳，其余各跳都在已建立的
  SSH 通道里，不再重复套同一个代理。
- ICMP 与连接诊断的裸 TCP **有意不走代理** —— 协议不支持 / 诊断语义即测直连链路。
- ⚠️ 别与**动态 SOCKS 转发**（`-D`）混淆，那是隧道功能，方向相反。

### 端口转发隧道

本地 `-L` / 远程 `-R` / 动态 SOCKS5 `-D`。**数据面与计量都在 SSH 库（`src/VelaShell.Ssh`）里** ——
宿主的 `Infrastructure/Ssh/LibraryPortForwardHandle.cs` 只是薄适配，把库的转发器（共同基类 `PortForwarder`）的
连接数、字节数与错误事件接到 `IPortForwardHandle` 上。换库之前 Tmds.Ssh 不给任何计数，宿主只好自己监听、自己搬运，那一套已经删了。

| 方向 | 实现 |
| --- | --- |
| 本地 `-L` | `LocalPortForwarder.Start`：库在本机监听，每条连接开一个 `direct-tcpip` 通道 |
| 动态 `-D` | `LocalPortForwarder.StartDynamic`：库里的 SOCKS5 服务端（RFC 1928，仅 `CONNECT` + 无认证），握手有时限 |
| 远程 `-R` | `RemotePortForwarder.StartAsync`：`tcpip-forward` 让服务端监听，回连的通道由库直接接到本机目标 —— 不再经本机临时监听接力 |

> ⚠️ 搬运保留**半关闭语义**：一侧读完就对另一侧发 EOF / `Shutdown(Send)`，而不是整条拆链；
> 链路断了则两端都中止（本机 socket 发 RST），不把截断的数据当成完整的交出去。规格见 [`ssh/spec/07-forwarding.md`](../ssh/spec/07-forwarding.md) §2.2、§6。

详见 [隧道功能规划.md](隧道功能规划.md)。

## 4. 终端内文件传输（ZMODEM / XMODEM / YMODEM）

`rz`/`sz`、`rb`/`sb`、`rx`/`sx` 三套协议**全部自己实现**，跨三个工程分布，协议中立契约共享：

| 位置 | 内容 |
| --- | --- |
| `Core/FileTransfer/` | 共享契约：`IByteDuplex`、`IFileTransferSink / Source / Observer`、`FileTransferSession / Item`、CRC-16/XMODEM、ZFILE / YMODEM block-0 文件信息编解码、`TransferTrace` |
| `Core/ZModem/` | ZMODEM 引擎（帧、ZDLE 转义、CRC-16/32、`ZModemSender` / `ZModemReceiver`），**只依赖 `IByteDuplex`** |
| `Core/XYModem/` | XMODEM / XMODEM-1K / YMODEM / YMODEM-G 引擎（定长块、逐块 ACK/NAK） |
| `Terminal/FileTransfer/` | `ZModemDetector`（输出流里嗅探 ZRQINIT / ZRINIT 引导）+ `TerminalTransferRouter`（坐在桥读循环与仿真器之间，会话期间把字节交给引擎，结束复位）+ `ShellStreamByteDuplex` |
| `App/Services/FileTransfer/` | 文件源（上传）、落盘目录（下载）、进度上报 |

**ZMODEM 自动接管**：引导序列在输出流里可识别。
**XMODEM / YMODEM 只能手动发起**（命令面板 →「文件传输」）：链路上没有可识别的引导序列 ——
`sb`/`sx` 静默等接收方的 `C`，`rb`/`rx` 发一个裸 `C` 与普通终端输出无从区分，任何自动检测都会误触发。
先在远端敲命令，再点面板条目。

因为引擎只要一个 `IByteDuplex`，同一条路同时服务 SSH、本地 ConPTY 与插件提供的协议会话
（Telnet / 串口）。排障置 `VELASHELL_TRANSFER_TRACE=1`（旧名 `VELASHELL_ZMODEM_TRACE=1` 仍可用）
打印协议帧。

> ⚠️ **测试教训**：互操作期望值必须按 lrzsz `zm.c`/`zmodem.h` 与 ymodem.txt **手工构造**。
> 用自家编码器生成期望值时，编解码同时错也照样全绿 —— CRC 双重增广的 bug 当初正是这么溜进来的。

## 5. 停靠与窗口壳

### VelaDock（零第三方依赖）

替换了 `Dock.Avalonia`，方案见 [dock-replacement-plan.md](dock-replacement-plan.md)。

| 层 | 内容 |
| --- | --- |
| `Docking/Model/` | **纯 INPC，可单测**：`DockWorkspace` / `DockGroup` / `DockSplit` / `DockDocument`；空的次级组自动折叠、单子分栏自动提升 |
| `Docking/Controls/` | `DockWorkspaceControl`（按树渲染 Grid + GridSplitter，**按文档缓存视图**）、`DockGroupControl`（标签条 + 溢出）、`DockTabItem`、`DockDragController` + `DockDropOverlay`（拖拽重排插入线、跨组并入、五区拖放分屏，Esc 取消） |

文档就是活动的 SSH / 本地 / SFTP 会话，每个 `TerminalTabView` 按文档缓存、跨标签切换复用。
**浮动窗口按产品决策不实现** —— 多屏需求由五区拖放分屏承担。

### 窗口壳

主窗在 Windows 与 Linux 上是**自绘无边框窗口**（`WindowDecorations="None"`），**不是原生 chrome**；
macOS 上改用系统外框与原生红绿灯，见下文「各平台的外框」。

> ⚠️ 这是在 **Windows（Win32）** 上踩出来的结论，不要在 Windows 上试图改回去。Avalonia 12.x 的 `ExtendClientArea` /
> `WindowDecorationsElementRole` 托管装饰在 Win32 上会拦截标题栏输入（按钮点不动、窗口拖不动），
> `BorderOnly` 还丢 `WS_CAPTION`（HTCAPTION 拖动与最小 / 最大化动画失效）。整套机制在 Win32 上不可用。
> 这条只管 Win32：macOS 与 Linux 的外框走各自的原生机制（下一小节），与它不冲突。
> 另一处坑：`VisualRoot as Window` **恒为 null**（视觉根是 TopLevelHost），
> 取窗口必须走逻辑树 `FindLogicalAncestorOfType<Window>()` —— 这条曾让标题栏按钮「看似无输入」数小时。

改以**自绘 + 原生行为补齐**：`Views/TitleBarView` 画 28px 标题栏（左 logo + 产品名，
右 全局功能图标组 + 自绘最小化 / 最大化 / 关闭），
空白区 `BeginMoveDrag`（原生移动循环，Win11 边缘贴靠有效）、双击切最大化、
**Win11 Snap Layouts 经 `MainWindow` 的 WndProc 钩子处理 `HTMAXBUTTON`**、
窗口四周 5px + 四角 10px 自绘缩放抓取区（最大化时关闭）。Windows 上全部对话框同为自绘无边框。

#### 各平台的外框

透明窗口 + 卡片 16px 边距 + 在边距里自绘 `VelaShadowWindow` 的写法只在 Windows 上成立：DWM 对无边框透明窗口什么都不加。
Linux 的合成器只知道整个窗口矩形，会沿它描边、切圆角、做背景模糊，透明边距就成了卡片外面的一圈「空白」；
不支持透明的 X11（WSLg 即是）干脆把那圈边距画成实色。macOS 的 `None` 没有系统阴影与圆角，
而透明窗口在 macOS 上滚动掉帧，只能不透明 —— 结果是没有阴影的直角矩形。所以每个平台改用自己的原生机制：

|            | Windows | macOS | Linux 原生 Wayland | Linux X11（含从 Wayland 回退） |
|------------|---------|-------|--------------------|-------------------------------|
| 主窗口     | 自绘无边框（同上） | `Full` + `ExtendClientAreaToDecorationsHint`：原生红绿灯、系统圆角与阴影；自绘的三个窗口按钮与缩放抓取区关闭，标题栏左侧给红绿灯让位（全屏时撤掉）、高度跟系统标题栏一致（当前 28pt），红绿灯与标题同一条中线 | 自绘无边框 | 自绘无边框 |
| 模态对话框 | 透明窗口 + 卡片 16px 边距 + 自绘阴影 | `BorderOnly` + 扩展客户区：系统圆角与阴影、不显示红绿灯、窗口不透明 | `BorderOnly` + 装饰主题 `VelaWaylandWindowDecorations`：Avalonia 画 16px 阴影与 1px 描边，并经 `xdg_surface.set_window_geometry` 告诉合成器真正的窗口范围 | 不透明直角矩形 |
| 非模态窗口 | 同对话框 | `Full` + 扩展客户区：原生红绿灯，自绘的窗口按钮隐去、标题栏左侧让位 | 同对话框 | 同对话框 |

最大化 / 全屏时，卡片在各平台都铺满成直角。自绘的缩放抓取区只在普通态、且系统不提供边缘缩放时出现：
macOS 的系统外框与 Wayland 的装饰层都自带（主窗口在 Wayland 上仍是 `None`，照用自绘的）。

**实现**：窗口构造时调 `WindowChrome.Apply(window, kind[, 缩放抓取区])`（`Views/WindowChrome.cs`），
按平台设装饰模式、透明度、背景（Wayland 还有装饰主题），跟着窗口状态走，并给窗口挂样式类：
平台类 `chrome-windows` / `chrome-macos` / `chrome-wayland` / `chrome-x11`；`chrome-traffic-lights`（显示系统红绿灯）；
`window-card-flat`（卡片是直角：macOS / X11 恒是，最大化 / 全屏时各平台都是）；`window-fullscreen`。
XAML 这一侧的约定（样式在 `Themes/WindowChrome.axaml`）：

- 卡片写 `Classes="window-card"`，边距、圆角、描边、阴影一律不写 —— 本地值优先级高于样式，写死了就不跟着平台切换；
- 贴着卡片四角、有不透明背景的子元素（标题栏、状态栏、左侧导航……）不写 `CornerRadius`，改挂
  `window-card-top` / `-bottom` / `-left` / `-right` / `-bottom-left`：卡片圆角时取内半径 7，直角时跟着直角；
- 非模态窗口自绘的最小化 / 最大化 / 关闭挂 `window-caption`，标题栏最左边放一个
  `<Panel Classes="traffic-light-spacer" />` 给红绿灯让位；
- **全部窗口的标题栏一个高度**（主窗口、独立窗口、对话框，2026-09-26 起统一为 28）：标题栏挂 `window-titlebar`，
  高度只在样式里定一处，XAML 里不写、标题行用 `Auto`；窗口按钮是 27×27 的方块（边长 = 标题栏 − 1px 底边）。
  放不下的副标题与操作按钮挪到标题栏下面一行（资源监视、录制回放、远程编辑、连接诊断）。
  macOS 上显示红绿灯的窗口改成系统标题栏的实际高度（`WindowDecorationMargin.Top`，当前也是 28），
  红绿灯位置改不了，只能让标题栏去对齐它。
  两处例外，头部没有窗口按钮、不算标题栏，保持 48：设置窗口左上角那条是左侧导航的抬头兼拖动区；
  消息框（提示 / 确认 / 输入）没有关闭按钮，关闭一律走按钮栏或 Esc。

窗口尺寸按 Windows 写（含 16px 边距），macOS / X11 上宽高各减 32，Wayland 上各减 34
（Wayland 未扩展客户区时 `Width` / `Height` 只算内容，1px 描边画在外面），卡片的可见尺寸三个平台一致；
代码里另有按 Windows 口径写的尺寸时（插件声明的面板窗口尺寸、新建连接窗口的高度上限），用 `WindowChrome.SizeReductionOf` 减掉。
隔离插件的窗口在另一个进程里（`VelaShell.PluginHost` 的 `PluginHostShellWindow`），按依赖纪律不引用主程序，
同一套规则在那里用代码另写了一份（Wayland 的装饰主题也是代码构建的）。
`WindowChromeCoverageTests` 扫描全部窗体，防止写死的外框回来。

**依据**（Avalonia 12.1.3 源码）：

- **macOS**：`None` 关掉系统阴影、没有标题栏样式，也就没有系统圆角；`BorderOnly` 是 `Titled | FullSizeContentView`、有系统阴影；
  红绿灯只在 `Full` 时显示，位置不能调（`ExtendClientAreaTitleBarHeightHint` 只改标题栏背景材质的高度）。
  扩展客户区时按下左键，先对界面做命中测试：命中按钮就是普通点击，只有什么都没命中、或命中 `TitleBar` 角色的元素，
  才交给系统拖动 / 双击缩放 —— 标题栏右上的功能按钮照常可点。
- **Wayland**：非 `Full` 时永久切到客户端装饰；`BorderOnly` 画阴影、描边、缩放抓取区，不画标题栏；
  阴影宽度经 `set_window_geometry` 交给合成器，合成器以描边为窗口边界（GTK4 程序同样这么做）。
  不要在同一个窗口上来回切换外框模式。Avalonia 12.1.3 的 Wayland 后端要求合成器提供 `xdg_wm_base` 3 版以上，
  WSLg 的 Weston 不满足，那里一律回退 X11。
- **X11**：`BorderOnly` 同样会让 Avalonia 画装饰，但 X11 后端没有实现阴影范围，也不写 `_GTK_FRAME_EXTENTS`，
  合成器拿到的仍是整个矩形，所以只能走不透明矩形。窗口句柄描述符是 `"XID"`，Wayland 后端不提供句柄，据此区分两者。

**验收**：外框改造本身在 Windows 上与改造前逐像素一致（无头渲染对照，全部窗口的暗 / 亮两套主题；
只有录制回放与远程编辑两扇窗的**最大化**态变了 —— 它们以前最大化时不铺满，现在与其它非模态窗口一致）。
随后按用户要求把全部窗口的标题栏统一为 28，Windows 上的标题栏高度随之有意改变。
macOS 上主窗口、设置窗口与消息框已实机验收（2026-09-26）；其余窗口按同一套机制接入，
非模态窗口的红绿灯、统一 28 后的红绿灯对齐与 Linux 原生 Wayland 桌面待下一轮实机确认。

文字菜单栏已整体移除，功能由命令面板（`Ctrl+P` / `Ctrl+K`）承担。

## 6. 主题与设计令牌

**12 套具名主题**（7 暗 5 亮）+ **16 套内置终端配色**，成对联动。

一套主题只需人工填 **25 个种子色**（`UiThemePalette`，在 Core）：底色阶梯、四档文字、
两档描边、强调色与语义色。其余六十多个令牌（`*Dim`、`VelaHeat1-5`、`VelaGauge*`、
`VelaTrace*`、`VelaShell*` 等）由 `ThemeTokenApplier`（宿主侧）按固定规则派生。

> **为什么要派生**：手抄会错，而且错了看不出来。`#644AC922` 这种把透明度写在末尾的错拼，
> 编译期无感、运行期是一片绿。派生出来的令牌天生自洽，加一套主题只需填种子色。

- **XAML 与 C# 里不许出现颜色字面量**，一律 `DynamicResource` 绑定令牌
  （规范见代码仓库的 [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md)）。
- 强调色是一层独立的覆盖层，用户自定义时按亮度自动配对前景色。
- 「跟随主题」是**显式状态**而非隐式默认 —— 隐式的那一版会让「选了 Dracula 却没反应」。

## 7. 持久化

全部持久化走 **[SonnetDB](https://github.com/IoTSharp/SonnetDB)** 嵌入式多模型数据库
（`SonnetDB.Core`，经 `Tsdb.Open` 开在 `~/.velashell/sonnetdb`）。
旧 JSON（`sessions.json`、`settings.json`、`state.json`、`known_hosts.json`、
`quick-commands.json`）首次运行一次性导入后改名 `*.migrated.bak`。

```mermaid
graph TD
    I1["<b>Core（契约）</b><br/>ISessionRepository · ISettingsService<br/>IRecentConnectionService · IAuditLogService<br/>IAppDataStore · ISessionRecordingStore<br/>IQuickCommandRepository · ISecretProtector"]
    E["<b>Infrastructure/Persistence</b><br/>SonnetDbEngine（单例）<br/>退出时 Dispose 刷 WAL"]

    subgraph Doc["文档集合（业务 / 配置）"]
        D1["session_groups"]
        D2["session_profiles<br/>$.groupId 索引"]
        D3["app_config<br/>settings / state / sync"]
        D4["known_hosts · ui_config"]
        D5["quick_commands（schema v2）"]
        D6["tunnels（每 profile 一份）"]
        D7["recordings（录制元数据）"]
    end

    subgraph TS["时序 measurement（时间序列）"]
        T1["conn_history<br/>最近连接，供侧栏"]
        T2["audit_log<br/>安全审计"]
        T3["session_recording_chunks<br/>tag: recording_id<br/>field: offset_ms + Base64 data"]
    end

    I1 --> E
    E --> Doc
    E --> TS

    style I1 fill:#2d6a4f,color:#fff
    style E fill:#1d3557,color:#fff
```

**敏感字段静态加密**：密码、私钥口令、同步令牌经 `ISecretProtector`（AES-256-GCM + 本地密钥文件
`~/.velashell/secret.key`，密文前缀 `enc1:`）落盘。

> ⚠️ **仓储加密必须写副本，不可原地改传入的 profile** —— 内存里那份明文正被活动连接使用，
> 原地加密会把它改成密文，表现是「重连突然认证失败」。

**设备本地状态**（侧栏分区折叠、记住的高度）存在 `app_config/state`，**不进 Gist 同步**。

快捷命令经 `IQuickCommandRepository` 装载，它自己负责 v1 SonnetDB 与遗留 JSON 的迁移、
备份、schema 校验与 Gist 兼容的 v1/v2 快照 —— UI 与同步服务**从不直接碰** SonnetDB 文档。

### ⚠️ SonnetDB 的几处方言坑

| 坑 | 说明 |
| --- | --- |
| `ORDER BY time` | 要求 SELECT 列表**包含 time 列** |
| `DELETE FROM measurement` | 可能不受支持 —— 录制存储的保留清理以 **drop + 回写压缩**兜底，防孤儿数据块磁盘只增不减 |
| 时序 tag 值 | **不允许空串**（临时连接因此不写 `profile_id`） |
| `FieldType` 命名 | 在 `SonnetDB.Storage.Format`，是 `Int64` 不是 `Long`，写值用 `FieldValue.FromLong` |
| 锁粒度 | **刻意保留全局信号量** —— 文档集合与时序共享同一 Tsdb 实例（同一 WAL / 存储引擎），SonnetDB 未承诺内部线程安全，按集合分锁有并发损坏风险。真正的热点是设置读（每次连接、每个传输文件都读一次），已在 `SonnetDbSettingsService` 加 JSON 缓存解决 |

## 8. 插件运行时（宿主侧）

宿主侧的插件运行时在 `Infrastructure/Plugins/`。**双模装载**：

```mermaid
flowchart TD
    PM["PluginManager<br/>发现 / 装载 / 启停 / 卸载"] --> Gate["PluginPermissionGate<br/>危险能力逐项授权"]
    Gate --> Ctx["PluginContext<br/>能力面组装"]

    Ctx --> InProc["<b>进程内</b><br/>可收集 AssemblyLoadContext"]
    Ctx --> Iso["<b>独立进程</b><br/>VelaShell.PluginHost"]

    InProc --> UI1["UI 直接并入停靠工作区"]
    Iso --> RPC["命名管道 RPC<br/>心跳 · 自愈重启 · 空闲回收"]
    RPC --> UI2["独立卡片窗口 / 嵌入宿主"]

    Ctx --> Caps["<b>能力面</b><br/>Sessions · Terminal · RemoteExec · RemoteFs<br/>RemoteTunnel · Protocols · Workspaces<br/>Storage · TimeSeries · Secrets<br/>Commands · Events · Ui · Clipboard · Log"]

    style PM fill:#1d3557,color:#fff
    style Gate fill:#7c4a03,color:#fff
    style Caps fill:#2d6a4f,color:#fff
```

装载方式由插件清单的 `hostMode` 声明，**两种模式共用同一套 SDK 契约，插件源码零改动**。

**依赖解析**：按插件自己的 `deps.json` 解析，只有 SDK 契约与 `Avalonia*` 框架程序集
回落到宿主（`PluginAssemblyLoadContext.SharedPrefixes`），保证跨边界类型同一。

**协议与工作台扩展**：`Protocols` / `Workspaces` 两个能力让插件**注册新的连接类型** ——
Telnet / 串口 / Redis / S3 就是这么接进会话树的。插件会话经 `PluginTerminalShellStream`
适配成 `IShellStreamWrapper`，复用既有的桥 / VT 引擎 / X-Y-ZMODEM / 重连整条链路。
id 强制插件前缀，冒名等于劫持别家的连接配置；释放时撤销该插件的全部注册 ——
这是可收集 ALC 能真正回收的前提。

**分发**：`.vpx` 包（VelaShell 自有的插件包格式，容器内是 zip 载荷；
`PluginPackageExtractor` 三道闸各挡各的 —— 条目数闸挡「一百万个空文件」、
字节预算挡「一个条目吐十 GB」、路径校验挡 zip-slip）。

完整设计见 [`../plugins/`](../plugins/) 的 15 篇蓝图与[进度总览](../plugins/STATUS.md)。

## 9. 一条连接是怎么建起来的

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant VM as MainWindowViewModel
    participant WF as ConnectionWorkflowService
    participant Auth as InteractiveAuthenticator
    participant HK as IHostKeyPrompt
    participant SSH as TmdsSshClientWrapper
    participant Br as SshTerminalBridge
    participant T as 终端控件

    U->>VM: 双击会话 / 命令面板
    VM->>WF: 解析 profile（含跳板链 ≤5 跳、环检测）
    WF-->>VM: 缺凭据？
    VM->>Auth: 两步验证弹窗（用户名 → 认证方式）
    Auth-->>VM: 凭据
    VM->>SSH: ConnectAsync（AutoConnect=false）
    SSH->>HK: 主机指纹校验
    alt 首次连接
        HK-->>SSH: TOFU 记录 / 人工三选一
    else 指纹变化
        HK-->>SSH: 人工三选一（默认）/ 直接拒绝（开了阻断开关）
    end
    SSH-->>VM: 连接成功（写 audit_log）
    VM->>Br: 建桥，只读循环
    Br->>T: 字节流 → VT 引擎 → 自绘渲染
    T->>Br: PtySizeChanged(cols,rows) → 实时改窗
    VM->>VM: 「认证后执行命令」按 profile 延迟下发
```

几处不显然的约定：

- **`AutoConnect = false` 是显式设的。** Tmds.Ssh 的默认值是 `true`，会导致会话掉线后
  **每一次** SFTP 操作各自静默重连一次 —— 拖入 N 个文件就是 N 次隐式重连 + N 发异常。
  连接只由 `ConnectAsync` 发起。
- **桥的读循环不向 shell 预写 `\n`**（修过「末行提示符重复」）。
- **本地终端标签不自动重连** —— `exit` 是用户意图；远端 `exit` 同理。
- **关掉「连接中」的标签就该取消后台握手**，而不是让它连完再挂在那儿。
- 连接失败不崩溃：认证 / 网络 / 超时异常映射成可读提示写状态栏，
  `Program.cs` 另装了 `TaskScheduler.UnobservedTaskException` /
  `AppDomain.UnhandledException` 兜底。

## 10. 组合根与 DI

**所有 DI 注册集中在 [`src/VelaShell/App.axaml.cs`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell/App.axaml.cs)**，
各层通过各自的 `*ServiceCollectionExtensions` 贡献注册。服务一律接口注入，便于 Mock 与单测。

新增一个跨层服务的正确姿势：契约放 `Core`，实现放 `Infrastructure`，
注册写进该层的 `*ServiceCollectionExtensions`，**不要**在组合根里直接 `new`。

## 相关文档

| 文档 | 内容 |
| --- | --- |
| [架构设计.md](架构设计.md) | 工程化重构蓝图（为什么这么分层） |
| [dock-replacement-plan.md](dock-replacement-plan.md) | VelaDock 替换 Dock.Avalonia 的方案与集成面分析 |
| [交互与界面规格.md](交互与界面规格.md) | 交互逻辑与设计令牌 |
| [settings-audit.md](settings-audit.md) | 设置项审计台账 |
| [隧道功能规划.md](隧道功能规划.md) · [路由追踪设计.md](路由追踪设计.md) · [消息中心与资讯源.md](消息中心与资讯源.md) | 各功能的详细设计 |
| [`../plugins/02-architecture.md`](../plugins/02-architecture.md) | 插件系统的进程模型与组件划分 |
