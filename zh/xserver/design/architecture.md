# VelaShell.XServer 架构与原理

English: [`../../../en/xserver/design/architecture.md`](../../../en/xserver/design/architecture.md)

> 状态:**M1(核心协议)、M2(现代工具包)、「功能完备」一轮(XKB、XInput2 等十余个扩展、窗口管理器角色、全库性能复查)
> M3(接入宿主)与 M4(同步抓取、设备拓扑、XKB 改表、MIT-SHM、GLX)已完成**,2026-09-23 立项。宿主的「X Server」按钮默认启动的就是本库(每个 X 窗口一个 Avalonia 原生窗口),
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
- **可嵌入**:库不依赖任何 UI 框架、不依赖原生库。像素、窗口生命周期与输入经一个宿主接口(§6)交换。
- **可测**:协议层全部走内存传输单测;另有 `[TestCategory("Interop")]` 用例让真实的 Xlib / XCB 客户端
  (Docker 里的 `xdpyinfo`、`xterm`、`xeyes`…)连进来。

**非目标**

- 不做**真实显示设备**(DRM/KMS、帧缓冲设备)与硬件输入 —— 那是完整 X 服务端的事,我们只做嵌入式的那一半。
- 不做 XDMCP、多屏幕(Screen > 1)、索引色 / 可写颜色表(只提供 24 位 TrueColor)。
- 不做 DRI2 / DRI3(没有 GPU 可交给客户端)。GLX 只登记直接渲染(客户端自己软件渲染、再经 PutImage 送像素)并提供
  固定功能 GL 子集的软件间接渲染;MIT-SHM 只在 Linux 上、只给经 Unix 套接字连进来的本机客户端。Present 只做软件拷贝(没有显存可翻页)。

## 3. 净室规程

与 `VelaShell.Ssh` 同一套纪律(见宿主仓库 `src/VelaShell.XServer/AGENTS.md`):

1. **实现依据只能是公开规范**:X.Org 发布的 *X Window System Protocol, X Version 11*(含附录 B 编码)、
   各扩展规范(*BIG-REQUESTS*、*XC-MISC*、*X Nonrectangular Window Shape Extension*、*X Fixes Extension*、
   *The X Resize and Rotate Extension*、*The X Rendering Extension*(及其引用的 PDF Reference 混合模式公式)、
   *The X Keyboard Extension: Protocol Specification*、*The X Input Extension*(1.5 与 2.2)、*XTEST Extension*、
   *X Synchronization Extension*、*X Damage Extension*、*Composite Extension*、*Double Buffer Extension*、
   *The Present Extension*、*MIT-SCREEN-SAVER*、*DPMS*、*X-Resource*、*Generic Event Extension*、*The MIT Shared Memory Extension* 1.1,
   以及 XINERAMA —— 它没有独立的规范文档,线格式依据 X.Org 发布的 panoramiXproto 协议定义)、ICCCM、EWMH、freedesktop.org 的
   XSETTINGS 规范;GLX 部分依据 Khronos 发布的 *OpenGL Graphics with the X Window System* 1.4、*GLX Extensions for OpenGL
   Protocol Specification* 1.3(编码)与 *The OpenGL Graphics System* 1.5(间接渲染的 GL 语义),操作码与枚举值取自 Khronos
   注册表的 `gl.xml` / `glx.xml`。
   **每个协议实现文件的头部写明它实现的是哪份规范的哪一节。**
2. **写实现时不打开任何其它 X 服务端的源码**(X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv),也不打开任何
   OpenGL / GLX 实现(Mesa 等)的源码。
3. 规范里的常量(操作码、事件码、错误码、预定义原子、掩码位)按规范取值 —— 那是协议,不受版权保护,
   也不许为了「看起来不一样」去改。
4. **数据不等于代码**:内置位图字体取自 X.Org `font-misc-misc` 的 BDF(版权声明原文为
   "Public domain font. Share and enjoy."),以数据文件随库分发并在 `NOTICE.md` 里注明来源。

## 4. 分层

```
Protocol/     常量(操作码、事件码、错误码、掩码、预定义原子)、字节序感知的请求读取与回复 / 事件 / 错误写出
Server/       X11Server:监听(TCP 6000+N、Unix 套接字)与 ServeAsync(任意双工流)、连接建立与授权(MIT-MAGIC-COOKIE-1 / 仅本机)、
              单线程执行循环、连接层背压、宿主回调的延后调用(DeferredHost)、客户端表与序号、GrabServer、BIG-REQUESTS 长度;
              请求处理按领域拆成 partial 文件(Windows / Exposure / Events / Properties / Graphics / Text / Colors / Input /
              Extensions / Queries,以及各扩展:Shape / XFixes / RandR / Monitors / Render / Damage / CompositeDbe / Sync /
              Present / Xkb / XkbSetMap / XInput / XiHierarchy / SyncGrabs / XTest / ScreenSaver / Shm / Glx,
              窗口管理器角色 Ewmh、剪贴板互通 Clipboard、XSETTINGS 管理器 XSettings)
Windowing/    窗口模型(树、几何、属性、事件选择、被动抓取、顶层缓冲、SHAPE 的三种形状)
Resources/    GC、像素图、颜色表、光标、字体句柄、颜色名表、RENDER 的 picture 与字形集
Drawing/      32 位软件帧缓冲、区域(Region)、光栅化:16 种光栅操作、平面掩码、填充样式、裁剪;
              点 / 线(细线 Bresenham + 宽线多边形)/ 矩形 / 多边形扫描线填充 / 弧 / 图像块 / 文字;
              RENDER:像素格式、合成运算与混合模式、取样源(图像 / 纯色 / 渐变,repeat、变换、过滤)、梯形覆盖率
Gl/           GLX 间接渲染的软件 GL:渲染命令解码、显示列表、矩阵栈、光照、裁剪、三角形 / 线 / 点光栅化、纹理、逐片元操作、查询
Fonts/        BDF 解析、内置 misc-fixed 字体、XLFD 名称匹配、合成的 cursor 与 nil2 字体
Input/        键码 ↔ 键值表(evdev 风格键码)、XKB 的 evdev 键名、修饰键映射、抓取的数据结构(核心与 XI2)
Host/         面向宿主的接口:IXServerHost、XTopLevelWindow、XServerOptions、XMonitor、窗口管理器请求与枚举、XKeycodes
```

Unix 套接字:Windows 以外默认监听 `/tmp/.X11-unix/X{N}`,Linux 另在抽象命名空间里监听同名套接字(Xlib / XCB 对 `:N` 先试它);
`XServerOptions.UnixSocketPath` 可指定或关掉,`ListenTcp` 可关掉 TCP。宿主字体提供者接口还没有:核心字体只服务老程序,现代工具包都走 RENDER + 客户端栅格化,接入宿主之后也没有遇到非它不可的程序。

## 5. 线程模型

X 协议的语义是**全局串行**的:服务端按到达顺序逐条执行所有客户端的请求,一个请求的效果对之后的
所有请求可见。所以:

- **一个执行循环**(单线程,`Channel<工作项>`)执行全部请求、宿主输入与定时器。协议状态不加锁 ——
  所有可变状态只在这个线程上被碰。唯一跨线程的是顶层像素,由 `PixelLock` 保护:执行循环每次持锁按 **4 毫秒**
  的时间预算跑一批,然后放锁让宿主拷像素。宿主经 `XTopLevelWindow.ReadPixels` / `CopyPixels` 读像素时先登记「在等」:执行循环每执行完一项就看一眼,
  有人在等就提前放锁,并且等它读完(最多 20 毫秒)再拿 —— `lock` 不公平,执行循环放锁后几微秒内就会再拿,等锁的 UI 线程可能一直抢不到,整个宿主界面跟着卡。
- **宿主回调不在持锁时调**:执行循环里产生的通知(映射、几何、损伤、光标、WM 请求……)先攒进 `DeferredHost`,
  放锁之后按原顺序调用 —— 宿主在回调里同步等 UI 线程、UI 线程又在 `CopyPixels` 里等锁,这种死锁因此不会出现;
  宿主回调抛异常也不会拖垮执行循环。
- 每个连接一个**读取任务**(64 KB 缓冲):按长度字段切出完整请求,交给执行循环;一个**写出任务**:把执行循环
  放进该连接输出队列的消息拼进一块池化缓冲再写。执行循环从不在套接字上阻塞,慢客户端拖不住别人。
- **背压**:每个客户端已读进来、未执行的请求最多 1024 条(读取任务在 `SemaphoreSlim` 上异步等);排队待写的字节超过
  64 MB(客户端不读了)就断开它。断开是真断开 —— `XClient.Abort` 取消连接上挂着的读与写。
- **需要等的都异步等**:XTEST 与 Present 的延迟、SYNC 的计时器用 `Task.Delay`,到点后 `Post` 回执行循环;
  SYNC 的 Await 把该客户端后续请求暂存,条件成立时按原顺序放回。
- **GrabServer** 期间,执行循环只执行持有者的请求,其余客户端的请求原样暂存,Ungrab 后按原顺序放回。
- 损伤区域在一批工作项执行完后合并,一次性通知宿主(不是每个绘图请求一次);每个顶层一批最多累计 8 块矩形,
  超出时合成外接矩形 —— 精确并集在一批几百条请求时退化成 O(n²)。

## 6. 宿主接口(rootless)

库定义接口,宿主实现;方向是**库 → 宿主**的通知,和**宿主 → 库**的注入:

| 方向 | 内容 |
| --- | --- |
| 库 → 宿主 | 顶层窗口映射 / 取消映射 / 销毁;几何变化(客户端 ConfigureWindow);标题(`WM_NAME` / `_NET_WM_NAME`)、类名、瞬态父窗口、override-redirect;非矩形轮廓(`XTopLevelWindow.Shape`,SHAPE 的边界形状,null 为矩形);窗口管理器提示(`WindowType`、`States`、`Decorated` —— 自绘标题栏的窗口为 false、最小 / 最大尺寸与步长、图标、`Urgent`、`AcceptsFocus`、`Opacity`、`ClientFrameExtents`、进程号 / 机器名 / 角色、`HasAlpha`);客户端的窗口管理器请求(`WindowManagerRequest`:移动 / 缩放拖拽、改状态、激活、关闭、最小化 —— 接口的默认实现方法,老宿主不必改);损伤矩形(随后宿主经 `XTopLevelWindow.ReadPixels` 在像素锁里只读这几块,直接写进自己的位图;`CopyPixels` 整窗拷一份,给测试与诊断用);光标形状(cursor 字体字形号,−1 默认箭头,−2 隐藏);响铃;X 客户端复制了文本(`ClipboardChanged`) |
| 宿主 → 库 | 用户移动 / 缩放了原生窗口(库据此改几何并发 ConfigureNotify / Expose);关闭按钮(有 `WM_DELETE_WINDOW` 协议就发 ClientMessage,否则断开该客户端);指针移动 / 按键 / 滚轮(换成 Button 4/5,6 以上是水平滚轮与侧键);按键(X 键码);焦点进出;系统剪贴板有了新文本(`SetClipboardText`);窗口状态与外框尺寸(`SetTopLevelStates` / `SetFrameExtents`,写回 `_NET_WM_STATE` / `_NET_FRAME_EXTENTS`);显示器布局(`SetScreenLayout`,发 RANDR 事件)、DPI 与缩放(`SetDisplayScale`,发 XSETTINGS 与 RESOURCE_MANAGER)、键盘布局(`SetKeyboardMapping`,发 MappingNotify 与 XKB 通知) |

剪贴板互通由 `XServerOptions.SyncClipboard`(CLIPBOARD,默认开)与 `SyncPrimary`(PRIMARY,默认关)控制。

**每个顶层窗口有一块自己的像素缓冲**(相当于常开的 backing store + Composite):子窗口画在所属顶层的
缓冲里,裁剪到自己的可见区域。好处是被别的原生窗口遮住的内容不丢,换来的只是内存 —— 不必在每次
遮挡变化时让客户端重画。只在映射、变大、ClearArea(exposures)与子窗口取消映射露出父窗口时发 Expose。

## 7. 关键取舍

- **视觉只有 TrueColor**:深度 24(根视觉,掩码 `0xff0000 / 0xff00 / 0xff`)与深度 32(ARGB,给 RENDER 留着);
  像素图另可建深度 1 / 4 / 8 / 15 / 16(与 X.Org 的惯例一致 —— Xt 程序会建 4 / 8 深度的像素图,只认 1 / 24 / 32 时它们报 BadValue)。
  像素值就是 RGB,AllocColor 只是换算,不存在「颜色表用完」;可写颜色单元(AllocColorCells)一律 BadAlloc。
- **软件光栅化,不用 Skia**:核心绘图的语义是**逐像素精确**的(细线的 Bresenham 端点、GXxor 橡皮筋、
  平面掩码),抗锯齿的 2D 库给不出同样的像素。宿主拿到的是 32 位像素,怎么贴到屏幕上是宿主的事。
- **键码采用 evdev 编号(evdev + 8)**,与现代 Linux 上的 Xorg 一致 —— 远端客户端大多预期这套编号。
  宿主负责把物理键翻成 X 键码,键值表由库给出(US 布局起步,可替换)。
- **授权**:默认只监听 `127.0.0.1`,没配置 cookie 时按「仅本机」放行(与 X.Org 的主机访问控制行为一致,
  SSH X11 转发过来的连接在本机看来就是 127.0.0.1);配置了 cookie 时要求 `MIT-MAGIC-COOKIE-1` 且常数时间比较。
- **字体**:核心字体来自内置 BDF(`fixed` / `6x13` / `9x15` / `10x20` 等及其 XLFD 名);以后宿主可以经字体提供者
  接口追加(比如把 Cascadia Mono 栅格化成位图字体)。`cursor` 字体是虚拟的:只有度量,光标形状按字形号
  交给宿主映射成系统光标。现代工具包不用核心字体(走 RENDER + 客户端栅格化),所以核心字体只需覆盖老程序。
- **RENDER 通用路径按浮点逐像素合成,占绝大多数的两种情形走整数内核**:预乘 alpha,通用路径每通道 0–1;
  纯色源 + 单字节遮罩 + Over(Xft 画字、cairo 的抗锯齿图形)与 8888 图像 Src / Over(贴图、窗口间拷贝)
  写成 8 位整数运算。只有 alpha 的字形按每像素一字节存;一个 CompositeGlyphs 请求只算一次目标、只记一次损伤。
  梯形与三角形按 16 条子扫描线、水平方向解析地算覆盖率,只算目标上可写的那一块。
  源 picture 的裁剪、alpha-map、poly-edge / poly-mode / dither 接受但不生效。
- **RANDR 对客户端只读,布局由宿主给**:每台显示器一个 CRTC / 输出 / 模式(`XServerOptions.Monitors` 或运行中的
  `SetScreenLayout`),布局变了按 SelectInput 发变更事件;XINERAMA 报同一份布局。客户端改配置的请求回 Failed 或 BadAccess ——
  rootless 模式下窗口摆在哪、显示器怎么排由宿主决定。
- **服务端兼任 XSETTINGS 管理器**:占有 `_XSETTINGS_S0`、发布 `Xft/DPI`、`Gdk/WindowScalingFactor` 等几项,
  并在根窗口发布 RESOURCE_MANAGER(`Xft.dpi`,Xft 与 Qt 从这里读)。真实桌面总有一个设置守护进程,
  GTK / Qt 启动时会去找;真正的守护进程来抢这个选区时照常让出。
- **服务端兼任窗口管理器的协议那一半**:维护 `_NET_SUPPORTED`、`_NET_SUPPORTING_WM_CHECK`、`_NET_CLIENT_LIST`、
  `_NET_ACTIVE_WINDOW`、工作区与顶层的 `WM_STATE` / `_NET_FRAME_EXTENTS` 等;根窗口 ClientMessage 里的请求翻成
  `XWindowManagerRequest` 交给宿主,由宿主决定照不照办、办完用 `SetTopLevelStates` 写回。
  GTK3 的 HeaderBar、Qt 的无边框窗口都依赖这些属性存在。
- **XKB 由核心键位表推出**:不单独维护一份 XKB 键位表,四个规范类型(外加 AltGr 层用的两个四级类型)、修饰键动作、SymInterpret、指示灯、键名都从
  核心表算出来;核心表一变(xmodmap、宿主 `SetKeyboardMapping`),XKB 跟着变并发 MapNotify。
  XKB 的 SetMap 把上传的键值(按 §17 的列序)与修饰键映射写回核心表,再照常推出;SetCompatMap、SetNames 等其余改表请求不支持。
- **XInput2 的设备拓扑可改,但不是完整的多指针**:初始是主指针 2 / 主键盘 3 各挂一个从设备(4、5);XIChangeHierarchy
  可以增删主设备、把从设备挂到别的主设备或让它浮动(浮动时只报从设备的 XI2 事件、不产生核心事件)。
  指针位置、焦点与抓取仍是一份 —— 宿主只有一套物理输入,完整的 MPX 没有用处。XI2 事件与核心事件走同一条传播路径,
  同一个窗口上选了核心的收核心、选了 XI2 的收 XI2。
- **MIT-SHM 只给本机**:只在 Linux 上注册,并且只对经 Unix 套接字连进来的客户端可见 —— 远端经 SSH 来的客户端给的
  shmid 在这台机器上毫无意义。段的大小与属主取自 `/proc/sysvipc/shm`,对端 uid 经 SO_PEERCRED 取得,不是属主 / 创建者
  且权限没对其他人开放就 BadAccess(否则本机客户端可以借服务端之手读写别人的共享内存)。共享像素图与 1.2 的 fd 传递不做。
- **GLX 两条路**:Mesa 在没有 DRI3 / DRI2 时默认走 drisw —— 客户端用 llvmpipe 渲染(GL 4.5)、经 PutImage 送像素,
  服务端只要把配置、上下文、可绘对象登记好;远端经 SSH 转发来的程序同样可用。强制 `LIBGL_ALWAYS_INDIRECT` 时由 `Gl/` 的
  软件 GL 执行固定功能管线的一个子集,版本如实报 1.1(3D 纹理没做);求值器、累积缓冲、选择 / 反馈、mipmap LOD、点画不实现。
  一个 GLX 表面最多 4096 × 4096 像素,一个请求里显示列表展开执行的命令数上限 400 万(列表互相调用会指数级展开)。
- **剪贴板**:宿主 → X 时服务端自己占有 CLIPBOARD 并按 ICCCM 回应;X → 宿主时服务端以一个隐藏的 InputOnly 窗口为
  请求方取回(UTF8_STRING → STRING 退路,支持 INCR)。宿主把刚收到的文本写回来时不抢选区,避免与客户端来回争抢。

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
- **互操作**(`[TestCategory("Interop")]`,默认跳过):`scripts/xserver/interop/` 构建一个带 x11-apps 的容器,
  用例在本机起服务端、让容器里的真实客户端经 `host.docker.internal:N` 连进来,再把顶层窗口的像素存成 PNG
  供人看、并做粗粒度断言(非背景像素数)。
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
- **合成字体**:`cursor`(只有度量,光标形状按字形号交给宿主)与 `nil2`(xterm 的隐形指针用,全空字形)不来自 BDF。
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
- **M3:引擎可选,默认内置**:`ILocalXServer` 由一个选择器实现,按设置在内置与 VcXsrv 之间转发,正在运行的那个优先
  (改了设置不会去停一个正在显示窗口的 X 服务端)。VcXsrv 只在 Windows 上;其它平台只有内置。
- **M3:SSH 的 x11 通道经连接器直接接进服务端**:SSH 库的 `X11ForwardOptions.LocalConnector`
  (`velashell-docs/zh/ssh/spec/07` §7.5.9)每条通道拿一对内存双工流,一端交给 `X11Server.ServeAsync`。假 cookie 的核对照旧,
  服务端按本机连接放行。只在受信模式下用;非受信模式要 `xauth` 连显示,仍走 TCP。服务端照样监听环回 TCP 与 Unix 套接字,
  本机别的 X 程序可以用 `DISPLAY=localhost:N` 连进来。
  **2026-09-24 修正两处**:① 连接器每来一条通道才取**此刻**在运行的服务端,不记住解析显示时的那一个 —— SSH 会话比服务端活得久,
  用户在标题栏把 X Server 停掉再开之后,老会话的每条通道原先都接进已释放的服务端,远端只看到 `Failed to open display`、本机日志一字不记;
  此刻没在运行就拒绝这条通道并记一行日志。② 远端发来 `CHANNEL_EOF` 时连接器那一端也要读到 EOF(spec 07 §7.5.9 新增的一条决策):
  以前远端程序退出后,连接与窗口一直挂到整条 SSH 会话结束。
- **M3:键盘布局跟随 Windows**:宿主按物理键(扫描码)注入 X 键码;Windows 上用系统的 `ToUnicodeEx` 按当前布局算出主键区
  无修饰与 Shift 两层的键值,换进服务端的键位表(布局切换后下次激活 X 窗口时重算)。AltGr 层暂不生成 —— 服务端的 XKB 描述目前只推两层;
  其它平台按 US。
- **M3 真实窗口验证中修掉的**:① 释放像素图时一并销毁建在它上面的 Damage 对象是错的 —— `FreePixmap` 只删 ID,xeyes 用 Present 换帧时
  随后的 `DamageDestroy` 因此回 BadDamage、客户端退出;改为 Damage 随 `DamageDestroy` 或客户端断开释放。② Unix 套接字监听
  在套接字文件已存在时会先删掉它 —— 桌面自己的 Xorg 通常不开 TCP,TCP 那一侧的占用检查看不出它,于是会删掉桌面的套接字;
  改为先试着连一下,有人应答就不碰。③ 宿主侧:显示之后改原生窗口尺寸要设 `Width` / `Height`(设 `ClientSize` 只改属性值);
  我们自己改尺寸引起的 `Resized` 可能晚一拍才到,只把用户拖动与窗口状态变化回报给服务端,否则会拿旧尺寸把客户端刚设的新尺寸改回去。
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
  深度 24、模板 8,没有累积缓冲与多重采样;GLX 1.2 的视觉配置每个视觉一条,取它的双缓冲配置。
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
