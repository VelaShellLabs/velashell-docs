# VelaShell.XServer Architecture and Rationale

中文:[`../../../zh/xserver/design/architecture.md`](../../../zh/xserver/design/architecture.md)

> Status: **M1 (core protocol), M2 (modern toolkits) and the "feature-complete" round (a dozen more extensions including
> XKB and XInput2, the window-manager role, a library-wide performance review) complete**; project started 2026-09-23. Not yet wired into the
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
- No GLX / DRI3 / MIT-SHM. Present does software copies only (there is no video memory to flip).

## 3. Clean-room rules

The same discipline as `VelaShell.Ssh` (see `src/VelaShell.XServer/AGENTS.md` in the host repository):

1. **Only public specifications are implementation sources**: X.Org's *X Window System Protocol, X Version 11*
   (including Appendix B, the encoding), the extension specifications (*BIG-REQUESTS*, *XC-MISC*,
   *X Nonrectangular Window Shape Extension*, *X Fixes Extension*, *The X Resize and Rotate Extension*,
   *The X Rendering Extension* (and the PDF Reference blend-mode formulas it cites), *The X Keyboard Extension:
   Protocol Specification*, *The X Input Extension* (1.5 and 2.2), *XTEST Extension*, *X Synchronization Extension*,
   *X Damage Extension*, *Composite Extension*, *Double Buffer Extension*, *The Present Extension*, *MIT-SCREEN-SAVER*,
   *DPMS*, *X-Resource*, *Generic Event Extension*, and XINERAMA — which has no standalone specification document, so
   its wire format follows the protocol definitions X.Org publishes in panoramiXproto), ICCCM, EWMH and freedesktop.org's
   XSETTINGS specification. **Every protocol file's header names the specification and section it implements.**
2. **No other X server's source is opened while implementing** (X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv).
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
              Render / Damage / CompositeDbe / Sync / Present / Xkb / XInput / XTest / ScreenSaver, the window-manager
              role in Ewmh, clipboard exchange in Clipboard, the XSETTINGS manager in XSettings)
Windowing/    Window model (tree, geometry, attributes, event selections, passive grabs, top-level buffer, the three SHAPE shapes)
Resources/    GCs, pixmaps, colormaps, cursors, font handles, colour-name table, RENDER pictures and glyph sets
Drawing/      32-bit software framebuffer, regions, rasterizer: 16 raster ops, plane mask, fill styles, clipping;
              points / lines (thin Bresenham + wide-line polygons) / rectangles / scan-line polygon fill / arcs /
              image blocks / text; RENDER: pixel formats, compositing operators and blend modes, sources (image / solid /
              gradients with repeat, transform, filter), trapezoid coverage
Fonts/        BDF parsing, built-in misc-fixed fonts, XLFD name matching, synthesized cursor and nil2 fonts
Input/        Keycode ↔ keysym table (evdev-style keycodes), XKB evdev key names, modifier mapping, grab data structures (core and XI2)
Host/         Host-facing API: IXServerHost, XTopLevelWindow, XServerOptions, XMonitor, window-manager requests and enums, XKeycodes
```

Unix sockets: outside Windows the server listens on `/tmp/.X11-unix/X{N}` by default, and on Linux also on the same
name in the abstract namespace (Xlib / XCB try that first for `:N`); `XServerOptions.UnixSocketPath` sets or disables it
and `ListenTcp` can turn TCP off. A host font-provider interface does not exist yet; it comes when M3 needs larger sizes.

## 5. Threading model

X semantics are **globally serial**: the server executes every client's requests one by one in arrival order, and
each request's effects are visible to everything after it. Therefore:

- **One execution loop** (single thread, `Channel<work item>`) runs all requests, host input and timers. Protocol state
  takes no locks — all mutable state is touched only on that thread. The one thing shared across threads is top-level
  pixels, guarded by `PixelLock`: the loop holds it for a batch within a **4 ms** time budget, then releases it so the
  host can copy pixels.
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
| Library → host | Top-level window mapped / unmapped / destroyed; geometry changes (client ConfigureWindow); title (`WM_NAME` / `_NET_WM_NAME`), class, transient parent, override-redirect; non-rectangular outline (`XTopLevelWindow.Shape`, the SHAPE bounding shape, null when rectangular); window-manager hints (`WindowType`, `States`, `Decorated` — false for windows that draw their own title bar, min / max size and increments, icons, `Urgent`, `AcceptsFocus`, `Opacity`, `ClientFrameExtents`, process id / machine / role, `HasAlpha`); clients' window-manager requests (`WindowManagerRequest`: interactive move / resize, state changes, activate, close, minimize — a default interface method, so existing hosts need no change); damage rectangles (the host then copies from that window's pixel buffer); cursor shape (cursor-font glyph number, −1 default arrow, −2 hidden); bell; an X client copied text (`ClipboardChanged`) |
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
- **XKB is derived from the core keymap**: there is no separately maintained XKB keymap — the four canonical types,
  modifier actions, SymInterprets, indicators and key names are all computed from the core table; when the core table
  changes (xmodmap, the host's `SetKeyboardMapping`), XKB follows and sends MapNotify. XKB's own mapping-change requests
  (SetMap / SetCompatMap, …) are not supported.
- **XInput2's device topology is fixed**: master pointer 2 / master keyboard 3, each with one slave (4, 5). XI2 events
  travel the same propagation path as core events — on a given window, a core selection receives core events and an
  XI2 selection receives XI2 events; XIChangeHierarchy returns BadImplementation.
- **Clipboard**: host → X, the server itself owns CLIPBOARD and answers per ICCCM; X → host, the server fetches the text
  with a hidden InputOnly window as requestor (UTF8_STRING with STRING fallback, INCR supported). When the host writes
  back the text it just received, the server does not take the selection, so the two sides never fight over it.

## 8. Milestones

| Milestone | Content | Acceptance |
| --- | --- | --- |
| **M1 core protocol** ✅ | All core requests; BIG-REQUESTS, XC-MISC; windows / events / properties / selections; software drawing; built-in fonts; keyboard mapping; headless test host | `xdpyinfo`, `xterm`, `xeyes`, `xclock`, `xlogo` in Docker connect, draw, and cause zero protocol errors (achieved, see §10) |
| **M2 modern toolkits** ✅ | SHAPE, XFIXES, RANDR (read-only), RENDER; clipboard exchange with the host; XSETTINGS manager | Simple GTK3 / Qt5 programs work (`zenity`, `gedit`, `qt5ct` draw with zero protocol errors — achieved, see §10) |
| **Feature-complete** ✅ | XKEYBOARD, XInputExtension 2.2, XTEST, XINERAMA, SYNC, DAMAGE, Composite, DOUBLE-BUFFER, Present, MIT-SCREEN-SAVER, DPMS, X-Resource, Generic Event; the window-manager role; Unix sockets; runtime layout / DPI / keyboard-layout changes; library-wide performance review | xkbcomp, xinput, xdotool, xprintidle, xrestop read and write correctly; gedit (GTK3) and qt5ct (Qt5) run on XKB and XI2 with zero protocol errors (achieved, see §10) |
| **M3 host integration** | Avalonia host (native windows, input, HiDPI, window-manager requests); replace the VcXsrv path; trim the settings page | The host's "X Server" button no longer depends on an external program |
| M4 | MIT-SHM (only meaningful on the same machine, low priority), GLX (indirect rendering), synchronous grabs, XIChangeHierarchy, XKB mapping-change requests | As needed |

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