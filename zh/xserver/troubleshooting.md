# 内置 X Server 排障:远端图形程序经 SSH 转发

English: [`../../en/xserver/troubleshooting.md`](../../en/xserver/troubleshooting.md)

下面几条都来自真实排查(2026-09-24,远端 Ubuntu 桌面版上的 `gnome-calculator`,GTK4 + libadwaita)。
前两条是**远端环境**的问题,换 VcXsrv / MobaXterm 也一样;最后一条是宿主的缺陷,已修。

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 终端里打出 `libEGL warning: DRI3 error: Could not get DRI3 device` | Mesa 先试 DRI3 硬件加速。DRI3 要在同一台机器上传文件描述符,经 SSH 转发不可能做到 | **无害**,Mesa 随即退回软件渲染。不想看见就在远端设 `LIBGL_ALWAYS_SOFTWARE=1` |
| GTK4 程序要等 25 秒、50 秒才出窗口,期间几乎不占 CPU | 桌面门户(`xdg-desktop-portal`)起不来,程序对它的 D-Bus 调用各等满 25 秒超时 | 见[一](#一启动慢一次-25-秒) |
| 窗口出来了,但操作掉帧、不流畅 | GTK4 默认 GL 渲染,远端没有 GPU 时退到 llvmpipe,每一帧把**整窗像素**经 SSH 发过来 | 见[二](#二操作卡) |
| 在标题栏把 X Server 停掉再开之后,远端报 `Failed to open display` | 宿主缺陷:连接器记住的是停掉之前的那个服务端 | 已修([architecture.md](design/architecture.md) 决策记录「M3:SSH 的 x11 通道经连接器直接接进服务端」);旧版本重连 SSH 即可 |
| 远端程序退出了,本机窗口却一直不关 | 宿主缺陷:远端的 EOF 没传给 X Server | 已修(同上;[SSH spec 07 §7.5.9](../ssh/spec/07-forwarding.md#759-本机显示经连接器接入)) |

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
