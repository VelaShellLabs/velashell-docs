# VelaShell.XServer 架构与原理

English: [`../../../en/xserver/design/architecture.md`](../../../en/xserver/design/architecture.md)

> 状态:**M1(核心协议)与 M2(现代工具包)已完成**,2026-09-23 立项。宿主尚未接入 —— 目前宿主的「X Server」
> 按钮拉起的是用户自己装的 VcXsrv(见 [`../../host/交互与界面规格.md`](../../host/交互与界面规格.md) §4A.2)。
> M3 把本库接进宿主、替换那条路径之后,用户就不必再装任何东西。

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
  关闭由宿主(即扮演窗口管理器)负责。
- **可嵌入**:库不依赖任何 UI 框架、不依赖原生库。像素、窗口生命周期与输入经一个宿主接口(§6)交换。
- **可测**:协议层全部走内存传输单测;另有 `[TestCategory("Interop")]` 用例让真实的 Xlib / XCB 客户端
  (Docker 里的 `xdpyinfo`、`xterm`、`xeyes`…)连进来。

**非目标**

- 不做**真实显示设备**(DRM/KMS、帧缓冲设备)与硬件输入 —— 那是完整 X 服务端的事,我们只做嵌入式的那一半。
- 不做 XDMCP、多屏幕(Screen > 1)、索引色 / 可写颜色表(只提供 24 位 TrueColor)。
- M1–M2 不做 GLX / DRI3 / Present。

## 3. 净室规程

与 `VelaShell.Ssh` 同一套纪律(见宿主仓库 `src/VelaShell.XServer/AGENTS.md`):

1. **实现依据只能是公开规范**:X.Org 发布的 *X Window System Protocol, X Version 11*(含附录 B 编码)、
   各扩展规范(*BIG-REQUESTS*、*XC-MISC*、*X Nonrectangular Window Shape Extension*、*X Fixes Extension*、
   *The X Resize and Rotate Extension*、*The X Rendering Extension*(及其引用的 PDF Reference 混合模式公式)、
   *The X Keyboard Extension: Protocol Specification*、*MIT-SHM* 等)、ICCCM、EWMH 与 freedesktop.org 的 XSETTINGS 规范。
   **每个协议实现文件的头部写明它实现的是哪份规范的哪一节。**
2. **写实现时不打开任何其它 X 服务端的源码**(X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv)。
3. 规范里的常量(操作码、事件码、错误码、预定义原子、掩码位)按规范取值 —— 那是协议,不受版权保护,
   也不许为了「看起来不一样」去改。
4. **数据不等于代码**:内置位图字体取自 X.Org `font-misc-misc` 的 BDF(版权声明原文为
   "Public domain font. Share and enjoy."),以数据文件随库分发并在 `NOTICE.md` 里注明来源。

## 4. 分层

```
Protocol/     常量(操作码、事件码、错误码、掩码、预定义原子)、字节序感知的请求读取与回复 / 事件 / 错误写出
Server/       X11Server:监听(TCP 6000+N)与 ServeAsync(任意双工流)、连接建立与授权(MIT-MAGIC-COOKIE-1 / 仅本机)、
              单线程执行循环、客户端表与序号、GrabServer、BIG-REQUESTS 长度;请求处理按领域拆成 partial 文件
              (Windows / Exposure / Events / Properties / Graphics / Text / Colors / Input / Extensions,
              以及各扩展:Shape / XFixes / RandR / Render,剪贴板互通 Clipboard、XSETTINGS 管理器 XSettings)
Windowing/    窗口模型(树、几何、属性、事件选择、被动抓取、顶层缓冲、SHAPE 的三种形状)
Resources/    GC、像素图、颜色表、光标、字体句柄、颜色名表、RENDER 的 picture 与字形集
Drawing/      32 位软件帧缓冲、区域(Region)、光栅化:16 种光栅操作、平面掩码、填充样式、裁剪;
              点 / 线(细线 Bresenham + 宽线多边形)/ 矩形 / 多边形扫描线填充 / 弧 / 图像块 / 文字;
              RENDER:像素格式、合成运算与混合模式、取样源(图像 / 纯色 / 渐变,repeat、变换、过滤)、梯形覆盖率
Fonts/        BDF 解析、内置 misc-fixed 字体、XLFD 名称匹配、合成的 cursor 与 nil2 字体
Input/        键码 ↔ 键值表(evdev 风格键码)、修饰键映射、抓取的数据结构
Host/         面向宿主的接口:IXServerHost、XTopLevelWindow、XServerOptions、XKeycodes
```

Unix 套接字监听(`/tmp/.X11-unix/XN`)与宿主字体提供者接口都还没有 —— 前者在 Linux / macOS 接入时再加
(`ServeAsync` 已经能接任何流),后者等 M3 需要更大字号时再加。

## 5. 线程模型

X 协议的语义是**全局串行**的:服务端按到达顺序逐条执行所有客户端的请求,一个请求的效果对之后的
所有请求可见。所以:

- **一个执行循环**(单线程,`Channel<工作项>`)执行全部请求、宿主输入与定时器。不加锁 ——
  所有可变状态只在这个线程上被碰。
- 每个连接一个**读取任务**:按长度字段切出完整请求,交给执行循环;一个**写出任务**:把执行循环
  放进该连接输出队列的字节写到套接字。执行循环从不在套接字上阻塞,慢客户端拖不住别人。
- **GrabServer** 期间,执行循环只执行持有者的请求,其余客户端的请求原样暂存,Ungrab 后按原顺序放回。
- 损伤区域在一批工作项执行完后合并,一次性通知宿主(不是每个绘图请求一次)。

## 6. 宿主接口(rootless)

库定义接口,宿主实现;方向是**库 → 宿主**的通知,和**宿主 → 库**的注入:

| 方向 | 内容 |
| --- | --- |
| 库 → 宿主 | 顶层窗口映射 / 取消映射 / 销毁;几何变化(客户端 ConfigureWindow);标题(`WM_NAME` / `_NET_WM_NAME`)、类名、瞬态父窗口、override-redirect;非矩形轮廓(`XTopLevelWindow.Shape`,SHAPE 的边界形状,null 为矩形);损伤矩形(随后宿主从该窗口的像素缓冲拷贝);光标形状(cursor 字体字形号,−1 默认箭头,−2 隐藏);响铃;X 客户端复制了文本(`ClipboardChanged`) |
| 宿主 → 库 | 用户移动 / 缩放了原生窗口(库据此改几何并发 ConfigureNotify / Expose);关闭按钮(有 `WM_DELETE_WINDOW` 协议就发 ClientMessage,否则断开该客户端);指针移动 / 按键 / 滚轮(换成 Button 4/5);按键(X 键码);焦点进出;系统剪贴板有了新文本(`SetClipboardText`) |

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
- **RENDER 按浮点逐像素合成**:预乘 alpha,每通道 0–1,加两条快路径(纯色 Src / 不透明 Over 整块填;Over / Add
  下全透明的源像素跳过)。梯形与三角形按 16 条子扫描线、水平方向解析地算覆盖率。正确性优先,真成瓶颈再按格式特化。
  源 picture 的裁剪、alpha-map、poly-edge / poly-mode / dither 接受但不生效。
- **RANDR 只读**:一台覆盖整个根窗口的虚拟显示器。rootless 模式下窗口摆在哪由宿主决定,客户端问显示器只为取尺寸与 DPI。
- **服务端兼任 XSETTINGS 管理器**:占有 `_XSETTINGS_S0`、发布 `Xft/DPI` 等几项。真实桌面总有一个设置守护进程,
  GTK / Qt 启动时会去找;真正的守护进程来抢这个选区时照常让出。
- **剪贴板**:宿主 → X 时服务端自己占有 CLIPBOARD 并按 ICCCM 回应;X → 宿主时服务端以一个隐藏的 InputOnly 窗口为
  请求方取回(UTF8_STRING → STRING 退路,支持 INCR)。宿主把刚收到的文本写回来时不抢选区,避免与客户端来回争抢。

## 8. 里程碑

| 里程碑 | 内容 | 验收 |
| --- | --- | --- |
| **M1 核心协议** ✅ | 全部核心请求;BIG-REQUESTS、XC-MISC;窗口 / 事件 / 属性 / 选区;软件绘图;内置字体;键盘映射;无头测试宿主 | Docker 里 `xdpyinfo`、`xterm`、`xeyes`、`xclock`、`xlogo` 连上、画出内容、零协议错误(已达成,见 §10) |
| **M2 现代工具包** ✅ | SHAPE、XFIXES、RANDR(只读)、RENDER;剪贴板与宿主互通;XSETTINGS 管理器 | GTK3 / Qt5 的简单程序可用(`zenity`、`gedit`、`qt5ct` 画出内容、零协议错误 —— 已达成,见 §10) |
| **M3 接入宿主** | Avalonia 宿主(原生窗口、输入、HiDPI);替换 VcXsrv 路径;设置页收敛 | 宿主「X Server」按钮不再依赖外部程序 |
| M4 | XKB 与 XInput2(原计划在 M2,见 §10)、MIT-SHM(同机才有意义,低优先)、GLX(间接渲染)、同步抓取 | 视需求 |

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
  选区用的隐藏窗口)取 `0x40`–`0x56`,落在任何客户端的 resource-base 之外。
- **XKB 与 XInput2 移出 M2**:实测 Qt5 没有 XKB 时退回核心协议的键位表(只打一行 `XKeyboard extension not present`),
  GTK3 没有 XInput2 时用核心指针事件,键盘与点击都正常。两者都**不能只做一半** —— 扩展一出现在 `QueryExtension` 里,
  客户端就改走它的代码路径;XKB 的 `GetMap` / `GetNames` / `GetCompatMap` 任何一个回复不对,xkbcommon-x11 建键位表失败,
  键盘反而更糟。要做就一次做到 `xkb_x11_keymap_new_from_device` 能成功。Qt6 按其源码同样保留了核心键位表的退路,但还没实测。
- **XSETTINGS 由服务端提供**:GTK / Qt 启动时找 `_XSETTINGS_S0` 的属主,找不到会各踩一次 BadWindow / BadAtom。
  服务端占有它、只发布字体渲染相关的几项;HiDPI 的 `Gdk/WindowScalingFactor` 等 M3 接入宿主时再加。
- **诊断日志带请求轨迹**:`XServerOptions.Log` 打印协议错误时附上该客户端最近 8 条请求的操作码 —— 上面那两个错误就是这样定位的。
- **M2 验收(2026-09-23)**:单元测试 73 条;interop 用例 8 条(新增 `xclock -render`、`xeyes -render`、Xft 字体的 `xterm`),
  全部零协议错误;手动验证 `zenity`(GTK3)、`gedit`(GTK3,键盘注入打字)、`qt5ct`(Qt5)零协议错误,
  `xclip` 双向互通(含 12 MB 文本),`xrandr` 读得到配置。
