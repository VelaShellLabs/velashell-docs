# VelaShell.XServer Architecture and Rationale

中文:[`../../../zh/xserver/design/architecture.md`](../../../zh/xserver/design/architecture.md)

> Status: **M1 (core protocol), M2 (modern toolkits), the "feature-complete" round (a dozen more extensions including XKB and
> XInput2, the window-manager role, a library-wide performance review), M3 (host integration) and M4 (synchronous grabs, device
> topology, XKB mapping changes, MIT-SHM, GLX) complete**; project started
> 2026-09-23; the fixes from the second library-wide review were completed in 2026-10 (§10). The host's "X Server" button starts this library by default (one Avalonia native window per X window) and SSH X11
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
- An optional **one-window (rootful) mode** (`X11ServerOptions.Rootful`, off by default, see "One-window mode" in §7): the whole root
  window is handed to the host as one top-level and a remote window manager manages the top-levels as usual — for a full remote desktop
  (xfce, MATE) or a graphical installer.
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
  only on Linux and only for local clients connected over a Unix socket from the server's own IPC namespace. Present does
  software copies only (there is no video memory to flip).
- The SECURITY extension covers issuing and revoking MIT-MAGIC-COOKIE-1 authorizations and the untrusted level ("Trust levels" in §7);
  the XC-QUERY-SECURITY-1 authentication method and Application Groups (made only for the long-gone X firewall proxy) are not done, and
  GenerateAuthorization's group must be None.

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
   ICCCM, EWMH, freedesktop.org's XSETTINGS specification, freedesktop's XDND protocol version 5 (both directions of drag and drop: the host as
   the source dragging into X windows, and — through the root-window `XdndProxy` convention of version 4 on — taking drags out of X programs),
   the X Consortium's *The Input Method Protocol* 1.0 and *The XIM Transport Specification* 0.1 (the XIM input method server, `Server/XimServer.cs`;
   the two documents' prose and tables contradict each other on whose window a long packet is written to and which format the ClientMessage
   announcing it uses — the tables in Appendix D of the Input Method Protocol win; the XIMStyle bits and XN* attribute names come from chapter 13
   of *Xlib - C Language X Interface*), and the X Consortium's *Compound Text Encoding* (COMPOUND_TEXT
   encoding and decoding, `Protocol/XText.cs`); GLX follows Khronos' *OpenGL Graphics with the X Window System*
   1.4, the *GLX Extensions for OpenGL Protocol Specification* 1.3 (the encoding), *The OpenGL Graphics System* 1.5 (GL
   semantics for indirect rendering) and the GLX_ARB_create_context / GLX_ARB_create_context_profile extension specifications
   in the Khronos registry, with opcodes and enum values taken from the Khronos registry's `gl.xml` / `glx.xml`.
   **Every protocol file's header names the specification and section it implements.**
2. **No other X server's source is opened while implementing** (X.Org / XLibre / yserver / node-x11 / WeirdX / VcXsrv),
   and no OpenGL / GLX implementation's source either (Mesa and the like).
3. Constants from the specifications (opcodes, event codes, error codes, predefined atoms, mask bits) take their
   specified values — they are protocol facts, not copyrightable, and must not be changed to "look different".
4. **Data is not code**: the built-in bitmap fonts are X.Org's BDF files as is — `font-misc-misc` ("Public domain font. Share and enjoy."),
   `font-cursor-misc` ("These glyphs are unencumbered"), `font-adobe-75dpi` / `100dpi` (Adobe / DEC's permissive notice) — plus GNU Unifont
   (dual-licensed; used under the SIL OFL 1.1); the colour-name table is X.Org `rgb`'s `rgb.txt` as is (MIT / X11 licence). All ship as data
   files with their source and licence recorded in `NOTICE.md`, and the fonts' licence texts are embedded in the assembly with the data.
   The font data is generated only by `scripts/xserver/fonts/build-fonts.cs` from pinned upstream commits (BDF byte for byte, Brotli-compressed),
   never edited by hand; what it downloads is checked against SHA-256 values pinned in the script (for the X.Org repositories, over the files
   picked out of the archive — GitLab's on-the-fly archives are not byte-stable), and the data directory is touched only once everything matches.

## 4. Layering

```
Host/         Every public type, all in the root namespace VelaShell.XServer: IX11ServerHost, X11ServerOptions,
              XTopLevelWindow with XTopLevelSnapshot / XTopLevelChanges / XFrameExtents / XGravity / XWindowFunctions,
              XPixelReader / XPixelReadResult, XClientInfo, XCursor / XCursorShape / XCursorImage, XKeymap, XMonitor,
              window-manager requests and enums, XKeycodes, XRect. Everything else is internal
Protocol/     Constants (opcodes, event codes, error codes, masks, predefined atoms), byte-order-aware request
              reading and reply / event / error writing, the per-work-item work budget (WorkBudget, §5), STRING /
              UTF8_STRING / COMPOUND_TEXT encoding and decoding (XText)
Server/       X11Server: every public member lives in X11Server.cs (construction, lifecycle, host injection); the execution
              loop (WorkLoop), listening and connections (Connection, UnixSocket: TCP 6000+N, Unix sockets and ServeAsync /
              ServeAuthenticatedAsync over any duplex stream, connection setup (time-limited) and authorization (see §7), backpressure, BIG-REQUESTS,
              cleanup on disconnect), the resource table, XC-MISC and the memory accounts (Resources), request dispatch (Dispatch),
              deferred and per-batch coalesced host callbacks (DeferredHost), the extension registry (Extensions: the numbering
              table, the check that event / error numbers do not overlap, cleanup hooks — connection closed, a client's resources
              destroyed, window destroyed, resource freed, pixmap freed — and Generic Event);
              request handlers split into partial files by area (Windows / Exposure / Events / Properties / Graphics / Text /
              Colors / Cursors / Input / KeyboardControl / GrabFreeze, plus one or more per extension: Shape / XFixes / RandR /
              Monitors / Xinerama / Render / Damage / Composite / Dbe / Sync / Present / Xkb / XkbSetMap / XInput / XiHierarchy /
              XTest / ScreenSaver / Dpms / XRes / Shm), the top-level bridge to the host (TopLevels: handles, snapshots and their
              changes, damage delivery, the host's window-manager actions), the window-manager role in Ewmh, clipboard exchange in
              Clipboard, the XSETTINGS manager in XSettings; the self-contained GLX is a class of its own, GlxExtension
Windowing/    Window model (tree, geometry, attributes, event selections (XI2 ones stored per device ID), passive grabs,
              top-level buffer, the three SHAPE shapes)
Resources/    Resource types: GCs, pixmaps, colormaps, cursors, font handles, plus each extension's resources (RENDER pictures
              and glyph sets, SYNC, DAMAGE, XFIXES regions, Present event contexts, MIT-SHM segments, GLX contexts and
              drawables); the colour-name table (X.Org's rgb.txt, an embedded resource)
Drawing/      32-bit software framebuffer, y-banded regions, rasterizer: 16 raster ops, plane mask, fill styles, clipping;
              points / lines (thin Bresenham + wide lines built as polygons per line-style / join-style / cap-style) /
              rectangles / scan-line polygon fill / arcs / image blocks / text; RENDER: pixel formats, compositing operators and
              blend modes, sources (image / solid / gradients with repeat, transform, filter), trapezoid coverage
Gl/           Software GL for GLX indirect rendering: render-command decoding, display lists, matrix stacks, lighting,
              clipping, triangle (scan-line) / line / point rasterization, textures, per-fragment operations, queries
Fonts/        BDF parsing, the bundled X.Org bitmap fonts and GNU Unifont, XLFD name matching (fonts.dir / fonts.alias,
              derived single-byte charsets, the nearest-size fallback)
Input/        Keycode ↔ keysym table (evdev-style keycodes), keysym case (KeysymCase), XKB evdev key names, modifier mapping,
              grab data structures (core and XI2; PassiveGrabTable buckets passive grabs by detail)
```

Unix sockets: outside Windows the server listens on `/tmp/.X11-unix/X{N}` by default, and on Linux also on the same
name in the abstract namespace (Xlib / XCB try that first for `:N`); `X11ServerOptions.UnixSocketPath` sets it (a path that does
not fit a socket address throws at construction) or disables it. TCP is governed by `ListenTcp`: by default (null) it listens only
when an `AuthorizationCookie` is configured, `true` always listens (logging a warning at start when there is no cookie), `false`
never does. While listening on a transport tied to the display number, the server holds `/tmp/.X{N}-lock` on Unix-like systems per
the Xserver(1) convention (see §7). A host font-provider interface does not exist yet: core fonts only serve older programs, modern
toolkits use RENDER with client-side rasterization, and the bundled X.Org bitmap fonts and GNU Unifont (§7 "Fonts") already cover the
families older programs commonly use as well as CJK.

## 5. Threading model

X semantics are **globally serial**: the server executes every client's requests one by one in arrival order, and
each request's effects are visible to everything after it. Therefore:

- **One execution loop** (single thread, `Channel<work item>`) runs all requests, host input and timers. Protocol state
  takes no locks — all mutable state is touched only on that thread. The one thing shared across threads is top-level
  pixels, guarded by the pixel lock (not exposed publicly): the loop holds it for a batch within a **4 ms** time budget, then releases it so the
  host can copy pixels. A host reading pixels through `XTopLevelWindow.ReadPixels` / `CopyPixels` / `TryReadPixels` first registers that it is
  waiting: after every work item the loop checks, releases early if someone is waiting, and re-takes the lock only after the read finishes
  (bounded at 20 ms) — `lock` is unfair, the loop re-takes it within microseconds of releasing it, and a waiting UI thread could lose
  that race again and again, freezing the whole host UI.
- **Every work item gets a work budget**: the 4 ms batch budget only acts between work items, and a request holds the pixel lock from start
  to finish — a request whose cost is out of proportion to its bytes (huge coordinate ranges, overlapping rows, large composites,
  display lists called over and over) could freeze the host UI along with it. So each work item starts with a budget (2²⁸ units, a unit
  being roughly one pixel processed: about 270 million pixels, filling a 4K screen 32 times), and hot loops charge what they process:
  rasterizer rows / pixels / bitmap blocks, polygon edges and rows, RENDER composite pixels, trapezoid coverage and mask allocation, and for
  GLX render commands vertices (16), fragments (8), triangle scan rows and spans, the area of Clear, and the pixels decoded by DrawPixels /
  Bitmap / CopyPixels / TexImage. When the budget runs out the request gets Alloc; GLX render commands have no reply, so they record GL's
  OUT_OF_MEMORY and the rest of that request's render commands are skipped. A work item that holds the lock for more than 250 ms is
  logged with its client and opcode.
- **The host can read pixels with a time limit**: `XTopLevelWindow.TryReadPixels(reader, timeout)` returns `Busy` without calling the reader
  when it cannot get the lock in time. The Avalonia host waits at most 8 ms per frame; if it cannot get in, it skips that frame and keeps the
  damage for the next one — a slow client cannot hold the host UI hostage. Measured with four fully loaded windows read one after another
  each frame: median 0.44 ms, never once missed, so there is no separate "read several windows under one lock" API (the scenario is in
  `scripts/xserver/bench/bench.cs`).
- **Host callbacks are never made while holding the lock, and are coalesced per batch**: notifications produced by the loop (map, geometry,
  damage, cursor, WM requests…) are queued in `DeferredHost` and invoked in order after the lock is released — so a host that
  synchronously waits for its UI thread inside a callback, while that UI thread waits for the lock in `ReadPixels`,
  cannot deadlock. A host callback that throws is logged and swallowed, so is a host log delegate that throws itself, and each batch of
  the loop has its own safety net, so none of these takes the loop down. Within one batch: a window's `TopLevelChanged` calls are OR-ed into
  one (at the position of the first; not merged across a map / unmap in between); a map followed by an unmap cancels out; only the last
  `CursorChanged` and `ClipboardChanged` are delivered; at most 32 `WindowManagerRequested` are delivered and the rest are dropped with a log
  line; at most one `BellRequested` (the loudest), at least 100 ms apart — a client looping over title changes, map / unmap or bells cannot
  flood the host's UI thread. `ServerGrabStalled` is queued as usual.
- One **reader task** per connection (64 KB buffer) cuts complete requests by their length field and hands them to the
  loop; one **writer task** packs the messages the loop queued for that connection into one pooled buffer and writes it.
  The loop never blocks on a socket, so a slow client cannot stall the others. When a write fails (the peer is gone) the client is
  disconnected right away instead of waiting for the reader to notice — the reader may be parked on backpressure and not reading the socket at all.
- **Backpressure**: at most 1024 read-but-not-executed requests per client, 32 MB in total (the reader waits asynchronously on a
  `SemaphoreSlim` and on the byte budget, and only then allocates and reads — counting requests alone would allow 1024 requests of
  16 MB, 16 GB); a client is disconnected when a message arrives while the bytes **already queued** for it have reached 64 MB (it stopped
  reading). A single message may itself be larger than the limit (with three 4K monitors side by side the reply to `xwd -root` is about
  100 MB), so the overshoot is at most one message; requests that can produce huge replies have limits of their own (GetImage estimates the
  size first and returns BadAlloc above 256 MB). Disconnecting is real — `XClient.Abort` cancels the pending read and write on the connection.
  Request buffers are rented from `ArrayPool` and returned after execution (a full-window PutImage is several MB, and allocating
  each one would put every one of them on the large object heap); the request reader carries its own length and reads only that
  many bytes (bounds are checked against the remaining length, so a client-supplied length near 2³¹ cannot wrap around), and anything a
  request's content must outlive is copied out. Replies and events are assembled in a writer reused per thread; only the copy handed to the
  writer task is allocated.
- **The accept loops survive transient errors**: a failed accept (file descriptors exhausted, the peer reset before the accept) is logged and
  the loop carries on, waiting 100 ms first for errors like fd exhaustion; it only exits on cancellation or once the listener is closed. It used
  to exit for good on any error, after which no local X program could connect again and nothing was logged.
- **Connection setup has a time limit and a concurrency limit**: the setup message (the 12-byte header plus the authorization name and data)
  must arrive, and a failure reply must be written, within 10 seconds, or the connection is closed; at most 32 connections may be in setup at
  once, and any beyond that are closed on the spot (a connection stops counting once it is registered as a client); the authorization name and
  data are at most 256 bytes each, checked as soon as the header is in, and longer ones get a connection failure. The client limit (255) counts
  only established clients — without these, any local user could exhaust file descriptors and memory with tens of thousands of connections that
  send a header and stall, no cookie needed. Waiting for the execution thread to register the client is not timed; if the caller cancels during
  that wait, the client that does get registered is disconnected at once, so no connection-less client is left holding a number.
- **Everything that waits, waits asynchronously**: XTEST and Present delays and SYNC timers use `Task.Delay` and `Post`
  back to the loop when due; SYNC's Await parks that client's later requests and replays them in order once satisfied.
  An XTEST FakeInput with a delay parks that client's later requests the same way (per the spec, no other requests from the client
  are processed until the delay expires) without blocking the execution thread; Present's queued PresentPixmap and NotifyMSC requests are
  capped at 256 per client together. A client's pending timers are cancelled when it disconnects. Parked requests (GrabServer ending, an Await
  satisfied, an XTEST delay expiring) go back through a **ready queue**: the loop takes from it before the channel, and each request runs under
  the usual batch budget with the lock released between batches — they used to run all at once in a single work item, in the worst case
  255 clients × 1024 requests under the lock. A client's requests still in the channel are always later than its parked ones, so order is kept.
- **Diagnostics are never written under the lock**: protocol errors and other diagnostics are collected on the execution loop and
  handed to `X11ServerOptions.Log` after the lock is released (host logs often write files synchronously, and writing under the lock
  makes the UI thread wait on the disk). Lines that clients can trigger in bulk have two limits: at most 50 lines per second, and a byte
  allowance (shared by all clients: 32 KB up front, then 256 bytes per second), with the number of dropped lines reported in one line the next
  time something can be logged. Those lines have C0 / DEL / C1 control characters and bidirectional-text controls written as `\xNN` / `\uNNNN`
  and are cut to 512 characters (OpenFont names to 200) — a newline in a client-supplied string could forge log lines, and an ESC could inject
  control sequences into the terminal of whoever reads the log. Unexpected failures of requests, work items and disconnect cleanup log a full
  stack trace the first time per kind (location + exception type) and one line after that. Each request's opcodes are recorded as integers in a
  ring and turned into text only when an error is printed.
- During **GrabServer** the loop runs only the holder's requests; other clients' requests are parked as they are and
  replayed in order after the ungrab (through the ready queue, above). If the grab has been held for 10 seconds while others' requests are
  waiting, a line names the holder (with its connection label, e.g. `user@host:22`) and the number of waiting requests, and again every 60
  seconds until it lets go; each time the host is also told through `ServerGrabStalled` (`XServerGrabStall`: the holder's number and
  connection label, how long it has held the grab, how many requests are waiting) so it can prompt the user; with nobody waiting neither
  happens. When the holder hangs, the host can call `BreakGrabs` or `DisconnectClient` by number (§6). A holder grabbing again is a no-op.
- **While a synchronous grab freezes devices**, device events queue in a single list, pointer and keyboard in arrival order (host injection
  and XTEST queue alike); after release, one work item replays at most 64 of them and the rest go to the next work item — so client requests
  in between (the next AllowEvents) can get in. The queue holds at most 4096 events and, when full, decides by three classes: droppable ones
  (pointer motion, leave, key auto-repeat) — the oldest goes first; with nothing droppable, the newly arriving press is dropped and remembered,
  so its release and auto-repeats are dropped with it and X never sees it pressed; releases are always kept and may exceed the limit (no more
  releases can arrive than there are keys and buttons held). It used to drop releases too when full, leaving a key or button held after the thaw.
  A WarpPointer while frozen is queued the same way and computed against the position and windows at the time it is replayed.
- Damage is merged after a batch of work items and reported to the host once (not once per drawing request); each
  top-level accumulates at most 8 rectangles per batch and falls back to their bounding box beyond that — an exact
  union degrades to O(n²) over a batch of a few hundred requests.
- **Clients take turns (2026-10-10)**: work taken from the channel is queued per client, keeping arrival order within a queue; host and timer
  work has its own queue that goes first whenever it has items (injected input, configuration changes, due timers: small and time-sensitive,
  and bounded by the real input rate, so clients cannot starve); clients take turns, one item each. Requests put back after being held
  (GrabServer ended, a SYNC Await satisfied, an XTEST delay elapsed) are still taken first. The protocol only requires one client's requests
  to run in order; ordering between clients was never guaranteed. Connection teardown is queued after that client's pending requests (a
  client that sends requests and closes still has them take effect). Everything used to share one queue: with an indirect-GL glxgears
  queueing over a thousand rendering requests, an xdpyinfo connecting afterwards waited 25 seconds; now under 1.
- **Watchdog (2026-10-10)**: the execution thread records the item in progress (start time, client, opcode, sequence number); the timer thread
  looks once a second and past `WatchdogThreshold` (5 seconds by default) logs a line `watchdog: client#3 (user@host:22) opcode 53 has been
  running for 5 s; all clients are waiting` and counts `work.stalled`, once per item and without waiting for it to finish (the "slow work item"
  line used to be logged only after the item finished, so a real hang left nothing).

## 6. Host interface (rootless)

The library defines the interface and the host implements it; notifications flow **library → host**, injection flows
**host → library**:

| Direction | Content |
| --- | --- |
| Library → host (`IX11ServerHost`; callbacks are all named "subject + past participle"; all are made on the execution thread after the lock is released, coalesced per batch as in §5) | Top-level window mapped / unmapped (`TopLevelMapped` / `TopLevelUnmapped`; destruction and being reparented away count as unmapping; **not sent one by one at shutdown** — the host closes its native windows itself when it stops the server); the snapshot changed (`TopLevelChanged`, with `XTopLevelChanges` saying which groups changed: `Geometry` — position, size, `BorderWidth`, `NeedsPlacement`; `Title` — title, `ClassName`, `InstanceName`; `States`; `Icons`; `Shape` — bounding and input shapes; `Hints` — everything else). A window's properties live in the immutable snapshot `XTopLevelWindow.Snapshot` (see "Snapshot" below); clients' window-manager requests (`WindowManagerRequested`, a default interface method; see "Window-manager requests" below); damage rectangles (`TopLevelDamaged`; the host then reads just those rectangles through `XTopLevelWindow.ReadPixels` / `TryReadPixels` under the pixel lock, straight into its own bitmaps; `CopyPixels` copies the whole window, for tests and diagnostics); the cursor (`CursorChanged`, an `XCursor`: a semantic shape `XCursorShape`, plus an `XCursorImage` for bitmap / ARGB / glyph cursors — except glyphs of the `cursor` font that have a matching system cursor, for which the
host picks the system cursor by shape — its pixels a read-only `ReadOnlyMemory<uint>`); bell (`BellRequested`, the 0–100 volume computed from the base volume as the protocol specifies; 0 means silent — `xset b off`, Bell −100); an X client copied text (`ClipboardChanged`); a client has held GrabServer too long while others wait for it (`ServerGrabStalled`, see §5; the default implementation does nothing); a tray icon docked / went away (`SystemTrayIconAdded(icon, title)` / `SystemTrayIconRemoved`, with `SystemTray` on, see "System tray" in §7); the XIM input context taking the host input method's input changed or its insertion point moved (`InputMethodFocusChanged(XInputMethodFocus?)`: its top-level, whether the program draws the preedit, the insertion point; null = the focused program does not use XIM. With `InputMethodName` set; only the last one per batch, see "The XIM bridge for the host input method" in §7); an X program dragged something outside every X window and the data has been fetched (`OutgoingDragStarted(XOutgoingDrag)`: URIs, text, the dragging program's connection label, root coordinates), and that drag ended on the X side before the host reported back (`OutgoingDragEnded`; with `AcceptOutgoingDrags` on, see "Dragging out" in §7) |
| Host → library (`X11Server` methods; windows are named by their `XTopLevelWindow` handle; invalid arguments throw on the spot, a window that is already gone is silently ignored) | Input, `Inject*`: pointer motion / buttons (content-area coordinates, checked against X's 16-bit range — out of range throws `ArgumentOutOfRangeException`; the wheel as buttons 4/5, 6 and up are horizontal wheel and side buttons; a release for a button X does not consider pressed is not delivered, while a release for a window that is already gone still takes effect), pointer leaving (the last position is kept, the pointer counts as on the root with child None), keys (`InjectKey(keycode, pressed, repeat)`, X keycodes; `repeat` marks the host's auto-repeat, see §7); activity in the host's own UI (`NoteUserActivity`: resets the idle time without producing input events, queuing at most one work item per 250 ms); window-manager actions (names containing `TopLevel`): focus (`FocusTopLevel`, null = no focus; it also raises the window above the other normal top-levels in X and advances the last-focus-change time), the user moved / resized the native window (`MoveTopLevel` / `ResizeTopLevel`; the library updates geometry, sends a real ConfigureNotify followed by ICCCM's synthetic one, and Expose), close button (`CloseTopLevel`: ClientMessage when `WM_DELETE_WINDOW` is advertised — with a ping when `_NET_WM_PING` is too — otherwise the client is disconnected; ignored for override-redirect windows), force quit (`KillTopLevelClient`, KillClient semantics, taking the client's other windows with it), window states (`SetTopLevelStates` replaces the whole set; `ChangeTopLevelStates(window, add, remove)` changes only the given bits and keeps the rest, throwing on the spot when `add` and `remove` overlap; both write `_NET_WM_STATE` and `WM_STATE`, and `Focused` is maintained by the server) and frame extents (`SetTopLevelFrameExtents`, written to `_NET_FRAME_EXTENTS`); the host's environment changed (`Set*` without `TopLevel`): the keymap (`SetKeymap`, see §7), monitor layout (`SetScreenLayout`, sends RANDR events), DPI and scale (`SetDisplayScale`, updates XSETTINGS and replaces only the `Xft.*` entries in RESOURCE_MANAGER), lock keys (`SetLockState(capsLock, numLock)`: no synthesized key presses, clients get an XKB StateNotify), the system clipboard has new text (`SetClipboardText`; more than `X11Server.MaxClipboardBytes` (16 MB) of UTF-8 throws `ArgumentOutOfRangeException` on the spot); recovering from a hang (`BreakGrabs`: releases every pointer / keyboard grab and thaws the devices, releases GrabServer, reattaches floating slave devices to the virtual core devices; clients get the usual Ungrab-mode events and HierarchyChanged); the client list (`GetClientsAsync` → `XClientInfo`: number, connection label, whether it disconnected in Retain mode, resource count, accounted memory, mapped top-levels, whether it holds the whole server with GrabServer — `HoldsServerGrab`) and disconnecting by number (`DisconnectClient(int)`; for a client that disconnected in Retain mode, its leftover resources are destroyed); local drag-and-drop (`InjectDragOver(window, x, y, types)` / `InjectDragLeave()` / `InjectDrop(window, x, y, data)`: the server acts as the XDND source for the host; `IsDragAccepted` is the target's latest answer, see "Drag and drop" in §7); host input method commits and preedit (`InjectText(text)`: when the focused program is connected over XIM the whole run goes as XIM_COMMIT, otherwise keycodes are borrowed; `InjectPreedit(text, caret)`: an on-the-spot program draws it in its own input box through XIM's preedit callbacks, otherwise it is ignored, see §7); the host's answers to something the server handed it (`Complete*`, the first argument being that thing): whether the local side dropped what an X program dragged out (`CompleteOutgoingDrag(drag, dropped)`, called before the released button is injected back, see "Dragging out" in §7) |

**Lifecycle**: the execution thread starts at construction, so an instance must be disposed with `DisposeAsync` even if `StartAsync`
was never called. `StartAsync`'s failure modes are in §7 (a taken display number throws `SocketException` (`AddressAlreadyInUse`); being
configured to listen yet getting no transport up throws `IOException`); on failure the listeners already opened are closed again and the same
instance may try again. `StartAsync` and `DisposeAsync` are mutually exclusive, so racing them cannot leak a listener nobody closes.
`DisposeAsync` may be called several times and concurrently: later calls return only after the first one has finished, and exceptions during
shutdown are not rethrown; `Completion` completes when shutdown is done. `ServeAsync(stream, isLocal)` and `ServeAuthenticatedAsync(stream[, label])`
throw invalid arguments (a null stream, a disposed server) on the spot rather than inside the returned task; `label` describes where the connection
comes from (e.g. `user@host:22`), shows up in the log, `XClientInfo.Label` and the snapshot's `ClientLabel`, and is what "the same session" means
when the clipboard follows the focus (§7). With a third argument `trust` of `XClientTrust.Untrusted` the connection is an untrusted client
("Trust levels" in §7); `XClientInfo.Trust` reports each client's level.

**Handles**: an `XTopLevelWindow` is the same object from map until destruction (or until it is reparented away). It carries the server that
issued it (`Server`; passing it to another server's methods throws `ArgumentException`) and `IsAlive` (false once the window is destroyed,
reparented away or the server has shut down; it never turns back to true). A host using handles as dictionary keys compares them by reference,
not by `Id` (the XID) — stop the server and start another right away and the new windows' XIDs coincide with the old ones, while the UI queue may
still hold callbacks from the old server; callbacks should first check that `Server` is the one attached right now. Pixels: `ReadPixels` /
`TryReadPixels` / `CopyPixels` get nothing once the window is destroyed or reparented away (`false` / `NoBuffer` / (0, 0)); unmapping does not
count — the buffer is still there, holding what was drawn last before the unmap. InputOnly top-levels have no pixels.

**Snapshot** (`XTopLevelSnapshot`; the server replaces it whole on every change and the host reads it into a local before reading fields; an
unchanged shape or icon set keeps its list instance): geometry (`X` / `Y` are the top-left corner of the X window's **outer border edge**, `Width` /
`Height` the content area, `BorderWidth`, `NeedsPlacement` and `PlaceInFrame` — see "Placement" below), title (`_NET_WM_NAME` first, else
`WM_NAME` decoded by its type), `ClassName` / `InstanceName` (`WM_CLASS`), override-redirect, transient parent (`TransientFor`),
`SupportsDeleteWindow`, the owning client (`ClientId` / `ClientLabel`; a tray embedder belongs to the server, so its `ClientLabel` is that of the
docked icon's connection), `InputOnly`, `HasAlpha` (true only for depth-32 windows while the server acts as compositing manager — without one X
shows ARGB windows without regard to alpha; tray icons excepted), `WindowType`, `States` (states the client set
before mapping; with a `WM_HINTS` initial_state of IconicState the server adds `Hidden` at map time), `Decorated` (false when `_MOTIF_WM_HINTS`
asks for no decorations), size constraints (min / max, increments, `BaseWidth` / `BaseHeight` — min and base default to each other,
`MinAspect` / `MaxAspect`, all clamped to 0–32767), `WinGravity`, `UserPosition` / `ProgramPosition`, `WindowGroup`, `Functions` (the
`_MOTIF_WM_HINTS` functions, `XWindowFunctions`), icons (`_NET_WM_ICON`, only its first 4 MB parsed; without it the `WM_HINTS` icon_pixmap /
icon_mask are baked into one icon, at most 256 on a side; `XWindowIcon.Pixels` is a read-only `ReadOnlyMemory<uint>`), `Urgent`, `AcceptsFocus`, `Opacity`, `ClientFrameExtents`, process id / machine /
role, `Shape` (the bounding shape), `InputShape` (SHAPE 1.1's input shape intersected with the bounding shape, null when not set) and `Strut`
(`_NET_WM_STRUT_PARTIAL`, else `_NET_WM_STRUT`). Strings handed to the host are bounded (titles at most 4096 characters, class / machine / role
at most 256) and stripped of C0 / C1 control characters and bidirectional formatting characters (an RLO can show a title backwards in the taskbar
and fake a file extension); hint properties are read only as far as the values the specifications define.

**Window-manager requests** (subclasses of `XWindowManagerRequest`; the host is the window manager and decides whether to honour them):
`XMoveResizeRequest` (`_NET_WM_MOVERESIZE`: start a native move / resize; the server treats the held button as handed to the window manager only
when the request comes from the client holding the pointer grab), `XStateChangeRequest` (`_NET_WM_STATE` additions and removals; also sent, removing
`Hidden`, when a client maps a mapped normal top-level that is `Hidden` — ICCCM §4.1.4), `XActivateRequest` (`_NET_ACTIVE_WINDOW`, carrying
EWMH's source indication `Source`, the timestamp `Timestamp` and the server's verdict `UserInitiated`: the source is a pager, or the timestamp is
no earlier than the user's last key or button press in an X window; CurrentTime and stale timestamps do not count — a host should honour it only
when it is true and the user is using X windows at that moment, and otherwise just draw attention), `XRaiseRequest` (a client sent a
ConfigureWindow with stack-mode Above to a top-level: XRaiseWindow, XMapRaised, Java's toFront; stacking only, no focus asked for), `XFocusRequest`
(a client moved the keyboard focus to this top-level itself, so keys now go there), `XNotRespondingRequest` (pinged on close, no `_NET_WM_PING`
reply within 5 seconds; the host may ask the user whether to `KillTopLevelClient`), `XCloseRequest` (`_NET_CLOSE_WINDOW`) and `XMinimizeRequest`
(ICCCM's `WM_CHANGE_STATE`). `_NET_MOVERESIZE_WINDOW` and `_NET_REQUEST_FRAME_EXTENTS` need no decision from the host and are handled by the
server directly.

**Placement**: the host does not draw X's border; the native window's content area goes at (`X + BorderWidth`, `Y + BorderWidth`), and the
native frame stands in for the border. When `NeedsPlacement` is true (window creation, a client moving a top-level, reparenting to the root —
the position is what the client asked for and no window manager has placed it yet), the host aligns the native window's **frame** (not its
content area) with the requested position by `WinGravity`, per ICCCM §4.1.2.3: `PlaceInFrame(frame)` gives the position the X window should take
once wrapped in a frame `frame` wide on each side (`Static` leaves the content area in place; for NorthWest the result does not depend on the frame
size, so nothing jumps when the window is shown), and the host reports it back with `MoveTopLevel`, after which the flag is false. When the
request is (0, 0) and not user-specified (no `UserPosition`), the host may choose a position; `xterm -geometry +0+0` carries USPosition and is
honoured. Override-redirect windows are not placed by the window manager and never have the flag. `_NET_MOVERESIZE_WINDOW` is converted directly
using the gravity (0 = the window's own win_gravity) and the frame the host reported; a width or height outside 1–32767 makes the whole request
ignored, coordinates are clamped to 16 bits, and a growing buffer is checked against the memory account first.

**Naming**: `Inject*` is synthesized user input; methods whose names contain `TopLevel` and take an `XTopLevelWindow` first are the host acting as
window manager on that top-level, named "verb + TopLevel + object" (`MoveTopLevel`, `SetTopLevelStates`, `ChangeTopLevelStates`,
`KillTopLevelClient`); `Set*` without `TopLevel` means something in the host's environment changed and is being pushed into the server (keymap,
monitor layout, DPI, lock keys, clipboard contents). Renaming a public method is a breaking change for embedders: when a name does not fit the
rule, fix how the rule is written first.

Clipboard exchange is controlled by `X11ServerOptions.SyncClipboard` (CLIPBOARD, on by default), `SyncPrimary` (PRIMARY, off by default) and
`ClipboardFollowsFocus` (exchange only with the session that has the keyboard focus, on by default; see §7).

**Every top-level window owns a pixel buffer** (effectively always-on backing store + Composite; InputOnly top-levels have none): child windows draw
into their top-level's buffer, clipped to their visible region. Content hidden behind other native windows is never
lost, at the cost of memory (charged to the memory account, §7) — clients need not redraw whenever occlusion changes. Expose is only sent on map,
growth, ClearArea(exposures) and when an unmapped child reveals its parent; when a top-level's bounding / clip shape changes only the newly revealed
part is repainted, and changing the input shape repaints nothing. On resize the buffer moves rows in place when it can, and a new one gets a quarter
extra (the slack is not charged).

## 7. Key trade-offs

- **TrueColor visuals only**: depth 24 (the root visual, masks `0xff0000 / 0xff00 / 0xff`) and depth 32 (ARGB, reserved
  for RENDER); pixmaps may additionally have depth 1 / 4 / 8 / 15 / 16 (following X.Org's convention — Xt programs create
  depth-4 / 8 pixmaps and got BadValue while only 1 / 24 / 32 were allowed). Pixel values are RGB, AllocColor is just a
  conversion and colormaps never "run out"; writable colour cells (AllocColorCells) are always BadAlloc.
  Colour names are looked up in X.Org's `rgb.txt` (an embedded resource parsed on first lookup; spaces and case are ignored, so
  `dark slate gray` and `DarkSlateGray` are the same entry, and numbered variants such as `red3` and `VioletRed4` are all there).
- **Software rasterization, not Skia**: core drawing semantics are **pixel-exact** (thin-line Bresenham endpoints,
  GXxor rubber bands, plane masks), which anti-aliasing 2D libraries cannot reproduce. The host receives 32-bit pixels;
  how they reach the screen is the host's business. As the protocol specifies, integer coordinates are pixel centres: polygons, wide
  lines and arcs sample row `row` at y = row, edges are closed at the top and open at the bottom, spans closed on the left and open on
  the right (RENDER's coverage masks are centred at +0.5 as the RENDER specification says and are unaffected).
- **Core drawing follows the protocol's details, and costs follow what is visible**:
  - Intersect with the clip first: rectangles are intersected with the drawable area before rows are filled; a thin line only walks
    the part that crosses the drawable area (for this Bresenham the minor-axis position at step k has a closed form, so the start is
    found by bisection and the walk continues from there, producing the same pixels as walking from the beginning); clip tests bisect
    the bands; coordinates accumulated in CoordModePrevious saturate at ±2³⁰; round caps / round joins of wide lines have at most 1024
    vertices, and segments and joins whose bounding box misses the drawable area are skipped outright.
  - Wide lines and arcs honour line-style, join-style and cap-style: Miter extends the two outer edges until they meet (falling back to
    Bevel below 11°), Bevel adds the outer triangle, Round adds a disc; zero-length segments are dropped from the path, and a line that
    collapses to a point draws a disc with Round and a square with Projecting. Dashes are measured along the line; OnOffDash draws only
    the even dashes; DoubleDash draws the odd ones in the background — sourced per fill-style: Solid uses the background colour,
    Stippled the background colour masked by the stipple, Tiled / OpaqueStippled the same source as the even dashes. A wide arc that is
    not a full circle gets caps at both ends per cap-style, and its inner and outer boundaries are ellipses with the semi-axes plus / minus
    half the line width, not rounded. An arc whose bounding box has zero width or height is a line segment; the protocol leaves the outline
    to the implementation only when both are nonzero, so here it is the ideal one, the two lines half the line width from the path: each
    monotonic stretch is drawn the full line width, and where the path turns back between its ends (at ±90° for zero width, at 0° / 180° for
    zero height) the outline goes half a turn around that end of the segment, adding a disc one line width across. PolyLine with three or more points whose first and last coincide is drawn as a closed path (a join
    there rather than two caps; a thin line does not draw the end point twice). Arcs in a PolyArc whose end point coincides with the next
    one's start point "join correctly" as the protocol says: "coincide" means less than half a pixel apart on each axis (with 1/256 pixel
    of slack) — the end points are real numbers, so exact equality is too strict and rounding to the same pixel is unstable at .5; the last
    arc joining back to the first closes the chain. A chain of joined wide arcs is one path: consecutive arcs get a join per join-style
    (built on the real corners of the two bands' end faces, since an ellipse's end face is not necessarily perpendicular to the tangent),
    caps go only at the two ends of the chain and none when it is closed, and the whole chain is filled in one active edge table (no seam
    drawn twice under GXxor); consecutive pieces of the same ellipse are merged into one (four 90° arcs make exactly the pixels of one
    360° arc). Dashes are measured along the whole chain and carry across joins; each unjoined chain restarts at the dash-offset. A thin
    chain is one thin polyline whose joints are drawn once. A wide chain is filled in batches of at most 2²⁰ vertices. A single arc and
    unjoined arcs draw the same pixels as before (200,000 random comparisons).
  - CopyArea / CopyPlane: only the visible part of a source window can be copied (with ClipByChildren mapped children obscure it too, with
    IncludeInferiors their contents are copied along); the parts that cannot be copied (obscured, unviewable, a child sticking out of its
    top-level's buffer) are first painted with the destination window's background when it is not None, then reported one rectangle at a
    time as GraphicsExposure, and NoExposure is sent only when everything was copied. CopyPlane with a bit-plane ≥ 2^source depth gets
    BadValue. GetImage fills the part that sticks out of the top-level's buffer with 0 (obscured contents are undefined anyway).
  - A failing request has no effect: CreateGC / ChangeGC read and check all values before writing any into the GC, and so do SetDashes and
    RENDER's FreeGlyphs; out-of-range GC enums (function, line-style, cap-style, join-style, fill-style, fill-rule, subwindow-mode,
    arc-mode…) get BadValue; PutImage in Bitmap / XYPixmap format with left-pad ≥ 32 gets BadMatch.
  - With single-byte fonts a CHAR2B is read as a 16-bit number, high byte first; a non-zero byte1 is a missing glyph and uses default-char.
- **Keycodes use evdev numbering (evdev + 8)**, like X.Org on modern Linux — most remote clients expect it. The host
  translates physical keys to X keycodes; the keysym table comes from the library (US layout to start, replaceable). `XKeycodes` also has
  the keys of Japanese JIS / Brazilian ABNT2 / Korean keyboards (Ro, Yen, Henkan, Muhenkan, Hiragana_Katakana, KP_Equal, Hangul,
  Hangul_Hanja), F13–F24 and the media keys, and the initial keysym table has keysyms for all of them.
- **Keyboard: cadence and layout come from the host, semantics from the server**:
  - The auto-repeat cadence is the host's system setting: when a key that is already down gets another press, the host marks it with
    `InjectKey(keycode, true, repeat: true)`, and the server applies X's rules — with `xset r off`, auto-repeat turned off for that key, or a
    modifier key (per-key repeat is on by default for everything except modifiers) it is dropped; otherwise core clients without XKB
    DetectableAutoRepeat get a release / press pair, those with it get only the press, and XI2 KeyPress and RawKeyPress carry the KeyRepeat
    flag; the release in between does not end passive grabs. XKB's RepeatKeys / PerKeyRepeat are the same settings as the core ones; the
    repeat delay and interval are only recorded and reported.
  - ChangeKeyboardControl takes effect: every item is checked before any takes effect (−1 restores the default, other negative values are
    BadValue; led / key without mode is BadMatch); the bell's base volume / pitch / duration, the LEDs, key click and global and per-key
    auto-repeat are recorded, and GetKeyboardControl reports them truthfully. After `xset b off` the computed bell volume is 0 and the host
    stays silent.
  - SetPointerMapping takes effect: 9 buttons (GetPointerMapping and XI report the same mapping); a wrong length or repeated non-zero
    entries get BadValue, a button that would change while held gets Busy, and success sends MappingNotify. The host and XTEST supply
    physical buttons; core and XI2 events, state bits and the implicit grab use the mapped numbers, and raw events report the physical ones.
    SetModifierMapping gets BadValue for keycodes outside 8–255 and Busy when an affected key is held.
  - `SetKeymap` changes only the keys whose keysyms differ from what the host supplied last time (a key a client changed with xmodmap and the
    host did not change keeps the client's change); right Alt's keysym changes only when its role (AltGr / Alt_R) changes, and it only moves
    between Mod1 and Mod5, leaving the rest of the modifier map alone; when nothing changed no MappingNotify / MapNotify is sent. The host
    checks the system layout before every press in an X window (auto-repeats excluded) and, if it changed, pushes the new keymap before
    injecting the key (on Windows only the layout handle is compared; on macOS / Linux computing the keymap is costlier, so at most once a
    second; not when a layout is chosen in the settings).
  - Lock keys: CapsLock / NumLock start off in the server, so the host pushes the system's real state whenever an X window gets the focus
    (`SetLockState`, no synthesized key presses; Windows uses GetKeyState, macOS CGEventSourceFlagsState with NumLock treated as always on,
    Linux reads the desktop X display's XKB lock modifiers on a background thread).
  - Idle time: the host reports activity in its own UI through `NoteUserActivity`, so the idle time remote programs see through
    MIT-SCREEN-SAVER / SYNC's IDLETIME no longer counts X input alone.
  - On macOS, key combinations with Command may never get a KeyUp: the host remembers the keys pressed while Command is held and releases
    the ones still down when Command goes up (if Avalonia does deliver the KeyUp, this does nothing).
- **Authorization**: listens on `127.0.0.1` (TCP by default only with a cookie configured, §4) and Unix sockets. In order: ① a stream the caller
  has already authenticated (`ServeAuthenticatedAsync`, used by the host's SSH connector — the fake cookie was already checked at the SSH layer)
  is admitted; ② a correct `MIT-MAGIC-COOKIE-1` (compared in constant time) is admitted; ③ a peer known to be the user running the server is
  admitted — when the peer uid is available (SO_PEERCRED on Linux, getpeereid on macOS / FreeBSD) the uid decides and must equal this process's
  effective uid; only when it is not (Windows) does the socket file's 0600 mode, set between `bind` and `listen`, decide (a custom path on a file
  system such as 9p / drvfs, where chmod silently does nothing, would otherwise let anyone in);
  ④ a peer whose uid is known and belongs to another user is refused (sockets in the Linux abstract namespace have no file
  permissions, so without the uid check any local user could connect, read windows, log keystrokes and inject input through XTEST), and
  without a configured cookie the connection is closed right after accept; ⑤ with a configured cookie everything else is refused (loopback TCP
  included: any local process or user can reach that port); without one, as with X.Org's host access control, only local connections are
  admitted. The cookie must be 16–256 bytes (`MinAuthorizationCookieLength` / `MaxAuthorizationCookieLength`; an empty array used to admit any
  cookie with empty data) and is copied at construction, so later changes to the caller's array do not affect authorization. The access policy
  is fixed: ListHosts reports access control enabled with an empty host list, and ChangeHosts and SetAccessControl(Disable) get BadAccess —
  they used to succeed silently, so `xhost +` looked effective while nothing changed.
  The host's built-in server generates a fresh cookie on every start and writes it to the user's `.Xauthority` (`XAUTHORITY` first; encoded and
  decoded with the SSH library's `XAuthority`, and a file the strict parser cannot fully read is not rewritten; xauth's lock convention — first
  `-c`, then `-l`, and a lock younger than a minute is not treated as stale; on stop it removes only its own entry; when the network address
  changes it checks the host name and, if it changed, re-registers under the new name and removes the old entry), so local X programs send it
  automatically through Xlib; programs that cannot read that file are refused.
  ⚠️ A design-level caveat (not a defect): every SSH session with X11 forwarding shares this one trusted display — a compromised
  remote machine can, through its forwarding, see and operate X programs from other sessions (read window contents, log keystrokes,
  inject input). That is what trusted X11 forwarding means; do not enable X11 forwarding for remote machines you do not trust. What can be
  tightened is under "Trust levels" and "Several sessions sharing one display" below; to separate sessions completely the host can give each
  SSH session its own display ("One display per SSH session" below). Before 2026-10-10 the library had no SECURITY extension and the built-in
  engine did not set up X11 forwarding at all for connections in untrusted mode; now untrusted mode forwards through the connector as usual and
  the client that connects is untrusted.
- **Listening and display numbers**:
  - A display number whose names are taken is not used at all: when the Linux abstract name is already bound by someone, something listens
    behind the socket file, a leftover socket file cannot be deleted (it belongs to another user, who could listen on it again at any time), or
    bind collides again after deleting it, `StartAsync` throws `SocketException` (`AddressAlreadyInUse`) and withdraws the TCP and Unix listeners
    already opened. It used to log a line and start on the other transports anyway — an attacker could bind the abstract name without listening,
    wait for us to start, then listen and receive the cookie Xlib sends when it tries the abstract name first for `:N`.
  - The directory holding the socket file (checked after it is created, in case someone created it first) must be a directory rather than a
    symbolic link, owned by this user or by root with the sticky bit set; otherwise no socket file is opened and a line is logged (the owner comes
    from statx on Linux, lstat on macOS). On macOS `/tmp/.X11-unix` usually does not exist, and whoever creates it first owns it.
  - Probing whether a socket is in use (when the server starts, and when the host picks a display number) is always an asynchronous connect
    limited to 300 ms; on Linux a non-blocking connect returning EAGAIN, or the time running out, counts as "someone is listening" and their socket
    file is left alone — a blocking connect to a socket whose backlog is full waits forever, and the X Server button could never start again.
    For the abstract name the host instead tries to bind it (atomic; released at once if it succeeds).
  - On Unix-like systems the server holds `/tmp/.X{N}-lock` per Xserver(1) (contents: the PID right-aligned in ten characters plus a newline,
    mode 0444) and deletes it on shutdown or when start fails half way; a live holder, or contents it cannot read, makes `StartAsync` throw
    `SocketException` (`AddressAlreadyInUse`); a holder that is gone means a stale lock, which is replaced; an unwritable `/tmp` only logs a line.
    Xvfb and `xvfb-run -a` look only at this file when picking a display number.
  - Being configured to listen and getting neither TCP nor a Unix socket up throws `IOException`. When the host picks the display number
    automatically and only finds out at start that it is taken (someone got in between the probe and the bind, or the lock file / abstract name
    shows a use the probe cannot see), it retries with the next free number, up to 4 times; a number set by hand reports the error directly.
  - On Windows the TCP listener sets `SO_EXCLUSIVEADDRUSE`: otherwise another process of the same user (low-integrity ones included) can still
    bind a more specific address with SO_REUSEADDR and take over local connections (cookie included). When the host decides whether "another X
    display is already in use", a listener behind `localhost:0` only counts if it is a process in the current user's session — on a terminal
    server it may be another user's VcXsrv.
- **Trust levels (SECURITY extension, 2026-10-10, xs_plan F2)**: based on the X Consortium *Security Extension Specification* 7.1 (error and
  event numbering and the SecurityGenerateAuthorization request layout follow the xorgproto protocol headers — the spec's chapter 5 encoding table
  lists the value-mask after the two strings, while what xauth actually sends matches the headers, with it in the fixed part).
  - **The extension**: QueryVersion (1.0), GenerateAuthorization (MIT-MAGIC-COOKIE-1 only, a 16-byte random cookie; timeout defaults to 60 seconds,
    trust-level to untrusted, group must be None, event-mask has only AuthorizationRevoked), RevokeAuthorization (clients connected with it are
    disconnected too), the AuthorizationRevoked event; an authorization expires after timeout seconds without a connection; at most 256. Issued
    cookies are checked when the client is registered (on the execution thread); a mismatch is refused as before.
  - **Named by the host**: `ServeAuthenticatedAsync(stream, label, XClientTrust.Untrusted)` — an in-process connector need not issue a cookie
    (this is how SSH's `ssh -X` comes in).
  - **What an untrusted client is restricted to** (spec chapter 3): it can only name resources of untrusted clients — trusted clients' windows,
    pixmaps, GCs… are treated as nonexistent (checked where requests resolve IDs; the server's internal lookups are unchanged); QueryTree /
    GetGeometry / TranslateCoordinates are unrestricted. The root window (and the server's own windows) can be used only in the requests the spec
    lists, plus RANDR, XINERAMA, XFIXES Select*Input and XIQueryPointer / XISelectEvents / XIGetSelectedEvents (toolkits send them on the root
    window at startup, and a BadWindow makes Xlib's default error handler exit the program); on the root window only StructureNotify /
    PropertyChange can be selected, and XI2 keeps only device / hierarchy / property changes; only the events ICCCM prescribes can be sent to the
    root window. XTEST, MIT-SCREEN-SAVER, DPMS, X-Resource, Composite, MIT-SHM and SECURITY are invisible to it. When the keyboard is not its own
    (by current focus, pointer, grabs and selections, keyboard events reach no untrusted client): QueryKeymap / KeymapNotify read all zeros,
    GrabKeyboard and keyboard XIGrabDevice return AlreadyGrabbed, SetInputFocus has no effect, its passive keyboard grabs do not activate, and
    key-caused XKB StateNotify is not sent to it. Changing the keymap, modifiers, keyboard and pointer controls, XKB / XI device settings, host
    access control and KillClient(AllTemporary) return Access; GrabServer is ignored. CUT_BUFFER0–7 on the root window are hidden from it, and
    its property changes on server windows are treated as no-ops (except the `_VELASHELL_*` atoms on the selection window that the server uses to
    fetch the clipboard for the host — selections still work through the host); asking for a selection owned by a trusted client returns
    property None; GetImage fills areas obscured by other windows with 0; a None background it sets is drawn black.
  - `RestrictForwardedClients` stays: it is the middle tier "trusted, but no XTEST / raw keys / device hierarchy changes" for old programs that
    misbehave when untrusted; untrusted clients get all three restrictions anyway.
- **One display per SSH session (host, 2026-10-10, xs_plan F1 / decision Q5)**: on one display, trusted sessions can still see each other's
  windows, read each other's clipboard and inject input through XTEST. With the host setting on, x11 channels that come in with a session object
  go to that session's own `X11Server`: it listens on nothing (`ListenTcp = false`, `UnixSocketPath = ""`), is fed only through the connector,
  has its own root window, selections and clipboard, and XTEST and raw events reach only programs of the same session. Channels of the same
  session share one; it is created on the first channel and removed when the session disconnects or the server stops; at most 32 — beyond that a
  new session's X programs cannot connect (they do not fall back to the shared display, which would silently undo the isolation the user asked
  for). Local programs (`DISPLAY=:N`) and channels without a session still go to the shared one. The host gives each server its own
  `IX11ServerHost` (windows, keymap, DPI and clipboard are kept apart), and each of them gets all the host-side settings (keyboard layout, window
  mode, local input method, source labels). In one-window mode, closing one session's screen window closes only that session's display (the host
  tells `BuiltInLocalXServer` through `IEmbeddedXServerHost.SessionDisplayCloseRequested`, asking first when programs are connected); other
  sessions and the shared display are unaffected, and the session gets a new display the next time it opens an X program — before the 2026-10-10
  review, closing any session's desktop stopped the whole X Server.
- **Several sessions sharing one display — what is tightened** (on one display, between trusted sessions):
  - **The clipboard follows the session with the keyboard focus** (`X11ServerOptions.ClipboardFollowsFocus`, on by default): the host's text can be
    read only by the client owning the focused top-level and by clients with the same connection label (`xclip` / `xsel` in the same SSH session;
    labels come from `ServeAuthenticatedAsync(stream, label)`), and only copies from that session reach the host; with no X window focused nobody
    can read it. Otherwise a password copied locally would be readable by every session as soon as the user clicked any X window, and programs in
    background sessions could keep rewriting the local clipboard. While the server owns the clipboard on the host's behalf, XFIXES owner
    notifications also go only to clients that can read it — other sessions could otherwise learn exactly when the local clipboard got new
    content. While it is on, PRIMARY, SECONDARY and CLIPBOARD are also **isolated per
    session**: the selection table is keyed by "atom + scope", and each session (clients with the same connection label; local programs without a
    label all belong to one local session) has its own owner — SetSelectionOwner, GetSelectionOwner, ConvertSelection, SelectionClear, XFIXES
    notifications, and an owner window being destroyed or its owner disconnecting all take effect only within that session. Other sessions see no
    owner changes, get no SelectionClear because of them, and cannot read what was copied in another session. All other selections (`WM_S0`,
    XSETTINGS, drag and drop …) stay display-wide as the protocol defines them. Copy and paste across sessions goes through the host clipboard:
    clipboard content has a logical clock (new text from the host, a copy from the X side handed to the host, and a program in some session taking
    a synchronized selection each advance it), and when a session gets the focus (with PointerRoot, when the pointer moves to another top-level)
    and its copy is older than the latest text, the server takes the selection there on the host's behalf — copy in A, switch to B, paste, and the
    content arrives; the most recent copy wins, whichever session or the local machine it happened in. Programs in the same session copy and paste
    to each other as usual. Embedding scenarios with a single trusted client can turn it off.
  - **`RestrictForwardedClients`** (off by default): connections that come in through `ServeAuthenticatedAsync` cannot see XTEST (synthesized input
    is indistinguishable from the real keyboard, so a compromised remote machine could type commands into another session's xterm), receive no
    XI2 raw key events (which would let a client log every key typed in every X window without taking the focus), and get BadAccess for
    XIChangeHierarchy (which can disable physical input devices). It is off by default because remote tools such as xdotool rely on XTEST; the
    host has a switch for it in its settings.
  - **Focus-stealing prevention**: the server decides `XActivateRequest.UserInitiated` from the source and the timestamp (§6). The Avalonia host
    activates only when it is true and the user is using an X window at that moment, and otherwise just flashes the taskbar entry;
    override-redirect and "always on top" windows stay on top only while an X window is active, and drop behind when the user returns to a local
    window — X's native windows live in the same process as the host's main window, so the system's foreground lock does not stand between them,
    and a remote program used to be able to jump to the front while the user typed a sudo password, or cover the screen with a full-screen popup
    that looks like a system credential prompt.
  - **The server holds the root window's SubstructureRedirect and `WM_S0`**: another client selecting SubstructureRedirect on the root gets
    BadAccess (as on a real desktop that already runs a window manager), so an openbox / xfwm4 started by mistake on a remote machine cannot take
    over every session's new windows. save-set works as the protocol's "Connection Close" says: on disconnect, windows in the save-set that are
    inferiors of the client's windows are reparented to the closest ancestor the client did not create (keeping root coordinates) and mapped if
    unmapped, before resources are destroyed; core ChangeSaveSet on the client's own window gets BadMatch, and XFIXES ChangeSaveSet supports
    the target and map flags.
  - **The server's own windows are protected like the root**: `0x43` (shared by the clipboard bridge, XSETTINGS and the WM check) and Composite's
    overlay window cannot be reparented, reconfigured or have windows created under them (BadMatch); ChangeWindowAttributes may only change event
    selections (BadAccess otherwise); Map / Unmap are ignored; destruction skips them — reparenting one and then destroying its new parent used to
    take down the whole display's clipboard bridge, XSETTINGS and window-manager check.
  - **Selection timestamps are always checked**: a SetSelectionOwner with a time later than the server's current time, or earlier than the
    selection's last ownership change, has no effect, and the last change time survives the owner going away — a malicious client that took
    CLIPBOARD with a timestamp in the future no longer makes everyone else's real-time attempts silently fail. When the host takes ownership for
    the user it uses max(current time, last change time).
  - **Nobody else can break a drag**: only the client holding the pointer grab can make the server release the button and the implicit grab
    through `_NET_WM_MOVERESIZE`; requests from anyone else are still passed to the host but leave button state and grabs alone.
    `_NET_MOVERESIZE_WINDOW` geometry is checked against X's ranges (§6).
  - **There is a way out of a hang**: the host's `BreakGrabs`, `DisconnectClient` by number and `KillTopLevelClient`; a GrabServer held too long
    names its holder in the log and is reported to the host through `ServerGrabStalled` (§5). In the host (VelaShell) the way in is the flyout of the
    title-bar X Server button — disconnect connected programs one by one, "Unstick" — and a toast with "Disconnect it" when a grab is held too long. At most 16 clients may keep their resources in Retain mode (any further one is treated as Destroy on
    disconnect, with a log line), and X-Resource lists them — a loop of "connect → RetainPermanent → disconnect" 254 times used to use up every
    client number, after which nobody could connect.
  - GLX: any client can use another's context as a share list and MakeCurrent / CopyContext / DestroyContext another's context — consistent with
    the core protocol's trust model, which does not isolate clients from each other (context tags are per client, so they cannot be forged); this
    will be tightened together with a per-connection trust level, and the code marks the spots.
- **Every resource has a per-item limit and an overall account; exceeding either gets BadAlloc (or GL's OUT_OF_MEMORY) instead of taking the
  process down**:
  - Per-item limits: 255 clients (connections still in setup are counted separately, at most 32); windows nested at most 256 deep, 32768 windows
    per client (destruction and repainting walk an explicit stack, not recursion); regions at most 16384 rectangles, with a merge budget for
    union / intersect / subtract; pixel buffers 2²⁶ pixels; property values 32 MB (XI device properties too); client-created atoms 2¹⁸ with 16 MB
    of names in total (names from XFIXES SetCursorName go through the same gate); XI2 device IDs up to 255; XC-MISC at most 65536 IDs per
    request; passive grabs (core and XI2) 4096 per window per client, with at most 1024 subtracted combinations per grab; XFIXES selection
    listeners 1024 per client; 256 Damage objects per drawable; 256 queued Present requests per client and 64 entries per PRESENTNOTIFY; 16
    clients retaining resources; GetImage replies 256 MB; clipboard text 16 MB; GLX below.
  - Memory accounts: `X11ServerOptions.MaxClientMemory` (per client, 1 GiB by default) and `MaxTotalMemory` (all clients together, 2 GiB by
    default) — with per-item limits alone, one 16-byte CreatePixmap is 256 MB, and a dozen of them exhaust the memory of the server and the host
    process with it. Charged are: a fixed overhead per resource (64 bytes); pixmaps with a buffer of their own (not the kind NameWindowPixmap gives out, which borrows the window's buffer); top-level buffers (checked at map and
    resize); DOUBLE-BUFFER back buffers; Composite pixmaps of subwindows, and pixmaps from NameWindowPixmap that keep the old buffer to
    themselves after the top-level is resized or remapped (charged to the pixmap's owner); Present pixmaps freed before they were presented
    (charged to the client that sent the request); property values and XI device properties (charged to the client that wrote them); RENDER
    glyphs (charged to the client that added them, released when the last reference to the glyph set goes) and gradient stops; XFIXES regions;
    GLX objects and a RenderLarge being assembled (see GLX below). Everything is refunded when a resource leaves the resource table, a property is
    replaced or deleted, a window is destroyed or a client disconnects, and also when the server's own write replaces a value a client wrote.
    Client requests are checked before allocating and get BadAlloc with no effect when over; resizes initiated by the host are charged but never
    refused; the slack kept when resizing buffers is not charged. `XClientInfo.MemoryBytes` reports each client's account.
  - On top of that each work item has a work budget (§5).
- **Fonts**: core fonts are the X.Org bitmap fonts and GNU Unifont bundled with the library (xs_plan CP-16 / F21 option A): the whole misc-fixed set
  (`4x6` … `10x20` in regular / bold / oblique, the CJK `12x13ja` / `18x18ja` / `18x18ko` and JIS X 0208 `k14`), `nil2`, `cursor`; Adobe's 75 / 100 dpi
  Courier, Helvetica, New Century Schoolbook, Symbol and Times (8–24 pt); GNU Unifont 18 (16 pixels, the whole Basic Multilingual Plane).
  The 233 BDF files as is, Brotli-compressed, take about 5 MB (the library DLL grows by about 4.5 MB), laid out like X.Org's installed font
  directories as misc / 75dpi / 100dpi (their order is the font path order), each with an mkfontdir-style `fonts.dir`. An ISO10646-1 font also
  appears under the names of the single-byte charsets it covers **completely** (as `mkfontdir -e` does): ISO8859-1 is the first 256 code points of
  Unicode; ISO8859-2 / 3 / 4 / 5 / 6 / 7 / 8 / 9 / 13 / 15 and KOI8-R / U are derived through the bundled mapping tables (exported from .NET's code
  page tables; no ISO8859-10 / 11 / 14 / 16), and a derived font's CHARSET_REGISTRY / CHARSET_ENCODING change with it — 1877 names in all.
  Aliases follow the X.Org misc directory's `fonts.alias` verbatim (`fixed`, `variable` — Helvetica Bold 12 pt —, `5x7` …); aliases whose targets are
  not bundled (Sony, JIS, ISAS and OPEN LOOK fonts) are not listed and get BadName; `9x18` / `9x18bold` are also accepted, and `8x16` / `12x24`
  (Sony) fall back to the nearest misc-fixed. When a full 14-field XLFD has no exact match, the font with the nearest pixel height among those
  whose foundry, family, weight, slant and charset match is used (by PIXEL_SIZE, else converted from POINT_SIZE and RESOLUTION_Y — the
  screen resolution `X11ServerOptions.Dpi` when RESOLUTION_Y is left open —, with no guessing when neither is given); on a tie, when AVERAGE_WIDTH is given, the closest average width wins first, then the smaller size — so
  `-misc-fixed-medium-r-semicondensed--13-120-75-75-c-120-iso10646-1`, a double-width request made by doubling the average width, gets 12x13ja
  rather than 7x13. The average width only breaks ties and never trades a closer size for a closer width. For a name that asks by point size with the
  resolution left open (POINT_SIZE is a number, PIXEL_SIZE and RESOLUTION_Y are not; `variable` is one), the match whose RESOLUTION_Y is
  closest to the screen resolution is taken first — XLFD's POINT_SIZE is a physical size, so on a 96 dpi screen 12 pt gets the 100 dpi font
  (17 pixels) rather than, by name order, the 75 dpi one (12 pixels); when falling back to the nearest size, ties are broken by resolution too. Families that are not bundled (B&H's Lucida — its licence requires particular notices in
  user documentation and code comments —, Bitstream, scalable fonts) are still BadName. The name table is built once on first use; a font is
  decompressed and parsed the first time it is opened, and the result is shared by the whole process (several server instances parse it once);
  glyph bitmaps are packed by row (1 bit per pixel). Measured (Debug): the name table plus `fixed` 35 ms, Unifont's first open 180 ms and 9.6 MB,
  every name opened once (`xlsfonts -l`) 870 ms and 58 MB in all. None of this happens on the execution thread: when OpenFont or
  ListFontsWithInfo needs fonts that are not built yet, they are decompressed and parsed on the thread pool while this client's request and
  the ones after it are held (the same mechanism as SYNC's Await) and put back in order once the fonts are ready — other clients and the
  host UI do not wait on the pixel lock meanwhile. A quoted BDF property value is a string (an atom in QueryFont), even when it
  looks like a number (`CHARSET_ENCODING "1"`).
  Glyph cursors are baked into images as the protocol specifies (the origins of the source and mask glyphs coincide at the hotspot); one with
  no visible pixel (xterm's invisible pointer, a blank glyph of `nil2`) reaches the host as `Hidden`, and an undefined glyph gets BadValue.
  Glyphs of the `cursor` font (X.Org's cursor.bdf) also record the glyph number: those with a matching system cursor (left_ptr, xterm, watch, the
  resize edges and corners …) reach the host as a shape without an image — the system cursor follows the desktop's theme and scaling —, the rest
  (pencil, gumby, dotbox …) hand the host the image; XFIXES GetCursorImage always returns the baked image. Modern toolkits do not use core fonts
  (they use RENDER with client-side rasterization), so core fonts only need to cover older programs.
- **RENDER composites in integers on 8888, a8 and a1 targets**: pixels are always premultiplied. Onto a8r8g8b8 / x8r8g8b8 / a8 / a1
  targets, all 53 operators (Porter-Duff, Disjoint / Conjoint, the PDF blend modes, with or without component alpha) first fetch the
  source and the mask each as a row of 8-bit premultiplied pixels — gradients, transforms, repeat and every source format are handled
  while sampling (solid colours, 8888 and a8 are just rearranged, bilinear filtering uses 0–256 fixed-point weights, gradient colours
  are still computed in floating point and quantized once) — and then combine per pixel in integers. Src / Over / Add have dedicated
  code; the other Porter-Duff and the Disjoint / Conjoint operators compute source × Fa + destination × Fb per channel, with factors
  in 0–255 when there is no division, while the ones with a division (Saturate, Disjoint / Conjoint's min(1, n / d) and
  max(1 − n / d, 0)) use the unrounded source alpha × mask in units of 1/65025 and round once per channel at the end. The blend modes
  do not divide out cs = Cs / αs first; αs·αd·B(cs, cd) is rearranged into an integer expression in the 8-bit Cs, αs, Cd and αd (soft
  light's square root comes from a table, the four HSL modes multiply both sides by αs·αd), and the three terms are summed in units of
  1/255³ and rounded once. With component alpha the factors are per channel, from that channel's own source alpha. The alpha-only a8 / a1
  targets compute just the alpha channel (the blend modes' alpha is αs + αd − αs·αd, and component alpha does not affect it); a1 is 1
  for ≥ 0.5. With 8-bit pixel sources the result is at most 1 away from per-pixel floating-point compositing, and a1 is bit for bit
  identical (checked exhaustively); bilinear and gradient sources are quantized to 8 bits first, at most 2 away from all-floating-point.
  Of these, solid source + one-byte mask + Over (Xft text, cairo's anti-aliased shapes) and 8888 image Src / Over (image blits,
  window-to-window copies) skip row sampling and work directly on the stored pixels. Other target formats such as r5g6b5, x1r5g5b5 and
  a4 composite per pixel in floating point (0–1 per channel). When the source or mask is the
  target's own buffer, the part to be read is copied out first (the result must be as if the source were read before any write): the area that
  will actually be written is mapped back to source coordinates, using the bounding box of the transformed corners when there is a transform,
  and folded back per repeat — the whole picture is not copied. Alpha-only glyphs are stored one byte per pixel; one CompositeGlyphs request
  computes its target once and records damage once, and with a mask format the mask is allocated only for the writable part of the target.
  Trapezoids and triangles use 16 sub-scanlines per row with analytic horizontal coverage, computed only over the writable part of the target.
  Source / mask picture clipping also limits what is read when there is no transform and no repeat (RENDER 0.11 §7: the clip-mask "affects all
  graphics requests, including sources"); with a transform or repeat, and for the sources of trapezoids and glyphs, only the target is clipped.
  Alpha maps follow RENDER 0.11 "CreatePicture": the alpha map's alpha channel
  replaces the drawable's (the colour channels stay as they are — pixels are always premultiplied), its origin is relative to the drawable's,
  and both reading and writing are clipped to the alpha map's extent and clip. As a source, the transform and filter apply to the picture the
  drawable and the alpha map make together; the alpha map itself is not transformed or repeated, and with a transform the alpha read outside it
  is 0; a pixel outside the drawable with no repeat is fully transparent. As a destination, the area to be written (the request ∩ the writable
  region) is assembled into a temporary a8r8g8b8 image and composited there, then the colour is written back to the drawable (whose own alpha
  channel is left alone) and the alpha to the alpha map, with DAMAGE noted. A source with an alpha map that shares a buffer with the destination
  (the drawable or the alpha map is the target) is likewise copied out before writing. The alpha map must be a picture on a pixmap (else
  BadMatch), in any format; one that already has an alpha map cannot be attached (the spec leaves that undefined; here it is BadMatch). An alpha
  map applies one level only: if the picture used as an alpha map is given an alpha map of its own afterwards, that one is ignored when
  compositing — otherwise a long chain built one link at a time would recurse once per link and overflow the stack at around six thousand links,
  taking the whole host process down. poly-edge / poly-mode / dither are accepted but have no effect. Gradient stops must lie in 0–1 and be
  sorted (BadValue otherwise;
  equal stops — hard transitions — are fine), and sampling finds the stop by bisection; CreateCursor with a hotspot outside the image gets
  BadMatch; AddGlyphs bitmap sizes and glyph / stop counts are checked with arithmetic that cannot wrap around (BadLength when too large).
- **RANDR is mostly read-only for clients; the host supplies the layout**: one CRTC / output / mode per monitor (`X11ServerOptions.Monitors` or
  `SetScreenLayout` at runtime; every monitor must lie within the root window, at most 16, invalid ones throw `ArgumentException` on the spot).
  CRTC / output IDs stay with a monitor's name (unplugging one does not shift the others' IDs so that a client's ID now points at another
  monitor); change events are sent per SelectInput when the layout changes, a DPI change also sends ScreenChangeNotify, and a dot clock that
  overflows 32 bits is saturated. XINERAMA reports the same layout, with QueryScreens and GetScreenSize in the same order. `XMonitor.WorkArea`
  (the part of a monitor left for windows once the taskbar and docks are taken off) drives `_NET_WORKAREA`: what each monitor gives up at the edges
  of the virtual desktop is subtracted from the root window (EWMH has a single work-area rectangle, so a taskbar between two monitors cannot be
  subtracted), so menus, maximized windows and dialogs avoid the taskbar. Client configuration requests: SetScreenSize to the current size
  succeeds and any other size gets BadValue; SetCrtcGamma and SetOutputPrimary validate their arguments and are silently accepted without effect
  (BadAccess made colour-temperature tools and desktop sessions using Xlib's default error handler exit); the rest (SetScreenConfig,
  SetCrtcConfig…) get Failed or BadAccess — in rootless mode the host decides where windows go and how monitors are arranged.
- **The server doubles as the XSETTINGS manager**: it owns `_XSETTINGS_S0` and publishes `Xft/DPI`,
  `Gdk/WindowScalingFactor` and a few more, and publishes RESOURCE_MANAGER on the root window (`Xft.dpi`, read by Xft
  and Qt). Real desktops always run a settings daemon and GTK / Qt look for one at startup; a real daemon taking the
  selection over is let through. When the host changes the DPI only the `Xft.dpi` / `antialias` / `hinting` / `hintstyle` / `rgba` entries in
  RESOURCE_MANAGER are replaced; everything else the user merged with `xrdb -merge` (comments included) stays.
- **The server does the protocol half of a window manager**: at start it owns `WM_S0` and holds the root window's SubstructureRedirect (above),
  and it maintains `_NET_SUPPORTED`, `_NET_SUPPORTING_WM_CHECK` (the check window's `_NET_WM_NAME` is `X11ServerOptions.WindowManagerName`,
  `LG3D` by default — why is in §10), `_NET_CLIENT_LIST` (in order of first map), `_NET_ACTIVE_WINDOW`, `_NET_WORKAREA` and per-top-level
  `WM_STATE` / `_NET_FRAME_EXTENTS` and so on; requests in root-window ClientMessages become `XWindowManagerRequest`s for the host (§6), which
  decides whether to honour them and writes the result back with `SetTopLevelStates` / `ChangeTopLevelStates`. GTK3's HeaderBar and Qt's
  frameless windows depend on these properties being present. ICCCM / EWMH details:
  - `WM_HINTS`, `WM_NORMAL_HINTS` and `_MOTIF_WM_HINTS` are parsed in full, accepting only fields that are both flagged and actually present
    (the old 15-value format included), with sizes, frame extents and process ids clamped to valid ranges; title-like properties are decoded by
    their type (STRING / UTF8_STRING / COMPOUND_TEXT, ICCCM §2.7.1).
  - Mapping from Withdrawn with an initial_state of IconicState writes `_NET_WM_STATE_HIDDEN` and `WM_STATE = Iconic` (`xterm -iconic`); a top-level
    that is withdrawn loses `_NET_WM_STATE` and `_NET_WM_DESKTOP` (otherwise it would be remapped with a stale Hidden / Focused); InputOnly
    top-levels get no `WM_STATE` and are not in the client list.
  - Focus moving into an override-redirect popup (a menu grabbing the keyboard) does not change the active window, so the main window is not drawn
    as inactive.
  - On close, a window that lists `_NET_WM_PING` in `WM_PROTOCOLS` is pinged along with `WM_DELETE_WINDOW`; with no reply within 5 seconds an
    `XNotRespondingRequest` is sent.
  - When the host moves a window the server sends a real ConfigureNotify, as for any real move (clients with SubstructureNotify on the root and
    Present get it, and the window under the pointer is recomputed), followed by ICCCM §4.1.5's synthetic one.
  - Without client-side shadows (`ClientSideShadows`, off by default) GTK keeps a resize band about 4 pixels wide (times GTK's scale) along the
    inside edge of the window and sends `_NET_WM_MOVERESIZE` (four edges and four corners) when it is pressed; the host starts a native resize on
    the `XMoveResizeRequest` — undecorated windows with a self-drawn title bar can still be resized from their edges, the band is just narrow
    (verified with gtk3-widget-factory).
- **Core window requests follow the protocol's details**: DestroySubwindows destroys bottom-to-top in stacking order; CirculateWindow picks the
  window by occlusion (RaiseLowest raises the lowest child that is obscured, LowerHighest lowers the highest child that obscures another;
  occlusion uses the outer rectangles, ignores SHAPE, and InputOnly windows obscure nothing), sends only a CirculateRequest when a client has
  SubstructureRedirect, and repaints after the move; the automatic remap in ReparentWindow counts as a MapWindow from the client that reparented;
  ConfigureWindow with a stack-mode above 4 gets BadValue, a resize becomes a ResizeRequest to the client that selected ResizeRedirect, children
  move per their win-gravity with a GravityNotify when the parent's inside size changes (Unmap gravity unmaps them), and TopIf / BottomIf /
  Opposite are decided by occlusion; VisibilityNotify is sent — each top-level has its own native window and the server cannot know what covers
  what on the host, so a mapped top-level counts as fully visible, and a host minimize does not report FullyObscured (lest xterm and the like stop
  repainting and get no Expose when restored); ShapeCombine places the source shape using only the client's offset; SetCloseDownMode outside 0–2
  gets BadValue; ConvertSelection with a property other than None checks that the atom exists (BadAtom). X's stacking order of top-levels does
  not follow the native z-order: injected pointer events are first hit-tested in the top-level the host names (only when the pointer is outside
  it — captured during a drag — is the root searched by stacking order), top-levels the host minimized take no part in the search from the root,
  and `FocusTopLevel` raises the top-level above the other normal top-levels.
- **XKB is derived from the core keymap**: there is no separately maintained XKB keymap — the four canonical types (plus two four-level types for the AltGr level),
  modifier actions, SymInterprets, indicators and key names are all computed from the core table; when the core table
  changes (xmodmap, the host's `SetKeymap`), XKB follows and sends MapNotify. XKB's SetMap writes the uploaded
  keysyms (in the §17 column order) and modifier map back into the core table, which is then derived as usual; SetCompatMap,
  SetNames and the other mapping-change requests are not supported. ALPHABETIC covers every letter with case (besides Latin also Cyrillic,
  Greek, Latin-2 / 3 / 4 / 9 and Unicode keysyms, with case decided by code point — the Russian and Greek layouts the host supplies are exactly
  Unicode keysyms); a group with a single keysym that is a cased letter expands to two levels, lower and upper (core protocol section 5;
  xmodmap often writes a single column); FOUR_LEVEL_ALPHABETIC has six map entries (Shift + Lock + Mod5 → level 3). A latched modifier is
  cleared after one use.
- **Grabs, focus and the pointer follow the protocol**:
  - Timestamps: the server keeps a last-pointer-grab and last-keyboard-grab time (an active grab takes the request's time, passive and implicit
    grabs the time of the event that activated them); a Grab* with a time earlier than the last grab or later than now gets InvalidTime, and a
    device frozen by another client's grab gets Frozen; stale Ungrab*, ChangeActivePointerGrab and AllowEvents have no effect. A device can be
    frozen by several grabs at once and continues only when all of them let go.
  - Passive grabs conflict when they share any combination (AnyModifier / AnyButton / AnyKey count as registering every combination): a core
    conflict fails the whole request with BadAccess, while XI2 lists each conflicting combination in the reply (AlreadyGrabbed). An Ungrab that
    covers only part of a grab subtracts that part (at most 1024 subtracted combinations per grab, BadAlloc beyond); core Ungrab requests do not
    touch XI2 passive grabs and vice versa; out-of-range modifier combinations, event masks and details get BadValue. A replayed press
    (ReplayPointer / ReplayKeyboard) reports the state from before the event.
  - Implicit grabs (on a button press) also send Grab / Ungrab-mode crossing events when they activate and end: the Grab ones before the
    ButtonPress and only to the grabbing client, the Ungrab ones after the ButtonRelease; the grab ends only when all buttons are up (buttons 6
    and above are not in the state mask, and releasing button 6 used to end it).
  - Focus: SetInputFocus (and XI's SetDeviceFocus and XISetFocus) honours timestamps — one earlier than the last-focus-change time or later than now
    has no effect, so a SetInputFocus arriving late over SSH no longer pulls the focus back to a window the user has left; the host's
    `FocusTopLevel` advances that time too, and WM_TAKE_FOCUS carries the same time. RevertToParent reverting all the way up gives the focus to the
    root (that is, PointerRoot); XISetFocus accepts PointerRoot and reverts like RevertToParent when the focus window becomes unviewable. When a
    client moves the focus to another top-level, the host gets an `XFocusRequest`.
  - WarpPointer: honours the source window and source rectangle (a zero width / height is converted as the protocol says), clamps the result to the
    root window and, with an active pointer grab that has a confine-to, to the nearest edge of the confine-to window; it is queued while frozen;
    core and XI share one implementation. The host does not know about warps: the server's pointer and the real cursor can diverge, and confine-to
    only constrains warps, not the user's mouse — that needs the host's cooperation and is not done.
  - When the pointer leaves all top-levels, the last position is kept and the pointer counts as on the root (child None); QueryPointer and the
    root coordinates in events report that last position.
- **XInput2's device topology can change, but this is not full multi-pointer X**: it starts with master pointer 2 / master
  keyboard 3, each with one slave (4, 5); XIChangeHierarchy adds and removes master devices and attaches slaves to other
  masters or floats them (a floating slave reports only slave XI2 events and generates no core events). Pointer position,
  focus and grabs remain single — the host has one set of physical input, so full MPX would buy nothing. XI2 events travel
  the same propagation path as core events — on a given window, a core selection receives core events and an XI2 selection
  receives XI2 events; the core do-not-propagate mask stops only core events, not XI2. XI2 selections are stored per (client, window, deviceid)
  and delivered by merging the master / slave masks for the current device hierarchy, recomputed when it changes; a client selecting
  XIAllDevices gets one copy for the master and one for the slave. A change of the root window size sends XI_DeviceChanged; XIQueryPointer resets
  the motion hint like the core QueryPointer; XI_RawMotion carries the pointer device's absolute position (consistent with the Abs X / Abs Y axes
  XIQueryDevice declares), is sent only when the host or XTEST moved it, and not for warps — programs doing relative mouse input with "raw motion
  + warp back to the centre" no longer jitter. Crossing events when a passive button / key grab activates use Grab mode (as XI 2.2 says;
  XIPassiveGrabNotify is only for Enter / FocusIn passive grabs, which are not implemented). XI device properties are validated like core
  properties (BadAtom / BadValue / BadLength / BadMatch), stored in native byte order, charged to the memory account, and changes send
  XI_PropertyEvent.
- **MIT-SHM is for the local machine only**: it is registered only on Linux and is visible only to clients connected over a Unix socket from the
  server's own IPC namespace — a shmid from a remote client forwarded over SSH means nothing on this machine, and once `/tmp/.X11-unix` is mounted
  into a container, a shmid from a client in it refers to a segment on the host side (the `/proc/<pid>/ns/ipc` of the SO_PEERCRED pid is compared
  with the server's own; different or unverifiable means hidden). Segment size and owner are looked up line by line in `/proc/sysvipc/shm`
  (stopping at the match) and the peer uid comes from SO_PEERCRED; a peer that is neither owner nor creator, on a segment not opened to others,
  gets BadAccess (otherwise a local client could read and write someone else's shared memory through the server); when another client refers to a
  segment by its XID (not the one that attached it), the check runs again; after 16 failed Attach requests from one client within a second, its
  further Attach requests in that second get BadAccess without reading the table. QueryVersion reports the server's effective uid / gid.
  Shared pixmaps and 1.2's fd passing are not implemented.
- **Two GLX paths**: without DRI3 / DRI2, Mesa defaults to drisw — the client renders with llvmpipe (GL 4.5) and sends
  pixels with PutImage, so the server only has to register configs, contexts and drawables; programs forwarded over SSH
  work the same way. The extension string advertises GLX_ARB_create_context and GLX_ARB_create_context_profile: CreateContextAttribsARB
  (request 34) only registers direct contexts (version, flags and profile are handled by the client's driver), so programs that need a core
  profile (3.2+) — GLFW / SDL / Qt's CoreProfile, Blender… — get a context on the direct path; SetClientInfoARB / SetClientInfo2ARB (33 / 35)
  are accepted and ignored. With `LIBGL_ALWAYS_INDIRECT` forced, the software GL in `Gl/` executes a subset of the fixed-function pipeline and
  honestly reports version 1.1 with only the compatibility profile: an indirect context asking for a core profile from 3.2 on gets
  GLXBadProfileARB, a version above 1.1 GLXBadFBConfig, an undefined version or 1.x with forward-compatible BadMatch, and unknown attributes /
  flag bits BadValue.
  - Not implemented: 3D textures, the accumulation buffer, feedback mode, mipmap LOD, pixel-transfer scale / bias and PixelMap, DrawPixels
    and CopyPixels in depth / stencil / colour-index formats, point / line / polygon smoothing, Hint. The first use of feedback mode in each
    context logs one line (GL behaviour is unchanged and no GL error is raised). Selection mode, line / polygon stippling and evaluators were
    added on 2026-10-10 (see "Added on 2026-10-10" below). Edge flags (the
    GLU tessellator's interior diagonals are not drawn under PolygonMode(LINE)), GL_CLAMP with LINEAR blending in the border colour,
    GL_EXT_texture_object's vendor-private requests (11–14, handled as the core texture commands), PolygonOffsetEXT's bias in depth-range units,
    and GL_EXT_abgr as advertised in the extension string are all implemented.
  - GL errors follow the specification: Enable / Disable / IsEnabled accept only 1.1's capabilities and those of advertised extensions, anything
    else records INVALID_ENUM; invalid enums in state commands record INVALID_ENUM and leave the state alone; between Begin / End only vertex
    attributes (plus CallList(s) and End) are allowed, anything else records INVALID_OPERATION and is not executed. DrawArrays reads each vertex's
    data in the order the ARRAY_INFO entries appear (as Mesa's indirect GLX sends it; the encoding specification's VERTEX_DATA section lists a fixed
    order, while the ARRAY_INFO list itself is unordered). New names from GenLists / GenTextures continue after the largest name used. A bound
    texture name no longer in the share group is treated as "deleted, so binding reverts to 0".
  - Limits and memory: a GLX surface is at most 4096 × 4096 pixels — for a larger window (an 8K screen, maximized across monitors) the surface
    is clamped and covers only the lower-left part of the window (GL window coordinates start at the bottom left), with one log line the first
    time, instead of BadAlloc; one request may expand at most 4 million display-list commands (lists calling each other expand exponentially;
    every CallList counts), and render commands are also charged to the work budget (§5); 2¹⁹ vertices per primitive, line width and point size
    64, and ReadPixels / GetTexImage replies no larger than half the output backlog limit. GL memory goes into the per-client memory account: an
    indirect context itself counts 64 KB, with its default texture and primitive buffer charged to the context's client; a share group's lists and
    named textures are charged to the client that created the group (each share group also keeps its limits of 64 MB of display lists and
    65536 textures / 256 MB), each display list costs another 64 bytes (empty lists too, replacing the former limit of 65536 list names);
    surfaces are charged per pixel to the first client that needs them (13 bytes per pixel double-buffered, 9 single-buffered); a RenderLarge
    being assembled is charged by its declared length. A context leaving the resource table and no longer current releases its GL objects and
    refunds the account.
  - Presenting: the front buffer of a single-buffered surface after each Render request, and a double-buffered one on swap, compare the drawn
    range row by row with the pixels actually in the window and write — and damage — only the span that differs; no full-window copy, and X
    drawing elsewhere in the window is not overwritten, while parts changed by core drawing are restored on the next swap.
  - Drawables: surfaces are released with their drawable and their client (a reused XID gets a fresh surface), and a GLX pixmap's surface with
    FreePixmap. When the current drawable is gone, Render and non-rendering commands run as usual (drawing nowhere), while WaitGL / WaitX /
    SwapBuffers with a tag / CopyContext / UseXFont get GLXBadCurrentWindow / GLXBadCurrentDrawable. A surface for the GLX 1.2 style (an X window
    used directly as the drawable) takes the config of the window's visual. GetVisualConfigs publishes a single-buffered and a double-buffered
    config for each of the two visuals, so indirect GLX can pick a single-buffered visual too.
- **Present and SYNC queue and time things as the specifications say**:
  - Present: PresentPixmap waits for its wait-fence to trigger (or be destroyed), then picks the frame by target-msc / divisor / remainder — the
    MSC follows the frame clock the host reports (`NotifyHostFrame`, 2026-10-10; derived from the server clock at 60 Hz when the host does not report), a past target with divisor 0 means now, and with PresentOptionUST the three are converted from
    microseconds to frames; due presents are shown in request order, and earlier ones on the same window not yet shown complete with
    CompleteModeSkip, so old content never overwrites new. The pixmap is held until presented (the specification allows FreePixmap right after
    the request). An idle-fence / wait-fence that is not a fence gets SYNC's BadFence. Present's and DAMAGE's QueryVersion report the highest
    version the server supports, but no higher than the client asked for.
  - SYNC: a trigger keeps its value-type and wait-value as given, and the test value computed at initialization is stored separately (QueryAlarm
    still reports Relative); Relative with counter None gets BadMatch, a test value outside INT64 BadValue; CreateAlarm without a test-type defaults
    to PositiveComparison (the specification's default table); a comparison alarm that fires advances in one closed-form step and is disabled only
    when the advance would overflow INT64; AlarmNotify reports the updated state; DestroyCounter, or the disconnect of the counter's creator, sends
    an AlarmNotify with state Inactive to the alarms on it; Await sends CounterNotify per trigger by event-threshold (checked even when the
    condition already holds when the request runs); an empty Await gets BadValue, an empty AwaitFence returns at once, a trigger with counter None
    is always true, and destroying a fence releases the AwaitFence waiting on it; positive transitions on SERVERTIME / IDLETIME that have already
    been crossed no longer schedule timers (they used to wake up every millisecond).
- **Other extensions**: DAMAGE — after DamageSubtract the remaining damage is re-reported per level (rectangle by rectangle for Raw / Delta, the
  bounding box for BoundingBox / NonEmpty), and Damage objects are released by DamageDestroy or when the client's resources are destroyed;
  drawing only looks at the windows in the same top-level that carry a Damage. DOUBLE-BUFFER — the Background swap action paints the window
  background (a background pixmap tiled from its origin, ParentRelative looked up through the ancestors, None left alone), and a window listed
  twice gets BadMatch. Composite — pixmaps from NameWindowPixmap keep their contents after the top-level is resized, remapped or destroyed (from
  then on they keep the old buffer to themselves); while they still share the buffer, drawing into them is also recorded as damage on the
  top-level, so the host sees it. XFIXES — GetCursorImage / GetCursorImageAndName return the real image and hotspot of the cursor under the
  pointer (glyph cursors of the `cursor` font included; the invisible pointer has no image and stays 1×1 transparent); ChangeCursor /
  ChangeCursorByName take effect (windows using the cursor change with it, the host gets CursorChanged and listeners get CursorNotify);
  DestroyPointerBarrier accepts only pointer barriers. Extension cleanup hooks come in two kinds: connection closed (event selections, timers,
  waiting requests) and a client's resources destroyed (in Retain mode this comes later than the disconnect, when KillClient destroys the
  leftover resources).
- **Added on 2026-10-10 (xs_plan F4–F28)**:
  - **Committing text from the local input method (F5, first step)**: `InjectText(text)`. X programs only understand keycodes: each character
    gets a free keycode whose keysym is changed to the character's Unicode keysym (protocol appendix A: Latin-1 is the code point itself, others
    are code point + 0x01000000), then it is pressed and released; a character the keymap already produces with no modifiers held (NumLock
    aside) is typed with that key.
    - **One notification per batch**: a run of text first changes all the keycodes it newly borrows, then sends one core MappingNotify (covering
      those keycodes) and one XKB MapNotify that reports only the key symbols of that range (changed = KeySyms), and then presses and releases
      them in order — clients refetch the keymap once per batch.
    - **Changed keycodes are not changed back**: clients refetch the keymap only after the notification, and get it as it is when their request
      arrives; typing the same character again changes nothing and sends no notification. Only when free keycodes run out is the least recently
      used one reused, and only after it has been idle for 3 seconds (`TextKeyReuseMilliseconds`, far longer than a round trip over SSH plus
      the keymap download), was not used in the same batch, and the keyboard is not frozen by a synchronous grab; otherwise the text waits and is
      retried every 50 ms. When no keycode can be borrowed at all (a client bound every keycode), the rest of the text is dropped and a line is logged.
    - **In order**: while it waits, later `InjectText` calls and the host's `InjectKey` queue up behind it (XTEST and the pointer do not).
    - **Case**: a borrowed key has the same keysym on both levels and the XKB type ALPHABETIC (Lock counts as consumed by the key), so with
      CapsLock on Xlib / xkbcommon do not turn "é" into "É"; old core-protocol-only clients still uppercase it per protocol section 5.
    - **Untrusted clients** (SECURITY "Keyboard Security"): the keysym of a borrowed keycode is the character the user just typed. While keyboard
      events do not reach untrusted clients, they get no keymap notifications for it, and GetKeyboardMapping / XI GetDeviceKeyMapping / XKB GetMap
      show those keycodes as NoSymbol to them; only keycodes used while typing into an untrusted program are revealed (with a notification sent
      to untrusted clients only).
    - Newline (CR LF counts once) types Return and tab types Tab; other control characters are skipped; at most 4096 UTF-16 code units at a time.
      When the focused program is connected over XIM, no keycodes are borrowed and the whole run goes as XIM_COMMIT (next item); this is decided when
      the run reaches the head of the queue, so earlier text still waiting for keycodes is typed first.
    - Before the 2026-10-10 review: one notification per new character; a keycode was reused after 200 ms (over a slow link, for a run longer than
      the free keycodes, the keycode had already been changed to a later character when the client refetched the keymap for an earlier one); host
      keys pressed while it waited landed in the middle of the earlier text; borrowed keys were derived as ONE_LEVEL and uppercased under CapsLock;
      untrusted clients got the notifications and could read the keysyms.
  - **The XIM bridge for the host input method (F5 step two, decision Q1)**: with `X11ServerOptions.InputMethodName` set (the host passes
    `velashell`), the server is an XIM input method server.
    - **Preconnection** (The Input Method Protocol, "Default Preconnection Convention"): it owns the selection `@server=name` (the owner being a
      window of the server's own) and adds that atom to the root window's `XIM_SERVERS` (existing entries kept); the `LOCALES` target answers
      `@locale=` with a long list (C, POSIX, every ISO 639-1 language code, common language_territory names with and without `.UTF-8`) — committed
      text is delivered in the negotiated encoding regardless of the program's locale, so everything is accepted; `TRANSPORT` answers
      `@transport=X/`. Both properties have the target atom itself as their type. The `LOCALES` / `TRANSPORT` atoms are created when XIM is turned
      on: Xlib first checks with an only-if-exists InternAtom whether they exist, and if not assumes there is no input method server (the interop
      case caught this — xterm never connected — and unit tests could not have).
    - **X transport** (The XIM Transport Specification): the program sends `_XIM_XCONNECT` (format 32) to the selection owner window; the server
      creates a server communication window for the connection and answers `_XIM_XCONNECT`: that window, transport version 0.2, dividing size 20.
      From then on, packets of up to 20 bytes go as one `_XIM_PROTOCOL` (format 8, zero-padded); longer ones are written as a property (type STRING,
      format 8) on the receiver's communication window and announced with a format-32 `_XIM_PROTOCOL` carrying the length and the property name,
      which the receiver deletes as it reads; multiple ClientMessages (`_XIM_MOREDATA` … `_XIM_PROTOCOL`) are accepted too. The two documents'
      prose and tables disagree (the prose says the sender's own window, tables 1.7 / D.6 say the IMS window; table 1.8 says format 8 yet uses
      data.l); the tables in Appendix D of the Input Method Protocol win, and on reading the sender's window is checked as well. The server writes
      properties on a program's window rotating through 64 names, moving on when the previous one is still unread. Untrusted clients can use it:
      writing properties on the server communication window and sending ClientMessages to XIM windows are not stopped by SECURITY (each window
      belongs to that one connection).
    - **Protocol**: XIM_CONNECT (its byte order sets the connection's; no authentication), OPEN (answers every IM / IC attribute: the IM only has
      `queryInputStyle`; the IC has `inputStyle`, `clientWindow`, `focusWindow`, `filterEvents`, `preeditAttributes` / `statusAttributes`
      (nested), `spotLocation`, `lineSpace`, `fontSet`, `area`, `areaNeeded`, colors and cursor, `separatorofNestedList`, `resetState`,
      `preeditState`), ENCODING_NEGOTIATION (COMPOUND_TEXT when listed, else UTF-8, else −1), QUERY_EXTENSION (no extensions), GET / SET_IM_VALUES,
      CREATE / DESTROY_IC, SET / GET_IC_VALUES (a nested level runs to the separator), SET / UNSET_IC_FOCUS, RESET_IC, SYNC, TRIGGER_NOTIFY, CLOSE,
      DISCONNECT; anything unknown — or a first packet that is not CONNECT — answers XIM_ERROR BadProtocol. Supported styles: XIMPreeditCallbacks
      (on-the-spot, with StatusNothing or StatusCallbacks), XIMPreeditPosition (over-the-spot), XIMPreeditNothing (root-window style),
      XIMPreeditNone; Area (off-the-spot — it needs geometry negotiation and the host cannot draw into the program's window) is not supported and
      CREATE_IC answers BadStyle. No status area is drawn.
    - **Keys do not go through XIM**: as soon as an input context exists the server sends XIM_SET_EVENT_MASK with both the forward and the
      synchronous mask 0, and `filterEvents` answers KeyPressMask: keys are interpreted by the program with its own keymap, and composition happens
      on the host (the local input method swallows the composing keys locally) — forwarding keys to the server and back over SSH would add a
      round trip per key. Should a program forward events anyway, they are sent straight back, with SYNC_REPLY when synchronous.
    - **The input context taking input**: the one that reported focus in the top-level holding the keyboard focus (with PointerRoot, the top-level
      under the pointer), the last to report when several did; re-chosen when the X focus changes, a program sets / unsets focus, changes the
      insertion point, creates / destroys an input context, or a communication or focus window is destroyed. The `XInputMethodFocus` reported to
      the host: its top-level (the screen window in single-window mode), `ClientDrawsPreedit` (the style includes XIMPreeditCallbacks), `Cursor`
      (XNSpotLocation is the baseline origin of the preedit's first character, turned into a bar 1 wide and one line tall in the top-level's inner
      coordinates: the line height is the program's XNLineSpace or, without it, estimated from the DPI — 16 at 96 dpi; the top edge is 80 % of a
      line above the baseline; null when the program reported no insertion point). Equal values are not reported twice.
    - **Commit**: each visible run of `InjectText` goes to the input context taking input as XIM_COMMIT (XLookupChars, not synchronous) in the
      negotiated encoding — with COMPOUND_TEXT, Latin-1 as is and everything else in UTF-8 segments (`ESC % G … ESC % @`); newline and tab are still
      typed as keys (Return / Tab are already in the keymap), other control characters skipped. The program's preedit is erased first.
    - **Preedit**: `InjectPreedit(text, caret)` goes only to an on-the-spot input context: XIM_PREEDIT_START when nothing is shown yet, then
      XIM_PREEDIT_DRAW (caret, chg_first and chg_length counted in characters — Unicode code points — replacing the whole previous run; one
      XIMUnderline per character); an empty string sends a DRAW deleting the whole run (no string | no feedback), then XIM_PREEDIT_DONE. It is also
      cleared when the input context taking input changes, unsets focus, or on RESET_IC; RESET_IC answers an empty preedit (the host input method
      is still composing on its own).
    - **Coverage**: Xlib XIM clients — xterm, Emacs, Java (AWT / Swing), Tk, Motif, and programs using GTK 2/3's xim input module (Firefox and
      Chromium follow GTK); the server puts `Gtk/IMModule = xim` in XSETTINGS, so GTK 2/3 programs without `GTK_IM_MODULE` use it (in single-window
      mode the server is not the XSETTINGS manager and this entry is absent). Qt 5 / 6 and GTK 4 have no XIM and still get commits through borrowed
      keycodes. Programs must start with `XMODIFIERS=@im=velashell` — Xlib looks for an input method server only when `@im=` is set and otherwise
      does its own compose handling; the host sets it in its silent post-connect injection (see the host repository's `plan.md`).
    - Limits: 256 XIM connections, 1024 input contexts per connection; a packet assembled from multiple ClientMessages may grow up to the XIM
      packet length limit (about 256 KB) — beyond that the connection is dropped as garbage.
  - **Smooth scrolling (F6)**: pointer devices gain two relative axes, Rel Horiz Scroll / Rel Vert Scroll, with matching ScrollClasses (XI 2.1,
    increment 1.0 = one notch); `InjectScroll(window, x, y, dx, dy)` sends Motion and RawMotion with the scroll axes and emulates buttons 4–7 once
    a full notch has accumulated (the XI2 copy carries PointerEmulated); conversely, wheel buttons from devices also give XI2 clients that use the
    scroll axes one notch of scrolling.
  - **HTML and images on the clipboard, the server as clipboard manager (F14 / F15)**: `XClipboardContent` (text, HTML, PNG); `SetClipboard(content)`
    (`SetClipboardText` is its text-only case; images are capped at `MaxClipboardImageBytes`, 32 MB) and the `ClipboardContentChanged` callback (its
    default implementation forwards text to `ClipboardChanged`). As owner the server lists only the formats it has in TARGETS; as requestor it asks for
    TARGETS first, then text, `text/html` and `image/png` in turn, and hands them to the host together. With clipboard sync on it owns
    CLIPBOARD_MANAGER (freedesktop Clipboard Manager Specification): SAVE_TARGETS is not answered until this clipboard has been fetched, succeeds
    when it has been, and gets None when it cannot be saved; after the owner exits the server takes over on the host's behalf as before.
  - **Pointer warps and confine-to handed to the host (F8)**: the `PointerWarped(rootX, rootY)` callback — reported only when the requesting client
    holds the pointer at the time and is not a restricted forwarded program, and only the last one in a batch; `PointerConfinementChanged(area)`:
    a pointer grab with confine-to reports that window's interior (root coordinates) when it starts and null when it ends.
  - **Screensaver cooperation (F9)**: MIT-SCREEN-SAVER Suspend is counted per client (semantics from the libXss XScreenSaverSuspend(3) man page:
    calls pair up, other clients cannot resume it, disconnecting cancels it); while any is suspended the `ScreenSaverSuspensionChanged(true)` callback
    fires, false when all have resumed; ForceScreenSaver(Reset) calls back `ScreenSaverReset` at most every 5 seconds.
  - **`_NET_WM_SYNC_REQUEST` (F10)**: when the host resizes a top-level that declares it and has a SYNC counter, the window gets the sync request
    before the ConfigureNotify; while waiting `XTopLevelWindow.AwaitingRedraw` is true, and `TopLevelRedrawn` is called back when the client
    raises the counter to that serial or after 300 ms.
  - **Compositing manager (F11)**: `X11ServerOptions.CompositingManager` (off by default) owns `_NET_WM_CM_S0` when on, so GTK, Qt and Electron
    use ARGB visuals for rounded corners, shadows and transparent windows. While it is off, depth-32 windows are still handed to the host as
    opaque (the snapshot's `HasAlpha` is false) — without a compositing manager X shows them without regard to alpha; a GL program that asks for
    8 alpha bits (GLFW does by default, and only gets an ARGB visual) clears with alpha 0, and before the 2026-10-10 review the whole window showed
    what was behind it while paying for the system's compositing. When a remote compositor (`xfwm4 --replace` and the like) takes the selection and
    then exits, or someone sets the owner to None, the server takes it back and broadcasts MANAGER on the root window per ICCCM §2.8 (to clients
    that selected StructureNotify). `ClientSideShadows` stays off by default (on Windows the transparent shadow margin still catches the mouse).
  - **X-Resource LocalClientPid (F28)**: clients connecting over the Unix socket record the peer pid (Linux via SO_PEERCRED, macOS via
    LOCAL_PEERPID), and QueryClientIds returns it to requestors that are local clients themselves.
  - **Metrics (F27)**: `XServerMetrics` exposes the meter name `VelaShell.XServer`: `clients.active`, `connections.refused` (reason: authorization,
    too_many_clients, too_many_setups, bad_setup), `clients.disconnected` (killed, output_backlog), `protocol.errors` (code is the error name),
    `work.duration` (a millisecond histogram, not recorded without a subscriber), `work.stalled`. The watchdog is in §5.
  - **GLX multisampling and GLX_EXT_libglvnd (F22)**: two 4× multisampled double-buffered configs (24-bit `0x105`, 32-bit ARGB `0x106`) — direct
    rendering is truly multisampled by the client's Mesa, while indirect contexts draw single-sampled on them and honestly report SAMPLE_BUFFERS 0;
    `QueryServerString(GLX_VENDOR_NAMES_EXT)` reports `mesa`.
  - **Indirect GL selection mode, stippling and evaluators (F23)**: the name stack and SelectBuffer, hit records for primitives taken as far as
    clipping in selection mode (RenderMode returns −1 on overflow); LineStipple / PolygonStipple; Map1 / Map2, MapGrid, EvalCoord / EvalMesh /
    EvalPoint, GetMap, AUTO_NORMAL (GLUT's teapot and GLU's NURBS draw).
  - **Present follows the host's frame clock (F25)**: the host's compositor calls `NotifyHostFrame()` once per frame; the frame interval is the
    shortest of the last 32 (clamped to 20–500 Hz), and PresentPixmap requests whose target frame has arrived complete right at that report; the
    `FrameClockWanted(bool)` callback tells the host when per-frame reports are needed (NotifyMSC or queued PresentPixmap). There is one global frame
    clock.
  - **SIMD fast paths in RENDER (F24)**: solid-through-mask OVER, image OVER and the general path's Over / Add compute 4 pixels at a time with
    `Vector128`, bit-identical to the scalar versions; bilinear sampling and the GL rasterizer are untouched.
  - **Host: Unix socket only by default on Linux / macOS (F4, decision Q4)**: the "Also listen on a TCP port" setting is off by default; on Windows
    it is always open (WSL and Cygwin programs only use TCP). The TCP port listens on 127.0.0.1 only (containers on a bridge network cannot reach
    it; on Linux mount `/tmp/.X11-unix` into them and use `DISPLAY=:N`). The display address the host reports follows the listeners actually open
    (`X11Server.Display`: `:N` on Linux / macOS, `localhost:N.0` with TCP only). When the Unix socket cannot be created and TCP is off (on macOS
    when `/tmp/.X11-unix` belongs to another user or has an untrusted owner), a server fed only through the connector is started anyway: SSH X11
    forwarding works, local programs cannot connect, and the start result carries a warning (`XServerStartResult.Warning`) — before the 2026-10-10
    review the whole server failed to start, taking SSH forwarding down with it. When picking a display automatically, a number whose
    `/tmp/.X{N}-lock` is held by a live process (or by one that cannot be read) counts as taken; after the built-in engine stops, old sessions'
    channels no longer fall back to local TCP (on platforms without VcXsrv, loopback `6000+N` is never our server).
- **One-window mode (`X11ServerOptions.Rootful`, off by default; see the 2026-10-10 entry in §10 for the decision)**: the whole root window is
  handed to the host as one top-level through `X11Server.Screen` (its snapshot is at (0, 0) and as large as the root window), and the host opens
  one native window showing the whole desktop; the other top-levels are no longer handed to the host (no `TopLevelMapped` and the like, and no
  window-manager requests). The server stops acting as the window manager: it does not own `WM_S0`, does not write `_NET_SUPPORTED` /
  `_NET_SUPPORTING_WM_CHECK` / the client lists / `_NET_WORKAREA`, leaves the root window's SubstructureRedirect to clients (BadAccess in
  rootless mode), and EWMH requests sent to the root window are only delivered as usual to the window manager that selected it; nor is it the
  XSETTINGS manager or the tray (`SystemTray` has no effect) — those belong to the remote desktop's own daemons and panel. Focus belongs to the
  remote window manager (PointerRoot when there is none); the host window gaining or losing focus does not change the X focus.
  - **Picture**: top-levels still have their own buffers and the drawing path is unchanged. At the end of each batch, before the lock is
    released, the areas drawn on top-levels in that batch and the changes in the top-levels' mapping / position / size / shape / border and in
    the root background are turned into screen areas (when only the stacking order changed, just the intersection of two overlapping windows
    whose order flipped), and recomposed in the root window's buffer: first the root background (a pixmap tiled from the root origin, black when
    there is none), then the mapped top-levels from bottom to top — border (a border pixmap tiled from the window's inner origin) and interior,
    clipped by the bounding shape; depth-32 windows are composited with premultiplied over while there is a compositing manager (a remote
    compositor owns `_NET_WM_CM_S0`), and otherwise cover what is below like any other window (X ignores alpha). Composed areas go to the host as
    `TopLevelDamaged(screen, rects)`. GetImage on the root window returns the composed screen (background included).
  - **Host actions**: pointer and drag-and-drop injection coordinates are root coordinates (the target is searched from the root down);
    `ResizeTopLevel(screen, width, height)` makes the screen that large (one monitor covering it; clients get the root ConfigureNotify and RANDR
    notifications); move, close, state changes, frame extents and focus do nothing on the screen handle. Cursors are always reported on the
    screen handle.
  - **Not done**: drawing on the root window itself (programs that draw straight onto the root, such as xroach) is not shown — only the
    background is; effects a remote compositor (xfwm4 with compositing on) draws onto the Composite overlay window, such as shadows, are not
    shown — the server composes the picture itself, so the windows are still visible.
- **System tray (`X11ServerOptions.SystemTray`, off by default)**: when on, the server owns `_NET_SYSTEM_TRAY_S0` (owned by the server's own
  selection window, which carries `_NET_SYSTEM_TRAY_ORIENTATION` = horizontal and `_NET_SYSTEM_TRAY_VISUAL` = the default visual) and acts as
  the freedesktop System Tray Protocol 0.3 tray manager. On `SYSTEM_TRAY_REQUEST_DOCK` the server creates an embedder window (a server-owned
  override-redirect top-level, `SystemTrayIconSize` on each side, 24 by default, placed at the bottom-right of the screen — programs pop their
  menus at the icon's root coordinates), reparents the icon window into it per XEmbed 0.5 and fills it, maps it per the XEMBED_MAPPED bit of
  `_XEMBED_INFO` (following that bit afterwards; old programs without the property are treated as wanting to be mapped) and sends
  `XEMBED_EMBEDDED_NOTIFY` (data1 = the embedder, version 0). The embedder, when mapped, is not handed to the host as an ordinary top-level but
  as `SystemTrayIconAdded(handle, name)` (the name is the icon's `_NET_WM_NAME`, else WM_NAME decoded by its type, else WM_CLASS, length-capped
  and stripped of control and bidi formatting characters like window titles; the snapshot's `ClientLabel` is the label of the icon's connection,
  so the host can show the source in the tooltip): the host reads its pixels,
  receives its damage and injects pointer input into it as usual. When the icon window is destroyed, reparented away by the program (the
  spec's way of ending the protocol) or its program disconnects, the embedder goes away (`SystemTrayIconRemoved`). At most 64 icons; balloon
  messages (BEGIN / CANCEL_MESSAGE) are accepted but not shown. It is off by default because with a tray present programs "close to tray", and
  if the host does not show the icons those windows can no longer be found.
- **Drag and drop (XDND, host → X)**: when local text or files are dragged onto an X window, the server acts as the freedesktop XDND version 5
  **source** for the host (the source window is the server's own selection window). The host reports the position on every move; the server
  walks up from the deepest window under the pointer to the first one with `XdndAware` (version ≥ 3, the lower of both sides is used) as the
  target, and if it has a valid `XdndProxy` (the proxy's own `XdndProxy` points to itself) the messages go to the proxy. On a change of target
  it sends `XdndLeave` to the old one and `XdndEnter` to the new one (with the more-than-three-types bit set, the target reads the source
  window's `XdndTypeList`), then `XdndPosition` (root coordinates, server time, `XdndActionCopy`). After a position it waits for `XdndStatus`;
  moves in the meantime keep only the latest, sent once the status arrives; `IsDragAccepted` is what the latest status said. On release it
  reports the last position once more and waits for the last `XdndStatus` (at most 3 s, else `XdndLeave`): accepted means `XdndDrop`; not
  accepted, or dropped where there is no target, means `XdndLeave`. The target fetches the data through `XdndSelection` (owned by the server,
  shared display-wide): `TARGETS`, `TIMESTAMP` and the types the host supplied (the type is the target atom; `STRING` answers STRING, `TEXT`
  answers UTF8_STRING), large data via INCR; before the drop only `TARGETS` can be answered. The data is dropped on `XdndFinished`, when the next
  drag starts, or a minute after the drop. Drags between X programs use the core protocol only and do not involve this; dragging out of an X
  program into a local one is the next item.
- **Dragging out (XDND, X → host)**: `X11ServerOptions.AcceptOutgoingDrags` (the host turns it on in multi-window mode; in single-window mode the
  root window belongs to the remote desktop and it is off).
  - **Where the target is**: per the root-window convention of version 4 on, the root window's `XdndProxy` points at a window of the server's own
    (whose `XdndProxy` points at itself, with `XdndAware` = 5); when the pointer is dragged outside every X window (rootless, that is the local
    desktop or a local program), sources that look at the root window's `XdndProxy` (GTK and the like) send their messages to the proxy, with the
    root window in the window field. Java's AWT only looks for a target on "the child of the root window under the pointer" (with an empty
    MotionNotify child it does not look at all), so when an X program takes `XdndSelection` (starts a drag) the server slips the proxy window in
    as the bottom child of the root window covering all of it, with `WM_STATE` (Java only looks for `XdndAware` on such top-levels): the empty
    area then has a top-level taking drops, with the real X windows still above it. It is slipped in and out silently (no CreateNotify /
    MapNotify) and only exists during the drag — from one second on it checks every half second and is removed once the `XdndSelection` owner
    no longer holds the pointer and no drag-out is in progress; sources build their window cache when the drag starts, do not see it, and still
    find the same proxy through the root window.
  - **Flow**: `XdndEnter` records the source window, version and types (more than three: the source window's `XdndTypeList`); every
    `XdndPosition` gets an `XdndStatus` — accepted when the types include `text/uri-list` or text (UTF8_STRING, `text/plain;charset=utf-8`,
    COMPOUND_TEXT, STRING, `text/plain`, TEXT), the action always `XdndActionCopy` (the local side gets a copy; the remote original is never
    treated as moved and deleted), an empty rectangle (report on every move). On the first acceptance the server asks the `XdndSelection` owner
    for the data with that `XdndPosition`'s timestamp (the protocol allows fetching during the drag): the URI list when offered, plus the first
    text type offered, through the clipboard's selection fetching (10-second step timeouts, INCR, ending when the owner leaves). It then parses
    the URIs (RFC 2483: comment and empty lines removed) and the text (decoded by type) and hands them to the host through
    `OutgoingDragStarted(XOutgoingDrag)`, with the source program's connection label (for the host to find the SSH session and fetch files) and
    root coordinates; when nothing came back, later `XdndStatus` messages refuse.
  - **Result**: when the host's local drag-and-drop ends it calls `CompleteOutgoingDrag(drag, dropped)` and **then** injects the released button —
    the X program's following `XdndDrop` gets an `XdndFinished` from that result (version 5: bit 0 of l1 "accepted and done", l2
    `XdndActionCopy`, or None when not dropped). Once the host says cancelled, later `XdndPosition` messages refuse. Before the host reports back,
    `XdndLeave` (the pointer came back into an X window), `XdndDrop` (the user released before the local drag started — answered "not
    accepted") and the source window being destroyed all report `OutgoingDragEnded`, and the host does not start anything. A result for an old
    drag does not count for a new one (matched by the `XOutgoingDrag` object).
  - Real-client case: Swing's drag-and-drop dragged outside the X windows finds the slipped-in proxy, the text reaches the host, and after the
    host says dropped `exportDone` reports COPY. OpenJDK 17, after a successful `XdndFinished`, sends one ChangeWindowAttributes for window 0 while
    cleaning up (BadWindow 0x0); compared against Swing dragging text into its own text field (which never touches the server's proxy) — same
    error, unrelated to the server, and AWT swallows it.
- **Clipboard**: host → X, the server itself owns CLIPBOARD (and PRIMARY with `SyncPrimary`) and answers per ICCCM: TARGETS includes MULTIPLE
  (pairs converted one by one, a pair that cannot be converted gets its property written back as None) and COMPOUND_TEXT (Latin-1 as is,
  everything else in UTF-8 segments); the TEXT target answers STRING when the text fits in Latin-1 and UTF8_STRING otherwise; text over
  256 KB goes as ICCCM §2.5 INCR (256 KB chunks, a transfer whose requestor does not fetch the next chunk within 10 seconds is abandoned, at most
  32 transfers at once); encoding happens once, on first request. X → host, the server fetches the text with a hidden InputOnly window as
  requestor, each selection independently (properties `_VELASHELL_CLIPBOARD` / `_VELASHELL_PRIMARY`), giving up on a transfer when a step
  (waiting for SelectionNotify, waiting for the next INCR chunk) takes more than 10 seconds, and decodes by the type the owner returned
  (UTF8_STRING with STRING fallback; COMPOUND_TEXT only for its ASCII, Latin-1 and UTF-8 segments, other character sets becoming U+FFFD).
  Both directions are limited to `X11Server.MaxClipboardBytes` (16 MB of UTF-8). When the host writes back the text it just received, the server
  does not take the selection, so the two sides never fight over it; when a synchronized selection loses its X owner (the owner gives it up, the
  owner window is destroyed, the owner disconnects), the server takes it over on the host's behalf with the text last handed to the host, much
  like a clipboard manager — before, once an X program exited, other X programs could no longer paste what it had just copied. Exchanging only
  with the focused session (`ClipboardFollowsFocus`) is described above.

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
- **Interop** (`[TestCategory("Interop")]`, skipped by default): `scripts/xserver/interop/` builds a container with real clients
  (the `velashell-xclients` image: x11-apps, xterm, xdotool, xinput, mesa-utils and more, plus default-jdk — the Swing cases run
  straight from source with `java X.java` to check how Java judges the window manager); the tests start the server locally, let real
  clients in the container connect via `host.docker.internal:N`, save top-level pixels as PNG for humans and make coarse assertions
  (non-background pixels). Rebuild the image after changing its Dockerfile. XIM and dragging out each have real-client cases: xterm under
  `XMODIFIERS=@im=velashell` connects over XIM and Chinese committed by the host reaches the shell's `read` (over-the-spot, insertion point
  reported); a Swing text field connects on-the-spot, gets the preedit to draw, and holds exactly the two committed characters afterwards; Swing
  drags text out of the X windows and, once the host drops it, `exportDone` reports COPY. The XIM protocol details (three transports, byte orders,
  nested attributes, errors) also have byte-level unit tests (`XimTests`, the test client playing the Xlib side); drag-out flow control and
  cleanup are in `OutgoingDragTests`.
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
- **Synthesized fonts**: `cursor` (metrics only; cursor shapes are derived from the glyph numbers) and `nil2` (xterm's
  invisible pointer, all-blank glyphs) do not come from BDF files.
  (Since 2026-10-09 both are X.Org's BDF files as is; see §7 "Fonts" and "Wrapping up the second review's leftovers" at the end of this section.)
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
  (Corrected in 2026-10: the snapshot's X / Y are the outer edge of the X border; a position the client asked for aligns the native
  window's **frame** by ICCCM's gravity, and only (0, 0) that is not user-specified gets centred — the frame, too; see "Placement" in §6.
  A window asking for y = 0 used to get its title bar pushed off screen.)
- **M3: the engine is selectable, built-in by default**: `ILocalXServer` is implemented by a selector that forwards to the
  built-in engine or VcXsrv per the settings, the running one taking precedence (changing the setting never stops an X server
  that is showing windows). VcXsrv exists only on Windows; other platforms only have the built-in engine.
- **M3: SSH x11 channels go straight into the server through a connector**: the SSH library's
  `X11ForwardOptions.LocalConnector` (`velashell-docs/en/ssh/spec/07` §7.5.9) gets an in-memory duplex stream pair per channel and
  hands one end to `X11Server.ServeAsync`. The fake-cookie check is unchanged; the server admits the stream as a local connection
  (since 2026-09-26 it is handed to `ServeAuthenticatedAsync` and not checked again, see §7).
  Used in trusted mode only — untrusted mode needs `xauth` to reach a display and still goes over TCP (since 2026-10, untrusted mode
  with the built-in engine does not set up forwarding at all: the library has no SECURITY extension, so `xauth` cannot get an untrusted
  cookie; see "Authorization" in §7; since 2026-10-10 untrusted mode goes through the connector too — the connector hands "untrusted" to the
  server with the channel (`ServeAuthenticatedAsync(stream, label, XClientTrust.Untrusted)`) and no xauth runs, see "Trust levels" in §7).
  Added in 2026-10: the connector passes the SSH session's `user@host:port` as the connection label
  to `ServeAuthenticatedAsync(stream, label)`; once the built-in engine has stopped (say, after switching to VcXsrv), x11 channels of
  existing sessions go over local TCP to whichever X server is running now instead of being refused. The server keeps listening
  on loopback TCP and the Unix socket, so other local X programs can connect with `DISPLAY=localhost:N` (since 2026-09-26 they
  need the cookie the host writes to `.Xauthority`). (Since 2026-10-10 only the Unix socket is open by default on Linux / macOS — local
  programs use `DISPLAY=:N` and TCP has to be turned on in the settings, see F4 under "Added on 2026-10-10" in §7; falling back to local TCP
  after the built-in engine stops happens on Windows only.)
  **Two corrections on 2026-09-24**: ① the connector picks the server that is running **at the moment** each channel arrives,
  instead of remembering the one that was running when the display was resolved — an SSH session outlives the server, and after
  the user stopped and restarted the X Server from the title bar, every channel of an existing session used to go into a disposed
  server: the remote side only saw `Failed to open display`, and nothing was logged locally. If no server is running at that
  moment, the channel is refused and a log line is written. ② When the remote side sends `CHANNEL_EOF`, the connector's end must
  read EOF too (a new decision in spec 07 §7.5.9): previously, after the remote program exited, its connection and window stayed
  up until the whole SSH session ended.
- **M3: the keyboard layout follows Windows**: the host injects X keycodes by physical key (scan code); on Windows the system's
  `ToUnicodeEx` computes the unshifted and Shift levels of the main key block for the current layout, which replace the
  server's keymap (recomputed the next time an X window is activated after a layout switch; since 2026-10 it catches up on the next key
  press in an X window, see "Keyboard" in §7). The AltGr level is not generated
  yet — the server's XKB description only derives two levels; other platforms use US.
- **Fixed while verifying M3 on real windows**: ① destroying the Damage objects on a pixmap when the pixmap was freed was wrong —
  `FreePixmap` only drops the ID, and xeyes' Present-based frame swap follows it with `DamageDestroy`, which then got BadDamage
  and the client exited; Damage objects are now released by `DamageDestroy` or client disconnect. ② The Unix-socket listener
  deleted an existing socket file before binding — the desktop's own Xorg usually has TCP off, so the TCP-side check cannot see
  it, and the desktop's socket got deleted; it now tries to connect first and leaves the file alone if anyone answers (since 2026-10
  this probe is asynchronous with a 300 ms limit, and a display number whose names are taken is not used at all; see "Listening and
  display numbers" in §7).
  ③ Host side: resizing a shown native window takes `Width` / `Height` (setting `ClientSize` only changes the property value);
  the `Resized` caused by our own resize may arrive a beat later, so only user drags and window-state changes are reported to
  the server — otherwise the old size would overwrite the one the client just set. (Corrected in 2026-10: Avalonia's X11 backend always
  reports `Unspecified` in ConfigureNotify, so a user resizing by the border on Linux was never reported; the host now remembers the last size
  it set from the server's geometry and reports a `Resized` whose size differs from it (within one pixel counts as equal) as a user resize.)
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
  TrueColor visual, colour 8/8/8 (plus 8 bits of alpha on the ARGB visual), depth 24, stencil 8, no accumulation buffer;
  GLX 1.2 visual configs have one entry per visual, using its double-buffered config. Since 2026-10-10 there are also two 4×
  multisampled double-buffered configs (`0x105` / `0x106`, see "Added on 2026-10-10"), six FBConfigs in all.
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
- **Host API clean-up (2026-09-25)**: an API review went over the public surface; where behaviour stays the same only the shape changed.
  ① **One public surface**: every public type moved to the root namespace `VelaShell.XServer` (it used to be spread over `.Host`, `.Server` and
  `.Drawing`), and every public member of `X11Server` now lives in `X11Server.cs`; `PixelLock` (unused, and locking it directly bypassed the yield)
  and `XErrorCode` went back to internal. `XServerOptions` / `IXServerHost` became `X11ServerOptions` / `IX11ServerHost`, sharing the `X11Server`
  prefix and no longer clashing with the host application's own `XServerOptions`; the options are a sealed class validated once at construction
  (invalid values throw `ArgumentException`).
  ② **Consistent naming** (in 2026-10 the rule was rewritten to match the existing public surface, see "Naming" in §6): host methods fall into three groups — `Inject*` (synthesized input), `*TopLevel` (window-manager actions), `Set*`
  (configuration); callbacks are all "subject + past participle" (`Bell` → `BellRequested`, `WindowManagerRequest` → `WindowManagerRequested`).
  Windows are named by their `XTopLevelWindow` handle instead of an XID.
  ③ **Window properties are an immutable snapshot**: `XTopLevelWindow`'s two dozen properties used to be written one by one on the execution
  thread while the host read them on its UI thread, so it could see a new width with an old height, and the 16-byte `ClientFrameExtents` tuple
  could tear. The execution thread now builds an `XTopLevelSnapshot` and swaps it in whole, and `TopLevelChanged` says which groups changed;
  nothing is reported when nothing changed (setting the same title again no longer disturbs the host), and an unchanged shape keeps its list instance.
  ④ **Cursors**: the callback used to carry a magic int (a glyph number; −1 meant both "default" and "bitmap cursor", −2 hidden), the host had to
  keep its own glyph table, and bitmap / ARGB cursors (which libXcursor creates through RENDER CreateCursor whenever a cursor theme is installed)
  all showed as arrows. It is now an `XCursor`: the library derives the shape from the glyph number or from the name given through XFIXES
  SetCursorName, bitmap and ARGB cursors carry their image baked at creation, and the Avalonia host shows the image directly.
  ⑤ **The keymap changes at once**: switching layouts used to take three `SetKeyboardMapping` calls plus one `SetModifierMapping`, each sending
  every client a round of notifications; the layout name was an optional trailing parameter the host never passed, so with `de` selected
  `_XKB_RULES_NAMES` still said `us`. `SetKeymap(XKeymap)` now commits everything with the layout name; the keysyms and modifier bit for right Alt
  as AltGr come from the library, so the host no longer writes a modifier map by hand; when following the system, the host takes the name of the
  most similar bundled layout.
  ⑥ **Extension registry**: major opcodes and event / error numbers sit in one table and registration checks they do not overlap; extensions register
  cleanup hooks for disconnects and destroyed windows, so connection teardown and window destruction no longer keep their own lists.
  GLX became a class of its own, `GlxExtension`; each extension's resource types moved to `Resources/`; `Queries` was split into Xinerama / XRes,
  `CompositeDbe` into Composite / Dbe, DPMS moved out of ScreenSaver, `SyncGrabs` was renamed `GrabFreeze` (distinct from the SYNC extension),
  and the top-level bridge moved from Exposure to TopLevels.
  ⑦ Other: `Display` reflects the transports actually listening (`:N` / `localhost:N.0` / null when neither), with `DisplayNumber` alongside;
  `ServeAsync`'s `isLocal` has no default any more and the docs state the stream is not disposed; diagnostics go only through `Log` (some used to go to
  `Trace`, so failures in host-injected work items were invisible to the host); the bell volume is computed as the protocol specifies.
- **The 31 fixes from the library-wide review (2026-09-26)**: the 31 items recorded by the 2026-09-25 review were fixed one by one,
  each with a test that was first confirmed to fail with the fix reverted; limits and authorization are in §5 and §7.
  The ones that changed protocol behaviour or are worth remembering:
  ① **Extension error numbers moved up by one**: XFIXES gained BadBarrier (`DestroyPointerBarrier` now accepts only pointer barriers;
  it used to delete whatever ID it was given, so one request could delete the root window), and every later extension's errors moved
  up with it: XFIXES 128–129, RANDR 130–133, RENDER 134–138, SYNC 139–141, DAMAGE 142, DOUBLE-BUFFER 143, XKEYBOARD 144,
  XInputExtension 145–149, MIT-SHM 150, GLX 151–164. Clients always get these from `QueryExtension` and are unaffected.
  ② **Focus and crossing events follow the protocol**: focus events carry the details from the protocol's "Input Focus events"
  section (Ancestor / Virtual / Inferior / Nonlinear / NonlinearVirtual / Pointer…) with the virtual events, and KeymapNotify follows
  FocusIn; activating / releasing a grab sends focus and crossing events in Grab / Ungrab mode, and during a grab Enter / Leave go only
  to the grabbing client; a grab is released automatically when its window or confine-to becomes unviewable; with the keyboard
  grabbed, key events still take their source window from the focus.
  ③ **`FocusTopLevel` follows ICCCM's input models** (the WM_HINTS input hint, §4.1.2.4; WM_TAKE_FOCUS, §4.2.8): override-redirect
  windows are ignored; windows whose input hint is False are not given the focus directly; windows advertising `WM_TAKE_FOCUS` get
  the ClientMessage (with a real timestamp, not CurrentTime).
  ④ **CloseDownMode takes effect**: resources of RetainPermanent / RetainTemporary clients survive their disconnect, and KillClient
  destroys them by resource ID or with AllTemporary.
  ⑤ **A latched XKB modifier is cleared after one use**; SetMap validates the keycodes in the modifier map; XI2 button events report
  the buttons as they were before the event (as the protocol specifies).
  ⑥ **GLX**: MakeCurrent prepares the surfaces before changing any state (a BadAlloc used to leave the context stuck on a phantom tag);
  RenderLarge checks the assembled length against the one declared in the first part; RenderMode was checked against
  *GLX Extensions for OpenGL Protocol Specification* 1.3 §2.2.1, which confirmed the existing behaviour — no reply when the previous
  mode was render.
  ⑦ **The host tracks pressed buttons one by one**: when a native window loses capture or is deactivated, it releases in X the
  buttons still held; a release for a top-level that is already gone still takes effect (X used to keep the button pressed).
  ⑧ **Performance**: the benchmark (`scripts/xserver/bench/bench.cs`, with four new scenarios — gradient / transformed / masked
  composites and GLX single-buffered small triangles — and a bytes-allocated-per-request column): linear-gradient Over
  4.2k → 7.2k per second, 2× bilinear upscale 2.2k → 4.0k, ARGB + a8 mask 3.8k → 19.3k; GLX single-buffered one small triangle
  per request 4.2k → about 46k; full-window PutImage 1.1k → 1.6k (40% less CPU).
- **Fixes from the second library-wide review (2026-10)**: the security, correctness, performance and API findings of the review were fixed one by
  one, each with a test first confirmed to fail with the fix reverted; behaviour and limits are already written into §2–§7, so this entry only
  records the trade-offs and why.
  ① **Resources went from "per-item limits only" to "per-item limits + memory accounts"**, plus a work budget per work item and time-limited pixel
  reads for the host (§5, §7): the previous round's limits were all per item — one 16-byte CreatePixmap was 256 MB and a dozen of them exhausted
  the memory of the server and the host process; one request whose cost was out of proportion to its bytes could freeze the host UI for minutes.
  ② **The clipboard follows the session with the keyboard focus** (`ClipboardFollowsFocus`, on by default): exchange used to be automatic in both
  directions, so a password copied locally was readable by every session as soon as the user clicked any X window, and remote programs could keep
  rewriting the local clipboard. The host's "Copy on selection" now defaults to off, matching the library's `SyncPrimary` default (the host used to
  default it to on and map it to PRIMARY sync, the opposite of the library).
  ③ **`RestrictForwardedClients` is off by default**: remote tools such as xdotool rely on XTEST, so the default keeps today's behaviour (a product
  decision), and the host's settings have a switch. A per-connection trust level and "one display per session" are new features for later.
  ④ **`ListenTcp` became `bool?`, and a zero-value configuration does not listen on TCP**: it used to default to true with no cookie configured, so
  `new X11Server()` + `StartAsync()` let any local user connect over loopback TCP (TCP cannot tell which user the peer is), contrary to the SSH
  library's "zero values are the safe values" rule. The host always configures a cookie, so nothing changes there;
  `scripts/xserver/host-demo/demo.cs`, which serves a container through forwarding without a cookie, turns it on explicitly.
  ⑤ **The window-manager name defaults to `LG3D`**: once the server holds `WM_S0` and the root window's SubstructureRedirect, Java (AWT / Swing)
  concludes there is a window manager and decides from the check window's `_NET_WM_NAME` whether it reparents into a frame — any name it does not
  know is assumed to reparent, so Java keeps waiting for ReparentNotify and ignores ConfigureNotify. Measured with a Swing probe on OpenJDK 17:
  before the change Java saw no window manager and could not maximize; holding the selections without renaming (`VelaShell`) made it Other WM,
  assuming a 25-pixel title bar, with no relayout after maximize / resize; with `LG3D` it is recognised as LookingGlass (no frame), insets 0, and
  maximize and resize relayout as usual. Before renaming `X11ServerOptions.WindowManagerName`, verify with a Swing program; the interop image gained
  default-jdk and Swing cases for this.
  ⑥ **Untrusted X11 forwarding with the built-in engine is not set up, and the reason is stated** ("Authorization" in §7) instead of leaving the user
  with an `xauth` error; the "Trusted" hint now explains that all forwarded sessions share one display.
  ⑦ **GLX surfaces larger than 4096² are clamped with one log line** instead of BadAlloc — GL windows on 8K screens or maximized across monitors used
  to get an error for every GL request.
  ⑧ **Checked against the specifications and left as is**: crossing events when an XI2 passive button / key grab activates stay in Grab mode (XI 2.2
  says so); GenericEvent delivery is not gated on a prior GEQueryVersion (whether every client library sends it first cannot be verified under the
  clean-room rules, and gating it could leave programs hanging without their events; only the write-only field was removed); public methods are
  not renamed to fit the naming rule — the rule's wording was fixed instead ("Naming" in §6); no "read several windows under one lock" API (the
  benchmark says it is not needed, §5); edge resizing of GTK windows without client-side shadows works as measured (§7); using another client's
  GLX context is consistent with the trust model (§7).
  ⑨ **Left for later** (all need new features): closing an X popup menu when the user clicks a local window or the desktop (needs a global pointer
  hook; today a click in any X window reaches the menu's grabbing client and closes it, and popups stay on top only while the user is in X, so they
  never cover local programs); WarpPointer moving the host's real cursor, and confine-to constraining the user's mouse; joins between consecutive
  arcs in PolyArc; alpha maps on source pictures; more core font families and sizes; the last pixel column possibly clipped under fractional
  scaling on the host side (needs checking on real machines per platform); a single-buffered visual for indirect GLX; passing a video player's
  ForceScreenSaver / Suspend on to the host to inhibit the system screen saver; a host UI entry for "unstick" (the library side, `BreakGrabs` and
  `DisconnectClient`, is complete). (2026-10-09: the PolyArc joins, alpha maps, more core font families and the single-buffered visual are done;
  see the next entry.)
  ⑩ Host-side behaviour (force quit when an X program is not responding, confirming before stopping the X Server, focus-stealing prevention, no
  native window for desktop and InputOnly windows, click-through outside a shape…) is in
  [`../../host/interaction-and-ui-specs.md`](../../host/interaction-and-ui-specs.md) §4A.3, the settings in
  [`../../host/settings-audit.md`](../../host/settings-audit.md) (eleventh batch), troubleshooting in [`../troubleshooting.md`](../troubleshooting.md).
- **Wrapping up the second review's leftovers (2026-10-09)**: the items in ⑨ above that need no new feature, and the ones the review left partly
  done, are finished; the behaviour is in §6 and §7, and only the trade-offs and their reasons are recorded here.
  ① **Core fonts ship as data with the library instead of waiting for a host font provider** (xs_plan F21 option A): the option originally meant
  rasterizing Liberation and the like at 75 / 100 dpi; X.Org's own Adobe bitmaps are used instead — font names and metrics match a real X server,
  the licence is just as permissive, and no TrueType rasterizer has to be written. CJK and the other scripts come from GNU Unifont (dual-licensed,
  taken under the OFL) and misc-fixed's ja / ko fonts. B&H's Lucida is left out: its licence requires particular notices in user documentation
  and code comments. The data is generated by a script from pinned upstream commits and kept byte for byte; moving to a new version only changes
  the commit ids and hashes in the script. The cost is about 4.5 MB more in the library's DLL.
  ② **Fonts are built on the thread pool while the request is held**: with the full set bundled, a real client's `xlsfonts -l "*"` first parsed
  for 1.5 seconds on the execution thread while holding the pixel lock, and the host UI froze along with it. Parsing fonts is pure computation
  with no I/O, so running it on the thread pool is not fake async; holding requests reuses the mechanism of SYNC Await and XTEST delays, a
  request that has been put back is not checked a second time, and if building fails in the background the request runs as usual and gets the
  usual error.
  ③ **Cursor-font glyphs with a matching system cursor still show the system cursor**: GetCursorImage returns the baked X bitmap, but locally the
  system cursor is used — it follows the desktop's theme and scaling and is clearer than a 16-pixel X bitmap; only glyphs without a match hand
  the image to the host.
  ④ **"Joined" in PolyArc means less than half a pixel apart**: the protocol only says the end points "coincide", but they are real numbers —
  exact equality is too strict (adjacent pieces of the same ellipse differ in the last few bits), and rounding to the same pixel is unstable at
  .5; end points at multiples of 90° all sit on the half-pixel grid, where the criterion is exact equality. A single arc and unjoined arcs draw
  the same pixels as before (200,000 random comparisons).
  ⑤ **RENDER's integer paths cover every operator on a1 / a8 / 8888 targets**: at most 1 away from per-pixel floating-point compositing, a1 bit
  for bit identical (checked exhaustively), the combinations that used floating point about 2–12 times faster. Along the way a NaN in the floating-point
  ClipColor was fixed (a grey source in the four HSL modes on a target with αd = 0 came out with colour 0). r5g6b5, x1r5g5b5 and a4 targets
  are rare and still use floating point.
  ⑥ Selections isolated per session (other sessions see no owner changes and get no SelectionClear for them), alpha maps on source pictures
  and the single-buffered visual for indirect GLX were done in the same round (§7).
  ⑦ **Still left**: closing an X popup menu when the user clicks a local window or the desktop (needs a global pointer hook); new features
  such as WarpPointer / confine-to, passing the screen saver on to the host and a host entry for "unstick"; the last pixel column under
  fractional scaling (needs checking on real machines per platform); a wide arc whose bounding box has zero width or height is drawn at only
  half the line width (it was like that before the review and is unchanged in this round; changed on 2026-10-10, see the next item).
- **The last round on the review list (2026-10-10)**: the second review's list (xs_plan) was checked item by item once more and the items that
  need neither new features nor real machines were finished; the behaviour is in §6 and §7.
  ① **A wide arc whose bounding box has zero width or height is drawn the full line width**: the protocol hands a wide arc's outline to the
  implementation only when width and height are both nonzero and unequal; with one of them zero it is the ideal pair of lines. The half disc
  where the path turns back is the limit of an ellipse as its width goes to zero, and like an ellipse one pixel wide it reaches half the line
  width past that end. Arcs with both width and height nonzero are pixel for pixel unchanged.
  ② **RENDER alpha maps read as "the alpha channel is replaced", the same for source and destination**: as a source, the colour used to be
  divided by the drawable's alpha and multiplied by the alpha map's, so a pixel written through an alpha map and read back through it was not
  the same pixel; the spec only says the alpha channel is replaced. On the destination side the compositor is left alone — the area is assembled
  into a temporary a8r8g8b8 image and written back. Such destinations are rare (cairo, Qt and Java do not use them), and in exchange none of the
  compositor's fast paths has to know about alpha maps; the temporary image covers only the request, and the work is charged for two copies.
  ③ **Point-size font requests pick 75 / 100 dpi by the screen resolution**: the same point size has different pixel heights in the two, and
  name order always gave the 75 dpi one; now the one whose RESOLUTION_Y is closest to `X11ServerOptions.Dpi` is taken. Names that give a
  resolution or a pixel size resolve exactly as before.
  ④ **The font script checks the X.Org fonts' content**: a digest over the picked files (sorted by name, one line "name SHA-256 of the
  content" per file, then hashed as a whole), not over the archive; everything is downloaded and checked before the data directory is touched.
  Rerun with the pinned digests, the generated data is byte for byte the same as the repository's.
  ⑤ **`.Xauthority` checked against the real xauth**: the xauth in the interop image lists the entry we add, each side keeps the other's entries
  when rewriting, and xauth recognises the lock file names we use and does not write while they exist.
  ⑥ **Still left**: closing an X popup menu when the user clicks a local window or the desktop (needs a global pointer hook); new features
  (per-session isolation, a UI for the X client list, WarpPointer, passing the screen saver on, a compositing manager, a tray, a one-window
  mode… — most done on 2026-10-10, see the 2026-10-10 entries at the end of this section); on real machines, the last pixel column under fractional scaling, KeyUp for Command combinations on macOS, and the peer uid through
  getpeereid on macOS / FreeBSD; extending the interop range to GTK3 / GTK4, browsers, Motif / Tk / Emacs, desktop sessions, the tray and fcitx5.
  All of these are recorded in the host repository's `feature-plan.md`, "H. Built-in X server".
- **The X program list and force-quit UI (2026-10-10, xs_plan F3)**: the library only gained two things — `XClientInfo.HoldsServerGrab`, and the
  host callback `ServerGrabStalled` when GrabServer is held too long (before, only a log line: neither the host nor the user knew whom to
  disconnect). Like the other callbacks it goes through the deferred queue and is delivered after the pixel lock is released; the host's
  `GetClientsAsync` has to go back to the execution thread, so VelaShell looks up which program it is on a separate task instead of waiting
  inside the callback. The UI is in the host's "Interaction and UI specs" §4A.2: while the built-in engine runs, the title-bar X Server
  button opens a flyout; an external VcXsrv cannot list its programs, so there the button still starts and stops it.
- **One-window (rootful) mode (2026-10-10, xs_plan F13 / decision Q2)**: §2 used to state only the premise "rootless multi-window, the root
  window is never drawn" without making the whole desktop a non-goal. A full remote desktop (xfce / MATE) or a graphical installer did not work
  in rootless mode (a remote window manager cannot get SubstructureRedirect, desktop windows cover the local screen), while VcXsrv's "One large
  window" and MobaXterm's Windowed mode are shapes users know, so it was added as an **optional mode**, rootless staying the default. The drawing
  path is unchanged: top-levels keep their buffers and the server composes the whole screen in the root window's buffer like an always-on
  compositor (see "One-window mode" in §7); the window-manager role is handed over to the remote side entirely. Real-client case: twm takes
  over, xterm gets framed and composed into the screen, zero protocol errors.
- **Trust levels and one display per SSH session (2026-10-10, xs_plan F2 / F1, decision Q5)**: all forwarded sessions used to share one trusted
  display, and untrusted `ssh -X` could not be set up with the built-in engine. Decision Q5 tightens this in two steps: first the SECURITY
  extension's untrusted level (a remote `xauth generate … untrusted` gets a restricted cookie, and the host's connector can mark a channel
  untrusted directly), then the host's "one display per SSH session" — trusted sessions can only be kept apart on different displays.
  XC-QUERY-SECURITY-1 and Application Groups are not done; one display per session is off by default and takes effect at the next start.
  Real-client case: `xauth generate` issues an untrusted cookie, after which xdpyinfo sees no XTEST / SECURITY, `xwd -root` cannot grab the
  screen, and xterm (core fonts and Xft), `xclock -render`, xlogo and indirect glxgears draw normally with zero protocol errors.
- **What became of the draft's remaining new features (2026-10-10, xs_plan F4–F30)**: F4, the first step of F5 (committing text from the local
  input method), F6, F8–F11, F14, F15, F17, F18, the screenshot part of F20, F22, F25, F27 and F28 are done; F21, F26 and the create_context part of
  F22 already existed. Partly done: F19 (each monitor already reports its DPI from its own scaling; integer upscaling needs one zoom factor for the
  whole server and changes every root↔screen mapping in the host, too risky to verify only headlessly), F23 (feedback mode, mipmap LOD, PixelMap and
  depth / stencil DrawPixels / CopyPixels are not done), F24 (bilinear sampling and the GL rasterizer untouched), F25 (one global frame clock, not per
  monitor). Not done: F5's second step, the XIM bridge (3–5 weeks); F7 pen pressure / touch / gestures (needs the master device's classes switched on
  SlaveSwitch, with no real toolkit to check against; XI 2.4 is not on the spec list); F20's screen recording (needs an encoder); F29 WSL (no
  environment to test); F30 MIT-SHM 1.2 (Linux-only fd passing that the TCP-based container setup cannot exercise). Defaults: local input method on,
  source labels on, compositing manager off, the TCP port on Linux / macOS off (containers connecting over the network need to turn it on).
- **Defaults of the new settings and F19 (2026-10-10, maintainer)**: the defaults above stay, and one display per SSH session stays off. F19's
  integer upscaling waits: no single zoom factor for the whole server; it waits for "one display per SSH session" to give each display its own.
- **Fixes after reviewing the new settings (2026-10-10)**: going through the items above one by one, plus real-client cases, changed these
  (details under each item in §7): ① committing text from the local input method: a keycode is reused after 3 seconds idle instead of 200 ms —
  correctness depends on the client having finished refetching the keymap for the earlier character, and only enough time can guarantee that
  ("the client fetched the keymap once" does not hold up: it may have fetched for another reason before reaching this round's notification);
  one notification per batch; host keys queue behind waiting text; borrowed keys use the ALPHABETIC type; untrusted clients cannot see text typed
  into trusted programs. Real-client case: xev receives U4E2D and U6587, and still eacute with CapsLock on. ② With the compositing manager off,
  ARGB windows are shown opaque; when a remote compositor takes the selection and drops it, the server takes it back. ③ Tray icon names are
  filtered like window titles, and the host's tooltip shows the source. ④ When the Unix socket cannot be created and TCP is off, only SSH
  forwarding is served instead of failing to start. ⑤ Host-side fixes (per-session displays get all host settings, closing one session's screen
  window closes only that display, "show where X windows come from" applies immediately, the display address follows the actual listeners) are
  in the host repository's `plan.md`.
- **The XIM bridge for the host input method and dragging out (2026-10-10, xs_plan F5 step two / decision Q1, the other half of F16)**:
  ① XIM — the server itself is the XIM input method server rather than bridging to a remote fcitx / ibus: the remote host usually has neither,
  while the host has the local input method at hand. Only the X transport (built into Xlib, no extra port); keys are not forwarded over XIM
  (forward mask 0), composition stays local and only committed text and the preedit travel, so typing over a slow link gains no round trip.
  Only on-the-spot, over-the-spot and root-window styles: off-the-spot would need the host to draw a status and preedit area inside the
  program's window, which it cannot. Programs must start under `XMODIFIERS=@im=velashell` — sshd does not accept that variable's env request by
  default, so the host sets it in the silent post-connect injection (an existing value is kept). Interop caught one problem unit tests could not:
  Xlib checks with an only-if-exists InternAtom whether `LOCALES` exists, so the server has to create it first. Qt 5 / 6 and GTK 4 have no XIM
  and keep getting commits through borrowed keycodes.
  ② Dragging out — the root-window `XdndProxy` convention of XDND version 4 on, so there is no guessing whether a local window is under the
  pointer: whatever lands outside the X windows goes to the host. Java ignores the root window's `XdndProxy`, so during a drag the proxy window is
  slipped in at the bottom with `WM_STATE`, without structure events and only while the source holds the pointer. The action is always copy.
  The data is fetched on the X side (during the drag, as the protocol allows), so the host gets URIs and text; fetching remote files and starting
  the local drag-and-drop are up to the host (see the host repository's `plan.md`).
