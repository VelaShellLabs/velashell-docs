# VelaShell.XServer Architecture and Rationale

中文:[`../../../zh/xserver/design/architecture.md`](../../../zh/xserver/design/architecture.md)

> Status: **M1 (core protocol), M2 (modern toolkits), the "feature-complete" round (a dozen more extensions including XKB and
> XInput2, the window-manager role, a library-wide performance review), M3 (host integration) and M4 (synchronous grabs, device
> topology, XKB mapping changes, MIT-SHM, GLX) complete**; project started
> 2026-09-23. The host's "X Server" button starts this library by default (one Avalonia native window per X window) and SSH X11
> forwarding connects straight into it; the VcXsrv the user installed becomes an optional engine on Windows (see
> [`../../host/interaction-and-ui-specs.md`](../../host/interaction-and-ui-specs.md) §4A.2 and §14).

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
  Decorations, moving, resizing and closing are the host's job; what a window manager does at the protocol level
  (EWMH / ICCCM properties, parsing client hints, receiving clients' WM requests) is done by the server, which then
  hands the requests to the host (§6).
- **Embeddable**: no UI framework and no native library dependency. Pixels, window lifecycle and input are exchanged
  through one host interface (§6).
- **Testable**: the protocol layer is unit-tested over in-memory transports; `[TestCategory("Interop")]` cases let real
  Xlib / XCB clients (`xdpyinfo`, `xterm`, `xeyes`… in Docker) connect.

**Non-goals**

- No **real display hardware** (DRM/KMS, framebuffer devices) or hardware input — that belongs to a full X server;
  we only build the embeddable half.
- No XDMCP, no multiple screens (Screen > 1), no indexed colour / writable colormaps (24-bit TrueColor only).
- No DRI2 / DRI3 (there is no GPU to hand to clients). GLX only registers direct rendering (the client renders in software
  itself and sends pixels with PutImage) and offers software indirect rendering of a fixed-function GL subset; MIT-SHM exists
  only on Linux and only for local clients connected over a Unix socket. Present does software copies only (there is no video
  memory to flip).

## 3. Clean-room rules

The same discipline as `VelaShell.Ssh` (see `src/VelaShell.XServer/AGENTS.md` in the host repository):

1. **Only public specifications are implementation sources**: X.Org's *X Window System Protocol, X Version 11*
   (including Appendix B, the encoding), the extension specifications (*BIG-REQUESTS*, *XC-MISC*,
   *X Nonrectangular Window Shape Extension*, *X Fixes Extension*, *The X Resize and Rotate Extension*,
   *The X Rendering Extension* (and the PDF Reference blend-mode formulas it cites), *The X Keyboard Extension:
   Protocol Specification*, *The X Input Extension* (1.5 and 2.2), *XTEST Extension*, *X Synchronization Extension*,
   *X Damage Extension*, *Composite Extension*, *Double Buffer Extension*, *The Present Extension*, *MIT-SCREEN-SAVER*,
   *DPMS*, *X-Resource*, *Generic Event Extension*, *The MIT Shared Memory Extension* 1.1, and XINERAMA — which has no
   standalone specification document, so its wire format follows the protocol definitions X.Org publishes in panoramiXproto),
   ICCCM, EWMH and freedesktop.org's XSETTINGS specification; GLX follows Khronos' *OpenGL Graphics with the X Window System*
   1.4, the *GLX Extensions for OpenGL Protocol Specification* 1.3 (the encoding) and *The OpenGL Graphics System* 1.5 (GL
   semantics for indirect rendering), with opcodes and enum values taken from the Khronos registry's `gl.xml` / `glx.xml`.
   **Every protocol file's header names the specification and section it implements.**
2. **No other X server's source is opened while implementing** (X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv),
   and no OpenGL / GLX implementation's source either (Mesa and the like).
3. Constants from the specifications (opcodes, event codes, error codes, predefined atoms, mask bits) take their
   specified values — they are protocol facts, not copyrightable, and must not be changed to "look different".
4. **Data is not code**: the built-in bitmap fonts are BDF files from X.Org's `font-misc-misc` (copyright notice:
   "Public domain font. Share and enjoy."), shipped as data files with their source recorded in `NOTICE.md`.

## 4. Layering

```
Protocol/     Constants (opcodes, event codes, error codes, masks, predefined atoms), byte-order-aware request
              reading and reply / event / error writing
Server/       X11Server: listening (TCP 6000+N, Unix sockets) and ServeAsync (any duplex stream), connection setup and
              authorization (MIT-MAGIC-COOKIE-1 / local only), single-threaded execution loop, connection backpressure,
              deferred host callbacks (DeferredHost), client table and sequence numbers, GrabServer, BIG-REQUESTS lengths;
              request handlers split into partial files by area (Windows / Exposure / Events / Properties / Graphics /
              Text / Colors / Input / Extensions / Queries, plus one per extension: Shape / XFixes / RandR / Monitors /
              Render / Damage / CompositeDbe / Sync / Present / Xkb / XkbSetMap / XInput / XiHierarchy / SyncGrabs /
              XTest / ScreenSaver / Shm / Glx, the window-manager role in Ewmh, clipboard exchange in Clipboard, the
              XSETTINGS manager in XSettings)
Windowing/    Window model (tree, geometry, attributes, event selections, passive grabs, top-level buffer, the three SHAPE shapes)
Resources/    GCs, pixmaps, colormaps, cursors, font handles, colour-name table, RENDER pictures and glyph sets
Drawing/      32-bit software framebuffer, regions, rasterizer: 16 raster ops, plane mask, fill styles, clipping;
              points / lines (thin Bresenham + wide-line polygons) / rectangles / scan-line polygon fill / arcs /
              image blocks / text; RENDER: pixel formats, compositing operators and blend modes, sources (image / solid /
              gradients with repeat, transform, filter), trapezoid coverage
Gl/           Software GL for GLX indirect rendering: render-command decoding, display lists, matrix stacks, lighting,
              clipping, triangle / line / point rasterization, textures, per-fragment operations, queries
Fonts/        BDF parsing, built-in misc-fixed fonts, XLFD name matching, synthesized cursor and nil2 fonts
Input/        Keycode ↔ keysym table (evdev-style keycodes), XKB evdev key names, modifier mapping, grab data structures (core and XI2)
Host/         Host-facing API: IXServerHost, XTopLevelWindow, XServerOptions, XMonitor, window-manager requests and enums, XKeycodes
```

Unix sockets: outside Windows the server listens on `/tmp/.X11-unix/X{N}` by default, and on Linux also on the same
name in the abstract namespace (Xlib / XCB try that first for `:N`); `XServerOptions.UnixSocketPath` sets or disables it
and `ListenTcp` can turn TCP off. A host font-provider interface does not exist yet: core fonts only serve older programs, modern toolkits use RENDER with client-side rasterization, and no program has needed it since the host integration.

## 5. Threading model

X semantics are **globally serial**: the server executes every client's requests one by one in arrival order, and
each request's effects are visible to everything after it. Therefore:

- **One execution loop** (single thread, `Channel<work item>`) runs all requests, host input and timers. Protocol state
  takes no locks — all mutable state is touched only on that thread. The one thing shared across threads is top-level
  pixels, guarded by `PixelLock`: the loop holds it for a batch within a **4 ms** time budget, then releases it so the
  host can copy pixels. A host reading pixels through `XTopLevelWindow.ReadPixels` / `CopyPixels` first registers that it is waiting:
  after every work item the loop checks, releases early if someone is waiting, and re-takes the lock only after the read finishes
  (bounded at 20 ms) — `lock` is unfair, the loop re-takes it within microseconds of releasing it, and a waiting UI thread could lose
  that race again and again, freezing the whole host UI.
- **Host callbacks are never made while holding the lock**: notifications produced by the loop (map, geometry, damage,
  cursor, WM requests…) are queued in `DeferredHost` and invoked in order after the lock is released — so a host that
  synchronously waits for its UI thread inside a callback, while that UI thread waits for the lock in `CopyPixels`,
  cannot deadlock; a host callback that throws does not take the loop down either.
- One **reader task** per connection (64 KB buffer) cuts complete requests by their length field and hands them to the
  loop; one **writer task** packs the messages the loop queued for that connection into one pooled buffer and writes it.
  The loop never blocks on a socket, so a slow client cannot stall the others.
- **Backpressure**: at most 1024 read-but-not-executed requests per client (the reader waits asynchronously on a
  `SemaphoreSlim`); a client whose queued output exceeds 64 MB (it stopped reading) is disconnected. Disconnecting is
  real — `XClient.Abort` cancels the pending read and write on the connection.
- **Everything that waits, waits asynchronously**: XTEST and Present delays and SYNC timers use `Task.Delay` and `Post`
  back to the loop when due; SYNC's Await parks that client's later requests and replays them in order once satisfied.
- During **GrabServer** the loop runs only the holder's requests; other clients' requests are parked as they are and
  replayed in order after the ungrab.
- Damage is merged after a batch of work items and reported to the host once (not once per drawing request); each
  top-level accumulates at most 8 rectangles per batch and falls back to their bounding box beyond that — an exact
  union degrades to O(n²) over a batch of a few hundred requests.

## 6. Host interface (rootless)

The library defines the interface and the host implements it; notifications flow **library → host**, injection flows
**host → library**:

| Direction | Content |
| --- | --- |
| Library → host | Top-level window mapped / unmapped / destroyed; geometry changes (client ConfigureWindow); title (`WM_NAME` / `_NET_WM_NAME`), class, transient parent, override-redirect; non-rectangular outline (`XTopLevelWindow.Shape`, the SHAPE bounding shape, null when rectangular); window-manager hints (`WindowType`, `States`, `Decorated` — false for windows that draw their own title bar, min / max size and increments, icons, `Urgent`, `AcceptsFocus`, `Opacity`, `ClientFrameExtents`, process id / machine / role, `HasAlpha`); clients' window-manager requests (`WindowManagerRequest`: interactive move / resize, state changes, activate, close, minimize — a default interface method, so existing hosts need no change); damage rectangles (the host then reads just those rectangles through `XTopLevelWindow.ReadPixels` under the pixel lock, straight into its own bitmaps; `CopyPixels` copies the whole window, for tests and diagnostics); cursor shape (cursor-font glyph number, −1 default arrow, −2 hidden); bell; an X client copied text (`ClipboardChanged`) |
| Host → library | The user moved / resized the native window (the library updates geometry and sends ConfigureNotify / Expose); close button (ClientMessage when `WM_DELETE_WINDOW` is advertised, otherwise the client is disconnected); pointer motion / buttons / wheel (as buttons 4/5; 6 and up are horizontal wheel and side buttons); keys (X keycodes); focus in / out; the system clipboard has new text (`SetClipboardText`); window states and frame extents (`SetTopLevelStates` / `SetFrameExtents`, written back to `_NET_WM_STATE` / `_NET_FRAME_EXTENTS`); monitor layout (`SetScreenLayout`, sends RANDR events), DPI and scale (`SetDisplayScale`, updates XSETTINGS and RESOURCE_MANAGER), keyboard layout (`SetKeyboardMapping`, sends MappingNotify and XKB notifications) |

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
- **RENDER's general path composites per pixel in floating point; the two dominant cases use integer kernels**:
  premultiplied alpha, 0–1 per channel on the general path; solid source + one-byte mask + Over (Xft text, cairo's
  anti-aliased shapes) and 8888 image Src / Over (image blits, window-to-window copies) run in 8-bit integer
  arithmetic. Alpha-only glyphs are stored one byte per pixel; one CompositeGlyphs request computes its target once
  and records damage once. Trapezoids and triangles use 16 sub-scanlines per row with analytic horizontal coverage,
  computed only over the writable part of the target. Source-picture clipping, alpha maps, poly-edge / poly-mode /
  dither are accepted but have no effect.
- **RANDR is read-only for clients; the host supplies the layout**: one CRTC / output / mode per monitor
  (`XServerOptions.Monitors` or `SetScreenLayout` at runtime), with change events sent per SelectInput when the layout
  changes; XINERAMA reports the same layout. Clients' configuration requests get Failed or BadAccess — in rootless mode
  the host decides where windows go and how monitors are arranged.
- **The server doubles as the XSETTINGS manager**: it owns `_XSETTINGS_S0` and publishes `Xft/DPI`,
  `Gdk/WindowScalingFactor` and a few more, and publishes RESOURCE_MANAGER on the root window (`Xft.dpi`, read by Xft
  and Qt). Real desktops always run a settings daemon and GTK / Qt look for one at startup; a real daemon taking the
  selection over is let through.
- **The server does the protocol half of a window manager**: it maintains `_NET_SUPPORTED`, `_NET_SUPPORTING_WM_CHECK`,
  `_NET_CLIENT_LIST`, `_NET_ACTIVE_WINDOW`, the work area and per-top-level `WM_STATE` / `_NET_FRAME_EXTENTS` and so on;
  requests in root-window ClientMessages become `XWindowManagerRequest`s for the host, which decides whether to honour
  them and writes the result back with `SetTopLevelStates`. GTK3's HeaderBar and Qt's frameless windows depend on these
  properties being present.
- **XKB is derived from the core keymap**: there is no separately maintained XKB keymap — the four canonical types (plus two four-level types for the AltGr level),
  modifier actions, SymInterprets, indicators and key names are all computed from the core table; when the core table
  changes (xmodmap, the host's `SetKeyboardMapping`), XKB follows and sends MapNotify. XKB's SetMap writes the uploaded
  keysyms (in the §17 column order) and modifier map back into the core table, which is then derived as usual; SetCompatMap,
  SetNames and the other mapping-change requests are not supported.
- **XInput2's device topology can change, but this is not full multi-pointer X**: it starts with master pointer 2 / master
  keyboard 3, each with one slave (4, 5); XIChangeHierarchy adds and removes master devices and attaches slaves to other
  masters or floats them (a floating slave reports only slave XI2 events and generates no core events). Pointer position,
  focus and grabs remain single — the host has one set of physical input, so full MPX would buy nothing. XI2 events travel
  the same propagation path as core events — on a given window, a core selection receives core events and an XI2 selection
  receives XI2 events.
- **MIT-SHM is for the local machine only**: it is registered only on Linux and is visible only to clients connected over a
  Unix socket — a shmid from a remote client forwarded over SSH means nothing on this machine. Segment size and owner come
  from `/proc/sysvipc/shm` and the peer uid from SO_PEERCRED; a peer that is neither owner nor creator, on a segment not
  opened to others, gets BadAccess (otherwise a local client could read and write someone else's shared memory through the
  server). Shared pixmaps and 1.2's fd passing are not implemented.
- **Two GLX paths**: without DRI3 / DRI2, Mesa defaults to drisw — the client renders with llvmpipe (GL 4.5) and sends
  pixels with PutImage, so the server only has to register configs, contexts and drawables; programs forwarded over SSH
  work the same way. With `LIBGL_ALWAYS_INDIRECT` forced, the software GL in `Gl/` executes a subset of the fixed-function
  pipeline and honestly reports version 1.1 (3D textures are not implemented); evaluators, the accumulation buffer,
  selection / feedback, mipmap LOD and stippling are not implemented. A GLX surface is at most 4096 × 4096 pixels, and one
  request may expand at most 4 million display-list commands (lists calling each other expand exponentially).
- **Clipboard**: host → X, the server itself owns CLIPBOARD and answers per ICCCM; X → host, the server fetches the text
  with a hidden InputOnly window as requestor (UTF8_STRING with STRING fallback, INCR supported). When the host writes
  back the text it just received, the server does not take the selection, so the two sides never fight over it.

## 8. Milestones

| Milestone | Content | Acceptance |
| --- | --- | --- |
| **M1 core protocol** ✅ | All core requests; BIG-REQUESTS, XC-MISC; windows / events / properties / selections; software drawing; built-in fonts; keyboard mapping; headless test host | `xdpyinfo`, `xterm`, `xeyes`, `xclock`, `xlogo` in Docker connect, draw, and cause zero protocol errors (achieved, see §10) |
| **M2 modern toolkits** ✅ | SHAPE, XFIXES, RANDR (read-only), RENDER; clipboard exchange with the host; XSETTINGS manager | Simple GTK3 / Qt5 programs work (`zenity`, `gedit`, `qt5ct` draw with zero protocol errors — achieved, see §10) |
| **Feature-complete** ✅ | XKEYBOARD, XInputExtension 2.2, XTEST, XINERAMA, SYNC, DAMAGE, Composite, DOUBLE-BUFFER, Present, MIT-SCREEN-SAVER, DPMS, X-Resource, Generic Event; the window-manager role; Unix sockets; runtime layout / DPI / keyboard-layout changes; library-wide performance review | xkbcomp, xinput, xdotool, xprintidle, xrestop read and write correctly; gedit (GTK3) and qt5ct (Qt5) run on XKB and XI2 with zero protocol errors (achieved, see §10) |
| **M3 host integration** ✅ | Avalonia host (native windows, input, HiDPI, window-manager requests, clipboard, Windows keyboard layout); engine choice (built-in by default, VcXsrv optional); SSH x11 channels go straight into the server through a connector; trim the settings page | The host's "X Server" button no longer depends on an external program; real xterm / xeyes / gedit / qt5ct become native windows, and moving, resizing, closing and typing work (achieved, see §10) |
| **M4** ✅ | Synchronous grabs (freezing, and release / single-step / replay through AllowEvents / XIAllowEvents); XIChangeHierarchy; XKB SetMap; MIT-SHM 1.1 (Linux, local clients); GLX 1.4 (registration of direct rendering + software indirect rendering) | `xinput create-master / reattach / float / remove-master`, `setxkbmap … \| xkbcomp - $DISPLAY`, `x11perf -shmput10`, `glxinfo` / `glxgears` on both the direct and the indirect path with zero protocol errors (achieved, see §10) |

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
  used for selections) take a range starting at `0x40`, outside every client's resource base.
- **XKB and XInput2 moved out of M2**: in practice Qt5 without XKB falls back to the core-protocol keymap (printing one
  `XKeyboard extension not present` line), and GTK3 without XInput2 uses core pointer events; keyboard and clicks work.
  Neither **can be done halfway** — once an extension shows up in `QueryExtension` clients switch to its code path, and
  if any of XKB's `GetMap` / `GetNames` / `GetCompatMap` replies is wrong, xkbcommon-x11 fails to build a keymap and
  the keyboard gets worse than today. If it is done, it must go all the way to `xkb_x11_keymap_new_from_device`
  succeeding. Qt6's source keeps the same core-keymap fallback, but that is not yet verified in practice.
- **XSETTINGS comes from the server**: GTK / Qt look for the owner of `_XSETTINGS_S0` at startup and each hit a
  BadWindow / BadAtom when there is none. The server owns it and publishes only font-rendering settings; HiDPI's
  `Gdk/WindowScalingFactor` and friends were added later in the feature-complete round (see below).
- **Diagnostic log carries a request trail**: when `XServerOptions.Log` prints a protocol error it appends the opcodes
  of that client's last 8 requests — that is how the two errors above were pinned down.
- **M2 acceptance (2026-09-23)**: 73 unit tests; 8 interop cases (new: `xclock -render`, `xeyes -render`, `xterm`
  with an Xft font), all with zero protocol errors; verified by hand: `zenity` (GTK3), `gedit` (GTK3, typing via
  key injection) and `qt5ct` (Qt5) with zero protocol errors, `xclip` both directions (including 12 MB of text),
  `xrandr` reads the configuration.
- **Extension numbering in the feature-complete round** (continuing from M2): GE 134, XTEST 135, XINERAMA 136,
  MIT-SCREEN-SAVER 137 (event 69), DPMS 138, X-Resource 139, SYNC 140 (events 70–71, errors 138–140), DAMAGE 141
  (event 72, error 141), Composite 142, DOUBLE-BUFFER 143 (error 142), Present 144 (events via GE), XInputExtension 145
  (events from 74, errors 144–148), XKEYBOARD 146 (event 73, error 143). Server-owned resources add SYNC's SERVERTIME /
  IDLETIME counters and Composite's overlay window; the selection window doubles as `_NET_SUPPORTING_WM_CHECK`.
- **XKB and XInput2 were finally done in full**: following the "cannot be done halfway" judgement above, XKB was only
  switched on once xkbcommon-x11 built a keymap (`xkbcomp` dumps a self-consistent 436-line keymap), and XI2 once GTK3's
  gedit typed and opened menus under XI2. Real clients exposed during debugging: GetGeometry (found = False) returned
  extra data (xkbcomp: Extra reply data), missing SymInterprets (compatibility map not defined), missing
  `_XKB_RULES_NAMES` (setxkbmap), and the initial focus must be PointerRoot (xdotool typed nothing).
- **Window-manager requests reach the host through a default interface method**: `IXServerHost.WindowManagerRequest`
  has an empty default implementation, so existing host implementations need no change; `_NET_MOVERESIZE_WINDOW` and
  `_NET_REQUEST_FRAME_EXTENTS`, which need no decision from the host, are handled by the server directly.
- **Host callbacks deferred until after the lock is released**: the host used to be called directly from the loop while
  it held the lock. A host that synchronously waited for its UI thread in a callback, while that UI thread waited for
  `PixelLock` in `CopyPixels`, deadlocked both sides. Callbacks now queue in `DeferredHost` and run after the release.
- **Library-wide performance review (2026-09-23)**: in-process throughput benchmark `scripts/xserver/bench/bench.cs`.
  Main changes — bounded damage accumulation (the former exact union was the top cost of all drawing), visibility cached
  by generation, RENDER integer kernels and byte glyphs, I/O buffering and pooling, request queuing without closures, an
  active-edge-table polygon fill (a single 65535-sized arc used to occupy the loop for tens of seconds). Before → after on
  the same machine: fill 2,748 → 205,000 ops/s, PutImage 671 → 7,800, Xft glyphs (10 per request) 865 → 45,000, ARGB
  composite 591 → 26,000, pointer motion 510k → 1.03M, round trips 34k → 87k (the "after" numbers are warmed-up steady
  state; the same code differs by about 1.4× between cold and warm).
- **Feature-complete acceptance (2026-09-23)**: 121 unit tests; 8 interop cases with zero protocol errors; verified by
  hand: `xdpyinfo -ext all`, xdotool typing into xterm through XTEST + XKB, xprintidle, xrestop, xkbcomp,
  `xinput list / list-props / query-state / test-xi2`, gedit (GTK3, XI2 + EWMH; double-clicking the HeaderBar sends a
  maximize request that, once honoured, fills the screen) and qt5ct (Qt5, XI2).
- **M3: the host is an `IXServerHost` implementation, no extra process**: the host side
  (`src/VelaShell/Services/XServer/AvaloniaXServerHost.cs`) opens one Avalonia native window per X top-level; the bitmap maps
  one-to-one to X pixels and is scaled by DPI without interpolation; every callback is posted to the UI thread. The root window
  is the bounding box of all monitors, one RANDR output per monitor, recomputed when monitors come and go; the DPI follows the
  primary monitor's scaling (at integer scaling GTK's window scale is set too). X coordinates are the position of the
  **content area**; the system title bar and borders lie outside it and their size reaches clients through
  `_NET_FRAME_EXTENTS`. Ordinary windows mapped without a position (at 0,0) are placed the way a window manager would: dialogs
  centered over their parent, everything else centered in the primary monitor's work area.
- **M3: the engine is selectable, built-in by default**: `ILocalXServer` is implemented by a selector that forwards to the
  built-in engine or VcXsrv per the settings, the running one taking precedence (changing the setting never stops an X server
  that is showing windows). VcXsrv exists only on Windows; other platforms only have the built-in engine.
- **M3: SSH x11 channels go straight into the server through a connector**: the SSH library's
  `X11ForwardOptions.LocalConnector` (`velashell-docs/en/ssh/spec/07` §7.5.9) gets an in-memory duplex stream pair per channel and
  hands one end to `X11Server.ServeAsync`. The fake-cookie check is unchanged; the server admits the stream as a local connection.
  Used in trusted mode only — untrusted mode needs `xauth` to reach a display and still goes over TCP. The server keeps listening
  on loopback TCP and the Unix socket, so other local X programs can connect with `DISPLAY=localhost:N`.
  **Two corrections on 2026-09-24**: ① the connector picks the server that is running **at the moment** each channel arrives,
  instead of remembering the one that was running when the display was resolved — an SSH session outlives the server, and after
  the user stopped and restarted the X Server from the title bar, every channel of an existing session used to go into a disposed
  server: the remote side only saw `Failed to open display`, and nothing was logged locally. If no server is running at that
  moment, the channel is refused and a log line is written. ② When the remote side sends `CHANNEL_EOF`, the connector's end must
  read EOF too (a new decision in spec 07 §7.5.9): previously, after the remote program exited, its connection and window stayed
  up until the whole SSH session ended.
- **M3: the keyboard layout follows Windows**: the host injects X keycodes by physical key (scan code); on Windows the system's
  `ToUnicodeEx` computes the unshifted and Shift levels of the main key block for the current layout, which replace the
  server's keymap (recomputed the next time an X window is activated after a layout switch). The AltGr level is not generated
  yet — the server's XKB description only derives two levels; other platforms use US.
- **Fixed while verifying M3 on real windows**: ① destroying the Damage objects on a pixmap when the pixmap was freed was wrong —
  `FreePixmap` only drops the ID, and xeyes' Present-based frame swap follows it with `DamageDestroy`, which then got BadDamage
  and the client exited; Damage objects are now released by `DamageDestroy` or client disconnect. ② The Unix-socket listener
  deleted an existing socket file before binding — the desktop's own Xorg usually has TCP off, so the TCP-side check cannot see
  it, and the desktop's socket got deleted; it now tries to connect first and leaves the file alone if anyone answers.
  ③ Host side: resizing a shown native window takes `Width` / `Height` (setting `ClientSize` only changes the property value);
  the `Resized` caused by our own resize may arrive a beat later, so only user drags and window-state changes are reported to
  the server — otherwise the old size would overwrite the one the client just set.
- **M3 acceptance (2026-09-24)**: headless UI tests (map → native window size, title, pixels; close button → client
  disconnected → window gone); `scripts/xserver/host-demo/demo.cs` opens real native windows, and xterm, xeyes, gedit (GTK3,
  self-drawn title bar, menu popups) and qt5ct (Qt5) from a container render correctly with matching geometry; X-side
  move / resize, closing a native window (WM_DELETE_WINDOW) and keys on a native window reaching xterm were all verified.
- **The AltGr level (2026-09-24)**: XKB gains two types, FOUR_LEVEL and FOUR_LEVEL_ALPHABETIC (six types in the table). Keys with
  keysyms in core columns 5 and 6 become four-level — the column order follows the core-compatibility convention of XKB §17:
  group 1 levels 1–2, group 2 levels 1–2, group 1 levels 3–4; Mod5 selects levels 3 and 4. The host API gains
  `SetModifierMapping`. The Windows host reads the AltGr level with `ToUnicodeEx` under Ctrl+Alt, emits six columns when the
  layout has AltGr characters, and makes the right Alt `ISO_Level3_Shift` in Mod5; the fake left Ctrl that Windows adds for
  AltGr is not forwarded (otherwise X programs would see Ctrl+AltGr). Fixed along the way: a narrower ChangeKeyboardMapping
  (`xmodmap -e "keycode 108 = …"` sends one column per keycode) shrank the whole keymap to one column; the width now only grows.
  Verified: after `xmodmap`, `xkbcomp` dumps the four-level key while other keys keep two levels, and AltGr+q in xterm types `@`.
- **M4 extension numbers** (continuing): MIT-SHM 147 (event 91 Completion, error 149 BadShmSeg), GLX 148 (event 92
  PbufferClobber, errors 150–162). GLX FBConfigs start at `0x101`, four of them: one single- and one double-buffered per
  TrueColor visual, colour 8/8/8 (plus 8 bits of alpha on the ARGB visual), depth 24, stencil 8, no accumulation buffer and
  no multisampling; GLX 1.2 visual configs have one entry per visual, using its double-buffered config.
- **Synchronous grabs replace "asynchronous only" (2026-09-24)**: M1 treated Sync modes as asynchronous; M4 implements real
  freezing — one freezer and one input queue per device, with host-injected and XTEST input queued while frozen; AllowEvents 0–7
  and XIAllowEvents release, single-step (SyncPointer / SyncKeyboard / SyncBoth) and replay (a replay skips passive grabs on
  the grab window and its ancestors); when the grab ends (including client disconnect) the device thaws and its queue goes
  back to the execution loop.
- **XIChangeHierarchy's RemoveMaster**: in AttachToMaster mode a return device of 0 means the virtual core pointer / keyboard —
  `xinput remove-master` sends exactly that by default, and treating it as an invalid device returned BadMatch.
- **XKB SetMap keeps no separate XKB table**: types, actions, behaviours, explicit components and the virtual modifier map are
  read per the specification and not stored; keysyms become core columns and are written back, and XKB is then derived from
  the core table as usual — consistent with the "XKB is derived from the core keymap" principle, which is why
  `xkbcomp keymap $DISPLAY` takes effect.
- **Extensions can be visible per client**: an entry in the extension registry carries an optional visibility predicate that
  QueryExtension, ListExtensions and dispatch all honour. MIT-SHM uses it to appear only to clients connected over a Unix socket.
- **GetImage on the root window**: in rootless mode the root window has no buffer, and GetImage used to return BadMatch
  (`xwd -root` failed); it now composes the mapped top-levels in stacking order, with black elsewhere.
- **GLX strings include the trailing NUL**: in QueryServerString and GetString replies the STRING8 length counts the trailing
  NUL — client libraries use that memory as a C string.
- **M4 acceptance (2026-09-24)**: XServer unit tests +17 (synchronous grabs, device topology, SetMap, MIT-SHM (run on Linux only),
  root-window GetImage, seven GLX tests — clear and triangle, depth and display lists, a texture uploaded through RenderLarge,
  GL queries and ReadPixels, the display-list execution budget, …), real-client tests +3 (`glxinfo` on both paths, `glxgears`
  direct / indirect). By hand: `xinput` adding and removing master devices, attaching / floating slaves;
  `setxkbmap -print -layout de | xkbcomp - $DISPLAY` swaps y / z and brings in the AltGr level; in a Linux container `xdpyinfo`
  lists MIT-SHM and `x11perf -shmput10` runs; `glxgears` draws identical gears on both paths (lighting, flat shading, depth), and
  `glxheads` renders correctly through indirect rendering.
- **Keyboard layout: "auto" follows the system on all three platforms, and a layout can be chosen (2026-09-24)**: the host reads
  the current layout per platform — Windows `ToUnicodeEx`; macOS `TISCopyCurrentKeyboardLayoutInputSource` + `UCKeyTranslate` (the
  Option level maps to AltGr, the right Option becomes ISO_Level3_Shift and the left Option stays Alt; TIS may only be called on the
  main thread); Linux reads the keymap of the desktop's `$DISPLAY` (XWayland on Wayland) through libxkbcommon-x11 and keeps levels 3–4
  only when the desktop's right Alt is a Level3 key (xkeyboard-config keymaps always contain a virtual `<LVL3>` key, so scanning the
  whole table would mark US as having AltGr too). The "Keyboard layout" setting is now shared by both engines; with a chosen layout the
  built-in engine uses keymap tables shipped with the app (generated from xkeyboard-config through libxkbcommon by
  `scripts/xserver/keymaps/generate.cs`, 27 layouts). All three sources go through one outlet (four levels in the §17 core column
  order) into the server's core keymap, from which XKB is derived as usual; the server library itself is unchanged.
- **Rendering path review (2026-09-25)**: the whole chain from "a client draws" to "a frame on screen" was reviewed.
  ① **The host reads only damaged rectangles, in one copy**: new `XTopLevelWindow.ReadPixels` (hands the buffer to a callback under the
  pixel lock; width and height come from the buffer itself). The Avalonia host splits each window into 256 × 256 tiles, one bitmap per
  tile, and on each display frame (`RequestAnimationFrame`) copies the accumulated damage rectangles straight into the locked tiles
  under the pixel lock. Before, every damage batch (hundreds per second under load) copied the whole window twice — into a temporary
  array, then pixel by pixel with an alpha fix-up into one full-window bitmap — and the renderer then re-uploaded the whole window to the
  GPU every frame; now there is at most one copy per frame and only touched tiles are re-uploaded. Damage is coalesced across threads on
  the host side, with at most one delivery queued on the UI thread at a time. This also fixes a crash: the UI thread read `Width` /
  `Height` before copying, and a client enlarging the window in between made the copy run out of bounds.
  ② **The pixel lock yields to the host** (see §5): the loop releases early when the host is waiting and re-takes it only after the read.
  ③ **PutImage touches only what can be drawn**: PutImage and MIT-SHM PutImage decode and blit only the part of the image that falls inside
  the part of the target that can actually be drawn; 32-bit ZPixmap (Qt, GTK4 through llvmpipe, browsers pushing whole windows) is blitted row by row straight
  from the request data or the shared segment, without a temporary array. ShmPutImage used to copy the whole shared image out, decode all
  of it and copy the source rectangle out again — four passes over the whole image for a small update; bitmap-format PutImage used to
  allocate pixels for the full image (eight pixels per byte, so a 16 MB request became 512 MB). Format / depth validation now runs before
  the length is computed from the depth.
  ④ RENDER FillRectangles computes the target region once per request (it used to clone the visible region for every rectangle).
  ⑤ Request buffers are no longer zero-filled before the read overwrites them.
  Benchmark (`scripts/xserver/bench/bench.cs` gained three scenarios: full-window 800×600 PutImage, FillRectangles with 50 rectangles,
  and host pixel reads under load): full-window PutImage throughput +6–13 %, CPU −12–26 %; the in-process benchmark is bound by its test
  client and pipes, so the server-side savings show up mainly as CPU and lock-hold time, and the other scenarios are within noise. The
  host half (damage-only, per-frame, tiled uploads) is not covered by this benchmark.
