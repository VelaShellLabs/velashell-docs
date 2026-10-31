# RDP 远程桌面插件设计与使用

English: [RDP plugin: design and use](../../en/host/rdp-plugin-design-and-use.md)。

`velashell.rdp` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp),是进程内工作台插件,在 Windows、Linux、macOS 的 Avalonia 标签页中显示远程桌面。引擎是**纯托管 C# 的 RDP 客户端**,不依赖 FreeRDP,不含原生代码,随包只有一份托管程序集,全 RID 通用。

## 架构与范围

协议栈是独立的 C# 类库 `VelaShell.Rdp`(MIT 许可,随仓库发布):它 UI 无关、不依赖插件 SDK,实现本插件所需的 MS-RDPBCGR / MS-RDPELE / MS-RDPECLIP / MS-RDPEDISP / MS-RDPEFS 子集。插件引用该库,库 DLL 随插件签名 `.vpx` 一起分发,不单独出 NuGet 包。分层:

- **Wire** —— TPKT/X.224 分帧、快速路径帧、ASN.1 PER/BER。
- **Security** —— TLS 用 `SslStream`(OS 信任链,或精确的 SHA-256 叶证书 pin);NLA/CredSSP 用 `NegotiateAuthentication`(NTLM + TSRequest 公钥绑定)。
- **MCS/GCC** —— Connect-Initial/Response、erect/attach/channel-join。
- **PDU** —— 客户端信息、许可、能力协商、连接终结化。
- **Graphics** —— 交错 RLE 解码与 MPPC 批量解压、BGRA 帧缓冲;分片的快速路径更新会重组。
- **Input** —— 快速路径键盘/鼠标/Unicode。
- **Channels** —— 剪贴板(cliprdr)与磁盘映射(rdpdr)走分块的静态虚拟通道;动态虚拟通道(drdynvc,MS-RDPEDYC)上只开启显示控制通道(MS-RDPEDISP)以协商动态分辨率,其余动态通道(音频、GFX 图形管线)一律拒绝,服务器保持经典位图路径。

库里的 `RdpManagedSession` 以 SDK 无关的 `ConnectAsync(RdpClientOptions, …)` 实现 `IRdpSession` / `IRdpClipboardTransport` 契约;插件引用该库,提供桌面视图、输入队列、剪贴板桥接与编解码,并负责把宿主的连接请求映射成库的 `RdpClientOptions`、把库抛出的异常翻译成宿主协议异常。宿主管理连接表单、凭据、证书批准、会话标签与可选 SSH 跳板;插件提供工作台和远程桌面。连接、事件循环与收尾在会话工作任务上异步完成,由取消令牌中断。引擎在进程内运行,与宿主共享崩溃边界,不是进程隔离或文件系统沙箱。相关契约见 [SDK 参考](../sdk/sdk-reference.md)、[开发指南](../templates/dev-guide.md) 与 [当前信任边界](../plugins/12-security-threat-model.md)。

范围包括桌面分辨率跟随窗口(动态分辨率)、适应窗口/实际尺寸、键鼠/Unicode、Ctrl+Alt+Delete、断开后在桌面上就地重连、文本/图像/文件剪贴板、目录传输及仿 mstsc 的磁盘映射(勾选本机磁盘或填写自定义文件夹)。不协商音频播放/采集、多显示器或绘图指令加速(RemoteFX/GFX);服务器以经典 RLE 位图更新呈现,弱网时启用 MPPC 流压缩降低带宽。

## 连接与操作

通过宿主插件管理页安装插件 Release 中已签名的 `velashell.rdp-<版本>.vpx`,新建 RDP 连接,填写服务器、端口(默认 3389)、用户名和密码。域为独立的可选字段。需要跳板时,在“SSH 跳板机”选择已有 SSH 会话;隧道由宿主管理,证书仍核验原始目标身份。

| 设置 | 默认值与允许范围 |
| --- | --- |
| 域（`domain`） | 空,可选域 |
| 桌面宽度（`width`） | 1920;320–8192（初值与回退，连上后跟随窗口) |
| 桌面高度（`height`） | 1080;200–8192（初值与回退,连上后跟随窗口) |
| 连接超时（`timeout`） | 15 秒;1–120 |
| 安全层（`security`,高级） | `auto` 自动(默认);`nla` 仅 NLA;`tls` 仅 TLS,见[认证与证书策略](#认证与证书策略) |
| 双向剪贴板（`clipboard`） | `false`;明确开启后支持文本、图像与文件 |
| 本机磁盘（`drive:<盘>`,「磁盘映射」页） | 每块本机磁盘一个复选框,全部默认不勾选 |
| 自定义文件夹（`drives`,「磁盘映射」页） | 空;`名称=绝对路径`,多条用分号分隔 |
| SSH 跳板机（`jumpSession`） | 无,由宿主 SSH 会话选择器提供 |
| 已信任 SHA-256（`trustedThumbprint`） | 隐藏设置,由证书批准流程提供,精确叶证书 pin |

远程桌面默认跟随工作台标签页:连接后、以及窗口缩放稳定时(350ms 防抖),插件通过显示控制通道请求服务器把桌面调整到视口的设备像素,此时“适应窗口”即 1:1 清晰呈现。`width`/`height` 是连接时的初值,也是服务器不支持动态调整时的回退尺寸。此时“适应窗口”把不匹配的远程帧缓冲平滑缩放到可用区域(高质量插值),“100%”按实际像素显示并出现滚动条。工具栏“Ctrl+Alt+Delete”将组合键发往远程。会话断开后,桌面上出现“重新连接”浮层按钮(宿主会话标签同样可重连);重连会更新凭据及证书批准,重连失败时保留当前视图,按钮可再次点击。

### 剪贴板与文件传输

剪贴板**默认关闭**。开启后远程桌面可以读取及替换本地剪贴板,包括文件。

- 远程文本和图像自动接收。“接收剪贴板”显式接收远程文件/目录:先下载,再把本地文件引用放进系统剪贴板,供文件管理器粘贴。
- “发送剪贴板”、桌面取得焦点及 Ctrl+V 发布本地内容。本地内容未变化时不因取得焦点而重复发布,避免覆盖刚复制的远程内容。文件和目录通过 RDP 剪贴板文件内容请求传输,与磁盘映射独立。
- 下载保留在 `<插件 DataDirectory>/rdp-clipboard`。文件管理器粘贴仍引用它们时必须保留。累计缓存上限为 4 GiB,满后需手动清理;只删除已不再供剪贴板/粘贴使用的文件。接收操作不意味着文件已经放入用户的“下载”目录。

| 资源 | 上限 |
| --- | --- |
| 单次剪贴板载荷 | 16 MiB |
| 图像尺寸 | 任一方向最多 8192 |
| 解码图像像素 | 额外限制为 16 MiB,不能只满足编码后载荷上限 |
| 文件描述符（含目录） | 4096 |
| 单次文件传输 | 合计 1 GiB |
| 保留的剪贴板缓存 | 4 GiB |

拒绝不安全文件名、路径穿越和符号链接。在这些限制内保留目录结构。剪贴板传输由插件管理 RDP 通道,不是宿主 SFTP 传输队列,也不代表授予全部宿主文件 API 权限。

### 磁盘映射

连接对话框的「磁盘映射」页仿照 mstsc「本地资源 → 驱动器」,每块本机磁盘一个复选框。Windows 列出全部盘符,按类型命名,如「本地磁盘 (C:)」「可移动磁盘 (E:)」「网络驱动器 (Z:)」「光盘驱动器 (F:)」;Linux / macOS 列出根目录(`/`)、主目录(`~`)以及 `/media`、`/run/media`、`/mnt`、`/Volumes` 下的挂载点。全部**默认不勾选**。

枚举磁盘只读盘符与类型(或挂载点),不读取介质、不查询卷,因此断开的网络驱动器或空光驱不会拖慢插件激活。插件每 5 秒检查一次磁盘集合,只有变化时才重新注册连接表单(同一个实现实例:已打开的远程桌面标签不受影响,正在编辑的对话框也不会丢失已填内容),稍后插上的 U 盘在下次打开连接对话框时出现。连接时已勾选、但当前不存在的磁盘(U 盘已拔出、网络驱动器离线)直接跳过,不报错。

同一页的「自定义文件夹」用于只共享某个目录:写 `名称=绝对路径`,多条用**分号**分隔(宿主把文本设置渲染为单行输入框,无法换行)。Windows 示例:`work=C:\Work\Share;docs=D:\Docs`;Linux:`work=/home/alice/share`;macOS:`work=/Users/alice/share`。早先保存的按行分隔的值仍可识别。输入框下方常驻的提示会按当前系统给出写法与示例(占位符一输入就消失,所以示例放在提示里);填错时连接报错会引出写错的那一条并给出正确写法,名称重复时点名重复的名称,超过 32 个时说明上限。

名称必须唯一,也不能与已勾选磁盘的名称(`C`、`E`、`root`、`home` 等)重复;只允许 **1–8 位 ASCII 字母、数字或下划线**,不允许连字符;磁盘与文件夹合计最多 **32 个映射**。目录必须存在。远程 Windows 桌面通过 `\\tsclient\<名称>` 访问,xrdp 会话在 `~/thinclient_drives/<名称>`。每个映射授予远程对该磁盘或文件夹的**读写访问**;除了用户勾选的磁盘和填写的文件夹,不共享任何主目录、系统盘或可移动盘。路径限制在映射目录内,拒绝路径穿越与符号链接。映射磁盘或目录与经剪贴板发送文件是两个独立选择。

## 认证与证书策略

安全层由 `security` 决定:**自动**(默认)同时提供 TLS 与 NLA,由服务器选择(支持 NLA 的服务器会选 NLA);**仅 NLA** 要求服务器选 NLA,绝不回退;**仅 TLS** 只提供 TLS,凭据在已校验的 TLS 通道内随 Client Info 发送,由服务器自己完成登录(相当于服务器关闭了「要求 NLA」)。TLS 最低 1.2;禁用旧式 RDP 安全及“忽略所有证书”,各模式的证书校验完全相同。没有 pin 时,核验系统信任链、服务器用途、有效期和主机名/IP,由 .NET 的 `SslStream` 用各平台的 OS 信任存储完成。

**自动模式的回退。** 若服务器在客户端发出第一条 CredSSP 消息之后就拒绝——这条消息只是 NTLM NEGOTIATE,不含用户名,也不含任何凭据证明——表现为以 `errorCode` 0x80090302(SEC_E_UNSUPPORTED_FUNCTION)作答,或直接中止 TLS 会话(本地 SChannel 报 0x80090304「无法连接到本地安全机构」;对同一台服务器、字节完全相同的消息,两种形态会交替出现),说明服务器不接受传入的 NTLM。这是服务器策略,不是凭据问题。自动模式随即**只提供 TLS 重连一次**,证书照样校验,插件日志记一条警告。若服务器连 TLS 也拒绝(RDP_NEG_FAILURE `HYBRID_REQUIRED_BY_SERVER`),即服务器要求 NLA 却不接受 NTLM,错误会说明这是服务器端设置、与用户名和密码无关,并给出两种修法:在服务器上允许传入 NTLM(本地安全策略 › 安全选项 ›「网络安全: 限制 NTLM: 传入 NTLM 流量」设为「允许所有」),或关闭「仅允许运行使用网络级别身份验证的远程桌面的计算机连接」。密码错误(服务器在 NTLM 认证之后拒绝)仍报认证失败,**不会**触发回退。

**连接失败的诊断。** 错误指明失败的阶段(TCP、安全协商、TLS、NLA、MCS、登录、许可、能力交换)和原因链,含 SChannel / 套接字错误码;TLS 握手之后的失败还附上协商出的 TLS 版本。服务器在 NLA 中拒绝登录时,按它返回的状态码给出具体原因(用户名或密码错误、密码过期、账号锁定或停用、不允许远程登录)。**连接成功后又被服务器断开**时(服务器发来 Set Error Info PDU,MS-RDPBCGR 2.2.5.1.1),按它的 errorInfo 码给出原因并显示在断开浮层与状态栏:用户被注销、会话被另一连接接管、服务器拒绝或权限不足、空闲/登录超时、没有可用的远程桌面授权(RDS 授权或宽限期过期)等;其余码回退到中性描述加十六进制码。断开日志还记录**此前是否收到过登录通知**(服务器的 Save Session Info PDU):收到说明凭据已被接受、Windows 会话已启动,随后的断开是服务器结束了一个已建立的会话(多为策略),而非认证或协议失败;未收到则说明尚未走到登录。插件日志(`velashell.rdp`)记录每次失败的完整异常;TLS 握手之后的失败再附上加密套件和服务器发来的 TLS 记录头(只记类型与长度,例如 `Alert 26 B`,从不保存载荷),用来区分「服务器中止」与本地解密问题。

宿主批准的 pin 是叶证书 DER 字节的精确 SHA-256(64 位十六进制)。**pin 替代 CA/名称/有效期验证**,只做叶证书身份的定时比较,不是在上述检查之上附加一道条件。叶证书变化会拒绝连接,需要重新明确批准。SSH 转发不会把预期服务器名称改成回环地址。密码通过 API 传递,不进入命令行参数。NLA/CredSSP 用 .NET 的 `NegotiateAuthentication` 做 NTLM 与消息密封,TSRequest 公钥绑定按 MS-CSSP 实现。

## 构建与独立交付

保持仓库的 .NET 11、SDK 与 Avalonia 设置,不改依赖。**只有托管构建,没有原生步骤**:

```bash
dotnet build plugins/VelaShell.Plugin.Rdp -c Release
dotnet build plugins/VelaShell.Plugin.Rdp -c Release -t:PackVpx
```

唯一的平台相关处是 `UnixClipboardFile`:它在 Unix 上 P/Invoke 系统 `libc` 的 `open`/`fstat`,以 `O_NONBLOCK|O_NOFOLLOW` 安全打开用于本地→远程传输的剪贴板文件(避免 FIFO 阻塞、拒绝符号链接),Windows 用 .NET 文件 API。系统 libc 随操作系统提供,不是捆绑的原生库。

不再有 `rdp-native` 工作流、CMake、随包 OpenSSL/VC 运行库或六平台原生任务:插件以**一份托管 `.vpx`** 交付,全 RID 通用。CI 与发布只跑托管测试与打包。工具链细节见 [插件构建 README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md)。另见 [打包与签名](../templates/publishing.md)。

## 验证情况与剩余边界

单元测试覆盖设置、剪贴板编解码、headless 界面与 CredSSP/NLA 的 ASN.1 编码。活连接互操作测试门控于 `VELA_RDP_TEST_TARGET`(CI 跳过),对真实服务器验证整条链路:

- TLS 协商与叶证书 pin 信任;无 pin 拒绝自签证书并抛出 SHA-256 供宿主信任流程。
- 桌面渲染:交错 RLE **与 MPPC 批量压缩**(服务器开启 `bulk_compression` 时,实测解压压缩 PDU、像素精确)。
- 键鼠/Unicode 输入;文本与图像剪贴板往返;磁盘映射:勾选的磁盘可被远程读取(含系统盘根目录 `C:\` 的列目录),自定义文件夹读与写。
- **动态分辨率**:经显示控制通道请求调整桌面,实测 1024×768 ⇄ 1280×1024 往返生效,重激活后帧缓冲与状态栏随之更新。
- **宿主内渲染**:真实 `RdpWorkspaceView` 与 `RdpDesktopControl` 经 Avalonia headless Skia 渲染托管引擎的帧,即宿主装载的同一套控件树;实测视图会把远程桌面调整到视口尺寸。

NLA/CredSSP 除离线 ASN.1 单测外,对 FreeRDP shadow 服务器(`/sec:nla`,门控于 `VELA_RDP_TEST_NLA_TARGET`)做了活连接验证:NLA 登录到桌面;错误密码报认证失败且不回退;「仅 TLS」被要求 NLA 的服务器以 `HYBRID_REQUIRED_BY_SERVER` 拒绝。安全层回退与「首条 NLA 回复即被拒」(错误码、TLS 中止两种形态)由进程内假服务器测试覆盖;TLS 1.2 下 Windows 服务器可能选用的各加密套件(ECDHE/RSA × AES-GCM/AES-CBC)已逐一对 xrdp 实测。真实 Windows 服务器方面,核实过一台拒绝传入 NTLM 的服务器的行为(首条消息即回 0x80090302 或中止 TLS),**真实 Windows 服务器的 NLA 登录成功路径仍未在活连接上验收**。交错 RLE / MPPC 解码、MCS/GCC、能力协商与各通道线格式按 MS-RDPBCGR 及相关公开规范实现。这些验证不构成全平台或真实 Windows 端的最终认证。
