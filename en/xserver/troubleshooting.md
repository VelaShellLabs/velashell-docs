# Built-in X Server troubleshooting: remote GUI programs over SSH forwarding

中文:[`../../zh/xserver/troubleshooting.md`](../../zh/xserver/troubleshooting.md)

Everything below comes from a real investigation (2026-09-24, `gnome-calculator` — GTK4 + libadwaita — on a remote Ubuntu
desktop install). The first two are problems in the **remote environment** and look the same with VcXsrv / MobaXterm; the last
one was a host defect and is fixed.

| Symptom | Cause | What to do |
| --- | --- | --- |
| The terminal prints `libEGL warning: DRI3 error: Could not get DRI3 device` | Mesa tries DRI3 hardware acceleration first. DRI3 passes file descriptors on the same machine, which is impossible over SSH forwarding | **Harmless**; Mesa falls back to software rendering right away. To hide it, set `LIBGL_ALWAYS_SOFTWARE=1` on the remote side |
| A GTK4 program takes 25 or 50 seconds to show its window, using almost no CPU meanwhile | The desktop portal (`xdg-desktop-portal`) cannot start, and each of the program's D-Bus calls to it waits for the full 25-second timeout | See [1](#1-slow-start-25-seconds-at-a-time) |
| The window shows up, but interaction drops frames and is not smooth | GTK4 renders with GL by default; without a GPU on the remote side it falls back to llvmpipe and sends the **whole window's pixels** over SSH for every frame | See [2](#2-sluggish-interaction) |
| After stopping and restarting the X Server from the title bar, the remote side reports `Failed to open display` | Host defect: the connector remembered the server that was running before the stop | Fixed ([architecture.md](design/architecture.md), decision log entry "M3: SSH x11 channels go straight into the server through a connector"); on older versions, reconnect the SSH session |
| The remote program exited, but its local window stays open | Host defect: the remote EOF was not passed on to the X Server | Fixed (same entry; [SSH spec 07 §7.5.9](../ssh/spec/07-forwarding.md)) |

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
