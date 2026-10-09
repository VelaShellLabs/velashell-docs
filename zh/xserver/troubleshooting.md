# 内置 X Server 排障:远端图形程序经 SSH 转发

English: [`../../en/xserver/troubleshooting.md`](../../en/xserver/troubleshooting.md)

下面的条目都来自真实排查(最早一批是 2026-09-24,远端 Ubuntu 桌面版上的 `gnome-calculator`,GTK4 + libadwaita)。
分三组:**远端环境**的问题,换 VcXsrv / MobaXterm 也一样;内置 X Server **有意的行为** —— 多个会话共用一个显示时不让它们互相看见、
互相注入(见[三](#三多个会话共用一个显示));宿主与库的**缺陷**,已修。

**远端环境**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 终端里打出 `libEGL warning: DRI3 error: Could not get DRI3 device` | Mesa 先试 DRI3 硬件加速。DRI3 要在同一台机器上传文件描述符,经 SSH 转发不可能做到 | **无害**,Mesa 随即退回软件渲染。不想看见就在远端设 `LIBGL_ALWAYS_SOFTWARE=1` |
| GTK4 程序要等 25 秒、50 秒才出窗口,期间几乎不占 CPU | 桌面门户(`xdg-desktop-portal`)起不来,程序对它的 D-Bus 调用各等满 25 秒超时 | 见[一](#一启动慢一次-25-秒) |
| 窗口出来了,但操作掉帧、不流畅 | GTK4 默认 GL 渲染,远端没有 GPU 时退到 llvmpipe,每一帧把**整窗像素**经 SSH 发过来 | 见[二](#二操作卡) |
| 运行 GTK 程序(实测 `gnome-calculator`)之后终端停住、本机不出窗口,按 Ctrl+C 才回到提示符;VelaShell 的日志里什么也没有 | 远端机器上这个用户正登录着 Wayland 桌面。SSH 会话里没有 `WAYLAND_DISPLAY`,但 `XDG_RUNTIME_DIR` 与桌面是同一个目录,里面有桌面的 `wayland-0`;GTK 先试 Wayland,就连上了它 —— 窗口开到了远端机器自己的屏幕上,没走 X11 转发 | 见[四](#四窗口开到了远端机器自己的桌面上) |

**内置 X Server 有意的行为**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 连接里勾掉了「受信任(`-Y`)」,X11 转发开不起来,终端顶部一行黄字说「内置 X Server 只支持受信任转发」 | 非受信转发要远端的 `xauth` 向本机 X 服务端要一个受限 cookie,这要 X 服务端支持 SECURITY 扩展;内置 X Server 没有。原先照样去试,只看得到 `xauth` 的报错 | 给这个连接勾上「受信任」;或者改用支持 SECURITY 扩展的外部 X 服务器(本机还要装 `xauth`)。受信任意味着什么见[三](#三多个会话共用一个显示) |
| 本机复制的文字在远端 X 程序里粘贴不出来;或者在 X 程序里复制了,本机剪贴板里却没有 | 剪贴板只与**键盘焦点所在的那个会话**互通:焦点在别的会话的 X 窗口上、或者在本机窗口上(此刻没有 X 窗口有焦点)时,读不到、也写不进本机剪贴板 | 先点一下要粘贴的那个程序的窗口再粘贴;同一个 SSH 会话里的 `xclip` / `xsel` 算同一个会话。见[三](#三多个会话共用一个显示) |
| 在 X 程序里按鼠标中键粘不出本机复制的内容;在 X 程序里选中文字也不会自动进本机剪贴板 | 「选中即复制」(X 的 PRIMARY 选区与系统剪贴板互通)默认关 | 需要时在 设置 → X Server → 剪贴板 打开。代价:打开后本机复制的内容在 X 程序里点一下中键就能粘出来 |
| 远端的 `xdotool` 之类模拟输入的工具不工作,或者报没有 XTEST 扩展 | 设置里打开了「限制经 SSH 转发来的程序」:经 SSH 转发来的程序看不到 XTEST、收不到原始按键事件、不能改动输入设备 | 只连你信任的服务器、又需要这些工具时,在 设置 → X Server 关掉它(这是全局设置,对所有会话生效) |
| 点了 X 窗口的关闭按钮,程序没反应;几秒后弹出「程序无响应,要强制结束吗」 | 程序卡住了:它声明了 `_NET_WM_PING`,关闭时服务端随关闭请求 ping 它,5 秒内没回 | 「强制结束」断开这个程序 —— 它的其它窗口一起关闭,未保存的内容会丢。没声明 `_NET_WM_PING` 的程序卡住时不弹这个框,断开它所在的 SSH 会话即可关掉 |
| 所有 X 窗口都点不动、打不了字(常见于远端程序的菜单开着时 SSH 断网、笔记本睡眠、远端进程被挂起) | 那个程序还抓着鼠标 / 键盘(或者用 GrabServer 抓着整个服务端),它的连接没断,抓取就一直在 | 断开那个 SSH 会话(或者等 SSH 保活超时),连接一断抓取就解除。GrabServer 抓着超过 10 秒时,VelaShell 的日志里会点名是哪个连接(带 `user@host:端口`)。宿主界面上「解除卡住」的入口还没有 |
| 老程序启动时报 `Cannot convert string "-b&h-lucida-…" to type FontStruct` 之类,界面退回等宽的 `fixed` | 内置 X Server 随库带的核心字体是 X.Org 的 misc-fixed、cursor、Adobe 75 / 100 dpi 的 Courier / Helvetica / New Century Schoolbook / Symbol / Times 与 GNU Unifont;B&H 的 Lucida(许可要求附带特定声明)、Bitstream 等其余字体与可缩放字体没有带,请求它们照旧 BadName | 在远端的 X 资源(`~/.Xresources`)里把字体换成 `-adobe-helvetica-*` / `-adobe-times-*` / `-adobe-courier-*`;同一字族里没有的字号会退到最接近的。非要那几个字体时改用外部 X 服务器 |

**已修的缺陷**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 在标题栏把 X Server 停掉再开之后,远端报 `Failed to open display` | 宿主缺陷:连接器记住的是停掉之前的那个服务端 | 已修([architecture.md](design/architecture.md) 决策记录「M3:SSH 的 x11 通道经连接器直接接进服务端」);旧版本重连 SSH 即可 |
| 远端程序退出了,本机窗口却一直不关 | 宿主缺陷:远端的 EOF 没传给 X Server | 已修(同上;[SSH spec 07 §7.5.9](../ssh/spec/07-forwarding.md#759-本机显示经连接器接入)) |
| 在设置里把引擎换成 VcXsrv 之后,已经开着的 SSH 会话里新开的 X 程序报 `Failed to open display` | 宿主缺陷:会话开 shell 时绑定的连接器只认内置引擎 | 已修(同上):内置引擎停了之后,老会话的 x11 通道改走本机 TCP,连此刻在运行的那个 X 服务端 |
| Java(Swing / AWT)程序不能最大化;或者最大化、改尺寸之后内容不重排,像是给一个并不存在的标题栏留了位置 | Java 按窗口管理器的名字决定它会不会给窗口套外框,不认得的名字一律当成会套,于是一直等外框、不理尺寸变化 | 已修:内置 X Server 以 `LG3D` 自称(Java 认得的「不套外框」的名字),最大化与改尺寸都照常重排([architecture.md](design/architecture.md) 决策记录「第二轮全库审查的修复」⑤)。把本库嵌进别的程序时别改 `X11ServerOptions.WindowManagerName`;要改先用 Swing 程序验一遍 |
| Motif / Xaw / 不带 Xft 的 Tk 程序界面字体全退回 `fixed`、错位;默认字体的 `xterm` 里中文、希腊文、西里尔文显示不出来 | 库的缺陷:核心字体原先只有 5 个裁剪过的 misc-fixed(只有拉丁字母与制表符),请求 `-adobe-helvetica-*` 一律 BadName | 已修:随库带 X.Org 的整套 misc-fixed(含中日韩的 `12x13ja` / `18x18ja` / `18x18ko`)、Adobe 75 / 100 dpi 字体与 GNU Unifont,别名照 X.Org 的 `fonts.alias`([architecture.md](design/architecture.md) §7「字体」) |
| 录屏 / 远程桌面(`x11vnc`、`ffmpeg -f x11grab`)录不到老程序的鼠标光标;老程序要的铅笔、靶心之类光标在本机显示成箭头 | 库的缺陷:cursor 字体只有度量、没有字形,XFIXES 的 GetCursorImage 回 1×1 的透明像素 | 已修:cursor 字体是 X.Org 的 cursor.bdf,GetCursorImage 给出真实的光标图像;有对应系统光标的(箭头、I 形、沙漏……)本机仍显示系统光标,没有对应的显示 X 的位图 |
| Windows 终端服务器(多个用户同时登录)上,远端程序的窗口、键盘与剪贴板落到了别的用户的桌面上 | 宿主缺陷:判断「本机已经有别的 X 显示在用」时只看 `localhost:0` 有没有人在听,而那可能是别的用户开着的 VcXsrv | 已修:只有当前用户会话里的进程在听才算;否则照常自动启动内置引擎(换一个空闲的显示号) |

## 一、启动慢,一次 25 秒

用 `G_MESSAGES_DEBUG=all gnome-calculator` 看时间戳,卡住的两行是:

```
Gtk-DEBUG:      Failed to get an inhibit portal proxy: ... org.freedesktop.portal.Desktop 调用 StartServiceByName 时出错:已到超时限制
Adwaita-DEBUG:  Settings portal not found: ... 已到超时限制
```

远端的 `journalctl --user -u xdg-desktop-portal` 给出根因:门户前端起来了,但它的 GTK 后端(`xdg-desktop-portal-gtk`)是个图形程序,
要一个显示;只经 SSH 登录、没有桌面会话时,**systemd 用户会话里没有 `DISPLAY`**,后端起不来,一层层等超时。

`GDK_DEBUG=no-portals`、`ADW_DISABLE_PORTAL=1` 只能绕开一部分(有的 GTK 版本在「禁止休眠」那一处不看 `no-portals`)。
根治是把 SSH 会话的显示交给 systemd 用户会话,让门户能正常起来。在远端的 `~/.bashrc` 里加一次,以后每次登录自动生效:

```bash
# 经 SSH 转发 X11 时:把当前会话的显示交给 systemd 用户会话,桌面门户就能正常启动,GTK4 程序不再等 25 秒超时。
# 只在 SSH 登录、且这台机器上没有图形桌面会话时才做,不影响本地桌面。
if [ -n "$SSH_CONNECTION" ] && [ -n "$DISPLAY" ] \
   && ! systemctl --user -q is-active graphical-session.target 2>/dev/null; then
    systemctl --user import-environment DISPLAY
    systemctl --user --no-block stop xdg-desktop-portal-gtk xdg-desktop-portal 2>/dev/null
    export GSK_RENDERER=cairo    # 见下一节
fi
```

- 每次登录的 `DISPLAY` 可能不同(`localhost:10`、`localhost:11`…),所以每次都重新导入,并停掉可能还连着旧显示的门户 ——
  下次被用到时它带着新的显示自动重启。
- `graphical-session.target` 那个判断:有人在那台机器上直接登录了桌面时什么都不做,免得把本地桌面的门户改到 SSH 的显示上去。

实测:`gnome-calculator` 从约 53 秒降到约 3 秒。

## 二、操作卡

GTK4 默认用 GL 渲染。经 SSH 转发时 Mesa 拿不到 GPU,退到 llvmpipe 在远端 CPU 上渲染,再把**整个窗口**的像素经 PutImage 发过来 ——
鼠标划过一个按钮也是一整帧。换成 cairo 渲染后只发变化的部分:

| 远端渲染方式 | 鼠标在按钮上移动 3 秒 | 每帧 |
| --- | --- | --- |
| 默认(GL → llvmpipe) | 约 114 MB,约 39 MB/s | 约 745 KB(整窗像素) |
| `GSK_RENDERER=cairo` | 约 2.9 MB,约 1 MB/s | 约 18 KB |

(`gnome-calculator` 窗口 365×509;端到端测量:Docker 里的 Ubuntu 24.04 sshd,经 VelaShell.Ssh 的 X11 转发接进内置 X Server。)

X Server 这一侧没法替客户端选渲染器 —— Mesa 的软件 EGL 只靠核心协议就能工作,所以设 `GSK_RENDERER=cairo` 是远端的事,
上面那段 `~/.bashrc` 已经带上了。

## 三、多个会话共用一个显示

内置 X Server 只有一个显示(`:N`),所有开了 X11 转发的 SSH 会话都接进它。这正是受信 X11 转发的含义:同一个显示上的程序
能看到彼此的窗口、读剪贴板、往别的窗口里注入输入 —— 一台被攻破的远端机,经它的转发能操作别的会话里的 X 程序。
**不信任的服务器别开 X11 转发。**

在这个前提下,内置 X Server 收紧了几处(细节见 [architecture.md](design/architecture.md) §7「多个会话共用一个显示时的收紧」):

- **剪贴板只与键盘焦点所在的会话互通**:本机复制的内容只有焦点所在的那个会话读得到(同一个 SSH 会话里的 `xclip` / `xsel` 也算),
  X 这边的复制也只收那个会话的;焦点在本机窗口上时谁都读不到。设置页「启用剪贴板」的说明里写着这一点。
  各个会话的剪贴板彼此隔离:一个会话看不到、也读不到另一个会话里的复制。要跨会话复制粘贴,照常在 A 里复制、点一下 B 的窗口再粘贴,
  内容经本机剪贴板中转过去。
- **「选中即复制」默认关**(设置 → X Server → 剪贴板):开着时本机复制的内容在任何 X 程序里点一下中键就能粘出来。
- **「限制经 SSH 转发来的程序」**(设置 → X Server,只在用内置引擎时出现,默认关):打开后经 SSH 转发来的程序不能模拟输入(XTEST)、
  不能监听在 X 窗口里敲的所有按键、不能改动输入设备。连不完全信任的服务器时建议打开;打开后那些服务器上的 `xdotool` 之类会失效。
- **远端程序不能自己跳到前台**:它要求激活自己的窗口时,只有是你刚在 X 窗口里的操作引起的才照办,否则只闪一下任务栏;
  菜单、「总在最前」的窗口只在你正用着 X 窗口时置顶,回到本机窗口时退到后面 —— 免得你在终端里输 sudo 口令时,按键被跳到前台的
  远端窗口接走。
- **远端误跑的窗口管理器**(openbox、xfwm4……)接管不了别的会话的窗口:内置 X Server 自己占着窗口管理器的位置。

## 四、窗口开到了远端机器自己的桌面上

远端机器上同一个用户正登录着 Wayland 桌面时(比如一台装了 Ubuntu 桌面版、开机自动登录的虚拟机),经 SSH 运行 GTK 程序,
终端停住、本机却不出窗口。确认:

```bash
ls $XDG_RUNTIME_DIR | grep wayland                              # 有 wayland-0:这个用户在那台机器上开着 Wayland 桌面
systemctl --user is-active graphical-session.target             # active
WAYLAND_DEBUG=1 gnome-calculator 2>&1 | grep -m1 wl_display     # 有输出:程序连的是 Wayland,不是转发来的 X11 显示
```

SSH 会话里没有 `WAYLAND_DISPLAY`,但登录时 `XDG_RUNTIME_DIR` 被设成与桌面同一个目录(`/run/user/<uid>`)。GTK 先试 Wayland 后端,
而没有 `WAYLAND_DISPLAY` 时 Wayland 的客户端库默认去连这个目录里的 `wayland-0` —— 正好是那台机器的桌面。于是窗口开在远端的屏幕上
(看一眼那台机器的控制台就能看到),`DISPLAY` 指向的 X11 转发根本没用上,内置 X Server 那边也就什么都没收到。换 VcXsrv / MobaXterm 也一样。

在远端的 `~/.bashrc` 里加:

```bash
# 经 SSH 转发 X11 时,让 GTK 程序用转发来的显示,而不是这台机器自己桌面的 Wayland(wayland-0)。
if [ -n "$SSH_CONNECTION" ] && [ -n "$DISPLAY" ] && [ -z "$WAYLAND_DISPLAY" ]; then
    export GDK_BACKEND=x11
fi
```

- 只在 SSH 登录、开了 X11 转发(有 `DISPLAY`)时生效,不影响那台机器的本地桌面。
- 与[一](#一启动慢一次-25-秒)里那段不冲突:那段在有桌面会话时什么也不做(门户本来就起得来),这段正是给有桌面会话的情况用的,两段都加上即可。
- 只想临时试一次:`GDK_BACKEND=x11 gnome-calculator`。
- 用 `G_MESSAGES_DEBUG=all gnome-calculator 2>&1 | head` 看日志时注意:`head` 读够行数就退出,程序下一次往终端写就会被系统结束 ——
  窗口还没来得及出来,看上去像是没起来。