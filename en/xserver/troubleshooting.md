# Built-in X Server troubleshooting: remote GUI programs over SSH forwarding

中文:[`../../zh/xserver/troubleshooting.md`](../../zh/xserver/troubleshooting.md)

Everything below comes from real investigations (the first batch on 2026-09-24, `gnome-calculator` — GTK4 + libadwaita — on a remote
Ubuntu desktop install). There are three groups: problems in the **remote environment**, which look the same with VcXsrv / MobaXterm;
**deliberate behaviour** of the built-in X Server — keeping sessions that share one display from seeing into and typing into each other
(see [3](#3-several-sessions-sharing-one-display)); and **defects** in the host and the library, now fixed.

**Remote environment**

| Symptom | Cause | What to do |
| --- | --- | --- |
| The terminal prints `libEGL warning: DRI3 error: Could not get DRI3 device` | Mesa tries DRI3 hardware acceleration first. DRI3 passes file descriptors on the same machine, which is impossible over SSH forwarding | **Harmless**; Mesa falls back to software rendering right away. To hide it, set `LIBGL_ALWAYS_SOFTWARE=1` on the remote side |
| A GTK4 program takes 25 or 50 seconds to show its window, using almost no CPU meanwhile | The desktop portal (`xdg-desktop-portal`) cannot start, and each of the program's D-Bus calls to it waits for the full 25-second timeout | See [1](#1-slow-start-25-seconds-at-a-time) |
| The window shows up, but interaction drops frames and is not smooth | GTK4 renders with GL by default; without a GPU on the remote side it falls back to llvmpipe and sends the **whole window's pixels** over SSH for every frame | See [2](#2-sluggish-interaction) |
| After starting a GTK program (seen with `gnome-calculator`) the terminal hangs and no window appears locally until you press Ctrl+C; VelaShell's log shows nothing | The same user is logged in to a Wayland desktop on the remote machine. The SSH session has no `WAYLAND_DISPLAY`, but its `XDG_RUNTIME_DIR` is the desktop's directory, which holds the desktop's `wayland-0`; GTK tries Wayland first and connects to it — the window opens on the remote machine's own screen, bypassing X11 forwarding | See [4](#4-the-window-opens-on-the-remote-machines-own-desktop) |

**Deliberate behaviour of the built-in X Server**

| Symptom | Cause | What to do |
| --- | --- | --- |
| With "Trusted (`-Y`)" unticked for the connection, some remote programs misbehave: they cannot take screenshots, `xdotool` does not work, other programs' windows are invisible | That is what untrusted (`ssh -X`) means: the built-in X Server admits such connections at the SECURITY extension's untrusted level, so they cannot touch trusted programs' windows or keyboard or fake input (since 2026-10-10; before, the built-in X Server did not support untrusted mode and such connections did not forward at all) | Tick "Trusted" for connections that need these abilities; to keep trusted sessions from seeing each other too, turn on "One display per SSH session" (see [3](#3-several-sessions-sharing-one-display)) |
| Typing Chinese with the local input method in an X window, the remote program receives Latin letters; or the remote machine has its own fcitx / ibus and the two input methods fight | With "Use this computer's input method in X windows" (Settings → X Server, on by default) off, keys go to the remote side as they are; with it on, text composed by the local input method is typed into the X program directly, with the candidate window at the last click position | Turn it on when the remote machine has no input method; turn it off to use the remote input-method framework. Takes effect after restarting the X Server |
| On Linux / macOS, programs in a container cannot reach the built-in X Server over the network (`DISPLAY=host:N`) | Since 2026-10-10 Linux / macOS open only the Unix socket by default, not the TCP port (local programs use `DISPLAY=:N` and SSH forwarding uses the in-process connector, neither needs it) | Settings → X Server, turn on "Also listen on a TCP port" and restart the X Server |
| GTK programs have no rounded corners or shadows; Electron's transparent windows have a black background | Without a compositing manager toolkits do not use visuals with alpha | Settings → X Server, turn on "Transparent X windows" (off by default: windows with alpha cost the system an extra compositing layer) and restart the X Server |
| A remote program that pulls the pointer back to the middle (continuous dragging in Blender, CAD rotation, game camera control) still lets it run off on macOS or Wayland | When a program warps the pointer the host moves the system cursor along, so far only on Windows and on Linux X11 desktops; macOS is not done yet and Wayland does not let programs move the cursor | No workaround yet; Windows works as expected |
| Running a full desktop such as `startxfce4` in an SSH session, the window manager does not start, or the desktop window covers the whole local screen | In the default multi-window mode the built-in X Server is the window manager itself and shows desktop windows as ordinary windows | Settings → X Server, set "Window mode" to one large window / no title bar / fullscreen and restart the X Server: the whole X desktop lives in an "X Desktop :N" window and the remote window manager takes over |
| "The X programs froze for a moment" and you want to know who did it | One program's request took too long and every program was waiting for it | Look in the log for `watchdog:` lines (one per item while it is still running, naming the program and opcode) and `slow work item` lines |
| Text copied on this computer cannot be pasted into a remote X program, or text copied in an X program never reaches the local clipboard | The clipboard is exchanged only with **the session that has the keyboard focus**: while the focus is in another session's X window, or in a local window (no X window has the focus at that moment), nothing can be read from or written to the local clipboard | Click the window of the program you want to paste into first, then paste; `xclip` / `xsel` in the same SSH session count as the same session. See [3](#3-several-sessions-sharing-one-display) |
| A middle click in an X program does not paste what was copied locally, and selecting text in an X program no longer puts it on the local clipboard | "Copy on selection" (exchanging X's PRIMARY selection with the system clipboard) is off by default | Turn it on under Settings → X Server → Clipboard if you need it. The cost: whatever you copy locally can then be pasted into any X program with a middle click |
| Remote input tools such as `xdotool` do not work, or complain that the XTEST extension is missing | "Restrict programs from SSH sessions" is on in the settings: programs forwarded over SSH cannot see XTEST, receive no raw key events and cannot change input devices | If you only connect to servers you trust and need these tools, turn it off under Settings → X Server (it is a global setting that applies to every session) |
| Clicking an X window's close button does nothing, and after a few seconds a "Not Responding — force quit?" dialog appears | The program is stuck: it advertises `_NET_WM_PING`, the server pinged it along with the close request, and no reply came within 5 seconds | "Force Quit" disconnects the program — its other windows close too, and unsaved work is lost. A stuck program that does not advertise `_NET_WM_PING` gets no such dialog; disconnecting its SSH session closes it |
| No X window reacts to clicks or keys (typically after the SSH link dropped, the laptop slept or the remote process was suspended while one of its menus was open) | That program still holds a pointer / keyboard grab (or holds the whole server with GrabServer), and since its connection has not dropped, the grab stays | Disconnect that SSH session (or wait for the SSH keep-alive to time out); the grab goes when the connection does. When GrabServer is held for more than 10 seconds, VelaShell's log names the connection (with `user@host:port`). There is no "unstick" entry in the host UI yet |
| An older program reports something like `Cannot convert string "-b&h-lucida-…" to type FontStruct` at startup, and its UI falls back to the monospaced `fixed` | The core fonts bundled with the built-in X Server are X.Org's misc-fixed, cursor, Adobe's 75 / 100 dpi Courier / Helvetica / New Century Schoolbook / Symbol / Times, and GNU Unifont; B&H's Lucida (its licence requires particular notices), Bitstream and other X.Org fonts and scalable fonts are not bundled, so requests for them still get BadName | Switch the fonts in the remote X resources (`~/.Xresources`) to `-adobe-helvetica-*` / `-adobe-times-*` / `-adobe-courier-*`; sizes missing from a family fall back to the nearest one. If you really need those fonts, use an external X server |

**Fixed defects**

| Symptom | Cause | What to do |
| --- | --- | --- |
| After stopping and restarting the X Server from the title bar, the remote side reports `Failed to open display` | Host defect: the connector remembered the server that was running before the stop | Fixed ([architecture.md](design/architecture.md), decision log entry "M3: SSH x11 channels go straight into the server through a connector"); on older versions, reconnect the SSH session |
| The remote program exited, but its local window stays open | Host defect: the remote EOF was not passed on to the X Server | Fixed (same entry; [SSH spec 07 §7.5.9](../ssh/spec/07-forwarding.md)) |
| After switching the engine to VcXsrv in the settings, X programs started from SSH sessions that were already open report `Failed to open display` | Host defect: the connector bound when the session opened its shell only knew the built-in engine | Fixed (same entry): once the built-in engine has stopped, x11 channels of existing sessions go over local TCP to whichever X server is running now |
| Java (Swing / AWT) programs cannot maximize, or after maximize / resize their contents do not relayout, as if leaving room for a title bar that is not there | Java decides from the window manager's name whether the window manager wraps windows in a frame, and assumes it does for any name it does not know — so it keeps waiting for the frame and ignores size changes | Fixed: the built-in X Server calls itself `LG3D` (a name Java knows does not wrap windows), so maximize and resize relayout as usual ([architecture.md](design/architecture.md), decision log entry "Fixes from the second library-wide review", ⑤). When embedding the library elsewhere, do not change `X11ServerOptions.WindowManagerName`; if you must, verify with a Swing program first |
| Motif / Xaw / Tk-without-Xft programs fall back to `fixed` for every UI font and lay out badly; a default-font `xterm` cannot show Chinese, Greek or Cyrillic | Library defect: the core fonts used to be just five trimmed misc-fixed fonts (Latin letters and box drawing only), and `-adobe-helvetica-*` always got BadName | Fixed: X.Org's whole misc-fixed set (including the CJK `12x13ja` / `18x18ja` / `18x18ko`), Adobe's 75 / 100 dpi fonts and GNU Unifont ship with the library, and aliases follow X.Org's `fonts.alias` ([architecture.md](design/architecture.md) §7 "Fonts") |
| Screen recording / remote desktop (`x11vnc`, `ffmpeg -f x11grab`) does not capture older programs' mouse cursor; cursors such as the pencil or target show up locally as an arrow | Library defect: the `cursor` font had metrics but no glyphs, and XFIXES GetCursorImage returned a 1×1 transparent pixel | Fixed: the `cursor` font is X.Org's cursor.bdf and GetCursorImage returns the real image; cursors with a matching system cursor (arrow, I-beam, hourglass …) still show the system cursor locally, the others show the X bitmap |
| On a Windows terminal server (several users logged in at once), remote programs' windows, keyboard and clipboard end up on another user's desktop | Host defect: deciding whether "another X display is already in use" looked only at whether something listened on `localhost:0`, which may be another user's VcXsrv | Fixed: only a listener process in the current user's session counts; otherwise the built-in engine starts automatically as usual (on another free display number) |

## 1. Slow start, 25 seconds at a time

With `G_MESSAGES_DEBUG=all gnome-calculator`, the timestamps show two stalls:

```
Gtk-DEBUG:      Failed to get an inhibit portal proxy: ... StartServiceByName for org.freedesktop.portal.Desktop: Timeout was reached
Adwaita-DEBUG:  Settings portal not found: ... Timeout was reached
```

`journalctl --user -u xdg-desktop-portal` on the remote side gives the root cause: the portal front end starts, but its GTK back
end (`xdg-desktop-portal-gtk`) is a GUI program and needs a display. When the user only logs in over SSH and there is no desktop
session, **the systemd user session has no `DISPLAY`**, so the back end cannot start and every layer waits for its timeout.

`GDK_DEBUG=no-portals` and `ADW_DISABLE_PORTAL=1` only get around part of it (some GTK versions ignore `no-portals` at the
"inhibit" call). The real fix is to hand the SSH session's display to the systemd user session so the portal can start
normally. Add this once to `~/.bashrc` on the remote side; it applies on every login from then on:

```bash
# With SSH X11 forwarding: hand this session's display to the systemd user session so the desktop portal can start,
# and GTK4 programs stop waiting for 25-second timeouts.
# Only for SSH logins, and only when this machine has no graphical desktop session, so the local desktop is unaffected.
if [ -n "$SSH_CONNECTION" ] && [ -n "$DISPLAY" ] \
   && ! systemctl --user -q is-active graphical-session.target 2>/dev/null; then
    systemctl --user import-environment DISPLAY
    systemctl --user --no-block stop xdg-desktop-portal-gtk xdg-desktop-portal 2>/dev/null
    export GSK_RENDERER=cairo    # see the next section
fi
```

- `DISPLAY` may differ on every login (`localhost:10`, `localhost:11`, …), so it is imported again each time, and a portal that
  may still hold the old display is stopped — it restarts with the new display the next time something needs it.
- The `graphical-session.target` check: when someone is logged in to the desktop on that machine, do nothing, so the local
  desktop's portal is not redirected to the SSH display.

Measured: `gnome-calculator` went from about 53 seconds to about 3 seconds.

## 2. Sluggish interaction

GTK4 renders with GL by default. Over SSH forwarding Mesa has no GPU, falls back to llvmpipe on the remote CPU, and sends the
**whole window's** pixels with PutImage — even a mouse passing over one button costs a full frame. With cairo rendering only the
changed parts are sent:

| Remote rendering | Mouse moving over buttons for 3 seconds | Per frame |
| --- | --- | --- |
| Default (GL → llvmpipe) | about 114 MB, about 39 MB/s | about 745 KB (whole-window pixels) |
| `GSK_RENDERER=cairo` | about 2.9 MB, about 1 MB/s | about 18 KB |

(`gnome-calculator` window 365×509; end-to-end measurement: an Ubuntu 24.04 sshd in Docker, forwarded through VelaShell.Ssh's X11
forwarding into the built-in X Server.)

The X Server cannot choose the renderer for the client — Mesa's software EGL works with the core protocol alone — so setting
`GSK_RENDERER=cairo` is a remote-side setting; the `~/.bashrc` snippet above already includes it.

## 3. Several sessions sharing one display

The built-in X Server has a single display (`:N`), and every SSH session with X11 forwarding connects to it. That is exactly what
trusted X11 forwarding means: programs on the same display can see each other's windows, read the clipboard and inject input into
other windows — a compromised remote machine can, through its forwarding, operate X programs from other sessions.
**Do not enable X11 forwarding for servers you do not trust.**

Within that, the built-in X Server tightens a few things (details in [architecture.md](design/architecture.md) §7, "Several sessions
sharing one display — what is tightened"):

- **The clipboard is exchanged only with the session that has the keyboard focus**: what you copy locally can be read only by that
  session (`xclip` / `xsel` in the same SSH session included), and only copies from that session reach the local clipboard; while the
  focus is in a local window nobody can read it. The description of "Enable clipboard" on the settings page says so.
  The sessions' clipboards are isolated from each other: one session can neither see nor read what was copied in another. To copy and paste
  across sessions, copy in A as usual, click a window of B and paste — the content goes through the local clipboard.
- **"Copy on selection" is off by default** (Settings → X Server → Clipboard): when it is on, whatever you copy locally can be pasted
  into any X program with a middle click.
- **"Restrict programs from SSH sessions"** (Settings → X Server, shown only with the built-in engine, off by default): when on, programs
  forwarded over SSH cannot simulate input (XTEST), cannot listen to every key typed in X windows, and cannot change input devices.
  Recommended when connecting to servers you do not fully trust; tools such as `xdotool` on those servers stop working.
- **Remote programs cannot jump to the front by themselves**: when one asks for its window to be activated, that is honoured only if it
  was caused by what you just did in an X window; otherwise the taskbar entry just flashes. Menus and "always on top" windows stay on top
  only while you are using an X window and drop behind when you return to a local window — so the keys of a sudo password typed in a
  terminal cannot be taken by a remote window that jumped to the front.
- **A window manager started by mistake on a remote machine** (openbox, xfwm4…) cannot take over the other sessions' windows: the built-in
  X Server holds the window manager's place itself. (Except in one-window mode, where the remote side is meant to be the window manager; see
  the "full desktop" row in the table above.)
- **Untrusted forwarding** (connections with "Trusted" unticked, supported by the built-in X Server since 2026-10-10): such sessions' programs
  cannot touch trusted programs' windows, keyboard or clipboard, or fake input.
- **One display per SSH session** (Settings → X Server, only shown with the built-in engine, off by default, since 2026-10-10): when on,
  programs from different SSH sessions get separate displays, so even trusted sessions cannot see, read or inject into each other; X programs on
  this computer keep using the shared display. The cost is a little memory and a thread per session.

## 4. The window opens on the remote machine's own desktop

When the same user is logged in to a Wayland desktop on the remote machine (for example a VM with Ubuntu Desktop that logs in
automatically at boot), a GTK program started over SSH leaves the terminal hanging while no window appears locally. To confirm:

```bash
ls $XDG_RUNTIME_DIR | grep wayland                              # wayland-0 present: this user has a Wayland desktop open on that machine
systemctl --user is-active graphical-session.target             # active
WAYLAND_DEBUG=1 gnome-calculator 2>&1 | grep -m1 wl_display     # any output: the program talks Wayland, not the forwarded X11 display
```

The SSH session has no `WAYLAND_DISPLAY`, but at login its `XDG_RUNTIME_DIR` is set to the same directory as the desktop's
(`/run/user/<uid>`). GTK tries the Wayland back end first, and without `WAYLAND_DISPLAY` the Wayland client library connects to
`wayland-0` in that directory by default — which is that machine's desktop. So the window opens on the remote screen (look at that
machine's console to see it), the X11 forwarding that `DISPLAY` points at is never used, and the built-in X Server never hears from the
program. VcXsrv / MobaXterm behave the same.

Add this to `~/.bashrc` on the remote side:

```bash
# With SSH X11 forwarding: make GTK programs use the forwarded display instead of this machine's own Wayland desktop (wayland-0).
if [ -n "$SSH_CONNECTION" ] && [ -n "$DISPLAY" ] && [ -z "$WAYLAND_DISPLAY" ]; then
    export GDK_BACKEND=x11
fi
```

- It only applies to SSH logins with X11 forwarding (`DISPLAY` set), and leaves that machine's local desktop alone.
- It does not conflict with the snippet in [1](#1-slow-start-25-seconds-at-a-time): that one does nothing when a desktop session exists
  (the portal can start anyway), and this one is exactly for that case — add both.
- To try it just once: `GDK_BACKEND=x11 gnome-calculator`.
- When reading logs with `G_MESSAGES_DEBUG=all gnome-calculator 2>&1 | head`, note that `head` exits once it has its lines, and the
  program is killed the next time it writes to the terminal — before its window has had a chance to appear, so it looks as if it never
  started.