# VelaShell.XServer 架构与原理

English: [`../../../en/xserver/design/architecture.md`](../../../en/xserver/design/architecture.md)

> 状态:**M1(核心协议)、M2(现代工具包)、「功能完备」一轮(XKB、XInput2 等十余个扩展、窗口管理器角色、全库性能复查)
> M3(接入宿主)与 M4(同步抓取、设备拓扑、XKB 改表、MIT-SHM、GLX)已完成**,2026-09-23 立项;2026-10 做完第二轮全库审查的修复(§10)。宿主的「X Server」按钮默认启动的就是本库(每个 X 窗口一个 Avalonia 原生窗口),
> SSH 的 X11 转发直接接进它;用户自己装的 VcXsrv 退成 Windows 上的可选引擎(见
> [`../../host/交互与界面规格.md`](../../host/交互与界面规格.md) §4A.2 与 §14)。

## 1. 为什么要做

宿主要在本机显示远端经 SSH X11 转发过来的图形程序。调研(2026-09-23)的结论是 **.NET 生态里没有可用的
X 服务端库**:X11.Net 是 Xlib 绑定(客户端);yserver(Rust,MIT)只跑在 Linux DRM/KMS 上、不能嵌入;
node-x11 的 `lib/xserver`(JS,MIT)画在浏览器 canvas 上;WeirdX(Java)是 GPL。捆绑 VcXsrv 只解决
Windows、还会让安装包大几十 MB,与「解压即跑」的分发模型冲突。

于是有了这个库:一个**可嵌入、无原生依赖、跨平台**的 X11 服务端。宿主用 Avalonia 把它的顶层窗口
画成原生窗口(rootless),于是 Windows / macOS / Linux 一套代码。

## 2. 目标与非目标

**目标**

- 实现 X Window System Protocol 第 11 版的**核心协议**(全部 119 个核心请求),以及现代工具包
  实际依赖的扩展(见 §8 里程碑)。
- **rootless 多窗口**:每个顶层 X 窗口对应一个宿主原生窗口;根窗口不画。窗口装饰、移动、缩放、
  关闭由宿主负责;窗口管理器在协议层该做的(EWMH / ICCCM 属性、解析客户端提示、接住客户端的 WM 请求)由服务端做,
  再把请求转交宿主(§6)。
- 可选的**单窗口(rootful)模式**(`X11ServerOptions.Rootful`,默认关,见 §7「单窗口模式」):整个根窗口作为一个顶层交给宿主,
  远端的窗口管理器照常管理顶层 —— 跑完整的远端桌面(xfce、MATE)或图形安装器时用。
- **可嵌入**:库不依赖任何 UI 框架、不依赖原生库。像素、窗口生命周期与输入经一个宿主接口(§6)交换。
- **可测**:协议层全部走内存传输单测;另有 `[TestCategory("Interop")]` 用例让真实的 Xlib / XCB 客户端
  (Docker 里的 `xdpyinfo`、`xterm`、`xeyes`…)连进来。

**非目标**

- 不做**真实显示设备**(DRM/KMS、帧缓冲设备)与硬件输入 —— 那是完整 X 服务端的事,我们只做嵌入式的那一半。
- 不做 XDMCP、多屏幕(Screen > 1)、索引色 / 可写颜色表(只提供 24 位 TrueColor)。
- 不做 DRI2 / DRI3(没有 GPU 可交给客户端)。GLX 只登记直接渲染(客户端自己软件渲染、再经 PutImage 送像素)并提供
  固定功能 GL 子集的软件间接渲染;MIT-SHM 只在 Linux 上、只给经 Unix 套接字连进来、与服务端在同一个 IPC 命名空间里的本机客户端。
  Present 只做软件拷贝(没有显存可翻页)。
- SECURITY 扩展只做 MIT-MAGIC-COOKIE-1 授权的签发、撤销与非受信级别(§7「信任级别」);不做 XC-QUERY-SECURITY-1 认证方法与 Application Group
  (只为早已没人用的 X 防火墙代理而设),GenerateAuthorization 的 group 只能是 None。

## 3. 净室规程

与 `VelaShell.Ssh` 同一套纪律(见宿主仓库 `src/VelaShell.XServer/AGENTS.md`):

1. **实现依据只能是公开规范**:X.Org 发布的 *X Window System Protocol, X Version 11*(含附录 B 编码)、
   各扩展规范(*BIG-REQUESTS*、*XC-MISC*、*X Nonrectangular Window Shape Extension*、*X Fixes Extension*、
   *The X Resize and Rotate Extension*、*The X Rendering Extension*(及其引用的 PDF Reference 混合模式公式)、
   *The X Keyboard Extension: Protocol Specification*、*The X Input Extension*(1.5 与 2.2)、*XTEST Extension*、
   *X Synchronization Extension*、*X Damage Extension*、*Composite Extension*、*Double Buffer Extension*、
   *The Present Extension*、*MIT-SCREEN-SAVER*、*DPMS*、*X-Resource*、*Generic Event Extension*、*The MIT Shared Memory Extension* 1.1,
   以及 XINERAMA —— 它没有独立的规范文档,线格式依据 X.Org 发布的 panoramiXproto 协议定义)、ICCCM、EWMH、freedesktop.org 的
   XSETTINGS 规范、freedesktop 的 XDND 协议第 5 版(拖放的两个方向:宿主当源拖进 X 窗口;第 4 版起「拖到根窗口」的 `XdndProxy` 约定接 X 程序往本机拖出来),
   X Consortium 的 *The Input Method Protocol* 1.0 与 *The XIM Transport Specification* 0.1(XIM 输入法服务端,`Server/XimServer.cs`;
   两份文档的正文与表格在「长包写在谁的窗口上」「通知属性的 ClientMessage 用什么格式」上互相矛盾,以 Input Method Protocol 附录 D 的表为准;
   XIMStyle 的位与 XN* 属性名取自 *Xlib - C Language X Interface* 第 13 章),以及 X Consortium 的 *Compound Text Encoding*(COMPOUND_TEXT 的编解码,`Protocol/XText.cs`);
   GLX 部分依据 Khronos 发布的 *OpenGL Graphics with the X Window System* 1.4、*GLX Extensions for OpenGL
   Protocol Specification* 1.3(编码)、*The OpenGL Graphics System* 1.5(间接渲染的 GL 语义)与 Khronos 注册表里的
   GLX_ARB_create_context / GLX_ARB_create_context_profile 扩展规范,操作码与枚举值取自 Khronos 注册表的 `gl.xml` / `glx.xml`。
   **每个协议实现文件的头部写明它实现的是哪份规范的哪一节。**
2. **写实现时不打开任何其它 X 服务端的源码**(X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv),也不打开任何
   OpenGL / GLX 实现(Mesa 等)的源码。
3. 规范里的常量(操作码、事件码、错误码、预定义原子、掩码位)按规范取值 —— 那是协议,不受版权保护,
   也不许为了「看起来不一样」去改。
4. **数据不等于代码**:内置位图字体是 X.Org 的 BDF 原样 —— `font-misc-misc`("Public domain font. Share and enjoy.")、
   `font-cursor-misc`("These glyphs are unencumbered")、`font-adobe-75dpi` / `100dpi`(Adobe / DEC 的宽松许可)—— 与 GNU Unifont
   (双许可,取 SIL OFL 1.1);颜色名表是 X.Org `rgb` 的 `rgb.txt` 原样(MIT / X11 许可)。都以数据文件随库分发,在 `NOTICE.md` 里注明来源与许可,
   字体的许可原文随数据嵌进程序集。字体数据只由 `scripts/xserver/fonts/build-fonts.cs` 从固定的上游提交生成(BDF 逐字节不动、Brotli 压缩),不手改;下载的内容按脚本里固定的 SHA-256 核对(X.Org 的仓库核对从归档里挑出来的文件 —— GitLab 现做的归档不保证逐字节稳定),全部对上才动数据目录。

## 4. 分层

```
Host/         全部公开类型,一律在根命名空间 VelaShell.XServer:IX11ServerHost、X11ServerOptions、XTopLevelWindow 与 XTopLevelSnapshot /
              XTopLevelChanges / XFrameExtents / XGravity / XWindowFunctions、XPixelReader / XPixelReadResult、XClientInfo、
              XCursor / XCursorShape / XCursorImage、XKeymap、XMonitor、窗口管理器请求与枚举、XKeycodes、XRect。其余一律 internal
Protocol/     常量(操作码、事件码、错误码、掩码、预定义原子)、字节序感知的请求读取与回复 / 事件 / 错误写出、
              每项工作的工作量预算(WorkBudget,§5)、STRING / UTF8_STRING / COMPOUND_TEXT 的编解码(XText)
Server/       X11Server:公开成员全在 X11Server.cs(构造、生命周期、宿主注入);执行循环(WorkLoop)、监听与连接(Connection、
              UnixSocket;TCP 6000+N、Unix 套接字与 ServeAsync / ServeAuthenticatedAsync 任意双工流,连接建立(有时限)与授权(见 §7)、背压、
              BIG-REQUESTS、断开时的清理)、资源表、XC-MISC 与内存账(Resources)、请求分派(Dispatch)、宿主回调的延后调用与按批合并(DeferredHost)、
              扩展注册表(Extensions:编号表、事件 / 错误编号不重叠的检查、清理钩子 —— 连接断开、客户端的资源销毁、窗口销毁、资源释放、
              像素图释放 —— 与 Generic Event);
              请求处理按领域拆成 partial 文件(Windows / Exposure / Events / Properties / Graphics / Text / Colors / Cursors / Input /
              KeyboardControl / GrabFreeze,以及各扩展:Shape / XFixes / RandR / Monitors / Xinerama / Render / Damage / Composite / Dbe / Sync /
              Present / Xkb / XkbSetMap / XInput / XiHierarchy / XTest / ScreenSaver / Dpms / XRes / Shm),
              与宿主之间的顶层窗口桥(TopLevels:句柄、快照与变化、损伤交付、宿主作为窗口管理器的动作)、窗口管理器角色 Ewmh、
              剪贴板互通 Clipboard、XSETTINGS 管理器 XSettings;自成一体的 GLX 是独立的类 GlxExtension
Windowing/    窗口模型(树、几何、属性、事件选择(XI2 的按设备号分开存)、被动抓取、顶层缓冲、SHAPE 的三种形状)
Resources/    资源类型:GC、像素图、颜色表、光标、字体句柄,以及各扩展的资源(RENDER 的 picture 与字形集、SYNC、DAMAGE、
              XFIXES 区域、Present 事件上下文、MIT-SHM 段、GLX 上下文与可绘对象);颜色名表(X.Org 的 rgb.txt,嵌入资源)
Drawing/      32 位软件帧缓冲、按 y 分带的区域(Region)、光栅化:16 种光栅操作、平面掩码、填充样式、裁剪;
              点 / 线(细线 Bresenham + 宽线按 line-style / join-style / cap-style 拼成的多边形)/ 矩形 / 多边形扫描线填充 / 弧 / 图像块 / 文字;
              RENDER:像素格式、合成运算与混合模式、取样源(图像 / 纯色 / 渐变,repeat、变换、过滤)、梯形覆盖率
Gl/           GLX 间接渲染的软件 GL:渲染命令解码、显示列表、矩阵栈、光照、裁剪、三角形(扫描线)/ 线 / 点光栅化、纹理、逐片元操作、查询
Fonts/        BDF 解析、内置的 X.Org 位图字体与 GNU Unifont、XLFD 名称匹配(fonts.dir / fonts.alias、单字节字符集的派生、最接近字号的退路)
Input/        键码 ↔ 键值表(evdev 风格键码)、键值的大小写(KeysymCase)、XKB 的 evdev 键名、修饰键映射、
              抓取的数据结构(核心与 XI2;被动抓取按 detail 分桶的 PassiveGrabTable)
```

Unix 套接字:Windows 以外默认监听 `/tmp/.X11-unix/X{N}`,Linux 另在抽象命名空间里监听同名套接字(Xlib / XCB 对 `:N` 先试它);
`X11ServerOptions.UnixSocketPath` 可指定(放不进套接字地址时构造就抛)或关掉。TCP 由 `ListenTcp` 决定:默认(null)只在配了
`AuthorizationCookie` 时才听,`true` 总是听(不配 cookie 时启动记一行提醒),`false` 不听。监听着与显示号挂钩的传输时,类 Unix 上按
Xserver(1) 的约定持有 `/tmp/.X{N}-lock`(见 §7)。宿主字体提供者接口还没有:核心字体只服务老程序,现代工具包都走 RENDER + 客户端栅格化,
随库带的 X.Org 位图字体与 GNU Unifont(§7「字体」)已经覆盖老程序常用的字族与中日韩。

## 5. 线程模型

X 协议的语义是**全局串行**的:服务端按到达顺序逐条执行所有客户端的请求,一个请求的效果对之后的
所有请求可见。所以:

- **一个执行循环**(单线程,`Channel<工作项>`)执行全部请求、宿主输入与定时器。协议状态不加锁 ——
  所有可变状态只在这个线程上被碰。唯一跨线程的是顶层像素,由像素锁保护(不对外公开):执行循环每次持锁按 **4 毫秒**
  的时间预算跑一批,然后放锁让宿主拷像素。宿主经 `XTopLevelWindow.ReadPixels` / `CopyPixels` / `TryReadPixels` 读像素时先登记「在等」:
  执行循环每执行完一项就看一眼,有人在等就提前放锁,并且等它读完(最多 20 毫秒)再拿 —— `lock` 不公平,执行循环放锁后几微秒内就会再拿,
  等锁的 UI 线程可能一直抢不到,整个宿主界面跟着卡。
- **每项工作一份工作量预算**:4 毫秒的批预算只在两项工作之间起作用,一条请求从头到尾持着像素锁 —— 代价与请求字节不成比例的请求
  (大坐标范围、重叠的行、大面积合成、反复调用的显示列表)就能让宿主界面陪着冻住。所以每个工作项开始时给一份预算(2²⁸ 个单位,
  一个单位约等于处理一个像素,约 2.7 亿个像素、4K 屏整屏填满 32 遍),热循环按处理的工作量扣:光栅化的行 / 像素 / 位图块、多边形的边与行、
  RENDER 合成的像素与梯形覆盖率、遮罩分配,GLX 渲染命令的顶点(16)、片元(8)、三角形扫描的行与段、Clear 的面积、DrawPixels / Bitmap /
  CopyPixels / TexImage 解码的像素。扣光时请求回 Alloc;GLX 的渲染命令没有回复,按 GL 的约定记 OUT_OF_MEMORY,这个请求余下的渲染命令不再执行。
  持锁超过 250 毫秒的工作项记一行日志,点名客户端与操作码。
- **宿主可以限时读像素**:`XTopLevelWindow.TryReadPixels(reader, timeout)` 限时之内拿不到锁就返回 `Busy`、不调回调。Avalonia 宿主每帧最多等
  8 毫秒,等不到就跳过这一帧、损伤留到下一帧再取 —— 一个慢客户端拖不住宿主的界面。实测四个满载的窗口每帧逐个读,中位 0.44 毫秒、一次也没有读不到,
  所以不另设「一次锁内读多个窗口」的接口(基准 `scripts/xserver/bench/bench.cs` 有这个场景)。
- **宿主回调不在持锁时调,并且按批合并**:执行循环里产生的通知(映射、几何、损伤、光标、WM 请求……)先攒进 `DeferredHost`,
  放锁之后按原顺序调用 —— 宿主在回调里同步等 UI 线程、UI 线程又在 `ReadPixels` 里等锁,这种死锁因此不会出现;
  宿主回调抛异常记一行后吞掉,宿主的日志委托本身再抛也一样吞掉,执行循环每一批另套一层兜底,不会因此退出。
  同一批里:同一个窗口的 `TopLevelChanged` 按位或并成一条(放在第一次的位置,不跨过夹在中间的映射 / 取消映射合并);映射了又取消映射的一对
  互相抵消;`CursorChanged` 与 `ClipboardChanged` 只交最后一次;`WindowManagerRequested` 最多 32 个,多的丢掉并记一行;`BellRequested`
  最多一次(取最大音量),两次之间至少隔 100 毫秒 —— 一个循环改标题、反复映射、狂发响铃的客户端灌不满宿主的 UI 线程。`ServerGrabStalled` 照常排队。
- 每个连接一个**读取任务**(64 KB 缓冲):按长度字段切出完整请求,交给执行循环;一个**写出任务**:把执行循环
  放进该连接输出队列的消息拼进一块池化缓冲再写。执行循环从不在套接字上阻塞,慢客户端拖不住别人。写不出去(对端已断)时立即断开这个客户端,
  不等读端察觉 —— 读端可能正停在背压上,根本没去读套接字。
- **背压**:每个客户端已读进来、未执行的请求最多 1024 条、合计最多 32 MB(读取任务在 `SemaphoreSlim` 与字节预算上异步等,
  等到了才分配、才读 —— 只数条数的话,1024 条 16 MB 的大请求就是 16 GB);输出队列里**已经排着**的字节达到 64 MB(客户端不读了)时
  再来消息就断开它。单条消息本身可以比上限大(三块 4K 横排时 `xwd -root` 的回复约 100 MB),超出的部分最多一条消息;会产生超大回复的请求
  另有自己的上限(GetImage 先估大小,超过 256 MB 回 BadAlloc)。断开是真断开 —— `XClient.Abort` 取消连接上挂着的读与写。
  请求缓冲从 `ArrayPool` 租、执行完还回去(整窗 PutImage 一条就是几 MB,每条新分配就是每条进大对象堆);请求读取器自带长度,
  只读前这么多字节(越界检查按「剩余长度」比,客户端给的长度接近 2³¹ 也不回绕),请求内容要留下的一律拷出去。
  回复与事件在按线程复用的写入器里拼,只分配交给写出端的那一份。
- **接受循环挺过瞬时错误**:accept 失败(fd 用完、accept 之前对端就复位)记一行接着接,fd 用完这类先等 100 毫秒再试;只在取消、
  监听已经关掉时退出。原先遇到任何错误就永久退出,之后本机 X 程序再也连不进来、一行日志都没有。
- **连接建立有时限、有并发上限**:连接建立报文(12 字节的头与授权名 / 数据)要在 10 秒内收齐、失败回复要在这之内写完,否则断开;
  同时处在建立阶段的连接最多 32 条,超了的当场关掉(登记成客户端之后不再占名额);授权名与授权数据各至多 256 字节,读到头就判,
  超了回连接失败。客户端上限(255)只数已经建立的 —— 不设这几道,本机任何用户不带 cookie 开几万条只发个头的连接,就能耗尽 fd 与内存。
  等执行线程登记客户端那一步不计时;等的时候调用方取消了,登记成的客户端随即断开,不留一个没有连接的客户端占着编号。
- **需要等的都异步等**:XTEST 与 Present 的延迟、SYNC 的计时器用 `Task.Delay`,到点后 `Post` 回执行循环;
  SYNC 的 Await 把该客户端后续请求暂存,条件成立时按原顺序放回。XTEST 的 FakeInput 带延迟时同样暂存该客户端之后的请求
  (协议:延迟过去之前不处理这个客户端的其它请求),执行线程不阻塞;Present 排着的 PresentPixmap 与 NotifyMSC 每客户端合计最多 256 条。
  客户端断开时它挂着的计时一并取消。暂存的请求(GrabServer 结束、Await 等到了、XTEST 的延迟到了)放回**就绪队列**:执行循环先取它、
  再取通道,每条照常按批预算执行、批与批之间放锁 —— 原先在一个工作项里一口气执行完,最坏 255 个客户端 × 1024 条全程持锁。
  同一个客户端还在通道里的请求都比暂存的来得晚,顺序不变。
- **诊断日志不在持锁时写**:协议错误等诊断先在执行循环里攒着,放锁之后才交给 `X11ServerOptions.Log`(宿主的日志往往同步写文件,
  持锁写就是让 UI 线程陪着等磁盘)。客户端能成批触发的日志有两道限额:每秒最多 50 条,另有字节限额(全部客户端合计,先给 32 KB 的余量,
  之后每秒补 256 字节),没记的条数在下一次能记时补一行。这类日志整行把 C0 / DEL / C1 控制字符与双向文字控制符写成 `\xNN` / `\uNNNN`、
  截到 512 个字符(OpenFont 的名字另截到 200 个)—— 客户端给的字符串里的换行能伪造日志行,ESC 能往看日志的终端里注入控制序列。
  请求、工作项、断开收尾的意外失败,每种(位置 + 异常类型)第一次记完整的调用栈,之后只记一行。每条请求的操作码记成整数存进环形数组,
  只在出错要打印时才拼成文字。
- **GrabServer** 期间,执行循环只执行持有者的请求,其余客户端的请求原样暂存,Ungrab 后按原顺序放回(经就绪队列,见上)。
  抓着 10 秒还有别人的请求在等,就记一行点名持有者(带连接名,比如 `user@host:22`)与等着的请求数,之后每 60 秒再记一次,直到放开;
  同时经 `ServerGrabStalled`(`XServerGrabStall`:持有者的编号与连接名、抓了多久、等着的请求数)告诉宿主,宿主据此提示用户;没人在等时两样都不做。
  持有者挂住时宿主可以 `BreakGrabs` 或按编号 `DisconnectClient`(§6)。持有者自己再抓一次是空操作。
- **同步抓取冻结期间**的设备事件排进同一个队列,指针与键盘按到达的先后(宿主注入与 XTEST 一样排);放行后一个工作项
  最多回放 64 个,余下的排到下一个工作项 —— 中间客户端的请求(下一个 AllowEvents)能插进来。队列最多 4096 条,满了按三类取舍:
  可丢的(指针移动、离开、按键的自动重复)先丢最早的一个;没有可丢的时丢新来的按下,并记下它 —— 它的松开与自动重复来时一并丢掉,
  X 这边从没按下过;松开一律留着,可以超出上限(能排进来的松开不会多于按着的键与按钮)。原先满了连松开也丢,解冻后键或按钮一直按着。
  WarpPointer 冻结时同样排队,到时才按当时的位置与窗口算。
- 损伤区域在一批工作项执行完后合并,一次性通知宿主(不是每个绘图请求一次);每个顶层一批最多累计 8 块矩形,
  超出时合成外接矩形 —— 精确并集在一批几百条请求时退化成 O(n²)。
- **按客户端轮流取工作(2026-10-10)**:从通道取出的工作按客户端分队、队内保持到达顺序;宿主与计时器的工作另一条队,有活就先取
  (注入的输入、换进来的配置、到点的计时器:量小、要及时,量受真实输入速率限制,饿不死客户端);客户端之间轮流、每次取一项。
  放回来的暂存请求(GrabServer 结束、SYNC 的 Await 等到了、XTEST 的延迟到了)照旧最先取。协议只要求同一个客户端的请求按序执行,
  不同客户端之间的先后本来就不保证。连接收尾排在那个客户端还没执行的请求之后(发完请求就关连接的客户端,请求照样生效)。
  原先全部工作排一条队:间接 GL 的 glxgears 排满一千多条渲染请求时,之后连进来的 xdpyinfo 要等 25 秒,现在 1 秒内。
- **看门狗(2026-10-10)**:执行线程记下正在做的那一项(开始时间、客户端、操作码、序号),计时器线程每秒看一眼,超过 `WatchdogThreshold`
  (默认 5 秒)就记一行 `watchdog: client#3 (user@host:22) opcode 53 has been running for 5 s; all clients are waiting`、计入 `work.stalled`,
  同一项只记一次、不等它做完(原先「慢工作项」那一行要等它做完才记,真卡死时什么也没有)。

## 6. 宿主接口(rootless)

库定义接口,宿主实现;方向是**库 → 宿主**的通知,和**宿主 → 库**的注入:

| 方向 | 内容 |
| --- | --- |
| 库 → 宿主(`IX11ServerHost`,回调名一律「主语 + 过去分词」;全部在执行线程上、放锁之后调,同一批里按 §5 合并) | 顶层窗口映射 / 取消映射(`TopLevelMapped` / `TopLevelUnmapped`,销毁、被 reparent 走也算取消映射;**收工时不逐个发**,宿主停服时自己收掉原生窗口);快照变了(`TopLevelChanged`,附 `XTopLevelChanges` 说明变了哪几组:`Geometry` —— 位置、尺寸、`BorderWidth`、`NeedsPlacement`;`Title` —— 标题、`ClassName`、`InstanceName`;`States`;`Icons`;`Shape` —— 边界形状与输入形状;`Hints` —— 其余)。窗口的属性在 `XTopLevelWindow.Snapshot` 这份不可变快照里(见下文「快照」);客户端的窗口管理器请求(`WindowManagerRequested`,接口的默认实现方法,见下文「窗口管理器请求」);损伤矩形(`TopLevelDamaged`,随后宿主经 `XTopLevelWindow.ReadPixels` / `TryReadPixels` 在像素锁里只读这几块,直接写进自己的位图;`CopyPixels` 整窗拷一份,给测试与诊断用);光标(`CursorChanged`,`XCursor`:语义形状 `XCursorShape`,位图 / ARGB / 字形光标另带图像 `XCursorImage` —— cursor 字体里有对应系统光标的字形不带,
宿主按形状选系统光标 —— 像素是只读的 `ReadOnlyMemory<uint>`);响铃(`BellRequested`,按协议从基准音量换算出的 0–100;0 表示不响 —— `xset b off`、Bell −100);X 客户端复制了文本(`ClipboardChanged`);有客户端 GrabServer 抓着太久、别人都在等它(`ServerGrabStalled`,见 §5;默认实现什么也不做);托盘图标停靠进来 / 没了(`SystemTrayIconAdded(icon, title)` / `SystemTrayIconRemoved`,开着 `SystemTray` 时,见 §7「系统托盘」);接受本机输入法输入的 XIM 输入上下文换了或插入点动了(`InputMethodFocusChanged(XInputMethodFocus?)`:所在的顶层、程序画不画预编辑、插入点;null = 焦点所在的程序不用 XIM。开着 `InputMethodName` 时,同一批里只交最后一次,见 §7「本机输入法的 XIM 桥」);X 程序把东西拖到了所有 X 窗口以外、数据已经取来(`OutgoingDragStarted(XOutgoingDrag)`:URI、文字、拖出它的程序的连接名、根坐标),以及宿主还没交回结果时那一次在 X 这边先结束了(`OutgoingDragEnded`,开着 `AcceptOutgoingDrags` 时,见 §7「拖出」) |
| 宿主 → 库(`X11Server` 的方法,窗口用 `XTopLevelWindow` 句柄指名;参数不合法当场抛异常,窗口已不在时静默忽略) | 输入 `Inject*`:指针移动 / 按钮(内区坐标,按 X 的 16 位范围核对,超出抛 `ArgumentOutOfRangeException`;滚轮换成 Button 4/5,6 以上是水平滚轮与侧键;X 这边没按着的按钮松开不投递,窗口已经不在时的松开照样生效)、指针离开(位置留着最后一次的,所在窗口算根、child 为 None)、按键(`InjectKey(keycode, pressed, repeat)`,X 键码;`repeat` 标明这是宿主的自动重复,见 §7);用户在宿主自己的界面里有动静(`NoteUserActivity`,空闲计时归零、不产生输入事件,每 250 毫秒至多排一个工作项);窗口管理器的动作(名字带 `TopLevel`):焦点(`FocusTopLevel`,null = 没有焦点;同时在 X 里把它抬到普通顶层的最上面,推进 last-focus-change time)、用户移动 / 缩放了原生窗口(`MoveTopLevel` / `ResizeTopLevel`,库据此改几何、发真实的 ConfigureNotify 再补 ICCCM 的合成事件、发 Expose)、关闭按钮(`CloseTopLevel`:有 `WM_DELETE_WINDOW` 协议就发 ClientMessage —— 声明了 `_NET_WM_PING` 的同时 ping 它 —— 否则断开该客户端;override-redirect 的窗口忽略)、强制结束(`KillTopLevelClient`,KillClient 语义,连同这个客户端的其它窗口)、窗口状态(`SetTopLevelStates` 整组覆盖;`ChangeTopLevelStates(window, add, remove)` 只改给出的位、其余原样,`add` 与 `remove` 重叠当场抛;都写回 `_NET_WM_STATE` 与 `WM_STATE`,`Focused` 由服务端维护)与外框尺寸(`SetTopLevelFrameExtents`,写回 `_NET_FRAME_EXTENTS`);宿主那边的环境变了(不带 `TopLevel` 的 `Set*`):键位表(`SetKeymap`,见 §7)、显示器布局(`SetScreenLayout`,发 RANDR 事件)、DPI 与缩放(`SetDisplayScale`,发 XSETTINGS,只替换 RESOURCE_MANAGER 里的 `Xft.*` 几项)、锁定键(`SetLockState(capsLock, numLock)`,不合成按键,客户端收到 XKB 的 StateNotify)、系统剪贴板有了新文本(`SetClipboardText`,UTF-8 超过 `X11Server.MaxClipboardBytes`(16 MB)当场抛 `ArgumentOutOfRangeException`);卡住时的恢复(`BreakGrabs`:解除一切指针 / 键盘抓取并解冻设备、放开 GrabServer、把浮动的从设备挂回虚拟核心设备,客户端照常收到 Ungrab 模式的事件与 HierarchyChanged);客户端清单(`GetClientsAsync` → `XClientInfo`:编号、连接名、是否以 Retain 模式断开、资源数、记账内存、映射着的顶层、是否正抓着整个服务端 `HoldsServerGrab`)与按编号断开(`DisconnectClient(int)`,Retain 模式断开过的销毁它留下的资源);本机的拖放(`InjectDragOver(window, x, y, types)` / `InjectDragLeave()` / `InjectDrop(window, x, y, data)`,服务端替宿主扮演 XDND 的源;`IsDragAccepted` 是目标最近一次的回应,见 §7「拖放」);本机输入法上屏与预编辑(`InjectText(text)`:焦点所在的程序经 XIM 连着时整段以 XIM_COMMIT 交过去,否则借键码;`InjectPreedit(text, caret)`:on-the-spot 的程序经 XIM 的预编辑回调画在自己的输入框里,别的情况不理,见 §7);宿主对服务端交来的事的答复(`Complete*`,第一个参数是那件事):X 程序拖出来的东西,本机那边放下了没有(`CompleteOutgoingDrag(drag, dropped)`,在把松开的按钮注入回来之前调,见 §7「拖出」) |

**生命周期**:构造时执行线程就开始运行,构造出来的实例即使从没 `StartAsync` 也要 `DisposeAsync`。`StartAsync` 的失败模式见 §7
(显示号被占抛 `SocketException`(`AddressAlreadyInUse`),配置了要监听而一种传输也没开起来抛 `IOException`);失败时已经开起来的监听一并撤回,
同一个实例可以再调一次。`StartAsync` 与 `DisposeAsync` 互斥:同时调时开起来的监听不会漏下没人收。`DisposeAsync` 可以多次、并发地调,
后来的调用等第一次收完才返回,不重抛收尾中的异常;`Completion` 在收工做完时完成。`ServeAsync(stream, isLocal)` 与
`ServeAuthenticatedAsync(stream[, label])` 的参数不合法(null 流、服务端已释放)当场抛,不放进返回的任务里;`label` 是连接的来历
(比如 `user@host:22`),进日志、`XClientInfo.Label` 与快照的 `ClientLabel`,也是剪贴板跟着焦点走时认「同一个会话」的依据(§7)。
第三个参数 `trust` 为 `XClientTrust.Untrusted` 时这条连接是非受信客户端(§7「信任级别」),`XClientInfo.Trust` 报每个客户端的级别。

**句柄**:`XTopLevelWindow` 从映射到销毁(或被 reparent 走)一直是同一个对象。它带着发出它的服务端(`Server`;交给别的服务端的方法抛
`ArgumentException`)与 `IsAlive`(窗口销毁、被 reparent 走、服务端收工之后为 false,不会再变回来)。宿主拿它当字典键时按引用比较,
不按 `Id`(XID)—— 停掉服务端马上再起一个,新窗口的 XID 与旧的重合,而 UI 队列里可能还排着旧服务端的回调;回调里先核对 `Server`
是不是此刻附着的那个。像素:`ReadPixels` / `TryReadPixels` / `CopyPixels` 在窗口销毁或被 reparent 走之后读不到(`false` / `NoBuffer` /
(0, 0));取消映射不算 —— 缓冲还在,读到的是取消映射前最后画的内容。InputOnly 的顶层没有像素。

**快照**(`XTopLevelSnapshot`,服务端每次变更整份替换,宿主先取到局部变量再读;没变的形状、图标沿用同一个列表实例):
几何(`X` / `Y` 是 X 窗口**边框外沿**的左上角,`Width` / `Height` 是内区,`BorderWidth`,`NeedsPlacement` 与 `PlaceInFrame`,见下文「摆放约定」)、
标题(`_NET_WM_NAME` 优先,否则按类型解码的 `WM_NAME`)、`ClassName` / `InstanceName`(`WM_CLASS`)、override-redirect、瞬态父窗口(`TransientFor`)、
`SupportsDeleteWindow`、所属客户端(`ClientId` / `ClientLabel`;托盘图标的嵌入窗口是服务端的,`ClientLabel` 取停靠进来的图标所属的连接)、`InputOnly`、
`HasAlpha`(深度 32 的窗口、且服务端在当合成管理器时才为真 —— 没有合成管理器时 X 显示 ARGB 窗口不看 alpha;托盘图标例外)、`WindowType`、`States`(映射前客户端写好的状态;
`WM_HINTS` 的 initial_state 为 IconicState 时服务端在映射时加上 `Hidden`)、`Decorated`(`_MOTIF_WM_HINTS` 要求无装饰时为 false)、
尺寸约束(最小 / 最大、步长、`BaseWidth` / `BaseHeight` —— 最小与基准互为缺省、`MinAspect` / `MaxAspect`,一律夹到 0–32767)、
`WinGravity`、`UserPosition` / `ProgramPosition`、`WindowGroup`、`Functions`(`_MOTIF_WM_HINTS` 的 functions,`XWindowFunctions`)、
图标(`_NET_WM_ICON`,只解析前 4 MB;没有时把 `WM_HINTS` 的 icon_pixmap / icon_mask 烙成一幅,边长至多 256;`XWindowIcon.Pixels` 是只读的 `ReadOnlyMemory<uint>`)、`Urgent`、`AcceptsFocus`、`Opacity`、
`ClientFrameExtents`、进程号 / 机器名 / 角色、`Shape`(边界形状)、`InputShape`(SHAPE 1.1 的输入形状与边界形状的交集,没设为 null)、
`Strut`(`_NET_WM_STRUT_PARTIAL`,没有时取 `_NET_WM_STRUT`)。交给宿主的字符串只取有限的一段(标题至多 4096 个字符,类名 / 机器名 / 角色至多 256 个),
去掉 C0 / C1 控制字符与双向排版控制符(RLO 之类能在任务栏里把标题倒着显示、伪造扩展名);提示类属性只读规范定义的那几个值。

**窗口管理器请求**(`XWindowManagerRequest` 的派生类,宿主就是窗口管理器,照不照办由它定):
`XMoveResizeRequest`(`_NET_WM_MOVERESIZE`:开始原生的拖动 / 缩放;只有持着指针抓取的客户端发来时服务端才把按着的按钮当作交给了窗口管理器)、
`XStateChangeRequest`(`_NET_WM_STATE` 的增减;已映射、处于 `Hidden` 的普通顶层被客户端 MapWindow 时也发一个去掉 `Hidden` 的 —— ICCCM §4.1.4)、
`XActivateRequest`(`_NET_ACTIVE_WINDOW`,带 EWMH 的来源指示 `Source`、时间戳 `Timestamp` 与服务端的判断 `UserInitiated`:来源是分页器,
或时间戳不早于用户最近一次在 X 窗口里按键 / 按按钮的时间;CurrentTime、过期的不算 —— 宿主应当只在它为真、且用户此刻在用 X 窗口时照办,否则改为提醒)、
`XRaiseRequest`(客户端对顶层发了 stack-mode 为 Above 的 ConfigureWindow:XRaiseWindow、XMapRaised、Java 的 toFront;只是次序,不要求焦点)、
`XFocusRequest`(客户端自己把键盘焦点挪到了这个顶层,按键此刻送往它)、`XNotRespondingRequest`(关闭时 ping 了、5 秒内没回 `_NET_WM_PING`,
宿主可以问用户要不要 `KillTopLevelClient`)、`XCloseRequest`(`_NET_CLOSE_WINDOW`)、`XMinimizeRequest`(ICCCM 的 `WM_CHANGE_STATE`)。
`_NET_MOVERESIZE_WINDOW` 与 `_NET_REQUEST_FRAME_EXTENTS` 不需要宿主拿主意,服务端直接办。

**摆放约定**:宿主不画 X 的边框,原生窗口的内容区对准 (`X + BorderWidth`, `Y + BorderWidth`),原生窗口的外框就当作代替了它。
`NeedsPlacement` 为真时(建窗口、客户端挪顶层、reparent 到根窗口 —— 位置是客户端请求的,窗口管理器还没摆过),宿主按 ICCCM §4.1.2.3
把原生窗口的**外框**(不是内容区)按 `WinGravity` 对准请求的位置:`PlaceInFrame(frame)` 给出套上四边宽 `frame` 的外框之后 X 窗口该在的位置
(`Static` 时内容区不动;NorthWest 的结果与外框尺寸无关,显示前后不跳),摆好之后用 `MoveTopLevel` 报回,之后这一项为 false。
请求 (0, 0) 又不是用户指定的(没有 `UserPosition`)时宿主可以替它选位置;`xterm -geometry +0+0` 带着 USPosition,照办。
override-redirect 窗口不归窗口管理器摆,恒为 false。`_NET_MOVERESIZE_WINDOW` 按重力(0 = 窗口自己的 win_gravity)与宿主给的外框直接换算,
宽高不在 1–32767 的整个不理、坐标夹到 16 位、缓冲变大前先核内存账。

**命名**:`Inject*` 是合成的用户输入;名字里带 `TopLevel`、第一个参数是 `XTopLevelWindow` 的,是宿主作为窗口管理器对那个顶层的动作,
按「动词 + TopLevel + 宾语」起名(`MoveTopLevel`、`SetTopLevelStates`、`ChangeTopLevelStates`、`KillTopLevelClient`);不带 `TopLevel` 的 `Set*`
是宿主那边的环境变了、换进服务端(键位表、显示器布局、DPI、锁定键、剪贴板内容)。公开方法改名是对外的破坏性改动:命名对不上规则时先改规则的写法。

剪贴板互通由 `X11ServerOptions.SyncClipboard`(CLIPBOARD,默认开)、`SyncPrimary`(PRIMARY,默认关)与 `ClipboardFollowsFocus`
(只与键盘焦点所在的会话互通,默认开,见 §7)控制。

**每个顶层窗口有一块自己的像素缓冲**(相当于常开的 backing store + Composite;InputOnly 的顶层没有):子窗口画在所属顶层的
缓冲里,裁剪到自己的可见区域。好处是被别的原生窗口遮住的内容不丢,换来的只是内存(记在内存账上,§7)—— 不必在每次
遮挡变化时让客户端重画。只在映射、变大、ClearArea(exposures)与子窗口取消映射露出父窗口时发 Expose;顶层的边界 / 裁剪形状变了只重画新露出来的部分,
改输入形状不重画。改尺寸时缓冲尽量就地挪行,要新建时多留四分之一(多留的不记账)。

## 7. 关键取舍

- **视觉只有 TrueColor**:深度 24(根视觉,掩码 `0xff0000 / 0xff00 / 0xff`)与深度 32(ARGB,给 RENDER 留着);
  像素图另可建深度 1 / 4 / 8 / 15 / 16(与 X.Org 的惯例一致 —— Xt 程序会建 4 / 8 深度的像素图,只认 1 / 24 / 32 时它们报 BadValue)。
  像素值就是 RGB,AllocColor 只是换算,不存在「颜色表用完」;可写颜色单元(AllocColorCells)一律 BadAlloc。
  颜色名按 X.Org 的 `rgb.txt` 查(嵌入资源,首次查找时解析;去空格、不分大小写,`dark slate gray` 与 `DarkSlateGray` 是同一项,
  `red3`、`VioletRed4` 这类编号变体都在)。
- **软件光栅化,不用 Skia**:核心绘图的语义是**逐像素精确**的(细线的 Bresenham 端点、GXxor 橡皮筋、
  平面掩码),抗锯齿的 2D 库给不出同样的像素。宿主拿到的是 32 位像素,怎么贴到屏幕上是宿主的事。
  按协议,整数坐标就是像素中心:多边形、宽线与弧的第 row 行在 y = row 处采样,边按上闭下开、跨距按左闭右开取像素
  (RENDER 的覆盖率掩码按 RENDER 规范以 +0.5 为中心,不受影响)。
- **核心绘图照协议的细节,代价按看得见的部分算**:
  - 先与裁剪求交:矩形先与可画区域求交再逐行填;细线只走与可画区域相交的那一段(这种 Bresenham 第 k 步副轴走了几格有闭式解,
    二分出起点接着走,画出的像素与从头走一遍相同);裁剪判断按带二分;CoordModePrevious 累加出的坐标饱和在 ±2³⁰;
    宽线的圆帽 / 圆接头至多 1024 个顶点,外接矩形碰不到可画区域的段与接头直接跳过。
  - 宽线与弧按 line-style、join-style、cap-style 画:Miter 把两条外沿延长到相交(夹角小于 11° 时退成 Bevel),Bevel 补外侧的三角,
    Round 补圆;端点重合的段从路径里拿掉,整条缩成一点时 Round 画圆、Projecting 画方块。虚线沿线按长度量,OnOffDash 只画偶数段;
    DoubleDash 的奇数段按背景 —— 源按 fill-style:Solid 用背景色、Stippled 用背景色按点画遮、Tiled / OpaqueStippled 与偶数段相同。
    宽弧不是整圆时两端按 cap-style 加端帽,内外边界是半轴各加 / 减半个线宽的椭圆,不取整。外接框宽或高为 0 的弧是一条线段,协议说这时边界不由实现决定、就是与路径相距半个线宽的两条线:每段单调的走位画满一个线宽,在两端之间折返的地方(宽为 0 的在 ±90°、高为 0 的在 0° / 180°)绕着线段的一头转半圈,补一个直径为线宽的圆。PolyLine 三个点以上、首尾重合时按闭合路径画
    (那里是接头而不是两个端帽,细线不画第二遍终点)。PolyArc 里前一条的终点与后一条的起点重合的弧按协议「join correctly」:
    「重合」按各轴相差都不到半个像素(留 1/256 像素的余量)判 —— 端点是实数,逐位相等太苛刻,取整到同一个像素在 .5 上又不稳;
    最后一条接回第一条时整串闭合。相接的一串宽弧当一条路径:相邻两条之间按 join-style 加接头(搭在两条环带真实的端面角上,
    椭圆的端面不一定垂直于切线),只在整串两头加端帽,闭合时不加,整串放进同一张活动边表一次填(GXxor 下接缝不画两次);
    同一个椭圆上接着走的几段并成一条画(四段 90° 弧拼成的整圆与一条 360° 的弧逐个像素相同)。虚线沿整串量、跨过接点接着走,
    不相接的每串从 dash-offset 重新开始。细弧整串连成一条细折线,接点只画一次。一串宽弧攒到 2²⁰ 个顶点就先填一批。
    单独一条弧与互不相接的弧像素与原来相同(20 万组随机对拍)。
  - CopyArea / CopyPlane:源窗口只有看得见的部分可拷(ClipByChildren 时映射着的子窗口也挡着,IncludeInferiors 时连子窗口的内容一起拷);
    拷不到的部分(被挡住、不可见、子窗口伸出顶层缓冲)在背景不为 None 的目标窗口上先铺背景,再逐块发 GraphicsExposure,都拷到了才发
    NoExposure。CopyPlane 的 bit-plane ≥ 2^源深度回 BadValue。GetImage 读到伸出顶层缓冲的部分补 0(被遮住区域的内容本来就未定义)。
  - 出错的请求不产生效果:CreateGC / ChangeGC 先全部读出、核对完再写进 GC,SetDashes、RENDER 的 FreeGlyphs 同理;GC 的枚举值
    (function、line-style、cap-style、join-style、fill-style、fill-rule、subwindow-mode、arc-mode……)越界回 BadValue;PutImage 的
    Bitmap / XYPixmap 格式 left-pad ≥ 32 回 BadMatch。
  - 单字节字体的 CHAR2B 按高位在前的 16 位数取字,byte1 不为 0 时按缺字走 default-char。
- **键码采用 evdev 编号(evdev + 8)**,与现代 Linux 上的 Xorg 一致 —— 远端客户端大多预期这套编号。
  宿主负责把物理键翻成 X 键码,键值表由库给出(US 布局起步,可替换)。`XKeycodes` 另有日文 JIS / 巴西 ABNT2 / 韩文键盘的键
  (Ro、Yen、Henkan、Muhenkan、Hiragana_Katakana、KP_Equal、Hangul、Hangul_Hanja)、F13–F24 与多媒体键,起步的键值表配齐了它们的键值。
- **键盘:节奏与布局来自宿主,语义由服务端定**:
  - 自动重复的节奏是宿主的系统设置:同一个键按着又来按下时,宿主用 `InjectKey(keycode, true, repeat: true)` 标出;服务端按 X 的语义决定
    —— `xset r off`、这个键逐键关了、它是修饰键(逐键默认除修饰键外都开)就丢掉;否则没开 XKB DetectableAutoRepeat 的核心客户端收一对
    「松开、按下」,开了的只收按下,XI2 的 KeyPress 与 RawKeyPress 带 KeyRepeat 标志;中间那个松开不解除被动抓取。XKB 的 RepeatKeys /
    PerKeyRepeat 与核心是同一份设置,重复的延迟与间隔只记录、回报。
  - ChangeKeyboardControl 生效:各项先核对后生效(−1 恢复默认,别的负值 BadValue;只给 led / key 而缺 mode 是 BadMatch);响铃的基准音量 /
    音高 / 时长、LED、按键音、全局与逐键的自动重复都记下,GetKeyboardControl 如实回报。`xset b off` 之后响铃算出的音量是 0,宿主不出声。
  - SetPointerMapping 生效:9 个按钮(GetPointerMapping 与 XI 报同一份),长度不对或非零元素重复回 BadValue,要改的按钮正按着回 Busy,
    成功时发 MappingNotify。宿主与 XTEST 给的是物理按钮,核心与 XI2 事件、状态位、自动抓取按映射后的按钮号,原始事件报物理按钮号。
    SetModifierMapping 的键码不在 8–255 回 BadValue,涉及的键正按着回 Busy。
  - `SetKeymap` 只改宿主这次给的与上次不同的键(客户端用 xmodmap 改过、宿主没变的键保留);右 Alt 的角色(AltGr / Alt_R)变了才改它的键值,
    并只在 Mod1 与 Mod5 之间挪它,修饰键表的其余部分不碰;什么都没变时不发 MappingNotify / MapNotify。宿主在 X 窗口里每次按下(不含自动重复)
    先看一眼系统布局,变了先推新键位表再注入这个键(Windows 只比布局句柄;macOS / Linux 算一遍较贵,至多每秒看一次;手选了布局时不跟随)。
  - 锁定键:服务端起步时 CapsLock / NumLock 都关着,宿主在 X 窗口得到焦点时按系统的真实状态推一次(`SetLockState`,不合成按键;
    Windows 用 GetKeyState,macOS 用 CGEventSourceFlagsState、NumLock 当一直开着,Linux 在后台线程上读桌面 X 显示的 XKB 锁定位)。
  - 空闲:宿主经 `NoteUserActivity` 报本机界面里的活动,远端经 MIT-SCREEN-SAVER / SYNC 的 IDLETIME 看到的空闲时间不再只按 X 输入算。
  - macOS 上带 Command 的组合键可能收不到 KeyUp:宿主记下按着 Command 时按下的键,Command 松开时替还没松开的松开
    (Avalonia 已经给了 KeyUp 时这一步什么也不做)。
- **授权**:默认只监听 `127.0.0.1`(TCP 默认只在配了 cookie 时开,§4)与 Unix 套接字。依次判断:① 调用方已经验过身份的流(`ServeAuthenticatedAsync`,
  宿主的 SSH 连接器用它 —— 假 cookie 在 SSH 那一层已经核对过)放行;② 带了对的 `MIT-MAGIC-COOKIE-1`(常数时间比较)放行;
  ③ 能确定对端就是运行服务端的这个用户 —— 取得到对端 uid 时(Linux 的 SO_PEERCRED,macOS / FreeBSD 的 getpeereid)以 uid 为准,
  与本进程的有效 uid 相同才算;取不到(Windows)时才看套接字文件在 `bind` 与 `listen` 之间设成的 0600 —— 放行(自定义路径落在 9p / drvfs
  这类 chmod 不报错却不生效的文件系统上时,只看文件权限的话谁都连得上);
  ④ 知道对端 uid 而它是别的用户:拒(Linux 抽象命名空间里的套接字没有文件权限可言,不看 uid 的话本机任何用户都能连进来
  读窗口、记键盘、经 XTEST 注入输入),没配 cookie 时 accept 之后当场关;⑤ 配置了 cookie 时其余一律拒(环回 TCP 也一样:本机别的进程、别的用户都连得到那个端口);
  没配置时与 X.Org 的主机访问控制一致,只放行本机。cookie 要 16–256 字节(`MinAuthorizationCookieLength` / `MaxAuthorizationCookieLength`;
  空数组曾让任何带空数据的 cookie 通过),构造时拷一份,之后调用方再改那个数组不影响授权。访问策略是固定的:ListHosts 报访问控制开着、
  主机清单为空,ChangeHosts 与 SetAccessControl(Disable) 回 BadAccess —— 原先静默成功,`xhost +` 看起来生效了,其实什么也没变。
  宿主的内置服务端每次启动生成一个新 cookie,写进用户的 `.Xauthority`(`XAUTHORITY` 优先;编解码用 SSH 库的 `XAuthority`,严格解析认不全的文件不改写;
  按 xauth 的约定先建 `-c`、再建 `-l` 两个锁文件,一分钟以内的锁不当残留;停止时只撤自己那一条;网络地址变化时核对主机名,变了按新名字重登、撤掉旧的),
  本机 X 程序经 Xlib 自动带上;读不到这个文件的程序连不进来。
  ⚠️ 设计层面的提醒(不是缺陷):所有开了 X11 转发的 SSH 会话共享这一个受信的显示 —— 一台被攻破的远端机经它的转发,
  能看到、也能操作别的会话里的 X 程序(读窗口内容、记键盘、注入输入)。受信的 X11 转发本来就是这个含义;不信任的远端机别开 X11 转发。
  能收紧的部分见下面「信任级别」与「多个会话共用一个显示」;要彻底隔开,宿主可以给每个 SSH 会话一个显示(下面「每个 SSH 会话一个显示」)。
  2026-10-10 之前本库没有 SECURITY 扩展,内置引擎下选了非受信模式的连接干脆不开 X11 转发;现在非受信模式经连接器照常转发,连进来的是非受信客户端。
- **监听与显示号**:
  - 显示号的名字被占就整个显示号不用:Linux 的抽象名已被人 bind、套接字文件后面有人在听、残留的套接字文件删不掉(别的用户的,属主随时可以再 listen)、
    删掉之后 bind 又撞上,`StartAsync` 都抛 `SocketException`(`AddressAlreadyInUse`),已经开起来的 TCP 与 Unix 监听一并撤回。原先只记一行日志、
    照样用别的传输开起来 —— 攻击者先 bind 抽象名不 listen,等我们开起来再 listen,就能收到 Xlib 按 `:N` 先试抽象名时发来的 cookie。
  - 放套接字文件的目录(建完再核,防有人抢在建目录之前建好)必须是目录而不是符号链接,属主是本用户,或者是 root 且带粘滞位;否则不开套接字文件、
    只记日志(Linux 经 statx、macOS 经 lstat 取属主)。macOS 上 `/tmp/.X11-unix` 通常不存在,谁先建谁是属主。
  - 占用探测(服务端启动时、宿主挑显示号时)对套接字一律异步 connect、限时 300 毫秒;Linux 上非阻塞 connect 回 EAGAIN、或到时限没连上,
    都按「有人在听」算,不删它的套接字文件 —— 对 backlog 已满的套接字做阻塞 connect 会一直等,X Server 按钮因此再也启动不了。
    宿主对抽象名改成 bind 一下试试(原子;成了立刻放掉)。
  - 类 Unix 上按 Xserver(1) 的约定持有 `/tmp/.X{N}-lock`(内容是 PID,十位右对齐、换行结尾,0444),收工或启动半途失败时删掉;持有者还活着、
    或内容认不出时 `StartAsync` 抛 `SocketException`(`AddressAlreadyInUse`);持有者已经不在了是残留,删掉重建;`/tmp` 写不了只记日志。
    Xvfb、`xvfb-run -a` 挑显示号时只看这个文件。
  - 配置了要监听、结果 TCP 与 Unix 套接字一种也没开起来时抛 `IOException`。宿主自动选号时,开起来那一刻才发现被占(探测与绑定之间被抢先,
    或锁文件、抽象名这些探测看不出的占用)就换下一个空闲的号再试,至多 4 次;手动指定的号直接报错。
  - Windows 上 TCP 监听设 `SO_EXCLUSIVEADDRUSE`:否则同一个用户的别的进程(包括低完整性的)用 SO_REUSEADDR 绑更具体的地址照样绑得上,
    把本机连接(带着 cookie)接走。宿主判断「本机已经有别的 X 显示在用」时,`localhost:0` 后面的监听者要是当前用户会话里的进程才算 ——
    终端服务器上那可能是别的用户开着的 VcXsrv。
- **信任级别(SECURITY 扩展,2026-10-10,xs_plan F2)**:依据 X Consortium 的 *Security Extension Specification* 7.1(错误与事件的编号、
  SecurityGenerateAuthorization 的请求布局依据 xorgproto 的协议头 —— 规范第五章的编码表把 value-mask 列在两个字符串之后,
  xauth 实际发的与协议头一致,在定长部分)。
  - **扩展本身**:QueryVersion(1.0)、GenerateAuthorization(只签 MIT-MAGIC-COOKIE-1,16 字节随机 cookie;timeout 默认 60 秒、trust-level 默认非受信、
    group 只能是 None、event-mask 只有 AuthorizationRevoked)、RevokeAuthorization(连同用它连着的客户端一起断开)、AuthorizationRevoked 事件;
    授权在没有连接的状态下满 timeout 秒自动作废;至多 256 个。签出的 cookie 在登记客户端时(执行线程上)核对,对不上照旧拒。
  - **宿主直接指名**:`ServeAuthenticatedAsync(stream, label, XClientTrust.Untrusted)` —— 进程内的连接器用不着去签 cookie(SSH 的 `ssh -X` 就这样接进来)。
  - **非受信客户端受的限制**(规范第三章):只能指名非受信客户端的资源 —— 受信客户端的窗口、像素图、GC……一律当作不存在(请求里解析 ID 的地方
    统一把关,服务端内部的查找不变);QueryTree / GetGeometry / TranslateCoordinates 不受限。根窗口(与服务端自己的窗口)只在规范列出的请求里可用,
    另许 RANDR、XINERAMA、XFIXES 的 Select*Input 与 XIQueryPointer / XISelectEvents / XIGetSelectedEvents(工具包初始化都会对根窗口发,
    回 BadWindow 时 Xlib 默认的错误处理会让程序退出);根窗口上只能选 StructureNotify / PropertyChange,XI2 只留设备 / 层级 / 属性变化;
    往根窗口只能发 ICCCM 规定的那几种事件。XTEST、MIT-SCREEN-SAVER、DPMS、X-Resource、Composite、MIT-SHM、SECURITY 对它不可见。
    键盘不归它时(按当前焦点、指针、抓取与选择,键盘事件送不到任何非受信客户端):QueryKeymap / KeymapNotify 全 0、GrabKeyboard 与键盘的
    XIGrabDevice 回 AlreadyGrabbed、SetInputFocus 不生效、它的被动键盘抓取不激活、按键引起的 XKB StateNotify 不发给它。改键位表、修饰键、
    键盘与指针设置、XKB / XI 的设备设置、主机访问控制与 KillClient(AllTemporary)回 Access;GrabServer 忽略。根窗口上的 CUT_BUFFER0–7 对它隐藏,
    它改服务端窗口上的属性当作没发生(选区窗口上服务端替宿主取剪贴板用的 `_VELASHELL_*` 除外 —— 选区经宿主中转照常可用);
    要受信客户端占着的选区回 property None;GetImage 被别的窗口挡住的地方填 0;它把背景设成 None 时用黑色。
  - `RestrictForwardedClients` 保留:它是「受信、但不给 XTEST / 原始按键 / 改设备层级」的中间一档,给在非受信下工作不正常的老程序;
    非受信客户端本来就受这三项限制。
- **每个 SSH 会话一个显示(宿主,2026-10-10,xs_plan F1 / 决策 Q5)**:同一个显示上,受信的会话之间仍能互相看窗口、读剪贴板、经 XTEST 注入。
  宿主的设置开着时,带着会话对象进来的 x11 通道进这个会话自己的 `X11Server`:不监听任何端口(`ListenTcp = false`、`UnixSocketPath = ""`),
  只经连接器喂流,有自己的根窗口、选区与剪贴板,XTEST 与原始事件只碰得到同一个会话的程序。同一个会话的通道进同一个;第一条通道来时建,
  会话断开或停服时收掉;至多 32 个,超了新会话的 X 程序连不上(不落回共用的显示 —— 那等于悄悄取消了用户要的隔离)。本机程序(`DISPLAY=:N`)
  与没带会话的照旧进共用的那个。宿主给每个服务端配一个自己的 `IX11ServerHost`(窗口、键位表、DPI、剪贴板各管各的),设置里归宿主管的几项
  (键盘布局、窗口模式、本机输入法、来源标识)每个都拿到。单窗口模式下用户关掉某个会话的屏幕窗口,只收掉这个会话的显示(宿主经
  `IEmbeddedXServerHost.SessionDisplayCloseRequested` 告诉 `BuiltInLocalXServer`,有程序连着时先确认),别的会话与共用的显示不受影响,
  这个会话之后再开 X 程序时另建一个 —— 2026-10-10 核对之前关掉任何一个会话的桌面都停整个 X Server。
- **多个会话共用一个显示时的收紧**(同一个显示上、受信的会话之间):
  - **剪贴板跟着键盘焦点所在的会话走**(`X11ServerOptions.ClipboardFollowsFocus`,默认开):宿主的文本只给焦点所在顶层的客户端、以及与它连接名
    相同的客户端(同一个 SSH 会话里的 `xclip` / `xsel`,连接名见 `ServeAuthenticatedAsync(stream, label)`)读,X 这边的复制也只收那个会话的;
    没有 X 窗口有焦点时谁都读不到。否则本机复制的密码在用户点一下任意 X 窗口之后对所有会话可读,后台会话里的程序也能反复改写本机剪贴板。
    服务端替宿主占有剪贴板时,XFIXES 的属主通知也只发给读得到的客户端 —— 别的会话本可以据此精确得知「本机剪贴板何时有了新内容」。
    开着时 PRIMARY、SECONDARY、CLIPBOARD 还**按会话隔离**:选区表按「原子 + 作用域」存,每个会话(连接名相同的客户端;没有连接名的本机程序
    同属一个本机会话)各有各的属主 —— SetSelectionOwner、GetSelectionOwner、ConvertSelection、SelectionClear、XFIXES 通知、属主窗口销毁 /
    属主断开都只在那个会话里生效。别的会话看不到属主变化、收不到因此发的 SelectionClear,也读不到另一个会话里的复制。其余选区
    (`WM_S0`、XSETTINGS、拖放……)照协议全显示共享。跨会话的复制粘贴经宿主的剪贴板中转:剪贴板内容有一个逻辑时钟(宿主给了新文本、
    X 端的复制交给了宿主、哪个会话里的程序占有了同步的选区,各推进一格),某个会话拿到焦点(PointerRoot 时指针换了顶层)时它那一份
    比最新的文本旧,服务端就替宿主在那里占有 —— 在 A 里复制、切到 B 再粘贴,内容照样到;最近一次复制赢,不管它发生在哪个会话或本机。
    同一会话里的程序之间照常互相复制粘贴。
    只有一个受信客户端的嵌入场景可以关掉。
  - **`RestrictForwardedClients`**(默认关):经 `ServeAuthenticatedAsync` 进来的连接看不到 XTEST(伪造的输入与真实键盘无从区分,被攻破的远端机
    能往别的会话的 xterm 注入命令)、收不到 XI2 的原始按键事件(不抢焦点就能记下所有 X 窗口里敲的键)、XIChangeHierarchy 回 BadAccess
    (能让物理输入设备失效)。默认关是因为远端的 xdotool 之类靠 XTEST 工作;宿主在设置里给了开关。
  - **焦点窃取防护**:`XActivateRequest.UserInitiated` 由服务端按来源与时间戳判断(§6)。Avalonia 宿主只在它为真、且用户此刻在用 X 窗口时激活,
    否则只闪任务栏;override-redirect 与「总在最前」的窗口只在 X 窗口活动时置顶,用户回到本机窗口时退到后面 —— X 的原生窗口与宿主主窗口在同一个进程里,
    系统的前台锁拦不住它们之间的切换,远端程序原本能在用户输 sudo 口令时跳到前台接走按键,或者盖一个像系统凭据框的全屏弹层。
  - **服务端占住根窗口的 SubstructureRedirect 与 `WM_S0`**:别的客户端选根窗口的 SubstructureRedirect 回 BadAccess(与真实桌面上已有窗口管理器时一样),
    远端误跑的 openbox / xfwm4 接管不了所有会话的新窗口。save-set 按协议「Connection Close」生效:断开时把 save-set 里挂在它的窗口下的窗口
    reparent 到最近的一个不是它建的祖先(根坐标不变)、没映射的补映射,在销毁资源之前做;核心 ChangeSaveSet 对自己的窗口回 BadMatch,
    XFIXES 的 ChangeSaveSet 支持 target 与 map 标志。
  - **服务端自己的窗口与根窗口同等保护**:剪贴板桥 / XSETTINGS / WM 检查共用的 `0x43` 与 Composite 的覆盖窗口不能被 reparent、改几何、在其下建窗口
    (BadMatch),ChangeWindowAttributes 只许改事件选择(其余 BadAccess),Map / Unmap 不理,销毁时跳过 —— 原先先 reparent 再销毁父窗口,
    整个显示的剪贴板桥接、XSETTINGS 与窗口管理器检查就一起没了。
  - **选区的时间戳规则一律检查**:SetSelectionOwner 的时间晚于服务端当前时间、或早于这个选区最后一次换属主的时间都无效,最后一次换属主时间不随属主
    消失 —— 恶意客户端带一个未来的时间戳占了 CLIPBOARD,之后别人带真实时间去占不会再被静默忽略。宿主替用户占有时用 max(当前时间, 最后一次换属主时间)。
  - **别人的拖动打断不了**:`_NET_WM_MOVERESIZE` 只有持着指针抓取的客户端发来时才放开按钮、解除自动抓取;别人发来的照样转给宿主,但不动按钮状态与抓取。
    `_NET_MOVERESIZE_WINDOW` 的几何按 X 的范围核对(§6)。
  - **卡住时有出口**:宿主的 `BreakGrabs`、按编号 `DisconnectClient`、`KillTopLevelClient`;GrabServer 抓太久时日志点名持有者、并经 `ServerGrabStalled` 告诉宿主(§5)。
    宿主(VelaShell)的入口在标题栏 X Server 按钮的浮层:连着的程序逐个断开、「解除卡住」;抓太久时弹一条带「断开它」的提示。
    以 Retain 模式留着资源的客户端至多 16 个(超了的断开时按 Destroy 处理并记一行),X-Resource 列得出它们 —— 原先循环「连上 → RetainPermanent →
    断开」254 次就占满所有编号,谁也连不进来。
  - GLX:任何客户端都能拿别人的上下文当 share list、MakeCurrent / CopyContext / DestroyContext 别人的上下文 —— 与核心协议「客户端之间不隔离」的信任模型一致
    (上下文标签按客户端分表,伪造不可行),留到做连接级的信任档时一起收紧,代码里已注明。
- **资源有单件上限,也有总量的账;超了回 BadAlloc(或 GL 的 OUT_OF_MEMORY)而不是拖垮进程**:
  - 单件上限:客户端 255 个(处在连接建立阶段的另算,至多 32 条);窗口嵌套 256 层、每客户端 32768 个窗口(销毁与重画走显式栈,不递归);
    区域最多 16384 块,并 / 交 / 差另有归并预算;像素缓冲 2²⁶ 像素;属性值 32 MB(XI 设备属性同样);客户端建的原子 2¹⁸ 个、名字合计 16 MB
    (XFIXES SetCursorName 起的名字走同一个入口);XI2 设备 ID 到 255 为止;XC-MISC 一次最多 65536 个 ID;被动抓取(核心与 XI2)每窗口每客户端 4096 个,
    每个抓取减掉的组合至多 1024 个;XFIXES 的选区监听每客户端 1024 条;每个可绘对象上的 Damage 256 个;Present 排着的请求每客户端 256 条、
    一条 PRESENTNOTIFY 至多 64 项;以 Retain 模式留着资源的客户端 16 个;GetImage 的回复 256 MB;剪贴板文本 16 MB;GLX 见下文。
  - 内存账:`X11ServerOptions.MaxClientMemory`(每客户端,默认 1 GiB)与 `MaxTotalMemory`(全部合计,默认 2 GiB)—— 只限单件的话,一条 16 字节的
    CreatePixmap 就是 256 MB,十几条就能让服务端连同宿主进程一起耗尽内存。记账的有:每个资源一份固定开销(64 字节);有独立缓冲的像素图(不是 NameWindowPixmap 那种借用窗口缓冲的);
    顶层窗口的缓冲(映射与改尺寸时先核账);DOUBLE-BUFFER 的后缓冲;Composite 子窗口的像素图,以及 NameWindowPixmap 给出、顶层改尺寸 / 重新映射之后
    独占旧缓冲的像素图(记在像素图属主名下);呈现之前就被释放的 Present 像素图(记在发请求的客户端名下);属性值与 XI 设备属性(记在写它的客户端名下);
    RENDER 字形(记在加它的客户端名下,字形表最后一个引用没了才释放)与渐变的色标;XFIXES 区域;GLX 的对象与拼到一半的 RenderLarge(见下文 GLX)。
    资源离开资源表、属性被替换或删掉、窗口销毁、客户端断开时如数退还;服务端自己写的属性替换掉客户端写的值时也退还。客户端的请求在分配之前核账,
    超了回 BadAlloc、不产生效果;宿主发起的改尺寸照记不拒;缓冲改尺寸时多留的余量不记。`XClientInfo.MemoryBytes` 报每个客户端账上的数。
  - 每项工作另有工作量预算(§5)。
- **字体**:核心字体是随库带的 X.Org 位图字体与 GNU Unifont(xs_plan CP-16 / F21 方案 A):misc-fixed 全套(`4x6` … `10x20` 的常规 / 粗体 / 斜体,
  中日韩的 `12x13ja` / `18x18ja` / `18x18ko` 与 JIS X 0208 的 `k14`)、`nil2`、`cursor`;Adobe 75 / 100 dpi 的 Courier、Helvetica、New Century Schoolbook、
  Symbol、Times(8–24 磅);GNU Unifont 18(16 像素,覆盖整个基本多文种平面)。233 份 BDF 原样、Brotli 压缩后约 5 MB(库的 DLL 约多 4.5 MB),
  照 X.Org 安装后的目录分 misc / 75dpi / 100dpi(先后即字体路径的先后),每个目录一份 mkfontdir 格式的 `fonts.dir`。
  ISO10646-1 的字体还以它**完整覆盖**的单字节字符集的名字出现(与 `mkfontdir -e` 一样):ISO8859-1 是 Unicode 的前 256 个码位,ISO8859-2 / 3 / 4 / 5 / 6 / 7 / 8 /
  9 / 13 / 15 与 KOI8-R / U 按随库的映射表派生(.NET 的代码页表导出;没有 ISO8859-10 / 11 / 14 / 16),派生字体的 CHARSET_REGISTRY / CHARSET_ENCODING 跟着换;
  一共 1877 个名字。别名照 X.Org misc 目录的 `fonts.alias` 原文(`fixed`、`variable` —— Helvetica Bold 12 磅 ——、`5x7` …),目标没有随库带的
  (Sony、JIS、ISAS、OPEN LOOK 的字体)不列出、打开 BadName;另认 `9x18` / `9x18bold`,`8x16` / `12x24`(Sony)退到最接近的 misc-fixed。
  完整的 14 字段 XLFD 没有完全匹配时,在 foundry、family、weight、slant、charset 都对得上的里面取像素高度最接近的(按 PIXEL_SIZE,
  没给时按 POINT_SIZE 与 RESOLUTION_Y 换算 —— RESOLUTION_Y 留空时按屏幕分辨率 `X11ServerOptions.Dpi`,都没给不猜);一样近时,给了 AVERAGE_WIDTH 的先挑平均宽度最接近的,再取小的 —— 把平均宽度翻倍要双宽字体的
  `-misc-fixed-medium-r-semicondensed--13-120-75-75-c-120-iso10646-1` 拿到 12x13ja,而不是 7x13;平均宽度只在一样近的里面比,不会为了字宽换到更远的字号。按磅数要、分辨率留空的名字(POINT_SIZE 是数字,PIXEL_SIZE 与 RESOLUTION_Y 不是;`variable` 就是这样),匹配上的里面先挑 RESOLUTION_Y 离屏幕分辨率最近的 —— XLFD 的 POINT_SIZE 是物理尺寸,96 dpi 的屏幕上 12 磅取 100 dpi 的那份(17 像素),而不是按名字的先后取到 75 dpi 的 12 像素;退到最接近的字号时一样近的也再比分辨率。没有随库带的字族(B&H 的 Lucida —— 许可要求在用户文档与代码注释里附特定声明 ——、
  Bitstream、可缩放字体)照旧 BadName。名字表在第一次用到时建一次,字体第一次打开时才解压、解析,解析结果整个进程共享(多个服务端实例只解析一次);
  字形位图按行打包(每像素 1 位)。量下来(Debug):名字表加 `fixed` 35 ms,Unifont 第一次打开 180 ms、9.6 MB,全部名字打开一遍(`xlsfonts -l`)
  870 ms、共 58 MB。这些都不在执行线程上做:OpenFont 与 ListFontsWithInfo 要用到还没建好的字体时,字体在线程池上解压、解析,
  这个客户端的这条与之后的请求暂存(与 SYNC 的 Await 同一套),建好了按原顺序放回 —— 别的客户端与宿主界面不陪着等像素锁。BDF 里带引号的属性值是字符串(QueryFont 里回原子),看上去是数字的也一样(`CHARSET_ENCODING "1"`)。
  字形光标按协议烙成图像(源与掩码字形的原点重合即热点),一个像素也不显示的(xterm 拿 `nil2` 的空白字形做的隐形指针)给宿主 `Hidden`,
  字形没定义回 BadValue。`cursor` 字体(X.Org 的 cursor.bdf)的字形另记下字形号:有对应系统光标的(left_ptr、xterm、watch、缩放边角……)按形状交给宿主、
  不给图像 —— 系统光标跟着桌面的主题与缩放 ——,没有对应的(pencil、gumby、dotbox……)把图像交给宿主;XFIXES 的 GetCursorImage 一律给烙好的图像。
  现代工具包不用核心字体(走 RENDER + 客户端栅格化),所以核心字体只需覆盖老程序。
- **RENDER 在 8888、a8 与 a1 目标上用整数合成**:像素一律预乘 alpha。到 a8r8g8b8 / x8r8g8b8 / a8 / a1 目标时,全部 53 种运算
  (Porter-Duff、Disjoint / Conjoint、PDF 混合模式,带不带分量 alpha)都先把源与遮罩各取成 8 位预乘的一行 —— 渐变、变换、repeat、
  各种源格式都在取样里处理掉(纯色、8888 与 a8 直接换位,双线性用 0–256 的定点权重;渐变的颜色仍按浮点算、量化一次)—— 再逐像素整数合成。
  Src / Over / Add 有专门的写法;其余 Porter-Duff 与 Disjoint / Conjoint 每个通道是 源 × Fa + 目标 × Fb,不带除法的因子取 0–255,
  带除法的(Saturate、Disjoint / Conjoint 的 min(1, n / d)、max(1 − n / d, 0))拿没取整的 源 alpha × 遮罩 按 1/65025 算,每个通道攒齐后只取整一次。
  混合模式不先除出 cs = Cs / αs,而是把 αs·αd·B(cs, cd) 整理成 8 位 Cs、αs、Cd、αd 的整数式(柔光的平方根查表,HSL 四种两边同乘 αs·αd),
  三项按 1/255³ 攒齐、只取整一次。分量 alpha 时因子逐通道、按那个通道自己的源 alpha 算。只有 alpha 的 a8 / a1 目标只算 alpha 一个通道
  (混合模式的 alpha 是 αs + αd − αs·αd,分量 alpha 不影响它);a1 按 ≥ 0.5 取 1。8 位像素的源与逐像素浮点合成最多差 1,a1 逐位相同(穷举核对过);
  双线性与渐变的源先量化成 8 位,与全程浮点最多差 2。其中纯色源 + 单字节遮罩 + Over(Xft 画字、cairo 的抗锯齿图形)与 8888 图像 Src / Over
  (贴图、窗口间拷贝)不逐行取样、直接按存储整块算。r5g6b5、x1r5g5b5、a4 等别的目标格式按浮点逐像素合成(每通道 0–1)。
  源或遮罩与目标是同一块缓冲时先拷出要读的那一块(结果要像「先读完源再写」):按目标上真正写得到的范围对回源坐标,有变换时取四个角变换后的外接矩形,
  再按 repeat 折回 —— 不整张拷。只有 alpha 的字形按每像素一字节存;一个 CompositeGlyphs 请求只算一次目标、只记一次损伤,
  带遮罩格式时遮罩只按目标上可写的一块分配。梯形与三角形按 16 条子扫描线、水平方向解析地算覆盖率,只算目标上可写的那一块。
  源 / 遮罩 picture 的裁剪在没有变换、不重复时也限制读取(RENDER 0.11 §7:clip-mask「affects all graphics requests, including sources」);
  有变换或重复时、以及梯形与字形的源,仍只裁目标。alpha-map 照 RENDER 0.11「CreatePicture」:它的 alpha 通道取代
  drawable 的 alpha 通道(颜色通道照原样 —— 像素一律是预乘的),原点相对 drawable 原点,读写都受 alpha-map 的范围与裁剪限制。当源时,变换与过滤作用在
  drawable 与 alpha-map 拼好的那张上,alpha-map 本身不变换、不重复,有变换时 alpha-map 之外取到的 alpha 为 0;drawable 范围之外又不重复的像素整个透明。
  当目标时,要写的那一块(请求的范围 ∩ 可写区域)拼成一张临时的 a8r8g8b8 合成,颜色写回 drawable(drawable 自己的 alpha 通道不动)、alpha 写回 alpha-map
  并记 DAMAGE。挂了 alpha-map 的源与目标是同一块缓冲(drawable 或 alpha-map 就是目标)时同样先拷出要读的部分。alpha-map 必须是像素图上的 picture
  (否则 BadMatch),格式不限;已经带 alpha-map 的不能再挂上去(规范说结果未定义,这里回 BadMatch)。alpha-map 只作用一层:先挂上、再给作 alpha-map
  用的那张挂 alpha-map 时,后者在合成时不算 —— 否则一张张接下去的长链每一环递归一层,约六千多环就栈溢出,把宿主进程整个带崩。
  poly-edge / poly-mode / dither 接受但不生效。
  渐变的色标要在 0–1 之间并按大小排好(否则 BadValue,相等的硬过渡照收),取样时二分找色标;CreateCursor 的热点不在图里回 BadMatch;
  AddGlyphs 的位图大小与字形、色标个数按不会回绕的算法核对(超了 BadLength)。
- **RANDR 对客户端基本只读,布局由宿主给**:每台显示器一个 CRTC / 输出 / 模式(`X11ServerOptions.Monitors` 或运行中的
  `SetScreenLayout`;每台都要落在根窗口里,至多 16 台,不合法当场抛 `ArgumentException`)。CRTC / 输出的编号按显示器的名字沿用
  (拔掉一台不会让别的编号前移、客户端手里的 ID 指向另一台),布局变了按 SelectInput 发变更事件,DPI 变了补发 ScreenChangeNotify,
  点时钟超出 32 位时取最大值。XINERAMA 报同一份布局,QueryScreens 与 GetScreenSize 同一个次序。`XMonitor.WorkArea`(去掉任务栏、Dock 之后
  可以摆窗口的部分)决定 `_NET_WORKAREA`:各台显示器在虚拟桌面边缘上让出来的部分从根窗口里扣掉(EWMH 只有一个工作区矩形,
  两台显示器之间的任务栏扣不出来),菜单、最大化、对话框据此避开任务栏。客户端改配置:SetScreenSize 给当前尺寸照样成功、别的尺寸回 BadValue;
  SetCrtcGamma、SetOutputPrimary 照常核对参数、静默接受、不生效(回 BadAccess 时,用 Xlib 默认错误处理的调色温程序、桌面会话会因此退出);
  其余(SetScreenConfig、SetCrtcConfig……)回 Failed 或 BadAccess —— rootless 模式下窗口摆在哪、显示器怎么排由宿主决定。
- **服务端兼任 XSETTINGS 管理器**:占有 `_XSETTINGS_S0`、发布 `Xft/DPI`、`Gdk/WindowScalingFactor` 等几项,
  并在根窗口发布 RESOURCE_MANAGER(`Xft.dpi`,Xft 与 Qt 从这里读)。真实桌面总有一个设置守护进程,
  GTK / Qt 启动时会去找;真正的守护进程来抢这个选区时照常让出。宿主换 DPI 时只替换 RESOURCE_MANAGER 里的 `Xft.dpi` / `antialias` /
  `hinting` / `hintstyle` / `rgba` 几项,用户 `xrdb -merge` 进去的其余资源(连同注释)原样留着。
- **服务端兼任窗口管理器的协议那一半**:启动时占有 `WM_S0`、占住根窗口的 SubstructureRedirect(见上),维护 `_NET_SUPPORTED`、
  `_NET_SUPPORTING_WM_CHECK`(检查窗口上的 `_NET_WM_NAME` 是 `X11ServerOptions.WindowManagerName`,默认 `LG3D`,来由见 §10)、
  `_NET_CLIENT_LIST`(按第一次映射的先后)、`_NET_ACTIVE_WINDOW`、`_NET_WORKAREA` 与顶层的 `WM_STATE` / `_NET_FRAME_EXTENTS` 等;根窗口 ClientMessage
  里的请求翻成 `XWindowManagerRequest` 交给宿主(§6),由宿主决定照不照办、办完用 `SetTopLevelStates` / `ChangeTopLevelStates` 写回。
  GTK3 的 HeaderBar、Qt 的无边框窗口都依赖这些属性存在。ICCCM / EWMH 的细节:
  - `WM_HINTS`、`WM_NORMAL_HINTS`、`_MOTIF_WM_HINTS` 解析全,只认 flags 里给了、属性里也真有的字段(15 个值的老格式照样认),尺寸、外框、进程号一律夹进合法范围;
    标题类属性按类型解码(STRING / UTF8_STRING / COMPOUND_TEXT,ICCCM §2.7.1)。
  - 从 Withdrawn 映射、initial_state 为 IconicState 时写 `_NET_WM_STATE_HIDDEN` 与 `WM_STATE = Iconic`(`xterm -iconic`);顶层 withdraw 时删掉
    `_NET_WM_STATE` 与 `_NET_WM_DESKTOP`(否则重新映射时带着过期的 Hidden / Focused);InputOnly 的顶层不写 `WM_STATE`、不进客户端列表。
  - 焦点进了 override-redirect 的弹层(菜单抓键盘)不算换了活动窗口,主窗口不会画成非活动的样子。
  - 关闭时对在 `WM_PROTOCOLS` 里声明了 `_NET_WM_PING` 的窗口随 `WM_DELETE_WINDOW` 发 ping,5 秒内没回就发 `XNotRespondingRequest`。
  - 宿主移动窗口时像真的移动一样发真实的 ConfigureNotify(根窗口上选了 SubstructureNotify 的客户端、Present 都收得到,指针所在的窗口重算),
    再补 ICCCM §4.1.5 的合成事件。
  - GTK 不开客户端阴影(`ClientSideShadows`,默认关)时在窗口内沿留一圈约 4 像素(乘 GTK 的缩放倍数)的缩放区,按下就发 `_NET_WM_MOVERESIZE`
    (四边与四角),宿主经 `XMoveResizeRequest` 开始原生的缩放 —— 无边框的自绘标题栏窗口照样能拖边缩放,只是缩放区窄(gtk3-widget-factory 实测)。
- **核心窗口请求照协议的细节**:DestroySubwindows 按堆叠次序从下到上销毁;CirculateWindow 按遮挡挑窗口(RaiseLowest 抬起被挡着的最低的,
  LowerHighest 压下挡着别人的最高的;遮挡按外框矩形算、不看 SHAPE,InputOnly 不算挡着别人),有 SubstructureRedirect 的改道者时只发 CirculateRequest,
  挪了之后重画;ReparentWindow 的自动重映射算发起方发的 MapWindow;ConfigureWindow 的 stack-mode 大于 4 回 BadValue,别的客户端选了
  ResizeRedirect 时改尺寸变成发给它的 ResizeRequest,父窗口的内区尺寸变了子窗口按各自的 win-gravity 挪并发 GravityNotify(Unmap 重力取消映射),
  TopIf / BottomIf / Opposite 按遮挡判断;发 VisibilityNotify —— 顶层各有原生窗口,宿主里谁挡着谁这边不知道,映射着就算完全露出,宿主最小化也不报
  FullyObscured(免得 xterm 之类停止重画之后,还原时又没有 Expose 来补);ShapeCombine 只按客户端给的偏移放源形状;SetCloseDownMode 的取值超出 0–2
  回 BadValue;ConvertSelection 的 property 不是 None 时核对原子(BadAtom)。顶层的 X 堆叠次序不跟原生窗口的 z 序走:注入的指针事件先在宿主
  指名的那个顶层里命中(指针落在它外面 —— 拖动时捕获着 —— 才从根按堆叠找),宿主最小化的顶层不参与从根开始的命中,`FocusTopLevel` 把顶层抬到普通顶层的最上面。
- **XKB 由核心键位表推出**:不单独维护一份 XKB 键位表,四个规范类型(外加 AltGr 层用的两个四级类型)、修饰键动作、SymInterpret、指示灯、键名都从
  核心表算出来;核心表一变(xmodmap、宿主 `SetKeymap`),XKB 跟着变并发 MapNotify。
  XKB 的 SetMap 把上传的键值(按 §17 的列序)与修饰键映射写回核心表,再照常推出;SetCompatMap、SetNames 等其余改表请求不支持。
  ALPHABETIC 认所有有大小写之分的字母(拉丁之外还有西里尔、希腊、Latin-2 / 3 / 4 / 9 与 Unicode 键值,大小写按码位判 —— 宿主给的俄文、希腊文布局正是
  Unicode 键值);一组只有一个键值、而它是有大小写之分的字母时按「小写、大写」两级展开(核心协议第 5 节,xmodmap 常写单列);
  FOUR_LEVEL_ALPHABETIC 有六条映射(Shift + Lock + Mod5 → 第三级)。锁存的修饰键用一次即解除。
- **抓取、焦点与指针照协议**:
  - 时间戳:服务端记 last-pointer-grab / last-keyboard-grab time(主动抓取取请求里的时间,被动与自动抓取取激活它的事件的时间);
    Grab* 的时间早于上次抓取或晚于当前时间回 InvalidTime,设备被别的客户端的抓取冻着回 Frozen;过期的 Ungrab*、ChangeActivePointerGrab、
    AllowEvents 不生效。一个设备可以同时被几个抓取冻着,全部放开才继续。
  - 被动抓取按「有没有共同的组合」查冲突(AnyModifier / AnyButton / AnyKey 等于对所有组合各登记一次):核心的冲突整个请求回 BadAccess,
    XI2 把冲突的组合逐个列进回复(AlreadyGrabbed)。Ungrab 只盖住一个抓取的一部分时减掉那一部分(每个抓取至多记 1024 个减掉的组合,多了 BadAlloc);
    核心的 Ungrab 不动 XI2 的被动抓取,反之亦然;修饰组合、事件掩码、detail 越界回 BadValue。重放(ReplayPointer / ReplayKeyboard)的按下按事件之前的状态报。
  - 自动抓取(按钮按下)的激活与解除同样发 Grab / Ungrab 模式的 crossing:Grab 的在 ButtonPress 之前、只报给抓取方,Ungrab 的在 ButtonRelease 之后;
    所有按钮都松开才解除(6 号以上的按钮不在 state 里,原先松开 6 号就解除了)。
  - 焦点:SetInputFocus(与 XI 的 SetDeviceFocus、XISetFocus)按时间戳生效,早于 last-focus-change time 或晚于当前时间的不生效 —— 经 SSH 迟到的
    SetInputFocus 不再把焦点拉回用户已经离开的窗口;宿主的 `FocusTopLevel` 也推进这个时间,WM_TAKE_FOCUS 带同一个时间。RevertToParent 一路退到根时
    焦点是根(即 PointerRoot);XISetFocus 接受 PointerRoot,焦点窗口不可见时按 RevertToParent 退。客户端把焦点挪到另一个顶层时交给宿主一个 `XFocusRequest`。
  - WarpPointer:按源窗口与源矩形生效(宽高为 0 按协议换算),结果夹在根窗口里,有带 confine-to 的指针抓取时只挪到 confine-to 最近的边上;
    冻结时排队;核心与 XI 共用一份实现。宿主不知道 Warp:服务端的指针与真实光标会分叉,confine-to 也只约束 Warp、不约束用户的鼠标 —— 要宿主配合,还没做。
  - 指针离开所有顶层时位置留着最后一次的,所在窗口算根(child 为 None),QueryPointer 与事件里的根坐标都是最后的位置。
- **XInput2 的设备拓扑可改,但不是完整的多指针**:初始是主指针 2 / 主键盘 3 各挂一个从设备(4、5);XIChangeHierarchy
  可以增删主设备、把从设备挂到别的主设备或让它浮动(浮动时只报从设备的 XI2 事件、不产生核心事件)。
  指针位置、焦点与抓取仍是一份 —— 宿主只有一套物理输入,完整的 MPX 没有用处。XI2 事件与核心事件走同一条传播路径,
  同一个窗口上选了核心的收核心、选了 XI2 的收 XI2;核心的 do-not-propagate 只截断核心事件,不截断 XI2。
  XI2 的事件选择按(客户端, 窗口, deviceid)分开存,投递时按当前设备层级合成主 / 从掩码、取并集,层级变了重算;选 XIAllDevices 的客户端主从各收一份。
  根窗口尺寸变了发 XI_DeviceChanged;XIQueryPointer 与核心 QueryPointer 一样重置移动提示;XI_RawMotion 的值是指针设备的绝对位置
  (与 XIQueryDevice 声明的 Abs X / Abs Y 轴一致),只在宿主或 XTEST 挪了它时发,Warp 不发 —— 靠「原始移动 + Warp 回中心」做相对鼠标的程序因此不再抖。
  被动按钮 / 按键抓取激活时的 crossing 是 Grab 模式(XI 2.2 如此;XIPassiveGrabNotify 只用于 Enter / FocusIn 类的被动抓取,那一类没有实现)。
  XI 的设备属性按核心属性的规则校验(BadAtom / BadValue / BadLength / BadMatch)、按本机序存、计入内存账,增删改时发 XI_PropertyEvent。
- **MIT-SHM 只给本机**:只在 Linux 上注册,并且只对经 Unix 套接字连进来、与服务端在同一个 IPC 命名空间里的客户端可见 —— 远端经 SSH 来的客户端给的
  shmid 在这台机器上毫无意义;把 `/tmp/.X11-unix` 挂进容器后,容器里的客户端给的 shmid 指的却是宿主这边的段(比较 SO_PEERCRED 的 pid 的
  `/proc/<pid>/ns/ipc` 与自己的,不同或核对不了都隐藏)。段的大小与属主逐行查 `/proc/sysvipc/shm`(找到就停),对端 uid 经 SO_PEERCRED 取得,
  不是属主 / 创建者且权限没对其他人开放就 BadAccess(否则本机客户端可以借服务端之手读写别人的共享内存);段被别的客户端按 XID 引用时
  (不是 Attach 它的那一个)再核一次;同一个客户端一秒内 Attach 失败满 16 次之后,这一秒里的 Attach 直接回 BadAccess、不再读表。
  QueryVersion 回服务端的有效 uid / gid。共享像素图与 1.2 的 fd 传递不做。
- **GLX 两条路**:Mesa 在没有 DRI3 / DRI2 时默认走 drisw —— 客户端用 llvmpipe 渲染(GL 4.5)、经 PutImage 送像素,
  服务端只要把配置、上下文、可绘对象登记好;远端经 SSH 转发来的程序同样可用。扩展串声明 GLX_ARB_create_context 与 GLX_ARB_create_context_profile:
  CreateContextAttribsARB(请求 34)对直接上下文只登记(版本、标志与 profile 由客户端的驱动处理),要核心 profile(3.2+)的程序
  (GLFW / SDL / Qt 的 CoreProfile、Blender……)因此在直接路径上拿得到上下文;SetClientInfoARB / SetClientInfo2ARB(33 / 35)收下不用。
  强制 `LIBGL_ALWAYS_INDIRECT` 时由 `Gl/` 的软件 GL 执行固定功能管线的一个子集,版本如实报 1.1、只有兼容 profile:间接上下文要 3.2 起的核心 profile
  回 GLXBadProfileARB,版本高于 1.1 回 GLXBadFBConfig,没定义的版本或 1.x 带前向兼容回 BadMatch,不认识的属性 / 标志位回 BadValue。
  - 没有实现的:3D 纹理、累积缓冲、反馈模式、mipmap LOD、像素传输的缩放 / 偏置与 PixelMap、深度 / 模板 / 颜色索引格式的 DrawPixels
    与 CopyPixels、点 / 线 / 多边形平滑、Hint。反馈模式每个上下文第一次用到时记一行日志(GL 的行为不变,不报 GL 错误)。
    选择模式、线 / 多边形点画与求值器 2026-10-10 补上了(见下面「2026-10-10 补的功能」)。
    边标记(GLU 镶嵌器的内部对角线在 PolygonMode(LINE) 下不画)、GL_CLAMP 配 LINEAR 时与边框色混合、GL_EXT_texture_object 的厂商私有请求
    (11–14,按核心的纹理命令处理)、PolygonOffsetEXT 的 bias 按深度范围单位,以及扩展串里声明的 GL_EXT_abgr 都已实现。
  - GL 错误照规范:Enable / Disable / IsEnabled 只认 1.1 的开关与声明了的扩展的开关,别的记 INVALID_ENUM;状态命令的非法枚举记 INVALID_ENUM、
    状态不变;Begin / End 之间只许指定顶点属性(以及 CallList(s) 与 End),别的记 INVALID_OPERATION、不执行。DrawArrays 每个顶点的数据按 ARRAY_INFO
    出现的顺序读(与 Mesa 的间接 GLX 一致;编码规范 VERTEX_DATA 一节列的是固定次序,而 ARRAY_INFO 的列表本身无序)。GenLists / GenTextures 的
    新名字接在用过的最大名字之后。绑定的纹理名已不在共享组里时按「删掉即退回 0」处理。
  - 上限与内存:一个 GLX 表面最多 4096 × 4096 像素 —— 窗口更大时(8K 屏、跨屏最大化)表面夹到上限、只盖住窗口左下的那一块(GL 窗口坐标原点在左下),
    第一次夹的时候记一行日志,不再回 BadAlloc;一个请求里显示列表展开执行的命令数上限 400 万(列表互相调用会指数级展开,每次 CallList 都计入),
    渲染命令另按工作量扣预算(§5);一个图元 2¹⁹ 个顶点,线宽与点大小 64,ReadPixels / GetTexImage 的回复不超过输出积压上限的一半。
    GL 的内存进每客户端的内存账:间接上下文本身按 64 KB 记,它的默认纹理与图元缓冲记在上下文所属客户端名下;共享组的列表与有名字的纹理记在建组的客户端名下
    (每个共享组另有显示列表 64 MB、纹理 65536 个 / 256 MB 的上限),每个显示列表另记 64 字节(空列表也占账,取代原先 65536 个列表名的上限);
    表面按像素记在第一个要它的客户端名下(双缓冲每像素 13 字节、单缓冲 9 字节);拼到一半的 RenderLarge 按声明的长度记。上下文离开资源表且不再是当前时
    释放它的 GL 对象并退账。
  - 上屏:单缓冲的前缓冲在每个 Render 请求之后、双缓冲在交换时,只在画过的范围里逐行与窗口里实际的像素比,只写、只记不同的那一段的损伤 ——
    不整窗拷,也不盖掉窗口里别处 X 画的内容;窗口被核心绘图改过的地方,下次交换照样补回来。
  - 可绘对象:表面随可绘对象与客户端释放(同一个 XID 被重用时是新表面),GLX 像素图的表面随 FreePixmap 释放。当前可绘对象没了时 Render 与非渲染命令
    照常执行(画不到任何地方),WaitGL / WaitX / 带标签的 SwapBuffers / CopyContext / UseXFont 回 GLXBadCurrentWindow / GLXBadCurrentDrawable。
    GLX 1.2 写法(X 窗口直接当可绘对象)的表面取窗口视觉的那条配置。GetVisualConfigs 为两个视觉各发布单缓冲与双缓冲两条配置,间接 GLX 也选得到单缓冲视觉。
- **Present 与 SYNC 按规范排队、计时**:
  - Present:PresentPixmap 等 wait-fence 触发(或被销毁),再按 target-msc / divisor / remainder 选帧 —— MSC 按宿主报来的帧节拍走(`NotifyHostFrame`,2026-10-10;宿主不报时按 60 Hz 从服务端时钟推算),target 已过而
    divisor 为 0 时立即,PresentOptionUST 时三者按微秒换算成帧;到点的按请求先后呈现,同一窗口上更早的、还没呈现的按 CompleteModeSkip 了结,
    不拿旧内容盖掉新的。呈现之前一直持有像素图(规范允许请求之后立刻 FreePixmap)。idle-fence / wait-fence 不是栅栏回 SYNC 的 BadFence。
    Present 与 DAMAGE 的 QueryVersion 回服务端支持的、但不高于客户端要的版本。
  - SYNC:触发器的 value-type 与 wait-value 原样保留,初始化时算出的测试值另存(QueryAlarm 报的仍是 Relative),Relative 而 counter 为 None 回 BadMatch,
    测试值超出 INT64 回 BadValue;CreateAlarm 不给 test-type 时默认 PositiveComparison(规范的默认值表);比较型报警器触发后一步算出推进到哪,
    只在推进会超出 INT64 时停用;AlarmNotify 报更新之后的状态;DestroyCounter、或创建计数器的客户端断开时,挂着它的报警器发一条 state 为
    Inactive 的 AlarmNotify;Await 按 event-threshold 给每个触发器发 CounterNotify(请求执行时就成立也照样查);空的 Await 回 BadValue,
    空的 AwaitFence 立即放行,counter 为 None 的触发器恒为真,栅栏被销毁时放行等它的 AwaitFence;SERVERTIME / IDLETIME 上已经越过的正向跨越
    不再排计时器(原先每毫秒醒一次)。
- **其余扩展**:DAMAGE —— DamageSubtract 之后剩下的损伤按级别重报(Raw / Delta 逐块,BoundingBox / NonEmpty 报外接矩形),Damage 随 DamageDestroy 或
  客户端的资源销毁释放;绘图时只看本顶层里挂了 Damage 的窗口。DOUBLE-BUFFER —— Background 交换按窗口背景铺(背景像素图按原点平铺、ParentRelative 沿祖先找、
  None 不动),同一窗口在列表里出现两次回 BadMatch。Composite —— NameWindowPixmap 给的像素图在顶层改尺寸、重新映射、销毁之后保持原样
  (从那时起独占旧缓冲);还共享着缓冲时往里画同时记成顶层的损伤,宿主收得到。XFIXES —— GetCursorImage / GetCursorImageAndName 给出指针处光标的
  真实图像与热点(含 cursor 字体的字形光标;隐形指针没有图像,仍是 1×1 透明);ChangeCursor / ChangeCursorByName 生效(正在用的窗口跟着变,
  宿主收到 CursorChanged、登记者收到 CursorNotify);DestroyPointerBarrier 只认指针屏障。扩展的清理钩子分「连接断开」(事件选择、计时器、
  等着的请求)与「客户端的资源销毁」(Retain 模式下晚于断开,KillClient 销毁留下的资源时才调)两种。
- **2026-10-10 补的功能(xs_plan F4–F28)**:
  - **本机输入法上屏(F5 第一步)**:`InjectText(text)`。X 程序只认键码:每个字找一个空着的键码,把键值改成这个字的 Unicode 键值
    (协议附录 A:Latin-1 是码位本身,其余是码位 + 0x01000000)再按下、松开;键位表里本来就有、此刻不按修饰键(NumLock 除外)就打得出来的字直接按那个键。
    - **一批一次通知**:一段字先把新借的键码都改好,再发一次核心 MappingNotify(覆盖这几个键码)与一次只报这一段键值的 XKB MapNotify
      (changed 只有 KeySyms),然后依次按下、松开 —— 客户端每批只重取一次键位表。
    - **改过的键码不改回去**:客户端收到通知之后才去重新取键位表,取到的是请求到达那一刻的;同一个字再输入不再改、不发通知。空键码用完才挪用
      最久没用过的,而且它最后一次用过之后要空闲 3 秒(`TextKeyReuseMilliseconds`,远长于经 SSH 的一个来回加重取键位表)、这一批里没用过、
      键盘没被同步抓取冻结着;挪不了时这段字等着,每 50 毫秒再看一次。一个可借的键码都没有(客户端把每个键码都占了)时剩下的字丢掉、记一行日志。
    - **按顺序**:等的时候后来的 `InjectText` 与宿主的 `InjectKey` 排在后面(XTEST 与指针不排)。
    - **大小写**:借来的键两级同一个键值、XKB 类型是 ALPHABETIC(Lock 算作被这个键消耗),CapsLock 开着时 Xlib / xkbcommon 不把「é」转成「É」;
      只认核心协议的老客户端仍按协议第 5 节转大写。
    - **非受信客户端**(SECURITY「Keyboard Security」):借来的键码的键值就是用户刚输入的字。键盘事件不送到非受信客户端时,改键位表的通知不发给
      它们,GetKeyboardMapping / XI 的 GetDeviceKeyMapping / XKB GetMap 里这些键码是 NoSymbol;只有输入给非受信程序时用到的键码才让它们看见
      (这时补发只给非受信客户端的通知)。
    - 换行(CR LF 算一个)按 Return、制表按 Tab,其余控制字符不输入;一次至多 4096 个 UTF-16 码元。键盘焦点所在的程序经 XIM 连着时,
      字不借键码、整段走 XIM_COMMIT(见下一条);轮到这一段时才决定,排在前面还在等键码的字先输完。
    - 2026-10-10 核对之前:每个新字一轮通知;空闲 200 毫秒就挪用(慢链路上超过空键码数的一段字,客户端为前面的字重取键位表时那个键码已经改成了后面的字);
      等的时候宿主的按键插到前一段字中间;借来的键推成 ONE_LEVEL,CapsLock 下转大写;非受信客户端收得到通知、读得到键值。
  - **本机输入法的 XIM 桥(F5 第二步,决策 Q1)**:`X11ServerOptions.InputMethodName`(宿主给 `velashell`)非 null 时服务端当 XIM 输入法服务端。
    - **预连接**(The Input Method Protocol「Default Preconnection Convention」):占住选区 `@server=名字`(属主是服务端自己的窗口),把这个原子加进
      根窗口的 `XIM_SERVERS`(已有的照留);转换目标 `LOCALES` 回 `@locale=` 加一长串区域名(C、POSIX、全部 ISO 639-1 语言代码、常见的「语言_地区」
      带不带 `.UTF-8`)—— 组好的字按协商的编码交过去,与程序的区域无关,所以尽量都认;`TRANSPORT` 回 `@transport=X/`。两个回答的属性类型就是目标原子本身。
      `LOCALES` / `TRANSPORT` 两个原子在开 XIM 时就建好:Xlib 先用「只查不建」的 InternAtom 看它们在不在,不在就当没有输入法服务端
      (互操作用例里 xterm 因此一直不连,单元测试测不出来)。
    - **X 传输**(The XIM Transport Specification):程序把 `_XIM_XCONNECT`(格式 32)发到选区属主窗口,服务端为这条连接建一个服务端通信窗口,
      回 `_XIM_XCONNECT`:窗口、传输版本 0.2、分界长度 20。之后不超过 20 字节的包一条 `_XIM_PROTOCOL`(格式 8,补 0);更长的写成收方通信窗口上的
      属性(类型 STRING、格式 8),再发一条格式 32 的 `_XIM_PROTOCOL` 告诉长度与属性名,收方读的时候删掉;多条 ClientMessage(`_XIM_MOREDATA` …
      `_XIM_PROTOCOL`)也认。两份文档的正文与表格互相矛盾(正文说写在发方自己的窗口上,表 1.7 / D.6 说写在 IMS 窗口上;表 1.8 写格式 8 却用 data.l),
      以 Input Method Protocol 附录 D 的表为准,读的时候发方的窗口也看一眼。服务端往程序窗口上写属性时轮流用 64 个名字,上一包没读走就换下一个。
      非受信客户端也能用:它写服务端通信窗口上的属性、往 XIM 窗口发 ClientMessage 不按 SECURITY 拦(窗口只属于那条连接)。
    - **协议**:XIM_CONNECT(按它的 byte order 定这条连接的字节序;不做认证)、OPEN(回全部 IM / IC 属性:IM 只有 `queryInputStyle`;
      IC 有 `inputStyle`、`clientWindow`、`focusWindow`、`filterEvents`、`preeditAttributes` / `statusAttributes`(嵌套)、`spotLocation`、`lineSpace`、
      `fontSet`、`area`、`areaNeeded`、颜色与光标、`separatorofNestedList`、`resetState`、`preeditState`)、ENCODING_NEGOTIATION(列表里有 COMPOUND_TEXT 挑它,
      没有挑 UTF-8,都没有回 -1)、QUERY_EXTENSION(没有扩展)、GET / SET_IM_VALUES、CREATE / DESTROY_IC、SET / GET_IC_VALUES(嵌套的那一层到分隔符为止)、
      SET / UNSET_IC_FOCUS、RESET_IC、SYNC、TRIGGER_NOTIFY、CLOSE、DISCONNECT;不认识的回 XIM_ERROR BadProtocol,第一个包不是 CONNECT 也是。
      支持的风格:XIMPreeditCallbacks(on-the-spot,配 StatusNothing 或 StatusCallbacks)、XIMPreeditPosition(over-the-spot)、XIMPreeditNothing(根窗口风格)、
      XIMPreeditNone;不支持 Area(off-the-spot:要协商几何,宿主画不进程序的窗口),建的时候回 BadStyle。状态区一律不画。
    - **按键不经 XIM**:每个输入上下文建好就发 XIM_SET_EVENT_MASK(转发掩码、同步掩码都是 0),`filterEvents` 回 KeyPressMask:按键由程序按它自己的键位表解释,
      组字在宿主那边(本机输入法在本机就吃掉了组字用的键)—— 要是按键经 SSH 转到服务端再转回去,每个键多一个往返。程序万一还是转了事件过来,原样转回、
      要求同步的回 SYNC_REPLY。
    - **接受输入的输入上下文**:键盘焦点所在的顶层(焦点是 PointerRoot 时看指针所在的顶层)里报了焦点的那个,有几个时取最后报的;X 的焦点变了、
      程序报 / 撤焦点、改插入点、建 / 销毁输入上下文、通信窗口或焦点窗口销毁时重挑。报给宿主的 `XInputMethodFocus`:所在的顶层(单窗口模式下是屏幕窗口)、
      `ClientDrawsPreedit`(风格含 XIMPreeditCallbacks)、`Cursor`(XNSpotLocation 是预编辑第一个字的基线起点,换成顶层内区里一个宽 1、高一行的竖条:
      行高用程序给的 XNLineSpace,没有就按 DPI 估,96 dpi 时 16;上沿是基线往上八成行高;程序没报插入点时为 null)。相同的不重复报。
    - **上屏**:`InjectText` 的一段可见字以 XIM_COMMIT(XLookupChars、不要求同步)交给接受输入的输入上下文,按协商的编码 —— COMPOUND_TEXT 时 Latin-1 原样、
      其余放进 UTF-8 段(`ESC % G … ESC % @`);换行、制表照常按键(回车 / 制表键本来就在键位表里),别的控制字符不输入。先擦掉程序那边的预编辑再上屏。
    - **预编辑**:`InjectPreedit(text, caret)` 只交给 on-the-spot 的输入上下文:没在显示时先 XIM_PREEDIT_START,再 XIM_PREEDIT_DRAW(caret 与 chg_first /
      chg_length 按字数 —— Unicode 码位 —— 算,替换上一次的整段;每个字一个 XIMUnderline);空串时 DRAW 删掉整段(no string | no feedback)再 XIM_PREEDIT_DONE。
      接受输入的输入上下文换了、撤了焦点、RESET_IC 时同样收掉。RESET_IC 回空的预编辑(宿主的输入法自己还在组字)。
    - **覆盖面**:Xlib 的 XIM 客户端 —— xterm、Emacs、Java(AWT / Swing)、Tk、Motif,以及用 GTK 2/3 的 xim 输入模块的程序(Firefox、Chromium 跟着 GTK);
      服务端在 XSETTINGS 里写 `Gtk/IMModule = xim`,没设 `GTK_IM_MODULE` 的 GTK 2/3 程序就走它(单窗口模式下服务端不当 XSETTINGS 管理器,没有这一项)。
      Qt 5 / 6 与 GTK 4 没有 XIM,仍走借键码上屏。程序要在 `XMODIFIERS=@im=velashell` 下启动 —— Xlib 只在写了 `@im=` 时才找输入法服务端,
      没写就用它自己的组合键处理;宿主在连接后的静默注入里设上它(见宿主仓库 `plan.md`)。
    - 上限:256 条 XIM 连接、每条连接 1024 个输入上下文,多条 ClientMessage 拼一个包至多攒到 XIM 包长的上限(约 256 KB),超了当坏数据收掉这条连接。
  - **平滑滚动(F6)**:指针设备多两个相对轴 Rel Horiz Scroll / Rel Vert Scroll 与对应的 ScrollClass(XI 2.1,increment 1.0 = 一格);
    `InjectScroll(window, x, y, dx, dy)` 发带滚动轴的 Motion 与 RawMotion,攒够一格再模拟一次按钮 4–7(XI2 那份带 PointerEmulated);
    反过来设备给的滚轮按钮也给用滚动轴的 XI2 客户端发一格滚动。
  - **剪贴板带上 HTML 与图片,服务端当剪贴板管理器(F14 / F15)**:`XClipboardContent`(文本、HTML、PNG);`SetClipboard(content)`
    (`SetClipboardText` 是它的文本特例,图片上限 `MaxClipboardImageBytes` 32 MB)与回调 `ClipboardContentChanged`(默认实现有文本时转给
    `ClipboardChanged`)。服务端当属主时 TARGETS 只列有的格式;当请求方时先要 TARGETS,再依次要文本、`text/html`、`image/png`,取完一起交给宿主。
    开着剪贴板同步时占有 CLIPBOARD_MANAGER(freedesktop Clipboard Manager Specification):SAVE_TARGETS 在这份剪贴板取完之前不回答,
    取到了回成功,存不了回 None;属主退出后照旧由服务端以宿主的身份接管。
  - **指针 Warp 与 confine-to 交给宿主(F8)**:回调 `PointerWarped(rootX, rootY)` —— 只在发请求的客户端此刻抓着指针、且不是被限制的转发程序时报,
    同一批里只交最后一次;`PointerConfinementChanged(area)`:带 confine-to 的指针抓取开始时报那个窗口的内区(根坐标),解除时报 null。
  - **屏保协作(F9)**:MIT-SCREEN-SAVER 的 Suspend 按客户端计数(语义照 libXss 手册 XScreenSaverSuspend(3):成对调用、别的客户端恢复不了、
    断开即作废),任何一个挂着时回调 `ScreenSaverSuspensionChanged(true)`,都恢复时 false;ForceScreenSaver(Reset) 至多每 5 秒回调一次 `ScreenSaverReset`。
  - **`_NET_WM_SYNC_REQUEST`(F10)**:宿主改一个顶层的尺寸时,声明了它、有 SYNC 计数器的窗口先收到同步请求再收到 ConfigureNotify;
    等的期间 `XTopLevelWindow.AwaitingRedraw` 为真,客户端把计数器推到序号或 300 毫秒到点时回调 `TopLevelRedrawn`。
  - **合成管理器(F11)**:`X11ServerOptions.CompositingManager`(默认关)打开时占住 `_NET_WM_CM_S0`,GTK、Qt、Electron 才用 ARGB 视觉画圆角、阴影与透明窗口。
    关着时深度 32 的窗口照样按不透明交给宿主(快照的 `HasAlpha` 为假)—— 没有合成管理器时 X 显示它们不看 alpha;要了 8 位 alpha 的 GL 程序
    (GLFW 默认就要,只拿得到 ARGB 视觉)清屏的 alpha 是 0,2026-10-10 核对之前整个窗口透出后面的东西,还白付系统合成的开销。
    远端跑了个合成器(`xfwm4 --replace` 之类)接走这个选区又退出、或者谁把属主设成了 None 时,服务端接回来,按 ICCCM §2.8 向根窗口广播
    MANAGER(选了 StructureNotify 的客户端收到)。`ClientSideShadows` 仍默认关(Windows 上透明的阴影区照样接住鼠标)。
  - **X-Resource 的 LocalClientPid(F28)**:经 Unix 套接字连进来的客户端记下对端 pid(Linux 经 SO_PEERCRED,macOS 经 LOCAL_PEERPID),
    QueryClientIds 回给本身也是本机客户端的请求方。
  - **计量仪表(F27)**:`XServerMetrics` 交出仪表源名 `VelaShell.XServer`:`clients.active`、`connections.refused`(reason:authorization、
    too_many_clients、too_many_setups、bad_setup)、`clients.disconnected`(killed、output_backlog)、`protocol.errors`(code 是错误名)、
    `work.duration`(毫秒直方图,没人订阅时不记)、`work.stalled`。看门狗见 §5。
  - **GLX 多重采样与 GLX_EXT_libglvnd(F22)**:两个 4 倍多重采样的双缓冲配置(24 位 `0x105`、32 位 ARGB `0x106`)—— 直接渲染由客户端的 Mesa 真的多重采样,
    间接上下文在这种配置上照常单采样画、如实报 SAMPLE_BUFFERS 0;`QueryServerString(GLX_VENDOR_NAMES_EXT)` 报 `mesa`。
  - **间接 GL 的选择模式、点画与求值器(F23)**:名字栈与 SelectBuffer、选择模式下图元走到裁剪为止的命中记录(溢出时 RenderMode 返回 −1);
    LineStipple / PolygonStipple;Map1 / Map2、MapGrid、EvalCoord / EvalMesh / EvalPoint、GetMap、AUTO_NORMAL(GLUT 的茶壶、GLU 的 NURBS 画得出来)。
  - **Present 跟宿主的帧节拍(F25)**:`NotifyHostFrame()` 由宿主的合成器每帧调一次,帧间隔取最近 32 次里最短的(限 20–500 Hz),报帧时当场做掉
    到了目标帧的 PresentPixmap;回调 `FrameClockWanted(bool)` 告诉宿主什么时候要逐帧报(有 NotifyMSC 或排队的 PresentPixmap 时)。只有一个全局帧时钟。
  - **RENDER 的 SIMD 快路径(F24)**:纯色过遮罩 OVER、图像 OVER、通用路径的 Over / Add 用 `Vector128` 一次算 4 个像素,与标量版逐位一致;
    双线性取样与 GL 光栅化没动。
  - **宿主:Linux / macOS 默认只开 Unix 套接字(F4,决策 Q4)**:设置「也开 TCP 端口」默认关;Windows 上一直开(WSL、Cygwin 的程序只会走 TCP)。
    TCP 端口只听 127.0.0.1(桥接网络里的容器连不到,Linux 上把 `/tmp/.X11-unix` 挂进去用 `DISPLAY=:N`)。宿主报的显示地址按实际开着的监听给
    (`X11Server.Display`:Linux / macOS 上 `:N`,只有 TCP 时 `localhost:N.0`)。Unix 套接字建不起来、TCP 又没开时(macOS 上 `/tmp/.X11-unix`
    归别的用户、属主不可信)照样起一个只经连接器喂流的服务端:SSH 的 X11 转发照常,本机程序连不上,启动结果带一条提示(`XServerStartResult.Warning`)——
    2026-10-10 核对之前整个服务端起不来,连 SSH 转发一并失败。自动选号时 `/tmp/.X{N}-lock` 由活着的进程持着(或读不出是谁)的号算占用;
    内置引擎停了之后老会话的通道不再退到本机 TCP(没有 VcXsrv 的平台上环回 `6000+N` 不会是我们的服务端)。
- **单窗口模式(`X11ServerOptions.Rootful`,默认关;决策见 §10 的 2026-10-10 一条)**:整个根窗口经 `X11Server.Screen` 作为一个顶层交给宿主
  (快照在 (0, 0)、与根窗口一样大),宿主开一个原生窗口显示整块桌面;其余顶层不再单独交给宿主(不发 `TopLevelMapped` 这些,也不提窗口管理器的请求)。
  服务端不再当窗口管理器:不占 `WM_S0`、不写 `_NET_SUPPORTED` / `_NET_SUPPORTING_WM_CHECK` / 客户端列表 / `_NET_WORKAREA`,
  根窗口的 SubstructureRedirect 让给客户端(rootless 下回 BadAccess),发给根窗口的 EWMH 请求只照常投递给选了它的窗口管理器;
  也不当 XSETTINGS 管理器与托盘(`SystemTray` 不起作用),那些归远端桌面自己的守护进程与面板。焦点归远端的窗口管理器(没有时是 PointerRoot),
  宿主窗口得失焦点不改 X 的焦点。
  - **画面**:顶层仍各有各的缓冲,绘图路径不变。每批执行完、放锁之前,把这一批画过的顶层区域,以及顶层的映射 / 位置 / 尺寸 / 形状 / 边框与
    根窗口背景的变化换成屏幕上的区域(只是堆叠次序变了时,只取相互重叠、上下颠倒了的两个窗口的交集),在根窗口的缓冲里重拼:先铺根窗口的背景
    (像素图按根窗口原点平铺,没有背景是黑的),再按堆叠次序从下往上画映射着的顶层 —— 边框(边框像素图按窗口内区原点平铺)与内区,按边界形状裁;
    深度 32 的在有合成管理器(远端的合成器占着 `_NET_WM_CM_S0`)时按预乘 over 叠上去,没有时与别的窗口一样整块盖上(X 不看 alpha)。
    拼好的区域经 `TopLevelDamaged(屏幕, 矩形)` 交给宿主。GetImage 读根窗口拿到的就是拼好的屏幕(连背景)。
  - **宿主的动作**:指针、拖放的注入坐标就是根坐标(从根往下找落点);`ResizeTopLevel(屏幕, 宽, 高)` 把屏幕改成这么大(一台显示器覆盖全部,
    客户端收到根窗口的 ConfigureNotify 与 RANDR 通知);移动、关闭、改状态、外框宽、给焦点对屏幕句柄不起作用。光标一律报在屏幕句柄上。
  - **没做的**:根窗口本身的绘图(直接画在根上的程序,如 xroach)不显示,只显示背景;远端合成器(xfwm4 开着合成)画在 Composite 叠加窗口上的
    阴影等效果不显示 —— 画面由服务端自己拼,窗口照样看得到。
- **系统托盘(`X11ServerOptions.SystemTray`,默认关)**:开着时服务端占住 `_NET_SYSTEM_TRAY_S0`(属主是服务端自己的选区窗口,上面写
  `_NET_SYSTEM_TRAY_ORIENTATION` = 水平、`_NET_SYSTEM_TRAY_VISUAL` = 默认视觉),当 freedesktop System Tray Protocol 0.3 的托盘管理器。
  X 程序发 `SYSTEM_TRAY_REQUEST_DOCK` 时,服务端建一个嵌入窗口(服务端自己的顶层,override-redirect,边长 `SystemTrayIconSize`,默认 24,
  摆在屏幕右下角 —— 程序按图标的根坐标弹菜单),按 XEmbed 0.5 把图标窗口 reparent 进去撑满、按 `_XEMBED_INFO` 的 XEMBED_MAPPED 映射
  (之后跟着这一位映射 / 取消映射;没有这个属性的老程序当作要映射)、发 `XEMBED_EMBEDDED_NOTIFY`(data1 = 嵌入窗口,版本 0)。嵌入窗口映射时
  不当普通顶层交给宿主,而是 `SystemTrayIconAdded(句柄, 名字)`(名字取图标的 `_NET_WM_NAME`,退到按类型解码的 WM_NAME、WM_CLASS,
  与窗口标题一样限长、去掉控制字符与双向排版控制符;快照的 `ClientLabel` 是图标所属连接的标签,宿主据此在提示里标出来源):宿主照常读它的像素、
  收损伤、往里注入指针。图标窗口销毁、被程序 reparent 走(规范的结束方式)、程序断开时收掉嵌入窗口(`SystemTrayIconRemoved`)。至多 64 个图标;
  气泡消息(BEGIN / CANCEL_MESSAGE)收下不显示。默认关是因为有了托盘程序会「关闭到托盘」,宿主不显示的话窗口就找不回来。
- **拖放(XDND,宿主 → X)**:本机的文本、文件拖进 X 窗口时,服务端替宿主扮演 freedesktop XDND 第 5 版的**源**(源窗口是服务端自己的选区窗口):
  宿主每动一下报一次位置,服务端从指针下最深的窗口往上找第一个设了 `XdndAware`(版本 ≥ 3,取两边的小者)的窗口当目标,它设了有效的 `XdndProxy`
  (代理自己的 `XdndProxy` 指向自己)时消息投给代理;换目标时先 `XdndLeave` 旧的、再 `XdndEnter` 新的(多于三种类型时置位、目标读源窗口的
  `XdndTypeList`),然后 `XdndPosition`(根坐标、服务端时间、`XdndActionCopy`)。发了位置就等 `XdndStatus`,等的期间又动了只记最新的一个,
  状态回来再补发;`IsDragAccepted` 是最近一次状态说的「接受」。松手时先按最后的位置再报一次、等最后一条 `XdndStatus`(至多 3 秒,等不到就
  `XdndLeave`),接受就 `XdndDrop`,不接受、或放在没有目标的地方就 `XdndLeave`。目标经 `XdndSelection`(服务端占有,全显示共享)取数据:
  `TARGETS`、`TIMESTAMP` 与宿主交来的类型(类型就是目标原子;`STRING` 回 STRING、`TEXT` 回 UTF8_STRING),太大的走 INCR;还没放下时只答得出
  `TARGETS`。目标回 `XdndFinished`、下一次拖放开始或放下一分钟之后,数据丢掉。X 程序之间的拖放只靠核心协议,不经过这里;X 程序拖出到本机程序
  见下一条。
- **拖出(XDND,X → 宿主)**:`X11ServerOptions.AcceptOutgoingDrags`(宿主在多窗口模式下打开;单窗口模式下根窗口是远端桌面的,不接)。
  - **目标在哪**:按第 4 版起「拖到根窗口」的约定,根窗口的 `XdndProxy` 指向服务端自己的一个代理窗口(它的 `XdndProxy` 指向自己、`XdndAware` = 5);
    指针拖到所有 X 窗口以外时(rootless 下那里是本机桌面或本机程序),看根窗口 `XdndProxy` 的源(GTK 这一类)把消息投给代理,窗口字段是根窗口。
    Java 的 AWT 只把目标找在「指针下面那个根窗口的子窗口」上(MotionNotify 的 child 为空时根本不找),所以 X 程序占了 `XdndSelection`(开始拖了)时,
    服务端把代理窗口垫到根窗口子窗口的最底下、盖满根窗口,带 `WM_STATE`(Java 只对这样的顶层找 `XdndAware`):那片空白就有了一个接拖放的顶层,
    真的 X 窗口照样在它上面。悄悄垫上、撤下(不发 CreateNotify / MapNotify),只在拖动期间存在 —— 一秒后起每半秒看一次,`XdndSelection` 的属主
    不再抓着指针、也没有进行中的拖出就撤掉;源在拖动开始时建的窗口缓存里没有它,照旧按根窗口找到同一个代理。
  - **流程**:`XdndEnter` 记下源窗口、版本、类型(超过三种时读源窗口的 `XdndTypeList`);每条 `XdndPosition` 回 `XdndStatus` —— 类型里有
    `text/uri-list` 或文字(UTF8_STRING、`text/plain;charset=utf-8`、COMPOUND_TEXT、STRING、`text/plain`、TEXT)就接受,动作只给 `XdndActionCopy`
    (本机拿到的是副本,远端的原件不会被当成「移走了」删掉),矩形为空(每动一下都报)。第一次接受时用这条 `XdndPosition` 的时间戳向 `XdndSelection`
    的属主要数据(协议允许拖着时就取):有 URI 列表要它,文字挑第一个有的;走剪贴板取选区的那一套(每一步 10 秒超时、INCR、属主走了照样结束)。
    取到之后解出 URI(RFC 2483:去掉注释行与空行)与文字(按类型解码),经 `OutgoingDragStarted(XOutgoingDrag)` 交给宿主,连同源程序的连接名
    (宿主据此找到 SSH 会话、把文件取回本机)与根坐标;什么都没取到时之后的 `XdndStatus` 都不接受。
  - **结果**:宿主发起的本机拖放结束后调 `CompleteOutgoingDrag(drag, dropped)`,**再**把松开的按钮注入回来 —— X 程序随后发的 `XdndDrop` 按这个结果回
    `XdndFinished`(第 5 版:l1 第 0 位「接受并做完了」,l2 是 `XdndActionCopy`,没放成为 None)。宿主说取消了时,之后的 `XdndPosition` 回不接受。
    宿主还没交回结果时 `XdndLeave`(指针回到了 X 窗口里)、`XdndDrop`(用户在本机拖放开始之前就松了手,回「没接受」)、源窗口销毁,都报
    `OutgoingDragEnded`,宿主不再发起。旧的那一次的结果不算到新的一次头上(按 `XOutgoingDrag` 对象认)。
  - 真实客户端用例:Swing 的拖放拖到 X 窗口外面,找到垫着的代理,文字交给宿主,宿主说放下之后 `exportDone` 报 COPY。OpenJDK 17 收到成功的
    `XdndFinished` 之后收尾时对窗口 0 发一次 ChangeWindowAttributes(BadWindow 0x0);对照过 Swing 把字拖进它自己的文本框(完全不经过服务端的代理)
    也一样,与服务端无关,AWT 自己吞掉这个错误。
- **剪贴板**:宿主 → X 时服务端自己占有 CLIPBOARD(开了 `SyncPrimary` 时连同 PRIMARY)并按 ICCCM 回应:TARGETS 里有 MULTIPLE(逐对转换,
  转换不了的那一对把属性换成 None 写回)与 COMPOUND_TEXT(Latin-1 原样、其余放进 UTF-8 段);TEXT 目标在 Latin-1 装得下时回 STRING,装不下回
  UTF8_STRING;超过 256 KB 的按 ICCCM §2.5 的 INCR 分块交(每块 256 KB,块之间请求方 10 秒不取就作废,同时至多 32 个传输);编码只在第一次有人要时做一次。
  X → 宿主时服务端以一个隐藏的 InputOnly 窗口为请求方取回,每个选区各取各的(属性 `_VELASHELL_CLIPBOARD` / `_VELASHELL_PRIMARY`),每一步
  (等 SelectionNotify、等下一块 INCR)10 秒不来就放弃这次;按属主回的类型解码(UTF8_STRING → STRING 退路;COMPOUND_TEXT 只认 ASCII、Latin-1 与
  UTF-8 段,别的字符集换成 U+FFFD)。两个方向的上限都是 `X11Server.MaxClipboardBytes`(16 MB,UTF-8)。宿主把刚收到的文本写回来时不抢选区,
  避免与客户端来回争抢;同步的选区失去 X 属主(属主放弃、属主窗口销毁、属主断开)时,服务端以宿主身份接管,内容是最近一次交给宿主的文本,
  相当于剪贴板管理器 —— 原先 X 程序一退出,别的 X 程序就粘贴不到它刚复制的内容。只与焦点所在的会话互通(`ClipboardFollowsFocus`)见上文。

## 8. 里程碑

| 里程碑 | 内容 | 验收 |
| --- | --- | --- |
| **M1 核心协议** ✅ | 全部核心请求;BIG-REQUESTS、XC-MISC;窗口 / 事件 / 属性 / 选区;软件绘图;内置字体;键盘映射;无头测试宿主 | Docker 里 `xdpyinfo`、`xterm`、`xeyes`、`xclock`、`xlogo` 连上、画出内容、零协议错误(已达成,见 §10) |
| **M2 现代工具包** ✅ | SHAPE、XFIXES、RANDR(只读)、RENDER;剪贴板与宿主互通;XSETTINGS 管理器 | GTK3 / Qt5 的简单程序可用(`zenity`、`gedit`、`qt5ct` 画出内容、零协议错误 —— 已达成,见 §10) |
| **功能完备** ✅ | XKEYBOARD、XInputExtension 2.2、XTEST、XINERAMA、SYNC、DAMAGE、Composite、DOUBLE-BUFFER、Present、MIT-SCREEN-SAVER、DPMS、X-Resource、Generic Event;窗口管理器角色;Unix 套接字;运行中换布局 / DPI / 键盘布局;全库性能复查 | xkbcomp、xinput、xdotool、xprintidle、xrestop 读写正确;gedit(GTK3)与 qt5ct(Qt5)走 XKB 与 XI2 零协议错误(已达成,见 §10) |
| **M3 接入宿主** ✅ | Avalonia 宿主(原生窗口、输入、HiDPI、窗口管理器请求、剪贴板、Windows 键盘布局);引擎选择(内置默认,VcXsrv 可选);SSH 的 x11 通道经连接器直接接进服务端;设置页收敛 | 宿主「X Server」按钮不再依赖外部程序;真实的 xterm / xeyes / gedit / qt5ct 画成原生窗口,移动、缩放、关闭、键盘走得通(已达成,见 §10) |
| **M4** ✅ | 同步抓取(冻结与 AllowEvents / XIAllowEvents 的放行、单步、重放);XIChangeHierarchy;XKB SetMap;MIT-SHM 1.1(Linux、本机);GLX 1.4(直接渲染的登记 + 软件间接渲染) | `xinput create-master / reattach / float / remove-master`、`setxkbmap … \| xkbcomp - $DISPLAY`、`x11perf -shmput10`、`glxinfo` / `glxgears` 直接与间接两条路径零协议错误(已达成,见 §10) |

## 9. 测试策略

- **单元测试**(`tests/VelaShell.XServer.Tests/`):内存双工流做传输,测试自带一个最小的「协议客户端」
  逐字节构造请求、解析回复与事件。绘图用例直接断言帧缓冲像素。
- **互操作**(`[TestCategory("Interop")]`,默认跳过):`scripts/xserver/interop/` 构建一个带真实客户端的容器(镜像 `velashell-xclients`:
  x11-apps、xterm、xdotool、xinput、mesa-utils 等,另有 default-jdk —— Swing 用例直接 `java X.java` 跑,验 Java 对窗口管理器的判定),
  用例在本机起服务端、让容器里的真实客户端经 `host.docker.internal:N` 连进来,再把顶层窗口的像素存成 PNG
  供人看、并做粗粒度断言(非背景像素数)。改了 Dockerfile 要重建镜像。XIM 与拖出各有真实客户端用例:xterm 在 `XMODIFIERS=@im=velashell` 下经 XIM 连上,
  宿主上屏的中文进了 shell 的 `read`(over-the-spot,报了插入点);Swing 文本框按 on-the-spot 连上,预编辑交给它画、上屏后文本框里正好是那两个字;
  Swing 往 X 窗口外拖文字,宿主放下之后 `exportDone` 报 COPY。XIM 的协议细节(三种传输、大小端、嵌套属性、错误)另有逐字节的单元测试
  (`XimTests`,测试客户端扮演 Xlib 那一侧),拖出的流控与收尾见 `OutgoingDragTests`。
- 规格与实现不一致时**以规范为准**,改实现,并在本文 §10 记一笔。

## 10. 决策记录

- **2026-09-23 立项**:库名 `VelaShell.XServer`,放在宿主仓库 `src/VelaShell.XServer/`,MIT 授权,
  净室规程同 `VelaShell.Ssh`。
- **入口类叫 `X11Server`,不叫 `XServer`**:后者与命名空间 `VelaShell.XServer` 同名,任何在该命名空间下的代码
  (包括测试)写 `XServer` 都会解析成命名空间。
- **抓取只实现异步模式**:SyncPointer / SyncKeyboard 与 AllowEvents 的冻结 / 放行都按异步处理。需要同步抓取的
  主要是窗口管理器与少数弹出菜单的实现,宿主本身就是窗口管理器,M1 不值得为它引入事件冻结队列。
- **焦点 PointerRoot 用根窗口表示**:`SetInputFocus(root)` 与 `SetInputFocus(PointerRoot)` 行为相同。宿主激活某个原生窗口时
  把焦点给对应的顶层(revert-to PointerRoot),与窗口管理器的常见做法一致。
- **Expose 偏多、不偏少**:子窗口移动 / 堆叠变化时直接重画并 Expose 新旧两块区域,不做「搬运原内容」的优化 ——
  客户端本来就要处理 Expose,多发一次只是多画一次,少发一次就是一块脏图。
- **合成字体**:`cursor`(只有度量,光标形状按字形号推出)与 `nil2`(xterm 的隐形指针用,全空字形)不来自 BDF。
  (2026-10-09 起两者都是 X.Org 的 BDF 原样,见 §7「字体」与本节末尾的「第二轮审查遗留项的收尾」。)
- **M1 验收(2026-09-23)**:单元测试 40 条;真实客户端 `xdpyinfo` / `xterm` / `xeyes` / `xclock` / `xlogo` 零协议错误、
  画出内容;键盘注入经 xterm 到容器里的 sh 往返成功。
- **扩展的编号**:主操作码按实现顺序从 128 起分配 —— BIG-REQUESTS 128、XC-MISC 129、SHAPE 130、XFIXES 131、RANDR 132、
  RENDER 133;事件码 SHAPE 64、XFIXES 65–66、RANDR 67–68;错误码 XFIXES 128、RANDR 129–132、RENDER 133–137。
  客户端一律经 `QueryExtension` 取号,这些值只是本实现的分配。服务端自己的资源(RANDR 的 CRTC / 输出 / 模式、RENDER 的格式、
  选区用的隐藏窗口)取 `0x40` 起的一段,落在任何客户端的 resource-base 之外。
- **XKB 与 XInput2 移出 M2**:实测 Qt5 没有 XKB 时退回核心协议的键位表(只打一行 `XKeyboard extension not present`),
  GTK3 没有 XInput2 时用核心指针事件,键盘与点击都正常。两者都**不能只做一半** —— 扩展一出现在 `QueryExtension` 里,
  客户端就改走它的代码路径;XKB 的 `GetMap` / `GetNames` / `GetCompatMap` 任何一个回复不对,xkbcommon-x11 建键位表失败,
  键盘反而更糟。要做就一次做到 `xkb_x11_keymap_new_from_device` 能成功。Qt6 按其源码同样保留了核心键位表的退路,但还没实测。
- **XSETTINGS 由服务端提供**:GTK / Qt 启动时找 `_XSETTINGS_S0` 的属主,找不到会各踩一次 BadWindow / BadAtom。
  服务端占有它、只发布字体渲染相关的几项;HiDPI 的 `Gdk/WindowScalingFactor` 后来随「功能完备」一轮加上(见下)。
- **诊断日志带请求轨迹**:`XServerOptions.Log` 打印协议错误时附上该客户端最近 8 条请求的操作码 —— 上面那两个错误就是这样定位的。
- **M2 验收(2026-09-23)**:单元测试 73 条;interop 用例 8 条(新增 `xclock -render`、`xeyes -render`、Xft 字体的 `xterm`),
  全部零协议错误;手动验证 `zenity`(GTK3)、`gedit`(GTK3,键盘注入打字)、`qt5ct`(Qt5)零协议错误,
  `xclip` 双向互通(含 12 MB 文本),`xrandr` 读得到配置。
- **功能完备一轮的扩展编号**(接着 M2):GE 134、XTEST 135、XINERAMA 136、MIT-SCREEN-SAVER 137(事件 69)、DPMS 138、
  X-Resource 139、SYNC 140(事件 70–71,错误 138–140)、DAMAGE 141(事件 72,错误 141)、Composite 142、DOUBLE-BUFFER 143
  (错误 142)、Present 144(事件走 GE)、XInputExtension 145(事件 74 起,错误 144–148)、XKEYBOARD 146(事件 73,错误 143)。
  服务端自己的资源另加 SYNC 的 SERVERTIME / IDLETIME 计数器、Composite 的叠加窗口,选区窗口兼作 `_NET_SUPPORTING_WM_CHECK`。
- **XKB 与 XInput2 最终一次做全**:按上面「不能只做一半」的判断,XKB 做到 xkbcommon-x11 建表成功(`xkbcomp` 导出 436 行自洽的键位表)
  才打开;XI2 做到 GTK3 的 gedit 在 XI2 下打字、点开菜单。调试中真实客户端暴露的:GetGeometry(found = False)
  多回了数据(xkbcomp 报 Extra reply data)、缺 SymInterpret(compatibility map not defined)、缺 `_XKB_RULES_NAMES`(setxkbmap)、
  初始焦点应为 PointerRoot(xdotool 打不进字)。
- **窗口管理器请求经接口的默认实现方法交给宿主**:`IXServerHost.WindowManagerRequest` 有空的默认实现,已有的宿主实现不必改;
  `_NET_MOVERESIZE_WINDOW` 与 `_NET_REQUEST_FRAME_EXTENTS` 这种不需要宿主拿主意的由服务端直接办。
- **宿主回调延后到放锁之后**:原来在持锁的执行循环里直接调宿主。宿主若在回调里同步等 UI 线程、而 UI 线程正在
  `CopyPixels` 里等 `PixelLock`,两边互等。改为 `DeferredHost` 攒着、放锁后调。
- **全库性能复查(2026-09-23)**:进程内吞吐基准 `scripts/xserver/bench/bench.cs`。主要改动 —— 损伤有界累计(原先的精确并集是
  所有绘图的头号开销)、可见区域按代号缓存、RENDER 整数内核与字节字形、I/O 缓冲与池化、请求入队不分配闭包、多边形活动边表
  (一个 65535 大小的弧原先能把执行线程占住几十秒)。本机前后:填充 2,748 → 205,000 次/秒,PutImage 671 → 7,800,
  Xft 字形(每请求 10 个)865 → 45,000,ARGB 合成 591 → 26,000,指针移动 51 万 → 103 万,往返 3.4 万 → 8.7 万
  (复查后的数字是预热过的稳态;同一份代码冷热相差约 1.4 倍)。
- **功能完备验收(2026-09-23)**:单元测试 121 条;interop 8 条零协议错误;手动验证 `xdpyinfo -ext all`、xdotool 经 XTEST + XKB
  往 xterm 打字、xprintidle、xrestop、xkbcomp、`xinput list / list-props / query-state / test-xi2`、gedit(GTK3,XI2 + EWMH;
  双击 HeaderBar 发出最大化请求、照办后铺满)、qt5ct(Qt5,XI2)。
- **M3:宿主是一个 `IXServerHost` 实现,不另起进程**:宿主侧(`src/VelaShell/Services/XServer/AvaloniaXServerHost.cs`)每个 X 顶层窗口
  开一个 Avalonia 原生窗口,位图与 X 像素一一对应、按 DPI 缩放不插值;回调全部 Post 到 UI 线程。根窗口 = 所有显示器的外接矩形,
  每台显示器一个 RANDR 输出,显示器增减时重算;DPI 取主显示器的缩放(整数倍时 GTK 的窗口缩放跟着设)。
  X 坐标是**内容区**的位置,系统标题栏与边框在外,尺寸经 `_NET_FRAME_EXTENTS` 告诉客户端。
  没给位置(映射在 0,0)的普通窗口像窗口管理器那样摆:对话框压在父窗口正中,其余在主显示器工作区正中。
  (2026-10 修正:快照的 X / Y 是 X 边框的外沿;客户端请求的位置按 ICCCM 的重力对准原生窗口的**外框**,只有 (0, 0) 而又不是用户指定的位置
  才替它居中,居中的也是外框 —— 见 §6「摆放约定」。原先请求 y = 0 的窗口标题栏落到屏幕外。)
- **M3:引擎可选,默认内置**:`ILocalXServer` 由一个选择器实现,按设置在内置与 VcXsrv 之间转发,正在运行的那个优先
  (改了设置不会去停一个正在显示窗口的 X 服务端)。VcXsrv 只在 Windows 上;其它平台只有内置。
- **M3:SSH 的 x11 通道经连接器直接接进服务端**:SSH 库的 `X11ForwardOptions.LocalConnector`
  (`velashell-docs/zh/ssh/spec/07` §7.5.9)每条通道拿一对内存双工流,一端交给 `X11Server.ServeAsync`。假 cookie 的核对照旧,
  服务端按本机连接放行(2026-09-26 起改为交给 `ServeAuthenticatedAsync`,不再查授权,见 §7)。只在受信模式下用;非受信模式要 `xauth` 连显示,仍走 TCP
  (2026-10 起内置引擎下的非受信模式直接不开转发:本库没有 SECURITY 扩展,`xauth` 签不出受限 cookie,见 §7「授权」;
  2026-10-10 起非受信模式同样经连接器 —— 连接器把「非受信」随通道交给服务端(`ServeAuthenticatedAsync(stream, label, XClientTrust.Untrusted)`),
  不跑 xauth,见 §7「信任级别」)。
  2026-10 补:连接器把 SSH 会话的 `user@host:port` 作为连接名交给 `ServeAuthenticatedAsync(stream, label)`;内置引擎停了(比如换成 VcXsrv)之后,
  老会话的 x11 通道改走本机 TCP 连此刻在运行的那个 X 服务端,而不是一律被拒。
  服务端照样监听环回 TCP 与 Unix 套接字,本机别的 X 程序可以用 `DISPLAY=localhost:N` 连进来(2026-09-26 起要带上宿主写进 `.Xauthority` 的 cookie)。
  (2026-10-10 起 Linux / macOS 上默认只开 Unix 套接字,本机程序用 `DISPLAY=:N`,TCP 要在设置里打开,见 §7「2026-10-10 补的功能」的 F4;
  内置引擎停了之后退到本机 TCP 只在 Windows 上做。)
  **2026-09-24 修正两处**:① 连接器每来一条通道才取**此刻**在运行的服务端,不记住解析显示时的那一个 —— SSH 会话比服务端活得久,
  用户在标题栏把 X Server 停掉再开之后,老会话的每条通道原先都接进已释放的服务端,远端只看到 `Failed to open display`、本机日志一字不记;
  此刻没在运行就拒绝这条通道并记一行日志。② 远端发来 `CHANNEL_EOF` 时连接器那一端也要读到 EOF(spec 07 §7.5.9 新增的一条决策):
  以前远端程序退出后,连接与窗口一直挂到整条 SSH 会话结束。
- **M3:键盘布局跟随 Windows**:宿主按物理键(扫描码)注入 X 键码;Windows 上用系统的 `ToUnicodeEx` 按当前布局算出主键区
  无修饰与 Shift 两层的键值,换进服务端的键位表(布局切换后下次激活 X 窗口时重算;2026-10 起在 X 窗口里按下时就跟上,见 §7「键盘」)。AltGr 层暂不生成 —— 服务端的 XKB 描述目前只推两层;
  其它平台按 US。
- **M3 真实窗口验证中修掉的**:① 释放像素图时一并销毁建在它上面的 Damage 对象是错的 —— `FreePixmap` 只删 ID,xeyes 用 Present 换帧时
  随后的 `DamageDestroy` 因此回 BadDamage、客户端退出;改为 Damage 随 `DamageDestroy` 或客户端断开释放。② Unix 套接字监听
  在套接字文件已存在时会先删掉它 —— 桌面自己的 Xorg 通常不开 TCP,TCP 那一侧的占用检查看不出它,于是会删掉桌面的套接字;
  改为先试着连一下,有人应答就不碰(2026-10 起这次探测异步、限时 300 毫秒,名字被占时整个显示号不用,见 §7「监听与显示号」)。
  ③ 宿主侧:显示之后改原生窗口尺寸要设 `Width` / `Height`(设 `ClientSize` 只改属性值);
  我们自己改尺寸引起的 `Resized` 可能晚一拍才到,只把用户拖动与窗口状态变化回报给服务端,否则会拿旧尺寸把客户端刚设的新尺寸改回去。
  (2026-10 修正:Avalonia 的 X11 后端在 ConfigureNotify 里一律给 `Unspecified`,Linux 上用户拖边框缩放因此从不回报;现在记下最后一次按服务端几何设的尺寸,
  `Resized` 来的尺寸与它不同(差一个像素以内算相同)就当作用户缩放回报。)
- **M3 验收(2026-09-24)**:无头 UI 用例(映射 → 原生窗口尺寸、标题、像素;点关闭 → 客户端断开 → 窗口收掉);
  `scripts/xserver/host-demo/demo.cs` 起真的原生窗口,容器里的 xterm、xeyes、gedit(GTK3,自绘标题栏、菜单弹层)、qt5ct(Qt5)
  画对且几何一致;X 端移动 / 缩放、原生窗口的关闭(WM_DELETE_WINDOW)、原生窗口上的按键到达 xterm 都已实测。
- **AltGr 层(2026-09-24)**:XKB 加 FOUR_LEVEL、FOUR_LEVEL_ALPHABETIC 两个类型(类型表 6 个)。核心键位表第 5、6 列有键值的键推成四级 ——
  列序按 XKB 规范 §17 的核心兼容约定:组 1 第 1、2 级,组 2 第 1、2 级,组 1 第 3、4 级;第三、四级由 Mod5 选。宿主 API 加
  `SetModifierMapping`。Windows 宿主用 `ToUnicodeEx` 按 Ctrl+Alt 取 AltGr 层,布局有 AltGr 字符时出 6 列、右 Alt 设成
  `ISO_Level3_Shift` 进 Mod5;系统为 AltGr 补的假左 Ctrl 不转发(否则 X 程序看到 Ctrl+AltGr)。顺带修掉:更窄的
  ChangeKeyboardMapping(`xmodmap -e "keycode 108 = …"` 每键码只发 1 列)会把整张键位表收成 1 列;现在列数只放宽不收窄。
  验证:`xmodmap` 设好之后 `xkbcomp` 导出四级键、其余键仍两级,xterm 里 AltGr+q 打出 `@`。
- **M4 的扩展编号**(接着前面):MIT-SHM 147(事件 91 Completion,错误 149 BadShmSeg)、GLX 148(事件 92 PbufferClobber,
  错误 150–162)。GLX 的 FBConfig 取 `0x101` 起,四个:两个 TrueColor 视觉各一单一双缓冲,颜色 8/8/8(ARGB 视觉再加 8 位 alpha)、
  深度 24、模板 8,没有累积缓冲;GLX 1.2 的视觉配置每个视觉一条,取它的双缓冲配置。2026-10-10 起另有两个 4 倍多重采样的双缓冲配置
  (`0x105` / `0x106`,见「2026-10-10 补的功能」),FBConfig 共六个。
- **同步抓取取代「只做异步」(2026-09-24)**:M1 时把 Sync 模式按异步处理;M4 做成真正的冻结 —— 每设备一个冻结者与一条输入队列,
  宿主注入与 XTEST 的输入冻结时排队;AllowEvents 0–7 与 XIAllowEvents 放行、单步(SyncPointer / SyncKeyboard / SyncBoth)、
  重放(重放时跳过抓取窗口及其上级的被动抓取);抓取结束(含客户端断开)时解冻并把队列放回执行循环。
- **XIChangeHierarchy 的 RemoveMaster**:AttachToMaster 模式下返回设备给 0 表示挂回虚拟核心指针 / 键盘 ——
  `xinput remove-master` 默认就这么发,按非法设备处理会回 BadMatch。
- **XKB SetMap 不另存 XKB 表**:类型、动作、行为、显式成分、虚拟修饰映射按规范读完、不另存,键值换成核心列写回,
  之后照常由核心表推出 —— 与「XKB 由核心表推出」的总原则一致,`xkbcomp keymap $DISPLAY` 因此生效。
- **扩展可以按客户端决定可见性**:注册表里的扩展带一个可选的可见性判断,QueryExtension、ListExtensions 与分派都按它来。
  MIT-SHM 用它只给经 Unix 套接字连进来的客户端。
- **根窗口 GetImage**:rootless 下根窗口没有缓冲,原先 GetImage 回 BadMatch(`xwd -root` 失败);现在按堆叠次序把映射着的顶层拼起来,其余为黑。
- **GLX 的串带结尾的 NUL**:QueryServerString 与 GetString 回复里的 STRING8 长度把结尾的 NUL 算在内 —— 客户端库按 C 串用这块内存。
- **M4 验收(2026-09-24)**:XServer 单元测试 +17(同步抓取、设备拓扑、SetMap、MIT-SHM(只在 Linux 上跑)、根窗口 GetImage、
  GLX 7 条 —— 清除与三角形、深度与显示列表、经 RenderLarge 上传的纹理、GL 查询与 ReadPixels、显示列表执行预算等),
  真实客户端用例 +3(`glxinfo` 两条路径、`glxgears` 直接 / 间接)。手动:`xinput` 增删主设备、挂接 / 浮动从设备;
  `setxkbmap -print -layout de | xkbcomp - $DISPLAY` 之后 y / z 互换、AltGr 层就位;Linux 容器里 `xdpyinfo` 列出 MIT-SHM、
  `x11perf -shmput10` 跑通;`glxgears` 两条路径画出的齿轮一致(光照、平直着色、深度),`glxheads` 间接渲染正常。
- **键盘布局:「自动」三个平台都跟随系统,也可以手选(2026-09-24)**:宿主侧按平台取当前布局 —— Windows `ToUnicodeEx`;macOS
  `TISCopyCurrentKeyboardLayoutInputSource` + `UCKeyTranslate`(Option 层对应 AltGr,右 Option 成为 ISO_Level3_Shift,左 Option 仍是 Alt;
  TIS 只能在主线程上调);Linux 经 libxkbcommon-x11 读桌面 `$DISPLAY`(Wayland 上是 XWayland)的键位表,只有桌面的右 Alt 是 Level3 键时
  才保留第三、四层(xkeyboard-config 的表里总有一个虚拟 `<LVL3>` 键,扫全表会把美式也判成有 AltGr)。设置里的「键盘布局」改为两种引擎共用,
  手选时内置引擎用随程序带的键位表(`scripts/xserver/keymaps/generate.cs` 经 libxkbcommon 从 xkeyboard-config 生成,27 个布局)。
  三条来源经同一个出口(四层按 §17 的核心列序)换进服务端的核心键位表,XKB 照常推出;服务端库本身不变。
- **渲染路径复查(2026-09-25)**:从「客户端的一次绘图」到「屏幕上的一帧」整条链路过了一遍。
  ① **宿主只读损伤矩形、一趟拷贝**:新增 `XTopLevelWindow.ReadPixels`(在像素锁里把缓冲交给回调,宽高取自缓冲本身)。
  Avalonia 宿主把每个窗口切成 256 × 256 的小块、每块一张位图,跟着显示器的帧(`RequestAnimationFrame`)取攒下的损伤矩形,
  在像素锁里直接写进锁好的小块。原先每批损伤(负载重时一秒几百批)都整窗拷两遍 —— 先拷进中转数组,再逐像素补 alpha 写进一张整窗位图 ——
  渲染层随后每帧把整窗重传给 GPU;现在一帧最多取一次,只有被改到的块重传。宿主侧的损伤跨线程攒着,UI 线程上同一时刻最多排一次投递。
  顺带修掉一个崩溃:UI 线程先读 `Width` / `Height`、再拷像素,客户端恰在中间把窗口改大时越界抛异常。
  ② **像素锁让行**(见 §5):执行循环看到宿主在等就提前放锁、等它读完再拿。
  ③ **PutImage 只处理画得到的部分**:PutImage 与 MIT-SHM 的 PutImage 只解码、只贴目标上画得到的那一块;32 位 ZPixmap(Qt、GTK4 经 llvmpipe、浏览器整窗送像素)
  直接从请求数据或共享内存段按行贴进缓冲,不经中转数组。ShmPutImage 原先把整幅共享图像拷出、整幅解码、再拷出源矩形 —— 一次小块更新要扫四遍整幅图;
  位图格式的 PutImage 原先按整幅宽高分配像素(一个字节八个像素,16 MB 的请求放大成 512 MB)。格式与深度的校验提到按深度算长度之前。
  ④ RENDER FillRectangles 整个请求只算一次目标区域(原先每个矩形都克隆一次可见区域)。⑤ 读请求时不再先把缓冲清零。
  基准(`scripts/xserver/bench/bench.cs` 新增整窗 800×600 PutImage、50 个矩形的 FillRectangles、满载时宿主读像素三个场景):整窗 PutImage 吞吐 +6–13%、
  CPU −12–26%;进程内基准的瓶颈在测试客户端与管道,服务端省下的主要体现在 CPU 与持锁时间上,其余场景在噪声范围内。宿主那一半(只取损伤、按帧、分块上传)不在这个基准里。
- **宿主 API 整理(2026-09-25)**:按一次 API 评审把公开面理了一遍,行为不变的地方只改形状。
  ① **公开面收拢**:公开类型全部移到根命名空间 `VelaShell.XServer`(原来散在 `.Host`、`.Server`、`.Drawing` 三处),`X11Server` 的公开成员全部集中在 `X11Server.cs`;
  `PixelLock`(没人用、直接锁它还拿不到让行)与 `XErrorCode` 收回 internal。`XServerOptions` / `IXServerHost` 改名 `X11ServerOptions` / `IX11ServerHost`,
  与 `X11Server` 同一前缀,也不再与宿主应用自己的 `XServerOptions` 重名;选项改成 sealed class,构造时统一校验(不合法抛 `ArgumentException`)。
  ② **命名成体系**(2026-10 按现有公开面重写了规则的写法,见 §6「命名」):宿主方法分三类 —— `Inject*`(合成输入)、`*TopLevel`(窗口管理器的动作)、`Set*`(换配置);回调一律「主语 + 过去分词」
  (`Bell` → `BellRequested`,`WindowManagerRequest` → `WindowManagerRequested`)。窗口用 `XTopLevelWindow` 句柄指名,不再传 XID。
  ③ **窗口属性是不可变快照**:原来 `XTopLevelWindow` 的二十几个属性由执行线程逐个写、宿主在 UI 线程上直接读,可能读到新宽度配旧高度,
  16 字节的 `ClientFrameExtents` 元组还会被撕裂。现在执行线程每次造一份 `XTopLevelSnapshot` 整份替换,`TopLevelChanged` 附带变了哪几组;
  值没变不报(同样的标题再设一遍不再惊动宿主),形状没变沿用同一个列表实例。
  ④ **光标**:原来是一个带魔数的 int(字形号,−1 同时表示默认与位图光标,−2 隐藏),宿主得自己维护字形号对照表,位图 / ARGB 光标
  (装了光标主题时 libXcursor 走 RENDER CreateCursor)一律显示成箭头。现在是 `XCursor`:形状在库里从字形号或 XFIXES SetCursorName 起的名字推出,
  位图与 ARGB 光标在创建时烙好图像一起交给宿主,Avalonia 宿主直接显示图像。
  ⑤ **键位表一次换**:原来一次换布局要调三次 `SetKeyboardMapping` 加一次 `SetModifierMapping`,每次都给所有客户端发一轮通知;布局名是最后一个可选参数,
  宿主从来没传过,于是用户选了 `de`,`_XKB_RULES_NAMES` 仍是 `us`。现在 `SetKeymap(XKeymap)` 带着布局名一次提交;右 Alt 当 AltGr 的键值与修饰位由库给出,
  宿主不再手写修饰键表;跟随系统时宿主取随程序带的表里最像的布局名。
  ⑥ **扩展注册表**:主操作码与事件 / 错误编号收进一张表,注册时检查不重叠;扩展登记断开与窗口销毁的清理钩子,连接收尾与窗口销毁不再各维护一份清单。
  GLX 拆成独立的 `GlxExtension` 类;各扩展的资源类型移到 `Resources/`;`Queries` 拆成 Xinerama / XRes,`CompositeDbe` 拆成 Composite / Dbe,
  DPMS 从 ScreenSaver 里拆出,`SyncGrabs` 改名 `GrabFreeze`(与 SYNC 扩展区分),顶层窗口桥从 Exposure 挪到 TopLevels。
  ⑦ 其余:`Display` 按实际监听的传输给出(`:N` / `localhost:N.0` / 没监听为 null),另有 `DisplayNumber`;`ServeAsync` 的 `isLocal` 不再有默认值、
  写明不释放流;诊断只走 `Log`(原来一部分走 `Trace`,宿主注入的工作项出错时宿主看不到);响铃按协议从基准音量换算。
- **全库审查的 31 项修复(2026-09-26)**:2026-09-25 审查登记的 31 项逐项修掉,每项配用例、先撤修复确认用例会红;上限与授权见 §5、§7。
  其中改了协议行为或值得记住的:
  ① **扩展的错误码顺延一位**:XFIXES 补上 BadBarrier(`DestroyPointerBarrier` 只认指针屏障,原先对任意 ID 调删除,一个请求就能删掉根窗口),
  之后各扩展的错误码跟着顺延:XFIXES 128–129、RANDR 130–133、RENDER 134–138、SYNC 139–141、DAMAGE 142、DOUBLE-BUFFER 143、
  XKEYBOARD 144、XInputExtension 145–149、MIT-SHM 150、GLX 151–164。客户端一律经 `QueryExtension` 取号,不受影响。
  ② **焦点与 crossing 按协议**:焦点事件按协议「Input Focus events」一节给 detail(Ancestor / Virtual / Inferior / Nonlinear / NonlinearVirtual / Pointer…)、
  发虚拟事件,FocusIn 之后发 KeymapNotify;抓取激活 / 解除发 Grab / Ungrab 模式的焦点与 crossing 事件,抓取期间 Enter / Leave 只报给抓取方;
  抓取窗口或 confine-to 变得不可见时自动解除;键盘被抓着时按键事件的源窗口仍按焦点算。
  ③ **`FocusTopLevel` 按 ICCCM 的输入模型**(WM_HINTS 的 input 提示 §4.1.2.4、WM_TAKE_FOCUS §4.2.8):override-redirect 的不理;input 提示为 False 的不直接给焦点;
  登记了 `WM_TAKE_FOCUS` 的发 ClientMessage(时间戳不用 CurrentTime)。
  ④ **CloseDownMode 生效**:RetainPermanent / RetainTemporary 的客户端断开后资源保留,KillClient 按资源 ID 或 AllTemporary 销毁。
  ⑤ **XKB 锁存的修饰键用一次即解除**;SetMap 的修饰映射校验键码范围;XI2 按钮事件的 buttons 取事件之前的状态(协议如此)。
  ⑥ **GLX**:MakeCurrent 先备好表面再改状态(原先抛 BadAlloc 时上下文卡在一个幽灵 tag 上);RenderLarge 核对拼出来的长度与第一段声明的一致;
  RenderMode 对照 *GLX Extensions for OpenGL Protocol Specification* 1.3 §2.2.1 确认原有行为 —— 之前在渲染模式时没有回复。
  ⑦ **宿主按钮状态逐个按钮记**:原生窗口失去捕获或失活时替 X 松开还按着的按钮;顶层窗口已经没了的松开照样生效(否则 X 那边一直按着)。
  ⑧ **性能**:基准(`scripts/xserver/bench/bench.cs`,新增渐变 / 变换 / 遮罩合成、GLX 单缓冲小三角形四个场景与每次请求的分配字节一列)——
  线性渐变 Over 4.2k → 7.2k 次 / 秒、放大 2 倍双线性 2.2k → 4.0k、ARGB + a8 遮罩 3.8k → 19.3k;GLX 单缓冲每请求一个小三角形 4.2k → 约 46k;
  整窗 PutImage 1.1k → 1.6k(CPU 少四成)。
- **第二轮全库审查的修复(2026-10)**:审查登记的安全、正确性、性能与 API 几组问题逐项修掉,每项配用例、先撤修复确认用例会红;
  行为与上限已经写进 §2–§7,这里只记取舍与来由。
  ① **资源从「只限单件」改成「单件上限 + 内存账」**,另加每项工作的工作量预算与宿主限时读像素(§5、§7):上一轮的上限都是单件的 ——
  一条 16 字节的 CreatePixmap 就是 256 MB,十几条就能让服务端连同宿主进程耗尽内存;一条代价与字节不成比例的请求能让宿主界面陪着冻几分钟。
  ② **剪贴板跟着键盘焦点所在的会话走**(`ClipboardFollowsFocus` 默认开):原先默认双向自动同步,本机复制的密码在用户点一下任意 X 窗口之后对所有会话可读,
  远端程序也能反复改写本机剪贴板。宿主的「选中即复制」默认改为关,与库的 `SyncPrimary` 默认一致(宿主原先默认开、映射成 PRIMARY 同步,与库相反)。
  ③ **`RestrictForwardedClients` 默认关**:远端的 xdotool 之类靠 XTEST 工作,默认值维持现状(产品决策),宿主设置里给开关。
  连接级的信任档与「每个会话一个显示」是后续的新功能。
  ④ **`ListenTcp` 改成 `bool?`,零值配置不开 TCP**:原先默认 true 而 cookie 默认不配,`new X11Server()` + `StartAsync()` 的结果是本机任何用户都能经环回 TCP
  连进来(TCP 上分不出对端是哪个用户),与 SSH 库「零值取安全值」的约定不一致。宿主一直配着 cookie,行为不变;`scripts/xserver/host-demo/demo.cs`
  经转发给容器用、不配 cookie,显式打开。
  ⑤ **窗口管理器名默认 `LG3D`**:服务端占住 `WM_S0` 与根窗口的 SubstructureRedirect 之后,Java(AWT / Swing)认定有窗口管理器,再按检查窗口上的
  `_NET_WM_NAME` 决定它套不套外框 —— 不认得的名字一律当成会套外框,于是一直等 ReparentNotify、不理 ConfigureNotify。OpenJDK 17 的 Swing 探针实测:
  改之前判成没有窗口管理器,不能最大化;只占不改名(`VelaShell`)判成 Other WM,假定 25 像素的标题栏,最大化 / 改尺寸之后内容不重排;
  名字为 `LG3D` 时识别为 LookingGlass(不套外框),边距 0,最大化与改尺寸都照常重排。`X11ServerOptions.WindowManagerName` 改名之前先用 Swing 程序验一遍;
  互操作镜像为此加了 default-jdk 与 Swing 用例。
  ⑥ **内置引擎下非受信的 X11 转发直接不开、说清原因**(§7「授权」),不再让用户只看到 `xauth` 的报错;「受信任」的提示改写为说明所有转发的会话共用一个显示。
  ⑦ **GLX 表面超过 4096² 时夹到上限、记一行日志**,不再回 BadAlloc —— 8K 屏、跨屏最大化的 GL 窗口原先每个 GL 请求都报错。
  ⑧ **核对规范后不改的**:XI2 被动按钮 / 按键抓取激活时的 crossing 仍是 Grab 模式(XI 2.2 写明如此);GenericEvent 不按「是否发过 GEQueryVersion」把关
  (各客户端库是否都先发它,在净室规程下核实不了,贸然把关可能让程序收不到事件而卡住;只删掉了只赋值不读的字段);公开方法不为命名规则改名,
  改规则的写法(§6「命名」);不加「一次锁内读多个窗口」的接口(基准量下来不需要,§5);GTK 不开客户端阴影时的边缘缩放实测可用(§7);
  跨客户端使用 GLX 上下文与核心协议的信任模型一致(§7)。
  ⑨ **留到以后的**(都要新功能):点本机窗口或桌面就收起 X 的弹出菜单(要全局指针钩子;现在点任何一个 X 窗口都会送到菜单的抓取方、菜单收起,
  弹层也只在用户用 X 时置顶,不会盖住本机程序);WarpPointer 挪宿主的真实光标、confine-to 约束用户的鼠标;PolyArc 相接的弧之间的接头;
  源 picture 的 alpha-map;更多核心字族与字号;宿主侧分数缩放下最后一列像素可能被裁(要各平台实机核对);间接 GLX 的单缓冲视觉;
  视频播放器的 ForceScreenSaver / Suspend 转告宿主、抑制系统屏保;宿主界面上「解除卡住」的入口(库这边的 `BreakGrabs` 与 `DisconnectClient` 已齐)。
  (2026-10-09:其中 PolyArc 的接头、alpha-map、更多核心字族与单缓冲视觉已补,见下一条。)
  ⑩ 宿主侧的界面行为(X 程序无响应时的强制结束、停 X Server 前的确认、焦点窃取防护、桌面类与 InputOnly 窗口不开原生窗口、形状外点击穿透……)见
  [`../../host/交互与界面规格.md`](../../host/交互与界面规格.md) §4A.3,设置项见 [`../../host/settings-audit.md`](../../host/settings-audit.md) 第十一批,
  排障见 [`../troubleshooting.md`](../troubleshooting.md)。
- **第二轮审查遗留项的收尾(2026-10-09)**:上一条 ⑨ 里不靠新功能的几项、以及审查时只做了一部分的几项补齐;行为写进了 §6、§7,这里只记取舍与来由。
  ① **核心字体随库带数据,不等宿主的字体提供者**(xs_plan F21 方案 A):方案原文是把 Liberation 之类栅格化成 75 / 100 dpi;改用 X.Org 自己的 Adobe 位图 ——
  字体名与度量与真实的 X 服务端一致,许可同样宽松,也不用写一个 TrueType 栅格化器。中日韩与其余文字靠 GNU Unifont(双许可里取 OFL)与 misc-fixed 的
  ja / ko 字体。B&H 的 Lucida 不带:许可要求在用户文档与代码注释里附特定声明。数据由脚本从固定的上游提交生成、逐字节不动,换版本只改脚本里的提交号与哈希。
  代价是库的 DLL 多约 4.5 MB。
  ② **字体在线程池上建,请求暂存**:带上整套字体之后,真实客户端的 `xlsfonts -l "*"` 第一次在执行线程上持着像素锁解析了 1.5 秒,宿主界面陪着冻住。
  解析字体是纯计算、没有 I/O,放到线程池上不算假异步;暂存复用 SYNC Await 与 XTEST 延迟的那一套,放回的那一条不再查第二遍,后台建不出来时照常执行、
  按常规回错误。
  ③ **cursor 字体里有对应系统光标的字形仍显示系统光标**:GetCursorImage 给烙好的 X 位图,本机却仍用系统光标 —— 它跟着桌面的主题与缩放,
  比 16 像素的 X 位图清楚;没有对应的才把图像交给宿主。
  ④ **PolyArc 的「相接」按差不到半个像素判**:协议只说端点「重合」,而端点是实数 —— 逐位相等太苛刻(同一个椭圆相邻两段的端点差几个末位),
  取整到同一个像素在 .5 上又不稳;90° 倍数上的端点都落在半像素的格点上,对它们这个判据就是恰好相等。单独一条弧与互不相接的弧像素不变(20 万组随机对拍)。
  ⑤ **RENDER 的整数路径扩到 a1 / a8 / 8888 目标上的全部运算**:与逐像素浮点合成最多差 1、a1 逐位相同(穷举核对过),原先走浮点的那些组合
  快约 2–12 倍。顺带修了浮点版 ClipColor 的一处 NaN(灰色的源在 αd = 0 的目标上做 HSL 四种模式,颜色成了 0)。r5g6b5、x1r5g5b5、a4 目标少见,仍走浮点。
  ⑥ 选区按会话隔离(别的会话看不到属主变化、收不到因此发的 SelectionClear)、源 picture 的 alpha-map、间接 GLX 的单缓冲视觉同在这一轮补上(§7)。
  ⑦ **仍留着的**:点本机窗口或桌面就收起 X 的弹出菜单(要全局指针钩子);WarpPointer / confine-to、屏保转告宿主、「解除卡住」的宿主入口等新功能;
  分数缩放下最后一列像素(要各平台实机核对);外接框宽或高为 0 的宽弧只画出一半线宽(审查之前就如此,这一轮没有动;2026-10-10 已改,见下一条)。
- **审查清单的最后一轮收尾(2026-10-10)**:对着第二次审查的清单(xs_plan)逐条核过一遍,不靠新功能、不靠实机的几项补完;行为写进了 §6、§7。
  ① **外接框宽或高为 0 的宽弧画满整个线宽**:协议只在宽高都不为 0 且不相等时把宽弧的边界交给实现,有一个为 0 时就是理想的那两条线。
  折返处补半个圆,取的是宽(或高)趋于 0 的椭圆在那一头的极限,与宽为 1 的椭圆在那一头同样伸出半个线宽。宽高都不为 0 的弧像素不变。
  ② **RENDER 的 alpha-map 按「换掉 alpha 通道」理解,源与目标一致**:原先当源时把颜色先按 drawable 的 alpha 除、再乘 alpha-map 的 alpha,
  写进挂了 alpha-map 的目标、再当源读回来就不是同一个像素;规范说的只是 alpha 通道被取代。目标侧不改合成器,先拼成一张临时的 a8r8g8b8 再写回 ——
  这种目标极少见(cairo、Qt、Java 都不用),换来合成器的各条快路径不用知道 alpha-map;临时缓冲只按请求的范围分配,工作量按两遍拷贝扣。
  ③ **按磅数要字体时按屏幕分辨率挑 75 / 100 dpi**:同一个磅数在两份里的像素高度不同,原先按名字的先后总取 75 dpi 的;现在取 RESOLUTION_Y 离
  `X11ServerOptions.Dpi` 最近的。给了分辨率或像素高度的名字与原来完全一样。
  ④ **字体脚本核对 X.Org 字体的内容**:对挑出来的文件算摘要(按文件名排序,每个文件一行「文件名 内容的 SHA-256」再整体算),不对归档算;
  全部下载、核对完才动数据目录。按固定的摘要重跑一遍,生成的数据与仓库里的逐字节相同。
  ⑤ **`.Xauthority` 与真实的 xauth 对过**:互操作镜像里的 xauth 列得出我们登记的那条,两边改写都留着对方的记录;我们用的锁文件名 xauth 认得、看见就不写。
  ⑥ **仍留着的**:点本机窗口或桌面就收起 X 的弹出菜单(要全局指针钩子);新功能(按会话隔离、X 程序清单的界面、WarpPointer、屏保转告、合成管理器、托盘、单窗口模式……,2026-10-10 大多做了,见本节末尾 2026-10-10 的几条);
  要实机核对的分数缩放下最后一列像素、macOS 上 Command 组合键的 KeyUp、macOS / FreeBSD 经 getpeereid 取对端 uid;互操作靶场扩到 GTK3 / GTK4、浏览器、
  Motif / Tk / Emacs、桌面会话、托盘与 fcitx5。这些都记在宿主仓库的 `feature-plan.md`「H. 内置 X 服务端」。
- **X 程序清单与强制结束的界面(2026-10-10,xs_plan F3)**:库这边只补了两样 —— `XClientInfo.HoldsServerGrab`,与 GrabServer 抓太久时的宿主回调
  `ServerGrabStalled`(原先只记日志:宿主与用户都不知道该断开谁)。回调与别的一样经延后队列在放掉像素锁之后交出;宿主的 `GetClientsAsync` 要回到执行线程,
  所以 VelaShell 收到它之后另起任务查是哪个程序,不在回调里同步等。界面见宿主文档《交互与界面规格》§4A.2:标题栏 X Server 按钮在内置引擎运行中时开浮层,
  外部 VcXsrv 列不出程序,照旧一点开、一点关。
- **单窗口(rootful)模式(2026-10-10,xs_plan F13 / 决策 Q2)**:§2 原先只写了「rootless 多窗口、根窗口不画」这个前提,没把整块桌面写成非目标。
  完整远端桌面(xfce / MATE)、图形安装器在 rootless 下实际不能用(远端窗口管理器拿不到 SubstructureRedirect,桌面窗口盖住本机),
  而 VcXsrv 的「One large window」、MobaXterm 的 Windowed 都是用户熟悉的形态,于是作为**可选模式**加入,默认仍 rootless。
  实现没改绘图路径:顶层照旧各有缓冲,服务端像一个永远开着的合成器那样在根窗口的缓冲里拼出整块屏幕(见 §7「单窗口模式」);
  窗口管理器的角色整个让给远端。真实客户端用例:twm 接管、xterm 被套进外框、拼进屏幕,零协议错误。
- **信任级别与每个 SSH 会话一个显示(2026-10-10,xs_plan F2 / F1,决策 Q5)**:所有转发来的会话原先共用一个受信的显示,非受信的 `ssh -X`
  在内置引擎下开不起来。按决策 Q5 分两步收紧:先实现 SECURITY 扩展的非受信级别(远端 `xauth generate … untrusted` 签得出受限 cookie,
  宿主的连接器也能直接把一条通道标成非受信),再在宿主加「每个 SSH 会话一个显示」—— 受信的会话之间要隔开,只能放到不同的显示上。
  XC-QUERY-SECURITY-1 与 Application Group 不做;每个会话一个显示默认关、下次启动生效。真实客户端用例:`xauth generate` 签出非受信 cookie,
  之后 xdpyinfo 看不到 XTEST / SECURITY、`xwd -root` 截不了屏,xterm(核心字体与 Xft)、`xclock -render`、xlogo、间接 glxgears 照常画、零协议错误。
- **草案第十二节其余新功能的取舍(2026-10-10,xs_plan F4–F30)**:F4、F5 第一步(本机输入法上屏)、F6、F8–F11、F14、F15、F17、F18、F20 的截图、
  F22、F25、F27、F28 做了;F21、F26 与 F22 的 create_context 部分此前已经有。只做了一部分的:F19(各显示器的 DPI 已经按各自的缩放报;
  整数放大要一个全服务端统一的放大倍数,宿主所有根坐标与屏幕坐标的换算都要改,只靠无头用例验证风险太大)、F23(反馈模式、mipmap LOD、
  PixelMap、深度 / 模板格式的 DrawPixels / CopyPixels 没做)、F24(双线性取样、GL 光栅化没动)、F25(一个全局帧时钟,不分显示器)。
  没做的:F5 第二步 XIM 桥(3–5 周)、F7 压感 / 触摸 / 手势(要按 SlaveSwitch 换主设备的类,没有真实工具包可核对;XI 2.4 不在规范清单里)、
  F20 的录屏(要编码器)、F29 WSL(没有可测的环境)、F30 MIT-SHM 1.2(Linux 才有的 fd 传递,TCP 的容器环境测不了)。
  默认值:本机输入法开、标出来源开、合成管理器关、Linux / macOS 上的 TCP 端口关(经网络连的容器程序要打开它)。
- **新设置的默认值与 F19(2026-10-10,维护者)**:上一条的默认值确认保持,每个 SSH 会话一个显示默认关。F19 的整数倍放大先等:不做全服务端统一倍数的方案,
  等「每个 SSH 会话一个显示」给每个显示各自的倍数。
- **核对新设置之后的几处修正(2026-10-10)**:对着上面几项逐一核对、外加真实客户端用例,改了这些(细节见 §7 各条):
  ① 本机输入法上屏:挪用键码从「空闲 200 毫秒」改成「空闲 3 秒」—— 正确性靠的是客户端为前一个字重取完键位表,这只能靠足够长的时间保证
  (按「客户端取过一次键位表」判断不成立:它可能在处理到这一轮通知之前就为别的原因取过);一批字一次通知;等的时候宿主的按键排在后面;
  借来的键用 ALPHABETIC 类型;非受信客户端看不见输入给受信程序的字。真实客户端用例:xev 收到的是 U4E2D、U6587,CapsLock 开着时仍是 eacute。
  ② 合成管理器关着时 ARGB 窗口按不透明显示;选区被远端合成器接走又放下时服务端接回来。③ 托盘图标的名字照窗口标题过滤,宿主的提示标出来源。
  ④ Unix 套接字建不起来、TCP 又没开时只服务 SSH 转发,不再整个起不来。⑤ 宿主一侧(按会话分出来的显示拿到全部宿主设置、关掉某个会话的屏幕窗口只收它、
  「标出来源」立即生效、显示地址按实际监听报)见宿主仓库 `plan.md`。
- **本机输入法的 XIM 桥与拖出(2026-10-10,xs_plan F5 第二步 / 决策 Q1、F16 的另一半)**:
  ① XIM —— 服务端自己当 XIM 输入法服务端,而不是去桥接远端的 fcitx / ibus:远端多半没装,而宿主手里就有本机的输入法。只走 X 传输(Xlib 自带、
  不用另开端口);按键不经 XIM 转发(转发掩码 0),组字留在本机、只有上屏的字与预编辑过去,慢链路上打字不多一个往返。风格只做 on-the-spot、
  over-the-spot 与根窗口三种:off-the-spot 要宿主往程序的窗口里画状态区与预编辑区,做不到。程序得在 `XMODIFIERS=@im=velashell` 下启动 ——
  sshd 默认不放行这个变量的 env 请求,宿主借连接后的静默注入设上(已经设了的不动)。互操作逮到一个单元测试测不出的问题:Xlib 用「只查不建」的
  InternAtom 看 `LOCALES` 在不在,服务端得先把它建好。Qt 5 / 6 与 GTK 4 没有 XIM,照旧借键码上屏。
  ② 拖出 —— 用 XDND 第 4 版起的根窗口 `XdndProxy` 约定,不必猜指针下面是不是本机窗口:凡是落在 X 窗口以外的都交给宿主。Java 不看根窗口的
  `XdndProxy`,于是拖动期间把代理窗口悄悄垫在最底下、带 `WM_STATE`;不发结构事件,只在源抓着指针的期间存在。动作只给复制。
  数据在 X 那边就取好(拖着时取,协议允许),宿主拿到的是 URI 与文字;取回远端文件、发起本机拖放是宿主的事(见宿主仓库 `plan.md`)。
