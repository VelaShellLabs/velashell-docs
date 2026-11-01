# RDP 远程桌面插件设计与使用

English: [RDP plugin: design and use](../../en/host/rdp-plugin-design-and-use.md)。

`velashell.rdp` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp),是进程内工作台插件,在 Windows、Linux、macOS 的 Avalonia 标签页中显示远程桌面。引擎是**纯托管 C# 的 RDP 客户端**,不依赖 FreeRDP,不捆绑原生二进制,随包只有一份托管程序集,全 RID 通用。

## 架构与范围

协议栈是独立的 C# 类库 `VelaShell.Rdp`(MIT 许可,随仓库发布):它 UI 无关、不依赖插件 SDK,实现本插件所需的 MS-RDPBCGR / MS-RDPELE / MS-RDPECLIP / MS-RDPEDISP / MS-RDPEFS 子集。插件引用该库,库 DLL 随插件签名 `.vpx` 一起分发,不单独出 NuGet 包。分层:

- **Wire** —— TPKT/X.224 分帧、快速路径帧、ASN.1 PER/BER。
- **Security** —— TLS 用 `SslStream`(OS 信任链,或精确的 SHA-256 叶证书 pin);NLA/CredSSP 在 Windows 上用系统原生 SSPI 的 Negotiate 包,其他平台用 `NegotiateAuthentication`(NTLM);TSRequest 公钥绑定按 MS-CSSP v5+。
- **MCS/GCC** —— Connect-Initial/Response、erect/attach/channel-join。
- **PDU** —— 客户端信息、许可、能力协商、连接终结化。
- **Graphics** —— 交错 RLE 与 RDP 6.0 planar(32 位色)解码、MPPC 批量解压、GDI 绘图指令与位图缓存、指针(鼠标形状)解码,以及记录变化区域的 BGRA 帧缓冲;分片的快速路径更新会重组。
- **Input** —— 快速路径键盘/鼠标/Unicode。
- **Channels** —— 剪贴板(cliprdr)与磁盘映射(rdpdr)走分块的静态虚拟通道;动态虚拟通道(drdynvc,MS-RDPEDYC)上只开启显示控制通道(MS-RDPEDISP)以协商动态分辨率,其余动态通道(音频、GFX 图形管线)一律拒绝,服务器保持经典位图路径。

库里的 `RdpManagedSession` 以 SDK 无关的 `ConnectAsync(RdpClientOptions, …)` 实现 `IRdpSession` / `IRdpClipboardTransport` 契约;插件引用该库,提供桌面视图、输入队列、剪贴板桥接与编解码,并负责把宿主的连接请求映射成库的 `RdpClientOptions`、把库抛出的异常翻译成宿主协议异常。宿主管理连接表单、凭据、证书批准、会话标签与可选 SSH 跳板;插件提供工作台和远程桌面。连接、事件循环与收尾在会话工作任务上异步完成,由取消令牌中断。引擎在进程内运行,与宿主共享崩溃边界,不是进程隔离或文件系统沙箱。相关契约见 [SDK 参考](../sdk/sdk-reference.md)、[开发指南](../templates/dev-guide.md) 与 [当前信任边界](../plugins/12-security-threat-model.md)。

范围包括桌面分辨率跟随窗口(动态分辨率,按本机缩放发送 DPI,1:1 锐利绘制)、16/24/32 位颜色深度、远端鼠标形状、键鼠/Unicode、[远程按键悬浮球](#远程按键悬浮球)(Ctrl+Alt+Delete 等本机会截走的组合键与键入剪贴板)、断开后在桌面上就地重连、文本/图像/文件剪贴板、目录传输及仿 mstsc 的磁盘映射(勾选本机磁盘或填写自定义文件夹)。不协商音频播放/采集、多显示器或图形管线(RemoteFX/GFX/H.264);服务器以经典位图更新(交错 RLE,32 位色时为 RDP 6.0 planar)和 GDI 绘图指令呈现,弱网时启用 MPPC 流压缩降低带宽。

**绘图指令与位图缓存。** 客户端接受 DstBlt、PatBlt、OpaqueRect、MultiOpaqueRect、ScrBlt 与 MemBlt(移植自 FreeRDP 的 `Orders.cs`),并声明 600/600/2048 项的第 2 版位图缓存:服务器可以用一次屏幕内拷贝完成滚动、用缓存引用重绘重复的图块,而不必重发像素。xrdp 整个桌面都以缓存 MemBlt 绘制;较新的 Windows 服务器无论如何都只发位图。指令在帧缓冲上按画面与边界裁剪,屏幕内拷贝在重叠时安全,三元光栅运算(ROP3)按真值表求值。默认开启,库调用方可用 `RdpClientOptions.DrawingOrders = false` 关闭;重激活(调整分辨率)时保留缓存、重置指令状态。

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
| 颜色深度（`colorDepth`,「体验」页） | `32` 最高质量(默认,与 mstsc 一致);`24` 真彩色;`16` 增强色(最省带宽,渐变有色带) |

远程桌面默认跟随工作台标签页:连接后、窗口缩放稳定时(350ms 防抖)以及窗口移到另一 DPI 的显示器时,插件通过显示控制通道请求服务器把桌面调整到视口的**设备像素**,同时带上本机显示缩放(150% 屏幕 → 远端界面 150%,deviceScaleFactor 取 100/140/180 中最近的一档),CS_CORE 里也带同样的缩放,远端文字不再只有本机的三分之二大。远程桌面与视图在设备像素上相差无几时(动态分辨率之后的常态),按 **1 个设备像素对 1 个远端像素** 绘制、原点对齐像素网格、不做插值;只有服务器跟不上窗口、尺寸确实不符时才按比例缩放(高质量插值)。显示控制只要求宽度为偶数。新会话直接按上一个视图的尺寸与缩放连接,省掉「先按设置尺寸全屏绘制、立刻调整再绘制一次」;`width`/`height` 是还没有任何视图时、以及服务器不支持动态调整时的回退尺寸。

画面路径:解码与像素转换在接收线程上、帧缓冲锁外完成,帧缓冲记录变化区域的并集,界面只把这部分拷进位图(原先每帧克隆整个桌面、1080p 约 8 MB,再整帧拷贝一次);位图按目标矩形裁剪,不画行尾补齐列。标签切走(视图移出可视树)或窗口最小化时停止呈现,并在服务器支持时发送 Suppress Output 暂停显示更新(不耗带宽、不解码),回来后恢复并发 Refresh Rect 重绘。远端鼠标形状(文本 I 形、调整大小箭头、打字时隐藏等)按服务器的指针更新显示。键入的文字每个快速路径 PDU 最多 7 个字符。会话断开后,桌面上出现“重新连接”浮层按钮(宿主会话标签同样可重连);重连会更新凭据及证书批准,重连失败时保留当前视图,按钮可再次点击。

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

连接对话框的「磁盘映射」页仿照 mstsc「本地资源 → 驱动器」,每块本机磁盘一个复选框。Windows 列出全部盘符,按类型命名,如「本地磁盘 (C:)」「可移动磁盘 (E:)」「网络驱动器 (Z:)」「光盘驱动器 (F:)」;Linux / macOS 列出根目录(`/`)、主目录(`~`)以及 `/media`、`/run/media`、`/mnt`、`/Volumes` 下的挂载点。全部**默认不勾选**。

枚举磁盘只读盘符与类型(或挂载点),不读取介质、不查询卷,因此断开的网络驱动器或空光驱不会拖慢插件激活。插件每 5 秒检查一次磁盘集合,只有变化时才重新注册连接表单(同一个实现实例:已打开的远程桌面标签不受影响,正在编辑的对话框也不会丢失已填内容),稍后插上的 U 盘在下次打开连接对话框时出现。连接时已勾选、但当前不存在的磁盘(U 盘已拔出、网络驱动器离线)直接跳过,不报错。

同一页的「自定义文件夹」用于只共享某个目录:写 `名称=绝对路径`,多条用**分号**分隔(宿主把文本设置渲染为单行输入框,无法换行)。Windows 示例:`work=C:\Work\Share;docs=D:\Docs`;Linux:`work=/home/alice/share`;macOS:`work=/Users/alice/share`。早先保存的按行分隔的值仍可识别。输入框下方常驻的提示会按当前系统给出写法与示例(占位符一输入就消失,所以示例放在提示里);填错时连接报错会引出写错的那一条并给出正确写法,名称重复时点名重复的名称,超过 32 个时说明上限。

名称必须唯一,也不能与已勾选磁盘的名称(`C`、`E`、`root`、`home` 等)重复;只允许 **1–8 位 ASCII 字母、数字或下划线**,不允许连字符;磁盘与文件夹合计最多 **32 个映射**。目录必须存在。远程 Windows 桌面通过 `\\tsclient\<名称>` 访问,xrdp 会话在 `~/thinclient_drives/<名称>`。每个映射授予远程对该磁盘或文件夹的**读写访问**;除了用户勾选的磁盘和填写的文件夹,不共享任何主目录、系统盘或可移动盘。路径限制在映射目录内,拒绝路径穿越与符号链接。映射磁盘或目录与经剪贴板发送文件是两个独立选择。

启用磁盘映射时,同时注册 `rdpdr` 与配套的 `rdpsnd` 静态通道;部分 Windows 服务器在缺少后者时不会启动磁盘握手。这项兼容性注册不会开启音频播放。磁盘通道在等待许可与 Demand Active 时也接收并重组数据,避免丢弃提前到达的握手;服务器重新发送 Server Announce 时开启新一轮设备公告。插件日志 `<数据根>/logs/plugins/velashell.rdp.log` 中的 `RDPDR` 行记录映射数量、实际加载的引擎路径、协商通道号、登录通知、握手包类型、设备注册结果及失败的 I/O 状态码,不记录文件内容或凭据。库调用方可设置 `RdpClientOptions.DriveRedirectionLog` 接收这些后台线程诊断。只有通道号或登录通知并不表示设备已注册;应继续检查 Server Announce (`0x496E`)、设备公告与 Device Reply 的状态码。网络发现不是访问 `\\tsclient\E` 的前提。

在插件仓库调试时使用 [CLI 开发内环](../cli/cli.md):将 `VelaShell.Plugin.Rdp` 设为启动项目,选择 `VelaShell` 配置后按 F5。`--dev-root` 扫描构建目录的父目录(例如 `bin/Debug`),开发插件从影子副本加载。宿主目录或用户插件目录中的同 id 插件优先于开发根;联调前应移走冲突副本,并取消继续向宿主复制该插件的本机构建配置。

## 认证与证书策略

安全层由 `security` 决定:**自动**(默认)同时提供 TLS 与 NLA,由服务器选择(支持 NLA 的服务器会选 NLA);**仅 NLA** 要求服务器选 NLA,绝不回退;**仅 TLS** 只提供 TLS,凭据在已校验的 TLS 通道内随 Client Info 发送,由服务器自己完成登录(相当于服务器关闭了「要求 NLA」)。TLS 最低 1.2;禁用旧式 RDP 安全及“忽略所有证书”,各模式的证书校验完全相同。没有 pin 时,核验系统信任链、服务器用途、有效期和主机名/IP,由 .NET 的 `SslStream` 用各平台的 OS 信任存储完成。

**NLA 的认证实现。** NLA/CredSSP 的认证令牌与封包在 **Windows 上走系统原生 SSPI 的 Negotiate 包**,与 mstsc 一致:主机名为 DNS 名时以 `TERMSRV/<主机>` 为服务目标,域服务器可走 Kerberos;IP 地址保持空 target(Kerberos 对它无从谈起,显式凭据下 Negotiate 实际都落到 NTLM)。其他平台退回托管 NTLM(`NegotiateAuthentication`),对接受裸 NTLM 的服务器仍可用。公钥绑定按 MS-CSSP 的 v5+ SHA-256 nonce 绑定实现;**服务器的 CredSSP 版本低于 5(CVE-2018-0886 修复之前)一律拒绝**,也不会因此回退到 TLS——mstsc 默认的「已缓解」策略同样拒绝。TLS 只用 1.2/1.3 并拒绝重协商;Client Info 与 TSCredentials 里的明文密码发送后即清零,选项记录的 `ToString` 不输出密码。

**降级保护。** X.224 协商是明文的,中间人可以回答「只支持 TLS」——若用户在首次连接时信任了它的证书,密码就会随 Client Info 落到它手里。因此自动模式下,某台主机(主机+端口)**一旦用 NLA 登录成功就被记住**(插件存储),之后对它要求 NLA;它若改为只提供 TLS 或拒绝 NLA,连接会在发送任何密码之前停止,并提示这可能是有人在中间强迫降级。若确实是服务器关闭了「要求 NLA」,在连接的高级设置里把安全层改为 TLS 即可。从未用 NLA 登录过的主机(例如不接受我们 NTLM 的那类)仍按下面的回退处理。

**自动模式的回退。** 若服务器在客户端发出第一条 CredSSP 消息之后就拒绝——这条消息只是 NTLM NEGOTIATE,不含用户名,也不含任何凭据证明——表现为以 `errorCode` 0x80090302(SEC_E_UNSUPPORTED_FUNCTION)作答,或直接中止 TLS 会话(本地 SChannel 报 0x80090304「无法连接到本地安全机构」;对同一台服务器、字节完全相同的消息,两种形态会交替出现),说明服务器不接受传入的 NTLM。这是服务器策略,不是凭据问题。自动模式随即**只提供 TLS 重连一次**,证书照样校验,插件日志记一条警告。若服务器连 TLS 也拒绝(RDP_NEG_FAILURE `HYBRID_REQUIRED_BY_SERVER`),即服务器要求 NLA 却不接受 NTLM,错误会说明这是服务器端设置、与用户名和密码无关,并给出两种修法:在服务器上允许传入 NTLM(本地安全策略 › 安全选项 ›「网络安全: 限制 NTLM: 传入 NTLM 流量」设为「允许所有」),或关闭「仅允许运行使用网络级别身份验证的远程桌面的计算机连接」。密码错误(服务器在 NTLM 认证之后拒绝)仍报认证失败,**不会**触发回退。

**连接失败的诊断。** 错误指明失败的阶段(TCP、安全协商、TLS、NLA、MCS、登录、许可、能力交换)和原因链,含 SChannel / 套接字错误码;TLS 握手之后的失败还附上协商出的 TLS 版本。服务器在 NLA 中拒绝登录时,按它返回的状态码给出具体原因(用户名或密码错误、密码过期、账号锁定或停用、不允许远程登录)。**连接成功后又被服务器断开**时(服务器发来 Set Error Info PDU,MS-RDPBCGR 2.2.5.1.1),按它的 errorInfo 码给出原因并显示在断开浮层与状态栏:用户被注销、会话被另一连接接管、服务器拒绝或权限不足、空闲/登录超时、没有可用的远程桌面授权(RDS 授权或宽限期过期)等;其余码回退到中性描述加十六进制码。断开日志还记录**此前是否收到过登录通知**(服务器的 Save Session Info PDU):收到说明凭据已被接受、Windows 会话已启动,随后的断开是服务器结束了一个已建立的会话(多为策略),而非认证或协议失败;未收到则说明尚未走到登录。插件日志(`velashell.rdp`)记录每次失败的完整异常;TLS 握手之后的失败再附上加密套件和服务器发来的 TLS 记录头(只记类型与长度,例如 `Alert 26 B`,从不保存载荷),用来区分「服务器中止」与本地解密问题。

宿主批准的 pin 是叶证书 DER 字节的精确 SHA-256(64 位十六进制)。**pin 替代 CA/名称/有效期验证**,只做叶证书身份的定时比较,不是在上述检查之上附加一道条件。叶证书变化会拒绝连接,需要重新明确批准。SSH 转发不会把预期服务器名称改成回环地址。密码通过 API 传递,不进入命令行参数。NLA/CredSSP 用 .NET 的 `NegotiateAuthentication` 做 NTLM 与消息密封,TSRequest 公钥绑定按 MS-CSSP 实现。

## 对恶意服务器的边界

引擎与宿主同进程,所以服务器发来的一切都有上限,畸形输入只结束会话(或只丢掉那一项),不会冻结界面、耗尽内存或越出映射文件夹(2026-11-01 审计后加固):

- **画面**:位图矩形先按目标矩形与画面裁剪再解码,单边 ≤8192,每个更新的像素总量 ≤4 个桌面面积;解码在锁外完成;短的 RLE 流剩余部分涂黑,不会把别处内存残留画到屏幕上。服务器可强加的桌面面积 ≤8K UHD。绘图指令每个更新的绘制量 ≤16 个桌面面积,位图缓存总量 ≤256 MiB,单个缓存位图边长 ≤4096;未声明的指令类型结束会话(指令流无法跳过未知指令)。
- **通道与消息**:静态虚拟通道重组以第一块声明的总长为准(≤32 MB),实际缓冲不得超过它;rdpdr 队列另有 64 MB 字节上限;动态通道只为已打开的显示控制通道缓冲,消息 ≤64 KB;CredSSP 消息 ≤1 MB。
- **磁盘映射**:单次读取 ≤1 MiB,同时打开的文件 ≤4096,写入起点最多超过文件末尾 256 MB,每个请求都有应答;路径逐段校验——拒绝 `.`/`..`、控制字符,Windows 上再拒绝备用数据流(`:`)、设备名(CON、NUL、COM¹ 等)与结尾的点/空格;映射根之下的符号链接/联接在打开、创建与重命名目标时一律拒绝(映射根本身可以是链接;OneDrive 占位文件不算链接);非 Windows 上前缀比较区分大小写。
- **剪贴板文件名**:拒绝 Unicode 格式字符(如 U+202E 可把 `.exe` 伪装成 `.txt`)、`CONIN$`/`CONOUT$`、`COM¹–³`/`LPT¹–³`;文件/目录冲突检查为线性复杂度。
- X.224 明文 cookie 只放可打印 ASCII 且限长。

## 构建与独立交付

保持仓库的 .NET 11、SDK 与 Avalonia 设置,不改依赖。**只有托管构建,没有原生步骤**:

```bash
dotnet build plugins/VelaShell.Plugin.Rdp -c Release
dotnet build plugins/VelaShell.Plugin.Rdp -c Release -t:PackVpx
```

Windows 虚拟文件剪贴板通过 `WindowsVirtualFileClipboard` 调用系统 OLE/user32/kernel32。另一平台相关处是 `UnixClipboardFile`:它在 Unix 上 P/Invoke 系统 `libc` 的 `open`/`fstat`,以 `O_NONBLOCK|O_NOFOLLOW` 安全打开用于本地→远程传输的剪贴板文件(避免 FIFO 阻塞、拒绝符号链接),Windows 用 .NET 文件 API。系统 libc 随操作系统提供,不是捆绑的原生库。

不再有 `rdp-native` 工作流、CMake、随包 OpenSSL/VC 运行库或六平台原生任务:插件以**一份托管 `.vpx`** 交付,全 RID 通用。CI 与发布只跑托管测试与打包。工具链细节见 [插件构建 README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md)。另见 [打包与签名](../templates/publishing.md)。

## 验证情况与剩余边界

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

加固与画质相关行为另有离线测试:`RdpHardeningTests`(含真实目录联接,以及发送超大 TSRequest、CredSSP v3 的假服务器)、`RdpRenderingTests`(1:1 绘制、DPI、Suppress Output、颜色深度设置)、`PlanarTests`(FreeRDP 测试向量与模糊测试)、`RdpPointerAndInputTests`(指针掩码规则、缓存、批量键入)。

NLA/CredSSP 除离线 ASN.1 单测外,对 FreeRDP shadow 服务器(`/sec:nla`,门控于 `VELA_RDP_TEST_NLA_TARGET`)做了活连接验证:**原生 SSPI(SPNEGO)路径与托管 NTLM 兜底路径都能登录到桌面**(Windows 上两者都测);错误密码报认证失败且不回退;「仅 TLS」被要求 NLA 的服务器以 `HYBRID_REQUIRED_BY_SERVER` 拒绝。安全层回退与「首条 NLA 回复即被拒」(错误码、TLS 中止两种形态)由进程内假服务器测试覆盖;TLS 1.2 下 Windows 服务器可能选用的各加密套件(ECDHE/RSA × AES-GCM/AES-CBC)已逐一对 xrdp 实测。真实 Windows 服务器方面,核实过一台拒绝传入 NTLM 的服务器的行为(首条消息即回 0x80090302 或中止 TLS),**真实 Windows 服务器的 NLA 登录成功路径仍未在活连接上验收**。交错 RLE / MPPC 解码、MCS/GCC、能力协商与各通道线格式按 MS-RDPBCGR 及相关公开规范实现。这些验证不构成全平台或真实 Windows 端的最终认证。
