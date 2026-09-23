# VelaShell.XServer Architecture and Rationale

中文:[`../../../zh/xserver/design/architecture.md`](../../../zh/xserver/design/architecture.md)

> Status: **M1 (core protocol) and M2 (modern toolkits) complete**; project started 2026-09-23. Not yet wired into the
> host — today the host's "X Server" button launches the VcXsrv the user installed (see [`../../host/interaction-and-ui-specs.md`](../../host/interaction-and-ui-specs.md) §4A.2).
> Once M3 wires this library into the host and replaces that path, users no longer need to install anything.

## 1. Why build it

The host needs to show GUI programs from remote hosts arriving through SSH X11 forwarding. The survey (2026-09-23) found
**no usable X server library in the .NET ecosystem**: X11.Net is an Xlib binding (client side); yserver (Rust, MIT) only
runs on Linux DRM/KMS and cannot be embedded; node-x11's `lib/xserver` (JS, MIT) draws into a browser canvas; WeirdX
(Java) is GPL. Bundling VcXsrv only solves Windows and adds tens of MB to the installer, which conflicts with the
unzip-and-run distribution model.

Hence this library: an **embeddable, native-dependency-free, cross-platform** X11 server. The host draws its
top-level windows as native windows through Avalonia (rootless), giving one code path for Windows / macOS / Linux.

## 2. Goals and non-goals

**Goals**

- Implement the **core protocol** of the X Window System Protocol, Version 11 (all 119 core requests), plus the
  extensions modern toolkits actually depend on (see milestones in §8).
- **Rootless multi-window**: every top-level X window maps to one native host window; the root window is never drawn.
  Decorations, moving, resizing and closing are the host's job (it acts as the window manager).
- **Embeddable**: no UI framework and no native library dependency. Pixels, window lifecycle and input are exchanged
  through one host interface (§6).
- **Testable**: the protocol layer is unit-tested over in-memory transports; `[TestCategory("Interop")]` cases let real
  Xlib / XCB clients (`xdpyinfo`, `xterm`, `xeyes`… in Docker) connect.

**Non-goals**

- No **real display hardware** (DRM/KMS, framebuffer devices) or hardware input — that belongs to a full X server;
  we only build the embeddable half.
- No XDMCP, no multiple screens (Screen > 1), no indexed colour / writable colormaps (24-bit TrueColor only).
- No GLX / DRI3 / Present in M1–M2.

## 3. Clean-room rules

The same discipline as `VelaShell.Ssh` (see `src/VelaShell.XServer/AGENTS.md` in the host repository):

1. **Only public specifications are implementation sources**: X.Org's *X Window System Protocol, X Version 11*
   (including Appendix B, the encoding), the extension specifications (*BIG-REQUESTS*, *XC-MISC*,
   *X Nonrectangular Window Shape Extension*, *X Fixes Extension*, *The X Resize and Rotate Extension*,
   *The X Rendering Extension* (and the PDF Reference blend-mode formulas it cites), *The X Keyboard Extension:
   Protocol Specification*, *MIT-SHM*, …), ICCCM, EWMH and freedesktop.org's XSETTINGS specification. **Every protocol
   file's header names the specification and section it implements.**
2. **No other X server's source is opened while implementing** (X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv).
3. Constants from the specifications (opcodes, event codes, error codes, predefined atoms, mask bits) take their
   specified values — they are protocol facts, not copyrightable, and must not be changed to "look different".
4. **Data is not code**: the built-in bitmap fonts are BDF files from X.Org's `font-misc-misc` (copyright notice:
   "Public domain font. Share and enjoy."), shipped as data files with their source recorded in `NOTICE.md`.

## 4. Layering

```
Protocol/     Constants (opcodes, event codes, error codes, masks, predefined atoms), byte-order-aware request
              reading and reply / event / error writing
Server/       X11Server: listening (TCP 6000+N) and ServeAsync (any duplex stream), connection setup and
              authorization (MIT-MAGIC-COOKIE-1 / local only), single-threaded execution loop, client table and
              sequence numbers, GrabServer, BIG-REQUESTS lengths; request handlers split into partial files by area
              (Windows / Exposure / Events / Properties / Graphics / Text / Colors / Input / Extensions, plus one per
              extension: Shape / XFixes / RandR / Render, clipboard exchange in Clipboard, the XSETTINGS manager in XSettings)
Windowing/    Window model (tree, geometry, attributes, event selections, passive grabs, top-level buffer, the three SHAPE shapes)
Resources/    GCs, pixmaps, colormaps, cursors, font handles, colour-name table, RENDER pictures and glyph sets
Drawing/      32-bit software framebuffer, regions, rasterizer: 16 raster ops, plane mask, fill styles, clipping;
              points / lines (thin Bresenham + wide-line polygons) / rectangles / scan-line polygon fill / arcs /
              image blocks / text; RENDER: pixel formats, compositing operators and blend modes, sources (image / solid /
              gradients with repeat, transform, filter), trapezoid coverage
Fonts/        BDF parsing, built-in misc-fixed fonts, XLFD name matching, synthesized cursor and nil2 fonts
Input/        Keycode ↔ keysym table (evdev-style keycodes), modifier mapping, grab data structures
Host/         Host-facing API: IXServerHost, XTopLevelWindow, XServerOptions, XKeycodes
```

Unix-socket listening (`/tmp/.X11-unix/XN`) and a host font-provider interface do not exist yet — the former comes
with the Linux / macOS integration (`ServeAsync` already accepts any stream), the latter when M3 needs larger sizes.

## 5. Threading model

X semantics are **globally serial**: the server executes every client's requests one by one in arrival order, and
each request's effects are visible to everything after it. Therefore:

- **One execution loop** (single thread, `Channel<work item>`) runs all requests, host input and timers. No locks —
  all mutable state is touched only on that thread.
- One **reader task** per connection cuts complete requests by their length field and hands them to the loop; one
  **writer task** writes the bytes the loop queued for that connection to the socket. The loop never blocks on a
  socket, so a slow client cannot stall the others.
- During **GrabServer** the loop runs only the holder's requests; other clients' requests are parked as they are and
  replayed in order after the ungrab.
- Damage is merged after a batch of work items and reported to the host once (not once per drawing request).

## 6. Host interface (rootless)

The library defines the interface and the host implements it; notifications flow **library → host**, injection flows
**host → library**:

| Direction | Content |
| --- | --- |
| Library → host | Top-level window mapped / unmapped / destroyed; geometry changes (client ConfigureWindow); title (`WM_NAME` / `_NET_WM_NAME`), class, transient parent, override-redirect; non-rectangular outline (`XTopLevelWindow.Shape`, the SHAPE bounding shape, null when rectangular); damage rectangles (the host then copies from that window's pixel buffer); cursor shape (cursor-font glyph number, −1 default arrow, −2 hidden); bell; an X client copied text (`ClipboardChanged`) |
| Host → library | The user moved / resized the native window (the library updates geometry and sends ConfigureNotify / Expose); close button (ClientMessage when `WM_DELETE_WINDOW` is advertised, otherwise the client is disconnected); pointer motion / buttons / wheel (as buttons 4/5); keys (X keycodes); focus in / out; the system clipboard has new text (`SetClipboardText`) |

Clipboard exchange is controlled by `XServerOptions.SyncClipboard` (CLIPBOARD, on by default) and `SyncPrimary`
(PRIMARY, off by default).

**Every top-level window owns a pixel buffer** (effectively always-on backing store + Composite): child windows draw
into their top-level's buffer, clipped to their visible region. Content hidden behind other native windows is never
lost, at the cost of memory — clients need not redraw whenever occlusion changes. Expose is only sent on map, growth,
ClearArea(exposures) and when an unmapped child reveals its parent.

## 7. Key trade-offs

- **TrueColor visuals only**: depth 24 (the root visual, masks `0xff0000 / 0xff00 / 0xff`) and depth 32 (ARGB, reserved
  for RENDER); pixmaps may additionally have depth 1 / 4 / 8 / 15 / 16 (following X.Org's convention — Xt programs create
  depth-4 / 8 pixmaps and got BadValue while only 1 / 24 / 32 were allowed). Pixel values are RGB, AllocColor is just a
  conversion and colormaps never "run out"; writable colour cells (AllocColorCells) are always BadAlloc.
- **Software rasterization, not Skia**: core drawing semantics are **pixel-exact** (thin-line Bresenham endpoints,
  GXxor rubber bands, plane masks), which anti-aliasing 2D libraries cannot reproduce. The host receives 32-bit pixels;
  how they reach the screen is the host's business.
- **Keycodes use evdev numbering (evdev + 8)**, like X.Org on modern Linux — most remote clients expect it. The host
  translates physical keys to X keycodes; the keysym table comes from the library (US layout to start, replaceable).
- **Authorization**: listens on `127.0.0.1` by default; without a configured cookie it admits local connections only
  (matching X.Org's host access control — connections arriving through SSH X11 forwarding come from 127.0.0.1); with a
  cookie it requires `MIT-MAGIC-COOKIE-1`, compared in constant time.
- **Fonts**: core fonts come from the built-in BDFs (`fixed` / `6x13` / `9x15` / `10x20` and their XLFD names); later the
  host may add more through a font-provider interface (e.g. rasterizing Cascadia Mono into bitmap fonts). The `cursor`
  font is virtual: metrics only, and the host maps glyph numbers to system cursors. Modern toolkits do not use core
  fonts (they use RENDER with client-side rasterization), so core fonts only need to cover older programs.
- **RENDER composites per pixel in floating point**: premultiplied alpha, 0–1 per channel, with two fast paths (solid
  Src / opaque Over fill whole spans; fully transparent source pixels under Over / Add are skipped). Trapezoids and
  triangles use 16 sub-scanlines per row with analytic horizontal coverage. Correctness first; specialize per format
  only if it becomes a bottleneck. Source-picture clipping, alpha maps, poly-edge / poly-mode / dither are accepted but
  have no effect.
- **RANDR is read-only**: one virtual monitor covering the whole root window. In rootless mode the host decides where
  windows go; clients only ask about monitors to learn size and DPI.
- **The server doubles as the XSETTINGS manager**: it owns `_XSETTINGS_S0` and publishes `Xft/DPI` and a few more.
  Real desktops always run a settings daemon and GTK / Qt look for one at startup; a real daemon taking the selection
  over is let through.
- **Clipboard**: host → X, the server itself owns CLIPBOARD and answers per ICCCM; X → host, the server fetches the text
  with a hidden InputOnly window as requestor (UTF8_STRING with STRING fallback, INCR supported). When the host writes
  back the text it just received, the server does not take the selection, so the two sides never fight over it.

## 8. Milestones

| Milestone | Content | Acceptance |
| --- | --- | --- |
| **M1 core protocol** ✅ | All core requests; BIG-REQUESTS, XC-MISC; windows / events / properties / selections; software drawing; built-in fonts; keyboard mapping; headless test host | `xdpyinfo`, `xterm`, `xeyes`, `xclock`, `xlogo` in Docker connect, draw, and cause zero protocol errors (achieved, see §10) |
| **M2 modern toolkits** ✅ | SHAPE, XFIXES, RANDR (read-only), RENDER; clipboard exchange with the host; XSETTINGS manager | Simple GTK3 / Qt5 programs work (`zenity`, `gedit`, `qt5ct` draw with zero protocol errors — achieved, see §10) |
| **M3 host integration** | Avalonia host (native windows, input, HiDPI); replace the VcXsrv path; trim the settings page | The host's "X Server" button no longer depends on an external program |
| M4 | XKB and XInput2 (originally planned for M2, see §10), MIT-SHM (only meaningful on the same machine, low priority), GLX (indirect rendering), synchronous grabs | As needed |

## 9. Test strategy

- **Unit tests** (`tests/VelaShell.XServer.Tests/`): in-memory duplex streams as transport, plus a minimal "protocol
  client" that builds requests byte by byte and parses replies and events. Drawing cases assert framebuffer pixels
  directly.
- **Interop** (`[TestCategory("Interop")]`, skipped by default): `scripts/xserver/interop/` builds a container with
  x11-apps; the tests start the server locally, let real clients in the container connect via
  `host.docker.internal:N`, save top-level pixels as PNG for humans and make coarse assertions (non-background pixels).
- When the specification and the implementation disagree, **the specification wins**: fix the implementation and
  record it in §10.

## 10. Decision log

- **2026-09-23 project start**: library named `VelaShell.XServer`, placed in the host repository at
  `src/VelaShell.XServer/`, MIT licensed, clean-room rules as for `VelaShell.Ssh`.
- **The entry class is `X11Server`, not `XServer`**: the latter shares its name with the namespace `VelaShell.XServer`,
  so any code inside that namespace (tests included) would resolve `XServer` to the namespace.
- **Only asynchronous grabs**: SyncPointer / SyncKeyboard and the freeze / release of AllowEvents are treated as
  asynchronous. Synchronous grabs are mostly used by window managers and a few popup-menu implementations; the host
  itself is the window manager, and M1 does not justify an event-freezing queue.
- **Focus PointerRoot is represented by the root window**: `SetInputFocus(root)` behaves like
  `SetInputFocus(PointerRoot)`. When the host activates a native window, focus goes to the corresponding top-level
  (revert-to PointerRoot), as window managers commonly do.
- **Expose errs on the side of more, not less**: when a child moves or restacks we repaint and send Expose for both the
  old and new areas instead of moving the old contents — clients must handle Expose anyway; one extra means one extra
  redraw, one missing means a dirty patch on screen.
- **Synthesized fonts**: `cursor` (metrics only; the host maps glyph numbers to cursor shapes) and `nil2` (xterm's
  invisible pointer, all-blank glyphs) do not come from BDF files.
- **M1 acceptance (2026-09-23)**: 40 unit tests; real clients `xdpyinfo` / `xterm` / `xeyes` / `xclock` / `xlogo` cause
  zero protocol errors and draw content; keyboard injection round-trips through xterm to `sh` in the container.
- **Extension numbering**: major opcodes are assigned from 128 in implementation order — BIG-REQUESTS 128, XC-MISC 129,
  SHAPE 130, XFIXES 131, RANDR 132, RENDER 133; event codes SHAPE 64, XFIXES 65–66, RANDR 67–68; error codes XFIXES 128,
  RANDR 129–132, RENDER 133–137. Clients always obtain them through `QueryExtension`; these values are only this
  implementation's allocation. Server-owned resources (RANDR's CRTC / output / mode, RENDER's formats, the hidden window
  used for selections) take `0x40`–`0x56`, outside every client's resource base.
- **XKB and XInput2 moved out of M2**: in practice Qt5 without XKB falls back to the core-protocol keymap (printing one
  `XKeyboard extension not present` line), and GTK3 without XInput2 uses core pointer events; keyboard and clicks work.
  Neither **can be done halfway** — once an extension shows up in `QueryExtension` clients switch to its code path, and
  if any of XKB's `GetMap` / `GetNames` / `GetCompatMap` replies is wrong, xkbcommon-x11 fails to build a keymap and
  the keyboard gets worse than today. If it is done, it must go all the way to `xkb_x11_keymap_new_from_device`
  succeeding. Qt6's source keeps the same core-keymap fallback, but that is not yet verified in practice.
- **XSETTINGS comes from the server**: GTK / Qt look for the owner of `_XSETTINGS_S0` at startup and each hit a
  BadWindow / BadAtom when there is none. The server owns it and publishes only font-rendering settings; HiDPI's
  `Gdk/WindowScalingFactor` and friends come with the M3 host integration.
- **Diagnostic log carries a request trail**: when `XServerOptions.Log` prints a protocol error it appends the opcodes
  of that client's last 8 requests — that is how the two errors above were pinned down.
- **M2 acceptance (2026-09-23)**: 73 unit tests; 8 interop cases (new: `xclock -render`, `xeyes -render`, `xterm`
  with an Xft font), all with zero protocol errors; verified by hand: `zenity` (GTK3), `gedit` (GTK3, typing via
  key injection) and `qt5ct` (Qt5) with zero protocol errors, `xclip` both directions (including 12 MB of text),
  `xrandr` reads the configuration.
