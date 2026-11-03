# RDP 远程桌面插件设计与使用

English: [RDP plugin: design and use](../../en/host/rdp-plugin-design-and-use.md)。

`velashell.rdp` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp),是进程内工作台插件,在 Windows、Linux、macOS 的 Avalonia 标签页中显示远程桌面。引擎是**纯托管 C# 的 RDP 客户端**,不依赖 FreeRDP,不捆绑原生二进制,传输代码与托管 DTLS 依赖随包分发,全 RID 通用。

## 架构与范围

协议栈是独立的 C# 类库 `VelaShell.Rdp`(MIT 许可,随仓库发布):它 UI 无关、不依赖插件 SDK,实现本插件所需的 MS-RDPBCGR / MS-RDPELE / MS-RDPECLIP / MS-RDPEDISP / MS-RDPEFS / MS-RDPEA 子集。音频播放调用系统音频库,不捆绑原生二进制。插件引用该库,库 DLL 随插件签名 `.vpx` 一起分发,不单独出 NuGet 包。分层:

- **Wire** —— TPKT/X.224 分帧、快速路径帧、ASN.1 PER/BER。
- **Security** —— TLS 用 `SslStream`(OS 信任链,或精确的 SHA-256 叶证书 pin);NLA/CredSSP 在 Windows 上用系统原生 SSPI 的 Negotiate 包,Linux/macOS 用 `NegotiateAuthentication` 的 SPNEGO 协商系统 Kerberos 或托管 NTLM;TSRequest 公钥绑定按 MS-CSSP v5+。
- **MCS/GCC** —— Connect-Initial/Response、erect/attach/channel-join。
- **PDU** —— 客户端信息、许可、能力协商、连接终结化。
- **Graphics** —— 交错 RLE 与 RDP 6.0 planar(32 位色)解码、MPPC 批量解压、GDI 绘图指令与位图缓存、指针(鼠标形状)解码,以及记录变化区域的 BGRA 帧缓冲;分片的快速路径更新会重组。托管 MS-RDPEGFX 管线增加 ZGFX、表面/缓存、帧确认、ClearCodec、RemoteFX/Progressive 和 H.264 AVC420/AVC444/AVC444v2。
- **Input** —— 快速路径键盘/鼠标/Unicode。
- **Channels** —— 剪贴板(cliprdr)、音频播放(rdpsnd)与显式选择的磁盘、端口、legacy 打印机(rdpdr)走分块静态虚拟通道。动态虚拟通道(drdynvc,MS-RDPEDYC)支持显示控制(MS-RDPEDISP)、所选摄像头(MS-RDPECAM)和所选 Windows 便携设备(MS-RDPEPNP)。默认注册 GFX 图形管线(MS-RDPEGFX);未注册通道被拒绝。服务器未开启 GFX 时保留经典位图路径。

库里的 `RdpManagedSession` 以 SDK 无关的 `ConnectAsync(RdpClientOptions, …)` 实现 `IRdpSession` / `IRdpClipboardTransport` 契约;插件引用该库,提供桌面视图、输入队列、剪贴板桥接与编解码,并负责把宿主的连接请求映射成库的 `RdpClientOptions`、把库抛出的异常翻译成宿主协议异常。宿主管理连接表单、凭据、证书批准、会话标签与可选 SSH 跳板;插件提供工作台和远程桌面。连接、事件循环与收尾在会话工作任务上异步完成,由取消令牌中断。引擎在进程内运行,与宿主共享崩溃边界,不是进程隔离或文件系统沙箱。相关契约见 [SDK 参考](../sdk/sdk-reference.md)、[开发指南](../templates/dev-guide.md) 与 [当前信任边界](../plugins/12-security-threat-model.md)。

范围包括桌面分辨率跟随窗口(动态分辨率,按本机缩放发送 DPI,1:1 锐利绘制)、16/24/32 位颜色深度、远端鼠标形状、键鼠/Unicode、[远程按键悬浮球](#远程按键悬浮球)(Ctrl+Alt+Delete 等本机会截走的组合键与键入剪贴板)、断开后在桌面上就地重连、文本/图像/文件剪贴板、目录传输及仿 mstsc 的磁盘映射(勾选本机磁盘或填写自定义文件夹),以及可选的远程音频播放。另支持下文的 RD Gateway 传输、经典 RemoteApp/RAIL 与 PC/SC 智能卡重定向。不协商麦克风采集或多显示器。桌面图形默认自动协商 GFX;不支持 GFX 的服务器仍以经典位图更新(交错 RLE,32 位色时为 RDP 6.0 planar)和 GDI 绘图指令呈现,弱网时启用 MPPC 流压缩降低带宽。

**绘图指令与位图缓存。** 客户端接受 DstBlt、PatBlt、OpaqueRect、MultiOpaqueRect、ScrBlt 与 MemBlt(移植自 FreeRDP 的 `Orders.cs`),并声明 600/600/2048 项的第 2 版位图缓存:服务器可以用一次屏幕内拷贝完成滚动、用缓存引用重绘重复的图块,而不必重发像素。xrdp 整个桌面都以缓存 MemBlt 绘制;较新的 Windows 服务器无论如何都只发位图。指令在帧缓冲上按画面与边界裁剪,屏幕内拷贝在重叠时安全,三元光栅运算(ROP3)按真值表求值。默认开启,库调用方可用 `RdpClientOptions.DrawingOrders = false` 关闭;重激活(调整分辨率)时保留缓存、重置指令状态。

## 托管 GFX 与 H.264

「体验」页的图形管线设置(`graphicsMode`)默认 `auto`,注册 `Microsoft::Windows::RDS::Graphics`,公告 RDPGFX 8.0–10.7。选择 `classic` 并重连可强制经典位图/GDI。库调用方对应 `RdpClientOptions.EnableGraphicsPipeline`(默认 `true`),诊断回调为 `GraphicsLog`。服务器未开启 GFX 时保留经典路径;已经协商后的畸形图形流会结束会话,不在连接中途重放到另一条路径。

协议、ZGFX、表面与缓存、帧确认和颜色重建均为 C#。H.264 使用纯托管 NuGet `ManagedCodecs.H264` 0.6.0 与 `ManagedCodecs.Core`,无需安装 FFmpeg 或分发原生图形库。AVC444/v2 的主图与辅助图共享 H.264 参考链;Progressive 的子带差分历史随表面保留,删除编码上下文不会丢失。支持桌面及缩放输出映射、raw/Planar/Alpha、ClearCodec(含 NSCodec/RLEX)、RemoteFX RLGR1/RLGR3、Progressive simple/first/upgrade;不支持 Progressive V2(codec 13)和 RemoteApp 窗口映射。

三平台使用相同实现,H.264 当前为 CPU 软件解码。Windows 完整 RDP 回归和 Debian 上的 GFX/ZGFX/Planar 测试已在本地运行通过;Ubuntu 与 macOS 的 CI 图形测试也已通过。离线用例包含真实 IDR/P 参考链、AVC444 色度布局、缓存生命周期和帧缓冲/ACK 链路,不能替代真实 Windows 服务器互操作及 4K/视频吞吐验收。

缓存导入只用于跨连接保存的持久化位图。当前实现保留会话内表面/位图缓存,不发送空的 `CACHE_IMPORT_OFFER`,也不等待缓存导入回复。真实 Windows 服务器对照验证中,空请求触发了首帧后的 `0x112F` 断开;移除该请求后仍保持 GFX 10.6 并持续接收登录界面画面。此验证未发送账号密码,不代表登录后桌面或视频性能验收。

动态通道 v3 接收 `DATA_FIRST_COMPRESSED` / `DATA_COMPRESSED`,按每个通道独立保存 ZGFX 历史,先解压再按消息总长度重组;关闭或复用通道 ID 会丢弃历史。该层与 GFX 消息自身的 ZGFX 压缩相互独立,在 TCP 与已迁移的 UDP 通道上都适用,并遵守通道消息上限和剩余分片预算。

**连接诊断。** `managed graphics pipeline ready` 仅说明本地解码器已准备好。后续 `graphics DVC opened`、`server confirmed graphics capabilities`、`graphics reset`、`first graphics frame decoded` 分别标记通道打开、服务器能力确认、画面尺寸重置与首帧解码,不代表已验证所有 AVC 模式。`0x112D/0x112E/0x112F` 分别表示服务器不支持图形模式、无法重置图形子系统、图形子系统失败;仅凭 `0x112F` 不能判断是客户端协议兼容还是服务器编码器问题。排查时可选择 `classic` 后重连作对照,这不是根本修复。若日志已记录 NLA 拒绝后成功回退 TLS,不要把那次 NLA 拒绝当成后续断开的原因。UDP 建连前会把双栈 TCP 返回的 `::ffff:a.b.c.d` 规范化为 IPv4 地址,真正的 IPv6 地址保持不变;必要时选择 TCP 传输单独隔离 UDP 问题。

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
| 远程音频（`audio`,「体验」页） | `false`;开启并重连后通过本机默认设备播放远端 PCM 音频 |
| 本机磁盘（`drive:<盘>`,「磁盘映射」页） | 每块本机磁盘一个复选框,全部默认不勾选 |
| 自定义文件夹（`drives`,「磁盘映射」页） | 空;`名称=绝对路径`,多条用分号分隔 |
| SSH 跳板机（`jumpSession`） | 无,由宿主 SSH 会话选择器提供 |
| 已信任 SHA-256（`trustedThumbprint`） | 隐藏设置,由证书批准流程提供,精确叶证书 pin |
| 颜色深度（`colorDepth`,「体验」页） | `32` 最高质量(默认,与 mstsc 一致);`24` 真彩色;`16` 增强色(最省带宽,渐变有色带) |
| 图形管线（`graphicsMode`,「体验」页） | `auto` 默认协商托管 GFX/H.264;`classic` 强制经典位图/GDI,重连生效 |

远程桌面默认跟随工作台标签页:连接后、窗口缩放稳定时(350ms 防抖)以及窗口移到另一 DPI 的显示器时,插件通过显示控制通道请求服务器把桌面调整到视口的**设备像素**,同时带上本机显示缩放(150% 屏幕 → 远端界面 150%,deviceScaleFactor 取 100/140/180 中最近的一档),CS_CORE 里也带同样的缩放,远端文字不再只有本机的三分之二大。远程桌面与视图在设备像素上相差无几时(动态分辨率之后的常态),按 **1 个设备像素对 1 个远端像素** 绘制、原点对齐像素网格、不做插值;只有服务器跟不上窗口、尺寸确实不符时才按比例缩放(高质量插值)。显示控制只要求宽度为偶数。新会话直接按上一个视图的尺寸与缩放连接,省掉「先按设置尺寸全屏绘制、立刻调整再绘制一次」;`width`/`height` 是还没有任何视图时、以及服务器不支持动态调整时的回退尺寸。

画面路径:解码与像素转换在接收线程上、帧缓冲锁外完成,帧缓冲记录变化区域的并集,界面只把这部分拷进位图(原先每帧克隆整个桌面、1080p 约 8 MB,再整帧拷贝一次);位图按目标矩形裁剪,不画行尾补齐列。标签切走(视图移出可视树)或窗口最小化时停止呈现,并在服务器支持时发送 Suppress Output 暂停显示更新(不耗带宽、不解码),回来后恢复并发 Refresh Rect 重绘。远端鼠标形状(文本 I 形、调整大小箭头、打字时隐藏等)按服务器的指针更新显示。键入的文字每个快速路径 PDU 最多 7 个字符。会话断开后,桌面上出现“重新连接”浮层按钮(宿主会话标签同样可重连);重连会更新凭据及证书批准,重连失败时保留当前视图,按钮可再次点击。

**后台长时间保持连接。** 暂停显示更新的会话在链路上完全静默,而 NAT 网关、防火墙、云负载均衡与端口转发会在空闲几分钟后悄悄丢弃连接、不通知任何一端——回到标签时先看到冻结或黑屏的桌面,过一会儿才弹出断开提示。因此库对套接字开启 TCP keep-alive(空闲 30 秒后探测,每 10 秒一次、3 次无应答即判定断开),并在显示更新暂停期间每 30 秒重发一次“暂停”状态,让用户态端口转发也能看到流量;库调用方可用 `RdpClientOptions.KeepAliveInterval` 调整(`TimeSpan.Zero` 关闭)。链路仍然断掉时,约一分钟内就在后台发现,而不是等用户回来才发现。

**断网自动重连。** 网络层断开(`RdpConnectionLostException`:连接被重置、keep-alive 超时、服务器未给原因就关闭 TCP)后,标签像 mstsc 一样自行重连:最多 20 次,间隔 1、2、4、8、15 秒,之后每 30 秒一次,浮层显示“正在自动重新连接(第 n/20 次)”,手动“重新连接”按钮在等待期间同样可用。远端 Windows 会话在断开状态下保留,重连后桌面与程序保持原样;标签在后台时重连的新会话一开始就暂停显示更新。重连后一分钟内再次断开的会话沿用同一轮计数,链路持续失败时会放弃而不是无限循环。以下情况不自动重连:服务器带原因结束会话(注销、被接管、空闲时限、许可证等——Set Error Info 优先于其后的 TCP 关闭)、协议或通道错误、重连时服务器因凭据或证书拒绝。断开时正在进行的复制不会续传。所有断开(包括普通断网)都在插件日志里记录原因。

本地文字与输入法提交以 Unicode 发送。可打印按键缺少 `KeySymbol` 时,即使尚未收到首个预编辑更新,也交给本地文字输入处理,避免输入法首字母或普通文字重复发送。标记为 `ImeProcessed` 或 `DeadCharProcessed` 的按键保留在本地,包括输入法候选选择快捷键。其他 Ctrl/Alt/Meta 快捷键及导航键继续成对发送物理扫描码;NumLock 关闭时的小键盘导航按逻辑导航键处理。

### RD Gateway

在「RD Gateway」页填写 `gatewayHost` 与 `gatewayPort`（默认 443）。原来的服务器与端口仍是内网 RDP 目标，由网关解析。客户端通过 `/remoteDesktopGateway/` 使用 MS-TSGU RDGHTTP 的 HTTPS/WebSocket 传输，需要启用 WebSocket 传输的 Windows Server 2016+ 网关。网关连接仅走 TCP，失败不会回退直连；同时配置网关与 SSH 跳板会被拒绝。

`gatewayUseRdpCredentials` 默认 `true`，复用目标凭据；关闭后填写独立的 `gatewayUsername`、`gatewayPassword` 和可选 `gatewayDomain`，密码按秘密设置保存。HTTP 认证允许 Negotiate/Kerberos 与 NTLM，拒绝仅提供 Basic 的服务器和 HTTP 重定向。Linux/macOS Kerberos 需要下文的系统 Kerberos 配置，并为网关主机名配置 HTTP 服务主体。目标 RDP 的 TLS/NLA 仍单独校验原始目标身份。

网关证书必须通过系统信任链、主机名与有效期检查，或匹配独立的 `gatewayCertificateSha256` 叶证书 pin（64 位十六进制）。这个 pin 仅替代网关的系统证书检查，请通过可信渠道取得。网关证书拒绝不会进入目标 RDP 的证书批准流程；应安装网关 CA 或配置网关的独立 pin。

网关策略禁止当前请求的剪贴板或设备重定向时，在创建目标通道之前失败；请关闭相应重定向，或由管理员调整策略。不实现旧 RPC-over-HTTP、成对 HTTP IN/OUT 传输、网关 UDP、MFA/扩展认证、consent、NAP 与重新认证，要求这些能力的网关不受支持。测试覆盖协议报文、策略拒绝、取消及真实环回 TLS/WebSocket 传输；真实 AD HTTP 认证与生产 RD Gateway 互通尚未验证。

### RemoteApp / RAIL

在「RemoteApp」页的 `remoteAppProgram` 填写已发布程序别名，例如 `||notepad`。可选 `remoteAppArguments` 与 `remoteAppWorkingDirectory` 会发送给服务器，仍受其发布与参数策略约束。程序留空使用普通桌面；未填写程序但填写参数或工作目录会被拒绝。库调用方设置 `RdpClientOptions.RemoteApp`。

RemoteApp 协商 INFO_RAIL 和 RAIL/window 能力，执行 RAIL 握手与程序启动交互，跟踪服务器窗口的创建、更新和删除。工作区仅绘制应用窗口区域，支持选择、焦点与输入、移动缩放和关闭。这些应用显示在标签页工作区内，不是本机系统的独立顶层窗口。远端桌面尺寸在会话内固定，并为应用选择栏保留空间；修改尺寸需要重连。

RemoteApp 即使在 `graphicsMode=auto` 下也选择经典位图/GDI；不支持增强 GFX 窗口映射、本机桌面集成或托盘图标。程序启动失败会报错，不会默默改为完整桌面。协议、窗口生命周期和呈现回归已有测试；真实 Windows RDS 部署的已发布应用尚未验证。

### 智能卡重定向

在「磁盘映射」页开启 `smartCard` 并重连，显式允许远端会话使用本机 PC/SC 可访问的全部读卡器与卡片，默认关闭。客户端在 `rdpdr` 上注册 MS-RDPESC 智能卡设备（同时注册伴随 `rdpsnd`），并在用户登录前公告。库调用方设置 `RdpClientOptions.SmartCardRedirection`。

Windows 需要 Smart Card 服务和读卡器驱动（`winscard.dll`）；Linux 需要 `libpcsclite.so.1`、运行中的 pcsc-lite 服务与读卡器驱动；macOS 使用系统 PCSC 框架与兼容驱动。缺少系统库或服务不可用时，已选择智能卡的连接会明确失败。不随插件分发读卡器驱动或原生库。

核心操作包括上下文管理、读卡器/组枚举、状态变化等待与取消、连接/重连/断开、事务、状态、APDU 传输、控制和属性读取，支持 A/W 报文及会话内虚拟句柄。不支持的管理、缓存和 SetAttrib 操作返回不支持。断开会取消等待并释放上下文与卡片，不记录 PIN 或 APDU 内容。请仅对可信远端开启：此选项授予卡片操作访问权限，仍受卡片自身认证规则约束。这不增加智能卡 NLA/CredSSP 登录能力。离线协议与后端测试不能证明实体卡或 Windows RDS 互通，仍需部署验收。

Windows 与 Debian x64 原生烟测已加载生产 PC/SC 后端，并确认预期的服务不可用结果（`0x8010001D`）。Debian 另与已安装 C 头文件对照验证了 native-long 结构布局。烟测未启动服务或访问卡片；macOS 运行验证仍待完成。

### UDP 多传输与网络适配

「体验」页新增 `transport`（默认 `auto`，可选 `tcp`）和 `adaptiveNetwork`（默认 `true`）。类库分别对应 `RdpClientOptions.Transport`、`AdaptiveNetwork`。自动模式在服务器支持扩展协商和 Soft-Sync 时建立 MS-RDPEUDP v1 可靠 UDP（UDP-R/TLS）与不可靠 UDP（UDP-L/DTLS 1.2）侧隧道，通过 MS-RDPEMT 承载服务器选择迁移的动态虚拟通道；不声明 UDP v2 或 UDP Preferred。主连接、经典位图/GDI、键鼠和静态通道仍走 TCP，开启 UDP 不代表整桌面流量已切换。

UDP-R 使用 `SslStream`，UDP-L 使用纯托管依赖 `BouncyCastle.Cryptography` 2.7.0；UDP/DTLS 传输不增加原生库。两种隧道都必须匹配主 TLS 连接的 SHA-256 叶证书，创建 cookie 只在加密后发送。建链失败继续使用 TCP；成功确认后隧道故障会断开主连接，由现有重连逻辑恢复，避免跨传输重放设备操作。SSH 跳板自动强制 TCP，不绕过跳板直连目标 UDP；其他 TCP-only 代理调用方应设置 `TcpTunnel = true`。

**UDP-L 当前限制：** FEC 可恢复的丢包继续交付；进入应用阶段后，无法恢复的序号缺口会终止隧道并要求重连。DVC 分片没有消息标识，直接跳过缺口可能把下一消息拼入残缺消息。出站 UDP-L 动态通道消息限制为 600 字节，拒绝分片；大消息应使用 TCP/UDP-R。这不是任意残余丢包下的连续媒体播放保证。

`IRdpNetworkSource` 提供网络状态及变更事件，底栏显示 TCP/UDP-R/UDP-L、建链回退、服务器提供的 RTT、带宽和质量；未收到测量时保持未知。RTT ≥250ms 或带宽 <1000kbps 判为差，RTT ≥100ms 或带宽 <5000kbps 判为受限；恶化立即生效，改善需连续三个较好样本。弱网下鼠标移动按 25/50ms 合并，点击与键盘先按序送出待发移动，不被定时器拖延。自动网络检测允许服务器覆盖手工体验选项；关闭自适应后保留手工策略和不节流输入，仍可显示服务器测量。

离线验证包含乱序、重传、FEC、窗口、残余丢包保护、Soft-Sync 屏障、早期消息通道、真实环回 UDP 套接字上的 TLS/DTLS 双向通信（含 EMT 跨记录及同记录多消息）、证书不匹配拒绝及取消收尾。真实 Windows 服务端互通和 WAN 性能尚未验证。

### 远程音频播放

在连接的「体验」页勾选「远程音频」(`audio`,默认 `false`),保存并重新连接。声音输出到本机系统默认设备,不需要勾选磁盘映射。Windows 使用 WinMM、macOS 使用 AudioToolbox AudioQueue、Linux 使用 ALSA 的 `default` PCM;Linux 需要 `libasound.so.2` 和可用的默认播放配置,使用 PipeWire/PulseAudio 的桌面需要相应 ALSA 插件。没有本机输出设备的无头会话不能播放。

开启远程音频时同时注册 `rdpsnd` 与 `rdpdr`,即使没有映射磁盘、端口或打印机也完成空设备列表的 RDPDR 握手。MS-RDPEA §2.1 的 Windows 实现说明明确指出:没有 `rdpdr` 时,服务器不会在 `rdpsnd` 上发出数据。这不共享任何未选择的本机设备。诊断中应看到两个通道号均非零,随后出现音频格式协商。

`rdpsnd` 按 [MS-RDPEA](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpea/bea2d5cf-e3b9-4419-92e5-0e074ff9bc5b) 完成格式协商、质量模式、训练应答、WaveInfo/Wave 重组、Wave2 播放和 Wave Confirm。只接受服务器提供的 PCM:8/16 位、单/双声道、8–48 kHz;使用现有加密 RDP 通道,不额外开启 UDP 端口。不支持压缩音频编码、音调重定向或麦克风采集。音量由服务器应用和本机混音器控制。音频与桌面显示暂停独立,切换标签或最小化仍可播放。

音频接收、连续提交和完成确认采用独立流水线:处理一个 Wave 不再等待它播放完成,后续 Wave 与训练消息可以继续处理;Wave Confirm 仍按接收顺序、在对应块实际播放完成后发送,不以“已入队”代替完成。接收队列限制为 128 条消息且不超过 1 MiB;输出层另有覆盖待提交和在途音频的 128 块、1 MiB 与 2 秒累计时长预算(空队列允许单个合法的长音频块)。原生设备最多保留 32 个在途缓冲,设备已有至少 120ms 音频时暂停追加,单块本身可以超过这个水位。Windows 复用样本与头缓冲;macOS 分别跟踪每块的回调和实际播放终点;Linux 连续填充 PCM,以设备剩余帧数确定各块是否播放完毕,请求 40ms 的 ALSA 缓冲延迟。Linux 会重试临时写入中断并恢复 ALSA 欠载,仅保留尚未写入的样本,避免重复播放已接受的音频;设备移除或恢复失败仍会报告音频错误。服务器正常关闭与格式重协商先排空在途音频;取消和断连使未完成请求失效,由输出工作任务安全回收原生缓冲。输出设备不可用、格式不兼容或队列溢出会停止本次会话的音频,状态栏与日志报告原因,桌面连接仍然可用;修复默认设备后重新连接。库调用方可以订阅 `IRdpAudioSource.AudioPlaybackFailed`,晚订阅会重放已有错误。仅为磁盘/端口/打印机注册的伴随声音通道,在音频关闭时回报零个音频格式且不打开输出设备。

插件日志 `<数据根>/logs/plugins/velashell.rdp.log` 中的 `RDPSND` 行记录实际播放开关、通道注册与协商通道号、平台后端、格式协商数量、训练应答、首个音频块和首次播放完成确认。`playback=False` 表示本次连接未启用音频;通道号非零只表示通道已协商,应继续检查 `formats received` 和 `wave received`。首块 `silent=True` 表示该块 PCM 全部为数字静音;之后收到首个非静音块时会再记录 `silent=False`,不逐帧刷日志,也不记录样本或凭据。`first playback completed and confirmed` 表示本机设备报告完成且已发回确认,不能代替扬声器听音验证。持续播放时还会输出 `playout` 汇总(最多每 5 秒一次):`targetMs` 为抖动缓冲目标,`bufferedMs` 为设备与待提交音频的估算总时长,`correctionPpm` 为当前速率校正,`inFlight` 为已提交原生缓冲数。库调用方通过 `RdpClientOptions.DriveRedirectionLog` 接收这些诊断。

开发者可在插件仓库设置 `VELA_RDP_AUDIO_TONE_TEST=1`,运行 `dotnet test tests/VelaShell.Plugin.Rdp.Tests/VelaShell.Plugin.Rdp.Tests.csproj --filter FullyQualifiedName~SystemDefaultOutputPlaysProtocolTone`,通过 RDPSND 协议处理与本机默认设备播放两段低音量 440 Hz 测试音。此测试不连接远端、不采集音频,用于区分本机播放问题和远端协商问题;默认跳过。

若远端仍显示没有音频设备,先确认已开启音频并重新连接,再检查远端 Windows 的“允许音频和视频播放重定向”策略以及 Windows Audio 服务。自动测试验证协议、数据交付与故障处理。Linux CI 还通过进程私有的 null 设备调用真实 ALSA API,这不验证桌面音频服务或扬声器。macOS CI 通过 CoreAudio 查询默认输出设备,存在时执行 AudioQueue 静音播放、格式切换和重开测试,否则明确报告未执行实播验证。有输出设备的开发者可在任意支持平台设置 `VELA_RDP_AUDIO_TEST=1`,运行 `AudioOutputTests.SystemDefaultOutputPlaysAndReopens`。连续多缓冲已在 Windows 默认输出上完成静音播放、格式切换和重开验证;自动测试还覆盖 ALSA 部分写入/欠载恢复以及十分钟虚拟时钟下的 ±1000ppm 漂移。此次改动尚未在 Linux/macOS 真机上验证连续播放。

启动和欠载恢复时采用 20–80ms 自适应预缓冲,依据到达间隔与音频时长的偏差估计抖动;孤立短音效也会在有限等待后播放。持续播放时根据设备播放时钟与待播时长做慢速反馈,通过 PCM8/PCM16 线性插值与跨块的小数帧累计修正采样数量,速率修正限制为 ±0.5%,抑制发送端与输出设备时钟偏差造成的长期积压或欠载。格式切换、关闭和欠载恢复重置漂移控制状态。PCM 不重新压缩,但漂移校正会改变采样数量;这不是视频时钟同步,也不保证消除网络、服务器分块或设备自身的延迟。真实远端、扬声器听音、设备热插拔和长时间音画同步仍需在各平台现场验证。

### 远程按键悬浮球

插件不装全局键盘钩子,所以本机系统会先截走的组合键按了也到不了远端:Ctrl+Alt+Del 总由本机处理,Win+L 锁的是本机,Ctrl+Shift+Esc 打开本机任务管理器,Alt+Tab 切换本机窗口,Win+R / Win+D 由本机资源管理器处理,单按 Win 会让本机和远端的开始菜单一起弹出,Windows 11 的 PrintScreen 默认打开本机截图工具。这些键收在桌面边缘的**远程按键悬浮球**里:

- 平时是一个半透明圆球,默认贴在右边缘;可拖动,松手后吸附到较近的左/右边缘并记住高度。
- 描边用主题强调色(`VelaAccent`,跟随用户自定义的强调色):悬停时强调色描边淡入并向外扩出一圈光晕;展开时键盘图标旋转淡出、换成 ×,× 悬停时再转四分之一圈并铺上半透明强调色底;扇形里的按钮悬停时放大 8% 并亮起强调色描边。
- 点击后以两个半圆向内扇形展开,背后的桌面变暗。内圈是最常用的:Ctrl+Alt+Del、任务管理器(Ctrl+Shift+Esc)、切换窗口(Alt+Tab)、键入剪贴板;外圈是 Windows 键:开始(Win)、运行(Win+R)、显示桌面(Win+D)、锁定(Win+L)、截屏(PrintScreen)。点一项即发送并收起。
- 按 Esc 或点击变暗的桌面收起,这一下不会发到远端。视图较小时整个扇形(连同按钮与文字)按比例缩小,最小 0.6 倍,文字不互相遮挡。
- **键入剪贴板**经 SDK 剪贴板能力读取本机剪贴板文本(不记日志),逐字键入远端:连续文字按 Unicode 发送,换行按回车、制表符按 Tab。用于登录界面、UAC 提示、控制台等剪贴板共享不起作用的地方。含换行或超过 100 个字符时先在状态栏说明、需对同一段文字再点一次确认(换行会按回车发送,在控制台里会执行命令);超过 2000 个字符不键入。
- 原来顶部 36px 的工具栏只有 Ctrl+Alt+Delete 一个按钮,已经去掉,远程桌面得到状态栏以上的全部高度。

### 剪贴板与文件传输

剪贴板默认关闭；授权开启后使用普通 Ctrl+C / Ctrl+V，保留文本和图像的自动同步，不提供手动发送/接收按钮。

- **Windows 远端→本机**：远端 Ctrl+C 只读取文件/目录描述并发布 OLE 虚拟文件。用户在本机 Explorer 的目标目录按 Ctrl+V，Explorer 请求对应 IStream 时才通过 RDP FILECONTENTS 读取内容；目标目录与落盘完全由 Explorer 管理。插件不监听全局快捷键、不猜目标路径、不预下载缓存。
- **本机→远端**：焦点、窗口激活和远端 Ctrl+V 发布格式与名称/大小描述；目录枚举及安全源句柄准备不读文件内容。远端 Explorer 粘贴时发出 FILECONTENTS 请求才读取并发送字节，远端自行选择目标目录。Ctrl+V 仍等待格式列表 ACK；仅准备描述显示“等待粘贴”，字节数为零。
- 两个方向的文件描述都带 FD_SHOWPROGRESSUI（0x4000，MS-RDPECLIP 2.2.5.2.3.1，与 FreeRDP 一致），接收端的 Windows Shell 因此显示它自己的复制进度对话框；不带时大文件粘贴期间目标 Explorer 没有任何反馈，看起来像卡住。复制是否放到后台、目标 Explorer 窗口在复制期间能否继续操作由接收端 Shell 决定：远端取决于服务器 rdpclip 的数据对象；本机数据对象实现 IDataObjectAsyncCapability，允许 Explorer 在后台提取。插件状态栏的进度照常显示。
- 下载/上传全部放在现有**固定 24px 分辨率状态栏**内，每个方向在右侧占一组：方向 · 已传/总大小 · 当前文件名、3px 进度条、状态（传输中显示百分比，其余显示状态文字，按状态着色），以及与宿主传输面板相同的 10px “×”取消按钮（只在仍可取消的传输上出现）。无传输、单向或双向活动都不改变桌面可用高度；空间不足时先截断文件名、再截断状态文字，进度条和“×”不会被挤出；悬停提示给出精确字节数与完整状态。仅发布来源显示“可粘贴”；全部唯一区间交付后显示“文件内容已交付／已发送”，这是无需继续等待的终态，提示说明目标尚未报告复制结果，不代表目标磁盘持久化。远端上传协议没有最终复制确认；零内容来源在描述成功发送后即可进入“已发送”。本机 Shell 的成功 EndOperation（复制/移动效果）或非零 Paste Succeeded 才显示“复制完成（Shell 已确认）”；普通 DropEffect/TargetCLSID 不算完成确认。零字节、目录及 EOF 不要求额外内容块，未收到反馈仍可停留在“内容已交付”；失败/取消独立显示，新操作重置进度，迟到进度不会把终态改回“传输中”。状态结束不释放消费方持有的 IStream。
- 新 Ctrl+C 或剪贴板换主只撤换后续粘贴的来源；已取得的流持有独立租约。协商 CAN_LOCK_CLIPDATA 后，通过 LOCK/UNLOCK 和 FILECONTENTS 的 clipDataId 保留远端源版本；对端锁定的本机源句柄也跨新复制保留。撤换的版本等流及数据对象消费完成/释放后解锁；当前仍发布的版本保留用于再次粘贴。显式取消、断开或释放视图会取消活动流。对端不支持锁时，不能保证跨远端源更换继续传输；源程序退出或提供者失效即使有锁也可能失败，关闭源文件夹的真实行为仍需现场验证。
- 已发送的文件块请求单次响应等待 30 秒；只有响应超时才最多额外重试 2 次，分别可取消地退避 500/1000ms。重试保持同一源版本、文件索引、offset 与 length，每次使用新 streamId，迟到响应直接忽略；底栏显示“重试中”，只有成功交付的唯一区间计入进度。明确的 CB_RESPONSE_FAIL 没有瞬态分类，不重试；发送错误、显式取消、断开连接或无锁源版本变化也不重试。TCP 已提供可靠传输，服务器拒绝不是丢包证据；不承诺跨重连续传，不重传整文件、不重复写目标。
- 仅未锁定的本机上传源句柄闲置 30 秒后释放，下次远端 Ctrl+V 重新准备。双向生产文件路径均不创建磁盘缓存、中转或临时文件，也不累积整文件内存；原生 IStream 用最多 256 KiB 块直接填入目标应用提供的缓冲区。进度是插件取得/提供的字节，不是目标落盘确认。目标文件未显示不能证明用了缓存，也不能断言 Explorer 会读完整个流后才创建文件；创建、刷新、半成品清理及目标应用自己的临时文件均由目标应用控制。旧版本 DataDirectory 下的 rdp-clipboard 缓存不会主动删除，以免破坏仍被引用的文件。
- **Linux/macOS**：当前 Avalonia 集成没有可用的远端虚拟文件 provider，状态栏明确显示“远端文件粘贴仅支持 Windows”；不自动下载、不改成手动接收。文本、图像以及本机→远端按需文件传输不受影响。

Avalonia 12.1.4 的 `DataTransferItem.Set(format, Func<T>)` 支持延迟取值，但公开文件类型是 `IStorageItem`；应用自定义格式标识会转换，无法表达 Windows 原生 `FileContents` 的 `FORMATETC.lindex` 与 `TYMED_ISTREAM`。因此 Windows 使用独立 `WindowsVirtualFileClipboard`：OLE `IDataObject` 提供 `FileGroupDescriptorW`、带索引的内容流及复制效果。专用 STA 线程泵消息，流读取不阻塞 Avalonia UI 或 RDP 接收泵；关闭时不调用会强制物化内容的 OleFlushClipboard。所有权通过 OLE 身份和剪贴板所有者窗口检查；序列号用于避免发布时覆盖新的本地复制，不能把延迟渲染导致的序列号变化误判为换主。

OLE 支持 Performed DropEffect、Logical Performed DropEffect、Paste Succeeded 的 DWORD 回调及 16 字节 TargetCLSID。SetData 成功复制反馈后遵守 STGMEDIUM 所有权：fRelease=true 才释放媒介，拒绝时仍由调用方持有。反馈不等同 EndOperation，也不关闭消费方流；成功 EndOperation 释放操作租约，流继续持有独立租约，失败则取消提取。这修复了此前任何已接受反馈都会过早结束操作并释放流的问题。普通 DV_E_FORMATETC / DV_E_DVASPECT 探测通过 PreserveSig 直接返回 HRESULT，避免每次抛出托管 COMException；必需格式失败仍是错误，不能与服务器明确 FILECONTENTS 拒绝混为一谈。CLIPBRD_E_CANT_OPEN 使用最多 8 次有界异步退避重试，重试期间仍检查发布序列号，避免覆盖新复制；不读回自己的虚拟剪贴板来触发循环竞争。

版本 LOCK 在描述请求之前发送；clipDataId=0 是有效 ID，与不携带字段不同，LOCK 没有协议 ACK。按 [MS-RDPECLIP 文件内容请求处理](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpeclip/d19de33e-4027-43de-b150-668ee21bced8)，携带 clipDataId 的请求必须读取对应锁定版本，无法满足则返回 CB_RESPONSE_FAIL，响应不提供内部原因分类。低敏日志按适用事件记录 operation、streamId、lindex、offset、requestLength、attempt、responseFlags、lockCapability、clipDataId、超时/重试以及 LOCK/UNLOCK、迟到响应，不记录文件内容或凭据。现场可按 streamId 对齐请求与失败，再按 operation/clipDataId 检查版本生命周期；现有日志尚不足以确定真实服务器拒绝的内部原因，不能把已复现的本机反馈生命周期缺陷认定为现场失败的唯一根因。

FORMAT_DATA_RESPONSE 没有线上 request-id；本地附带的格式号不能作为独立响应身份。取消、超时或发送结果不明后保留原请求槽，后续接收在发新 LOCK/描述请求之前可取消地等待旧回复排空，不把迟到回复交给新请求。排空等待超过 30 秒会明确停止该连接的剪贴板共享，需重连恢复。格式列表通知按版本跳过过期队列项，迟到回复不再补投接收通知，同一已发布文件版本的重复接收不重新抓描述或读取内容。关闭先原子设置停止状态，再取消等待与流；不再启动 LOCK、描述请求或剪贴板发布，收尾等待工作队列及已登记的版本释放，每个 LOCK 至多一次 UNLOCK，重复释放调用等待同一收尾任务。新 Ctrl+C 仍仅撤换来源，既有流租约继续保留；明确关闭会取消所有流。正常取消不记运行失败警告，真实响应拒绝、超时和失步仍保留诊断。调试器的 first-chance OperationCanceledException 可以来自正常取消；没有调用栈的宿主 Infrastructure 日志不足以判定额外缺陷。

继续保持 16 MiB 单次载荷、8192 图像边长、16 MiB 解码图像像素、4096 文件/目录描述符、合计 1 GiB 文件大小上限。拒绝路径穿越、不安全名称、大小写冲突、文件作为父目录和符号链接，保持目录层级。剪贴板与磁盘映射、宿主 SFTP 队列相互独立。文件块最大 256 KiB，通道按 1600 字节逐帧发送并释放发送锁；接收泵只做有界入队，队列溢出明确停止剪贴板共享。

本机聚焦回归验证“复制零内容读取、请求才传输”、有界大 Read、多文件/目录、挂起流跨换主继续、版本锁/解锁、显式取消/断开/释放、OLE 完成回调、busy 重试及固定底栏几何。Windows 测试通过真实 COM vtable 调用 IDataObject 的 HGLOBAL 和 IStream 的读取、定位、释放，不更改用户系统剪贴板、不粘贴到用户目录。未连接公网服务器；真实 Explorer 与 RDP 服务器端到端粘贴仍需现场验收。

### 磁盘映射

**树形资源选择（依赖 SDK 2.1.7 及配套宿主）：**「磁盘映射」页按驱动器、打印机、便携设备分为可折叠的复选树，分类默认折叠并显示已选/总数。点击分类可全选或全不选，部分选中时显示半选状态，再点击会全选；展开长列表时在树内滚动。没有设备的类别不显示。摄像头仍为单选下拉，COM/LPT 端口与自定义文件夹仍使用文本输入。只改变选择界面，原保存键值、默认关闭和各类映射数量限制不变；已有选择可回显，后来插入的设备不会自动勾选。该界面依赖的 `SelectionGroup` 契约已随 SDK 2.1.7 发布，2.1.6 不含此契约；宿主及插件包引用已升级至 2.1.7，但 RDP 清单的 `minSdkVersion` 仍为 2.1.5，发布插件前须同步至 2.1.7 并完成配套宿主联调。

连接对话框的「磁盘映射」页仿照 mstsc「本地资源 → 驱动器」,每块本机磁盘一个复选框。Windows 列出全部盘符,按类型命名,如「本地磁盘 (C:)」「可移动磁盘 (E:)」「网络驱动器 (Z:)」「光盘驱动器 (F:)」;Linux / macOS 列出根目录(`/`)、主目录(`~`)以及 `/media`、`/run/media`、`/mnt`、`/Volumes` 下的挂载点。全部**默认不勾选**。

枚举磁盘只读盘符与类型(或挂载点),不读取介质、不查询卷,因此断开的网络驱动器或空光驱不会拖慢插件激活。插件每 5 秒检查一次磁盘集合,只有变化时才重新注册连接表单(同一个实现实例:已打开的远程桌面标签不受影响,正在编辑的对话框也不会丢失已填内容),稍后插上的 U 盘在下次打开连接对话框时出现。连接时已勾选、但当前不存在的磁盘(U 盘已拔出、网络驱动器离线)直接跳过,不报错。

同一页的「自定义文件夹」用于只共享某个目录:写 `名称=绝对路径`,多条用**分号**分隔(宿主把文本设置渲染为单行输入框,无法换行)。Windows 示例:`work=C:\Work\Share;docs=D:\Docs`;Linux:`work=/home/alice/share`;macOS:`work=/Users/alice/share`。早先保存的按行分隔的值仍可识别。输入框下方常驻的提示会按当前系统给出写法与示例(占位符一输入就消失,所以示例放在提示里);填错时连接报错会引出写错的那一条并给出正确写法,名称重复时点名重复的名称,超过 32 个时说明上限。

名称必须唯一,也不能与已勾选磁盘的名称(`C`、`E`、`root`、`home` 等)重复;只允许 **1–8 位 ASCII 字母、数字或下划线**,不允许连字符;磁盘与文件夹合计最多 **32 个映射**。目录必须存在。远程 Windows 桌面通过 `\\tsclient\<名称>` 访问,xrdp 会话在 `~/thinclient_drives/<名称>`。每个映射授予远程对该磁盘或文件夹的**读写访问**;除了用户勾选的磁盘和填写的文件夹,不共享任何主目录、系统盘或可移动盘。路径限制在映射目录内,拒绝路径穿越与符号链接。映射磁盘或目录与经剪贴板发送文件是两个独立选择。

启用磁盘映射时,同时注册 `rdpdr` 与配套的 `rdpsnd` 静态通道;部分 Windows 服务器在缺少后者时不会启动磁盘握手。这项兼容性注册不会开启音频播放。磁盘通道在等待许可与 Demand Active 时也接收并重组数据,避免丢弃提前到达的握手;服务器重新发送 Server Announce 时开启新一轮设备公告。插件日志 `<数据根>/logs/plugins/velashell.rdp.log` 中的 `RDPDR` 行记录映射数量、实际加载的引擎路径、协商通道号、登录通知、握手包类型、设备注册结果及失败的 I/O 状态码,不记录文件内容或凭据。库调用方可设置 `RdpClientOptions.DriveRedirectionLog` 接收这些后台线程诊断。只有通道号或登录通知并不表示设备已注册;应继续检查 Server Announce (`0x496E`)、设备公告与 Device Reply 的状态码。网络发现不是访问 `\\tsclient\E` 的前提。

在插件仓库调试时使用 [CLI 开发内环](../cli/cli.md):将 `VelaShell.Plugin.Rdp` 设为启动项目,选择 `VelaShell` 配置后按 F5。`--dev-root` 扫描构建目录的父目录(例如 `bin/Debug`),开发插件从影子副本加载。宿主目录或用户插件目录中的同 id 插件优先于开发根;联调前应移走冲突副本,并取消继续向宿主复制该插件的本机构建配置。

### COM / LPT 端口（Windows）

在「磁盘映射」页的「COM / LPT 端口」明确填写名称，例如 `COM3;LPT1`，默认留空。最多选择 32 个唯一名称，允许 `COM1`–`COM9999` 或 `LPT1`–`LPT9999`，拒绝路径、冒号、前导零别名。库调用方使用 `RdpClientOptions.Ports` 和 `PortMapping`。只有远程请求打开时才独占打开所选本地端口；其他程序占用时可能返回共享冲突。远程会话能读写和配置这些端口，服务器必须允许端口重定向；远程应用访问重定向的 COM/LPT 设备，而非文件共享。

串口和并口均遵循 [MS-RDPESP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpesp/04ae8f6b-a2fe-4989-9314-09bff11fa086)，使用 `rdpdr` 及兼容性配套 `rdpsnd`。端口公告不含设备专属数据。内部可复用 `IRdpdrDevice` 契约将非磁盘设备分发与文件系统处理分开。读写要求零偏移，单次最多 256 KiB，同时最多 64 个非磁盘请求。串口控制通过精确长度和值校验的允许清单：波特率、队列、线路格式、超时、字符、流控、break、DTR/RTS、XON/XOFF、即时字符、purge、事件掩码及等待、modem/通信状态与属性。并口支持打开、读写、关闭、查询信息及 INIT。未知、厂商专属和其余控制明确失败，不伪造成功。原生串口结构按定义的线上布局传输，包括去除 SERIAL_STATUS 对齐填充。

Windows 使用重叠 I/O，在关闭或断开时调用 `CancelIoEx`。读、写和 `WAIT_ON_MASK`（`WaitCommEvent`）与本机端口一样等待，只受远程应用自己设置的 COMMTIMEOUTS 约束，线路空闲时不会凭空报错；清空队列、更改事件掩码、关闭或断开会结束它们。不等待数据的配置类控制请求超过 30 秒即取消。挂起读取不会把控制和关闭请求串行堵在后面，读写返回实际字节数。即使取消，句柄和固定缓冲区也保留到内核真正完成；故障驱动若不完成取消，资源释放可能超过期限，不会提前释放内核仍使用的内存。原生打开和提交操作也依赖驱动响应。暂不支持热插拔重连、驱动专属透传或 Linux/macOS 端口后端。假后端测试覆盖选择、公告、畸形请求、短读写、控制拒绝、超时及关闭取消；真实硬件与 Windows 服务器互通仍需单独验收，测试不打开真实 COM/LPT 硬件。

### Legacy 打印机（Windows）

「磁盘映射」页为每个本地已安装打印队列提供独立勾选项，全部默认关闭。激活时及每分钟在后台只读查询队列元数据；队列变化后重新打开连接表单。最多选择 16 个队列。所选队列缺失时连接设置明确失败，不会改选其他打印机。排除 FILE:/PORTPROMPT: 输出端口及嵌套 TS 重定向队列。库调用方可在 UI 线程外调用 `WindowsPrinterCatalog.GetPrinters()`，把明示选择的 `PrinterMapping` 传入 `RdpClientOptions.Printers`。

实现 [MS-RDPEPC legacy 打印重定向](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpepc/993fd26f-b31b-4524-90ec-3efb59445f2a)。远端服务器必须允许打印机重定向，并安装匹配驱动。**不支持 Easy Print、XPS、驱动安装或驱动配置缓存。** 公告使用 PRINT=4、PRN数字 DOS 名及 UTF-16 零终止字符串，仅声明真实的默认/网络属性。远端 CREATE 打开明示选择的已安装队列并创建固定文档名的 RAW 任务；WRITE 补齐 spooler 短写；CLOSE 完成并释放任务。不向 Winspool 透传远端路径、文档名、驱动安装或任意 IOCTL。

每队列最多 4 个任务、单次写入 1 MiB、单任务 64 MiB；请求按到达顺序处理，畸形写入、异常及断线会中止未完成任务。Winspool 可能阻塞在驱动内部：reset/断线立即使旧请求失效并后台清理，但无法强制打断卡住的原生调用，须待其返回才能释放任务。测试使用假任务验证协议字节、限制、顺序及清理；本机仅实际执行过只读队列枚举，没有提交真实打印任务，也未验证真实服务器到打印机的端到端互通。

### 便携设备 / WPD（Windows）

「磁盘映射」页逐项列出 Windows Portable Device 接口，默认均不勾选。选择后允许远程会话通过 WPD 读取和修改该设备。此能力**不是通用 USB 重定向**：不接收任意 USB 设备、原生控制码或服务器指定的文件系统路径。激活时及每五秒在后台通过 SetupAPI 只枚举元数据，不打开设备。库调用方把 `RdpWpdDevices.Enumerate()` 返回的元数据 ID 填入 `RdpClientOptions.WpdDevices`，默认空集合。即使桌面分辨率固定，选择 WPD 仍会启用所需动态通道。连接时所选设备缺失会明确报错，不会替换成其他设备。

[MS-RDPEPNP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpepnp/39e5a1ae-bd17-4dd2-8ea5-63c150c337cf) 使用一个 `PNPDR` 控制通道及多个 `FileRedirectorChannel` 实例，每个文件实例独占一个句柄。交换 1.5 版本后，等待服务器认证消息才公告设备。文件通道版本 4 支持 CreateFile、读、写、WPD IOControl 和按请求 ID 取消，不声明 custom events。Windows 后端使用重叠 I/O，通过 `CancelIoEx` 精确取消原请求。只允许 SDK 定义的 [WPD read/read-write 消息 IOCTL](https://learn.microsoft.com/en-us/windows-hardware/drivers/portable/access-control)：`0x00404108` 和 `0x0040C108`。上限为选择 32 个接口、16 个文件通道、每句柄 4 个挂起请求、每次输入/输出 1 MiB。错误按 HRESULT 回复，畸形请求关闭对应文件通道。

Windows 8+ 使用 [CM_Register_Notification](https://learn.microsoft.com/en-us/windows/win32/api/cfgmgr32/nf-cfgmgr32-cm_register_notification) 无窗口监听接口到达与移除；先注册再取得初始快照，原生回调只排队。串行 worker 在空闲或活动设备拔出时撤销设备，取消 I/O 并关闭句柄。重插只对连接开始时已授权、系统唯一 PnP 实例身份（`CM_DEVCAP_UNIQUEID`）相同、硬件 ID 一致且当前恰有一个匹配接口的设备自动恢复。恢复时更新本地路径并公告全新的协议 DeviceId，旧句柄与旧响应不会指向新实例。友好名称不作为身份证据。缺少稳定身份、匹配歧义或未选择的新设备均不会自动共享；身份不明确的设备需重新选择建立新连接。修改已保存设置不会改变现有会话的授权集合。

通知注册失败会明确记录降级诊断，此时只保留 I/O 错误检测到的撤销，不具备空闲拔出检测或自动重插恢复。关闭会话时注销通知、收尾 worker 并释放句柄；有界事件队列溢出时保守撤销活动设备。fake 后端及 fake 通知测试覆盖认证、真实报文、选择白名单、句柄隔离、读写/IOCTL、取消、空闲拔出与重插、重复及未选设备事件、初始化竞态与关闭。没有实际打开、拔插或修改 WPD 设备；真实设备与 Windows 服务器互通仍未验收，受远端策略及驱动支持影响。

## 认证与证书策略

安全层由 `security` 决定:**自动**(默认)同时提供 TLS 与 NLA,由服务器选择(支持 NLA 的服务器会选 NLA);**仅 NLA** 要求服务器选 NLA,绝不回退;**仅 TLS** 只提供 TLS,凭据在已校验的 TLS 通道内随 Client Info 发送,由服务器自己完成登录(相当于服务器关闭了「要求 NLA」)。TLS 最低 1.2;禁用旧式 RDP 安全及“忽略所有证书”,各模式的证书校验完全相同。没有 pin 时,核验系统信任链、服务器用途、有效期和主机名/IP,由 .NET 的 `SslStream` 用各平台的 OS 信任存储完成。

**NLA 的认证实现。** NLA/CredSSP 的认证令牌与封包在 **Windows 上走系统原生 SSPI 的 Negotiate 包**,与 mstsc 一致:主机名为 DNS 名时以 `TERMSRV/<主机>` 为服务目标,域服务器可走 Kerberos;IP 地址保持空 target(Kerberos 对它无从谈起,显式凭据下 Negotiate 实际都落到 NTLM)。Linux/macOS 使用 `NegotiateAuthentication(Package = "Negotiate")` 的 SPNEGO 协商:优先尝试系统 GSSAPI Kerberos,不可用时由 .NET 协商托管 NTLM,无需额外安装 NTLM 提供程序。公钥绑定按 MS-CSSP 的 v5+ SHA-256 nonce 绑定实现;**服务器的 CredSSP 版本低于 5(CVE-2018-0886 修复之前)一律拒绝**,也不会因此回退到 TLS——mstsc 默认的「已缓解」策略同样拒绝。TLS 只用 1.2/1.3 并拒绝重协商;Client Info 与 TSCredentials 里的明文密码发送后即清零,选项记录的 `ToString` 不输出密码。

**Unix 提供程序初始化。** 引擎在创建认证上下文前启用 `System.Net.Security.UseManagedNtlm`。在当前 .NET 中,该开关让 `Negotiate` 使用托管 SPNEGO 和 NTLM,同时保留系统 GSSAPI Kerberos,并不强制使用 NTLM。主程序及隔离插件宿主的 `runtimeconfig.json` 也从进程启动时启用它,避免其他插件先调用认证 API 后缓存旧值。该开关影响同进程的其他 `NegotiateAuthentication` 客户端;Windows SSPI 路径不变。独立嵌入引擎的应用也应在启动配置中设置它。部署时更新宿主与插件并完整重启;普通 NTLM 环境无需安装 `gss-ntlmssp`。

**禁用 NTLM 的域环境。** Linux 需要可用的系统 Kerberos/GSSAPI 库(例如 Debian/Ubuntu 的 `libgssapi-krb5-2`),macOS 使用系统 GSSAPI。配置系统 Kerberos realm 与 KDC 发现(`krb5.conf` 或组织的系统配置),确保 DNS、KDC 网络连通和时钟同步。连接地址使用与服务器 `TERMSRV/<FQDN>` SPN 匹配的 DNS 名;IP、未注册的别名和重复 SPN 不能保证 Kerberos 成功。账号可用 `user@EXAMPLE.COM`,或用户名 `user` 加域/realm `EXAMPLE.COM`;不要假定 NetBIOS 短域名一定能映射到 Kerberos realm。显式账号密码通过 GSSAPI 获取服务票据;SSH 隧道仍使用原始服务器名称作为 SPN。可用 `kinit user@EXAMPLE.COM`、`kvno TERMSRV/rdp.example.com` 检查系统配置。服务器禁止 NTLM 时必须修复 Kerberos 配置;选择“仅 NLA”可明确禁止 TLS 回退。

**降级保护。** X.224 协商是明文的,中间人可以回答「只支持 TLS」——若用户在首次连接时信任了它的证书,密码就会随 Client Info 落到它手里。因此自动模式下,某台主机(主机+端口)**一旦用 NLA 登录成功就被记住**(插件存储),之后对它要求 NLA;它若改为只提供 TLS 或拒绝 NLA,连接会在发送任何密码之前停止,并提示这可能是有人在中间强迫降级。若确实是服务器关闭了「要求 NLA」,在连接的高级设置里把安全层改为 TLS 即可。从未用 NLA 登录过的主机(例如不接受我们 NTLM 的那类)仍按下面的回退处理。

**自动模式的回退。** 仅当首个令牌可确认为不含凭据证明的 NTLM Type 1(裸令牌,或只提供 NTLM 的 SPNEGO 包装),且服务器在第一条回复返回错误或中止 TLS 时,自动模式才允许**只提供 TLS 重连一次**,证书照样校验。Kerberos AP-REQ 在首包就可能携带身份凭据证明;其被拒、断连、未知机制或畸形令牌都不能触发这种回退。凭据提交后的失败和旧版 CredSSP 同样不允许降级。若服务器要求 NLA 又拒绝 NTLM,应先使用正确的 FQDN/SPN 和 realm 配置 Kerberos;没有 Kerberos 的环境只能由管理员决定是否调整服务器认证策略。已记住必须 NLA 的主机和“仅 NLA”模式均不会回退。

**连接失败的诊断。** 错误指明失败的阶段(TCP、安全协商、TLS、NLA、MCS、登录、许可、能力交换)和原因链,含 SChannel / 套接字错误码;TLS 握手之后的失败还附上协商出的 TLS 版本。服务器在 NLA 中拒绝登录时,按它返回的状态码给出具体原因(用户名或密码错误、密码过期、账号锁定或停用、不允许远程登录)。**连接成功后又被服务器断开**时(服务器发来 Set Error Info PDU,MS-RDPBCGR 2.2.5.1.1),按它的 errorInfo 码给出原因并显示在断开浮层与状态栏:用户被注销、会话被另一连接接管、服务器拒绝或权限不足、空闲/登录超时、没有可用的远程桌面授权(RDS 授权或宽限期过期)等;其余码回退到中性描述加十六进制码。断开日志还记录**此前是否收到过登录通知**(服务器的 Save Session Info PDU):收到说明凭据已被接受、Windows 会话已启动,随后的断开是服务器结束了一个已建立的会话(多为策略),而非认证或协议失败;未收到则说明尚未走到登录。插件日志(`velashell.rdp`)记录每次失败的完整异常;TLS 握手之后的失败再附上加密套件和服务器发来的 TLS 记录头(只记类型与长度,例如 `Alert 26 B`,从不保存载荷),用来区分「服务器中止」与本地解密问题。

宿主批准的 pin 是叶证书 DER 字节的精确 SHA-256(64 位十六进制)。**pin 替代 CA/名称/有效期验证**,只做叶证书身份的定时比较,不是在上述检查之上附加一道条件。叶证书变化会拒绝连接,需要重新明确批准。SSH 转发不会把预期服务器名称改成回环地址。密码通过 API 传递,不进入命令行参数。NLA/CredSSP 用 Windows SSPI 或 Unix 的 .NET Negotiate 完成 Kerberos/NTLM 与消息密封,TSRequest 公钥绑定按 MS-CSSP 实现。

## 许可协商

在**许可协商**阶段出现 `RDP licensing negotiation (msg 0x01) is not supported`，表示服务器要求执行旧版引擎尚未实现的 MS-RDPELE 许可交换。该阶段位于 TLS/认证之后，与 Unix NTLM 初始化失败是不同问题。托管引擎现支持 `SERVER_LICENSE_REQUEST` → `CLIENT_NEW_LICENSE_REQUEST`、平台挑战/响应及经过认证的新许可/升级许可；服务器的 `STATUS_VALID_CLIENT` 也可完成协商。其他许可错误保留错误码与状态转移码并明确失败，不会当作授权成功。重传/重置请求有上限，已开始的许可交换不能通过提前激活绕过；等待许可时仍分发早到的虚拟通道消息。

交换在已校验的 TLS 通道内使用 [MS-RDPELE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpele/88061224-4a2f-4a28-a52e-e896b75ed2d3) 规定的 RSA 预主密钥传输、密钥派生、逐 blob 重置的 RC4 与 16 字节 MAC。专有证书校验协议签名；X.509 链以 TLS 对端提供的许可根为锚进行校验，不安装根证书、不下载证书，也不替代 TLS 身份校验。无效许可证书会回复中止消息。空许可证书会明确拒绝：TLS 会话没有可按规范回退使用的 GCC 证书。暂不持久化 CAL，因此后续连接重新请求许可，实际授权仍由服务器策略决定。

## 对恶意服务器的边界

引擎与宿主同进程,所以服务器发来的一切都有上限,畸形输入只结束会话(或只丢掉那一项),不会冻结界面、耗尽内存或越出映射文件夹(2026-11-01 审计后加固):

- **画面**:位图矩形先按目标矩形与画面裁剪再解码,单边 ≤8192,每个更新的像素总量 ≤4 个桌面面积;解码在锁外完成;短的 RLE 流剩余部分涂黑,不会把别处内存残留画到屏幕上。服务器可强加的桌面面积 ≤8K UHD。绘图指令每个更新的绘制量 ≤16 个桌面面积,位图缓存总量 ≤256 MiB,单个缓存位图边长 ≤4096;未声明的指令类型结束会话(指令流无法跳过未知指令)。
- **通道与消息**:静态虚拟通道重组以第一块声明的总长为准(≤32 MB),实际缓冲不得超过它;rdpdr 队列另有 64 MB 字节上限;动态通道只为已打开的显示控制通道缓冲,消息 ≤64 KB;CredSSP 消息 ≤1 MB。
- **磁盘映射**:单次读取 ≤1 MiB,同时打开的文件 ≤4096,写入起点最多超过文件末尾 256 MB,每个请求都有应答;路径逐段校验——拒绝 `.`/`..`、控制字符,Windows 上再拒绝备用数据流(`:`)、设备名(CON、NUL、COM¹ 等)与结尾的点/空格;映射根之下的符号链接/联接在打开、创建与重命名目标时一律拒绝(映射根本身可以是链接;OneDrive 占位文件不算链接);非 Windows 上前缀比较区分大小写。
- **剪贴板文件名**:拒绝 Unicode 格式字符(如 U+202E 可把 `.exe` 伪装成 `.txt`)、`CONIN$`/`CONOUT$`、`COM¹–³`/`LPT¹–³`;文件/目录冲突检查为线性复杂度。
- X.224 明文 cookie 只放可打印 ASCII 且限长。

## 构建与独立交付

保持仓库的 .NET 11、SDK 与 Avalonia 设置;GFX 新增纯托管 `ManagedCodecs.H264` 0.6.0 依赖。**只有托管构建,没有原生步骤**:

```bash
dotnet build plugins/VelaShell.Plugin.Rdp -c Release
dotnet build plugins/VelaShell.Plugin.Rdp -c Release -t:PackVpx
```

Windows 虚拟文件剪贴板通过 `WindowsVirtualFileClipboard` 调用系统 OLE/user32/kernel32。另一平台相关处是 `UnixClipboardFile`:它在 Unix 上 P/Invoke 系统 `libc` 的 `open`/`fstat`,以 `O_NONBLOCK|O_NOFOLLOW` 安全打开用于本地→远程传输的剪贴板文件(避免 FIFO 阻塞、拒绝符号链接),Windows 用 .NET 文件 API。系统 libc 随操作系统提供,不是捆绑的原生库。

不再有 `rdp-native` 工作流、CMake、随包 OpenSSL/VC 运行库或六平台原生任务:插件以**一份托管 `.vpx`** 交付,全 RID 通用。包内包含 `ManagedCodecs.H264.dll`、`ManagedCodecs.Core.dll` 和 `VelaShell.Rdp.THIRD-PARTY-NOTICES.txt`,保留 MIT/Apache-2.0 与依赖版权声明。构建与打包仍为托管流程;Ubuntu 认证 CI 额外使用临时系统 MIT Kerberos KDC 做集成测试。工具链细节见 [插件构建 README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md)。另见 [打包与签名](../templates/publishing.md)。

## 验证情况与剩余边界

`ManagedNtlmTests` 在没有安装 `gss-ntlmssp` 的环境中验证裸 NTLM 和 Unix 默认 SPNEGO、NTLMv2 凭据证明、机制列表 MIC、额外协商轮次及连续封包/解包。`CredSspNegotiationTests` 覆盖 Kerberos 首包被拒或断连不降级,以及立即完成的上下文只发送一次令牌并使用 v5+ 绑定。`KerberosNlaTests` 配合 `scripts/Test-RdpKerberos.sh` 启动隔离的 MIT KDC,分别使用 UPN 和用户名加 realm,明确断言实际机制为 Kerberos,经 TLS 验证 CredSSP 公钥绑定及最终凭据加密。脚本只使用临时数据库、配置、票据缓存和服务 keytab,结束时清理。

本次在 Windows 与 WSL Debian/.NET 11 RC3 运行了认证回归,并在 Linux 通过两项真实 KDC 的 CredSSP 交换测试。现有测试工程另有四个与认证无关的文件无法编译(`GfxRemoteFxTests`、`DynamicChannelTests`、`RdpGfxTests`、`WpdHotplugTests`);本地验证临时排除它们,未修改这些文件。CI 的 Ubuntu/macOS 认证选择器已扩充,Ubuntu 增加隔离 KDC 步骤;本次尚未执行更新后的远程 CI、macOS Kerberos 或真实禁用 NTLM 的 Windows AD 桌面登录。

许可修复在 Windows 和 Linux .NET 11 RC3 容器上均通过 59 项许可/认证/通道聚焦测试。`LicensingTests` 与 `LicensingExchangeTests` 使用独立服务端实现校验 RSA 解密、密钥派生、RC4/MAC，覆盖 512/2048 位专有密钥和 X.509 链的完整交换、伪造签名/MAC、畸形消息、重置/重传上限、错误回复，以及真实连接循环在 Demand Active 前发送 MCS 应答。Ubuntu 和 macOS CI 均通过全部 47 项认证/许可测试，包括旧密钥交换用例。这些是模拟交换，并非连接日志中的真实 Windows 服务器；macOS 到 Windows 的桌面互通仍需现场复验。

大文件阻塞的本地根因是下载长期持有发布操作锁、焦点发布等待占据键鼠队列，以及完整虚拟通道消息长期占据发送锁；原 cliprdr 队列丢写和 worker 吞异常也会造成请求无响应。聚焦回归覆盖挂起大文件响应时本地发布仍可完成、取消挂起内容请求且不创建本地缓存、进度字节和乱序重传、工作台状态/取消/画面刷新、焦点发布与键鼠队列分离、分片间输入与同通道消息顺序、队列溢出显式失败。磁盘 DriveCapability v2 的 DeviceData 已按 UTF-16LE 编码，8 字节 PreferredDosName 仍为 ASCII，并以三个名称长度的精确报文字节测试覆盖；尚未验收真实 Windows 对该磁盘声明的接受情况。上述大文件修复未连接公网服务器，真实 Windows 双向大文件和慢网体验仍需现场复验。

剪贴板通道发送分片遵循协商的 `SHOW_PROTOCOL` 选项;接收分片先按 `CHANNEL_PACKET_*` 标志使用连接共享的 MPPC 历史解压,再重组交给剪贴板解析器。未协商长格式名时,ASCII 注册名使用 32 字节短名称字段,避免截断 `FileGroupDescriptorW`。本地回归覆盖压缩的空载荷格式列表 ACK、分片压缩的文件描述符、长/短格式名和双向文件请求。旧实现会把压缩 ACK 当作原始 PDU,解析失败后发布等待超时;这条故障路径已在本地复现。真实 Windows 服务器的双向文件 Ctrl+C/V 仍需复验,没有服务器通道抓包时不能断言其超时一定由压缩 ACK 引起。

单元测试覆盖设置、剪贴板编解码、headless 界面与 CredSSP/NLA 的 ASN.1 编码。活连接互操作测试门控于 `VELA_RDP_TEST_TARGET`(CI 跳过),对真实服务器验证整条链路:

- TLS 协商与叶证书 pin 信任;无 pin 拒绝自签证书并抛出 SHA-256 供宿主信任流程。
- 桌面渲染:交错 RLE **与 MPPC 批量压缩**(服务器开启 `bulk_compression` 时,实测解压压缩 PDU、像素精确)。
- 键鼠/Unicode 输入;文本与图像剪贴板往返;磁盘映射:勾选的磁盘可被远程读取(含系统盘根目录 `C:\` 的列目录),自定义文件夹读与写。
- **动态分辨率**:经显示控制通道请求调整桌面,实测 1024×768 ⇄ 1280×1024 往返生效,重激活后帧缓冲与状态栏随之更新。
- **宿主内渲染**:真实 `RdpWorkspaceView` 与 `RdpDesktopControl` 经 Avalonia headless Skia 渲染托管引擎的帧,即宿主装载的同一套控件树;实测视图会把远程桌面调整到视口尺寸。
- **颜色深度**:对 xrdp 以 16/24/32 位分别连接,24 位(交错 RLE)与 32 位(RDP 6.0 planar,实测 295 个 planar 矩形)解出的同一桌面逐像素一致,锁定 planar 的方向与通道顺序。
- **远端指针**:实测收到并解码服务器的 32×32 I 形光标(热点 15,16)。
- **绘图指令**:开启与关闭绘图指令各连一次 xrdp,桌面、以及滚动 2000 行之后的终端(xterm)逐像素一致;xrdp 每次发约 70 个指令更新、300–500 次缓存 MemBlt。xrdp 的 Xorg 后端不发 ScrBlt、填充与图案画刷,这几类只由单元测试覆盖(`OrderDecoderTests`:每种指令与缓存格式、边界、增量坐标、截断与 2000 次随机变异;`RdpDrawingOrderTests`:重叠拷贝、ROP3 真值表、画刷原点、裁剪、两条更新路径)。

加固与画质相关行为另有离线测试:`RdpHardeningTests`(含真实目录联接,以及发送超大 TSRequest、CredSSP v3 的假服务器)、`RdpRenderingTests`(1:1 绘制、DPI、Suppress Output、颜色深度设置)、`RdpBackgroundSessionTests`(TCP keep-alive、断开原因区分、自动重连及后台重连暂停更新)、`PlanarTests`(FreeRDP 测试向量与模糊测试)、`RdpPointerAndInputTests`(指针掩码规则、缓存、批量键入)。

NLA/CredSSP 除离线 ASN.1 单测外,对 FreeRDP shadow 服务器(`/sec:nla`,门控于 `VELA_RDP_TEST_NLA_TARGET`)做了活连接验证:**原生 SSPI(SPNEGO)路径与托管 NTLM 兜底路径都能登录到桌面**(Windows 上两者都测);错误密码报认证失败且不回退;「仅 TLS」被要求 NLA 的服务器以 `HYBRID_REQUIRED_BY_SERVER` 拒绝。安全层回退与「首条 NLA 回复即被拒」(错误码、TLS 中止两种形态)由进程内假服务器测试覆盖;TLS 1.2 下 Windows 服务器可能选用的各加密套件(ECDHE/RSA × AES-GCM/AES-CBC)已逐一对 xrdp 实测。真实 Windows 服务器方面,核实过一台拒绝传入 NTLM 的服务器的行为(首条消息即回 0x80090302 或中止 TLS),**真实 Windows 服务器的 NLA 登录成功路径仍未在活连接上验收**。交错 RLE / MPPC 解码、MCS/GCC、能力协商与各通道线格式按 MS-RDPBCGR 及相关公开规范实现。这些验证不构成全平台或真实 Windows 端的最终认证。

### 摄像头（Windows）

「磁盘映射」页提供摄像头单选项，默认「关闭」。激活时及每分钟在后台通过 Media Foundation 只枚举元数据；设备变化后重新打开表单。已保存选择只匹配该设备，缺失时不会回退到默认摄像头。库调用方可使用 `RdpCameraDevices.Enumerate()` 并设置 `RdpClientOptions.Camera`，null 表示关闭。固定桌面分辨率不影响摄像头通道。

实现 [MS-RDPECAM v1](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpecam/6c9dd9fc-0a9a-45c4-84b0-72cd3986dd39)，支持一个彩色视频流和设备原生 MJPEG 格式；不支持 H.264、YUY2、音频、多摄像头或 v2 属性。远端服务器与应用须支持摄像头重定向，策略须允许。枚举不会激活采集源；远端 Activate 请求才创建 Media Foundation 源和异步 source reader。每个样本请求返回一个完整帧或协议错误，同时最多等待一个请求。Stop/Deactivate 取消待完成采集，通道关闭或断线释放资源。激活/采集检测到设备移除时通知远端；现有会话空闲期间的热插拔通知尚未实现。

动态通道支持独立重组、同名多实例和双向分片。重组总量限制 32 MiB，摄像头消息限制 16 MiB，会话队列限制 64 条/32 MiB。队列溢出明确断开会话，不会静默丢分片。假后端测试覆盖协议布局、明示选择、生命周期、请求节奏、取消和分片。没有实际打开摄像头采样，真实服务器与摄像头的端到端互通仍未验证。
