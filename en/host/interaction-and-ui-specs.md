# VelaShell Interaction Logic and UI Specification

> This document is based on the `VelaShell-zh.pen` design file and is intended for delivery to an AI Agent for feature implementation.
> Target technology stack: **Avalonia** (UI) + **VelaDock** (draggable/split tab docking, replacing Dock.Avalonia; floating windows are disabled by product decision) + VT engine (parser / buffer / emulator / render).
> The design file is the sole visual baseline for this document. Any item phrased as "should be changed to..." is a **refactoring requirement** for the existing design structure. Implementations must follow this document.

---

## 0. Product Positioning and Design Direction

VelaShell is a modern SSH/SFTP terminal client for operations and development. Its core experience goals are:

- **Keyboard first**: almost every function can be completed through the command palette (Ctrl+P) or keyboard shortcuts.
- **High information density without crowding**: monospaced fonts carry data, UI fonts carry actions, and the color palette remains restrained.
- **Floating auxiliary tools**: file transfer, tunneling, and resource monitoring appear as on-demand floating panels that do not consume main terminal space.
- **Dark by default, light as an option**: the design provides dark and light themes, and all colors use tokens.

---

## 1. Global Design Standards (Design Tokens)

Use the theme resource dictionary consistently during implementation. Hard-coded colors are prohibited. The following variables are defined by the design:

### 1.1 Colors

| Token | Dark | Light | Purpose |
|---|---|---|---|
| `bg-page` | `#0A0E14` | `#F5F5F7` | Lowest-level window background |
| `bg-sidebar` | `#0D1117` | `#FFFFFF` | Sidebar / status bar |
| `bg-surface` | `#111620` | `#FFFFFF` | Floating panels / dialogs / cards |
| `bg-terminal` | `#080C12` | `#1E1E2E` | Terminal canvas |
| `bg-input` | `#151B26` | `#F0F0F2` | Input fields / footer bars |
| `bg-hover` | `#1A2233` | `#E8E8EC` | Hover state |
| `bg-active` | `#1C2A3F` | `#E0E7F0` | Selected/active state |
| `border-primary` | `#1E2A3A` | `#E0E0E4` | Standard divider |
| `border-secondary` | `#253345` | `#D0D0D6` | Floating panel outline |
| `accent` | `#00D4AA` | `#00B894` | Primary accent color (brand green) |
| `accent-dim` | `#00D4AA30` | `#00B89420` | Light accent background (label/badge background) |
| `accent-text` | `#0A0E14` | `#FFFFFF` | Text on accent buttons |
| `text-primary` | `#E0E6ED` | `#1A1A2E` | Primary text |
| `text-secondary` | `#8B9BB4` | `#6B6B80` | Secondary text |
| `text-tertiary` | `#5A6A80` | `#9999AA` | Tertiary text/default icon color |
| `text-muted` | `#3D4F63` | `#BBBBCC` | Muted hint/placeholder |
| `status-connected` | `#00D4AA` | Constant | Connected (green) |
| `status-connecting` | `#FDCB6E` | Constant | Connecting (yellow) |
| `status-disconnected` | `#FF6B6B` | Constant | Disconnected (red) |
| `info` | `#74B9FF` | `#3498DB` | Information blue |
| `warning` | `#FDCB6E` | `#E67E22` | Warning |
| `error` | `#FF6B6B` | `#E74C3C` | Error |

Terminal ANSI palette: `term-red/green/yellow/blue/magenta/cyan/white` = `#FF6B6B / #69FF94 / #FDCB6E / #74B9FF / #D980FA / #00D4AA / #E0E6ED`.

### 1.2 Typography

- `font-mono` = **JetBrains Mono**: terminal content, hostnames/IP addresses, ports, paths, shortcuts, numeric values, and tab names.
- `font-ui` = **Inter**: menus, buttons, settings, explanatory text, and group headings.
- Common font sizes: terminal 12–13, body 11–14, secondary descriptions 9–10, group headings 10 (`letterSpacing:1`, usually uppercase/bold).

### 1.3 Shapes and Spacing

- Corner radius: floating panels 6–8, buttons/input fields 3, badges 2, label dots circular.
- Floating panel shadows: **every overlay and self-drawn window card shares one token**, `VelaShadowWindow` (two layers: a near one that firms the edge, a far one that spreads). Dark `0 1 3 #40000000, 0 4 12 #66000000`; light `0 1 3 #1A000000, 0 4 12 #33000000`. **Reach (offsetY + blur) is capped at 16**, matching the card margin (the shadow width of the Linux Wayland decorations layer uses the same value) — a self-drawn transparent window can only paint its shadow inside the window rectangle, and anything past the margin is cut into a smear by the window edge. Opacity must differ per theme; the same dense black under a light card looks dirty. Values and reasoning live in `DESIGN.md` §4.5 in the code repository.
- Use **lucide** as the unified icon library, at sizes 11–16.
- Common row heights: toolbar rows 28–36, list rows 28–38, status bar 24.

---

## 2. Main Window Structure Overview

The main window is 1440×900 and can be freely resized. **〔2026-07 final, user decision〕The window uses a custom borderless frame**: `WindowDecorations="None"` (Avalonia 12.x `BorderOnly`/`ExtendClientArea` managed decorations intercept title bar input on Win32, preventing button clicks and window dragging; this was abandoned in testing). It uses custom drawing + the native `BeginMoveDrag` movement loop + a WndProc hook for Windows 11 snap support. **〔Since 2026-09〕macOS is the exception**: the main window uses the system frame and the native traffic lights (close / minimize / full screen, on the left of the title bar); the self-drawn minimize/maximize/close buttons are hidden and the logo makes room for the traffic lights (not in full screen), and the title bar takes the system title bar height (28pt today — the traffic lights cannot be moved, so the title bar aligns to them instead). Windows and Linux are unchanged. How each platform's frame works, and why, is in [`architecture.md`](architecture.md) §5 "Window shell". The first row is a 28px custom title bar (`TitleBarView`: left = product logo + name; right = global feature button group `GQQwj` + minimize/maximize/close buttons). **The text menu has been removed entirely** (its functions duplicate the command palette). From top to bottom, the client area is: **custom title bar → (sidebar ‖ right area) → status bar**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│Custom title bar 28px: ▣ VelaShell …gap… [Feature buttons GQQwj]  ─ □ ✕  │
├──────────────┬──────────────────────────────────────────────────────────┤
│              │Tab row: [Tab1][Tab2]…[+]   ……gap……   [◀][▶][▾]           │
│   Sidebar    ├──────────────────────────────────────────────────────────┤
│  Sidebar     │Terminal main area (VT render + optional line/time bar)   │
│  260px       ├──────────────────────────────────────────────────────────┤
│              │File browser / SFTP (collapsible, default 220px)          │
│              │Broadcast input bar (on demand, above status bar)         │
├──────────────┴──────────────────────────────────────────────────────────┤
│Status bar (full width, 24px)                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

- **The custom title bar spans the full-width first row** (`bg-sidebar` background + bottom divider): left = logo and product name; right = global feature button group `GQQwj` (see §4A; “broadcast” is implemented, “group sync” remains disabled) + minimize/maximize/close window controls (27×27 squares, with system red on close hover; on macOS these three are absent and the system traffic lights on the left take their place).
- **Dialog and standalone-window frames differ per platform** (every dialog, plus the standalone windows — task manager, resource monitor, route trace, directory sync, plugin manager, plugin panels, recording player, remote editor — including isolated plugins' windows): on Windows it is a card floating in a transparent window (corner radius 8, 1px border, a `VelaShadowWindow` shadow in the 16px margin); on macOS the system draws the corners, border and shadow around an opaque window — dialogs show no traffic lights, standalone windows use them in place of the self-drawn window buttons; on native Linux Wayland Avalonia's decorations layer draws the shadow and border, and the compositor treats the card edge as the window edge; on Linux X11 it is an opaque square rectangle. The visible card is the same size on every platform; a maximised or full-screen standalone window fills the screen with square corners.
- **Every window's title bar is 28px** (main window, standalone windows and dialogs, since 2026-09-26; previously the main window was 36, dialogs 48 and standalone windows 34–56): the title bar holds the icon, the title, the window-level actions and the window buttons (the × at the top right for dialogs). Window buttons are always 27×27 squares flush with the top-right corner, close hovers system red; the card clips to its rounded corner, so a square's hover fill follows the corner instead of poking out of the window. **Window-level actions live in the title bar** (since 2026-09-29): an icon-only group just left of the window buttons, styled like the main window's global feature button group `GQQwj` — each has a tooltip saying what it does and an automation name, and its icon turns accent while a toggle is on or while something is waiting to be done. Resource monitor: the host badge (green dot when connected / grey when not + host name) and pause / resume (while paused it becomes an accent "resume"); recording player: export / refresh / clean up / auto-record (accent while on); remote editor: save (accent while there are unsaved changes — it used to be an always-accent solid button); connection diagnostics: export report (disabled until there is a report) / re-run. The two-line headers of these four windows now keep only the subtitle in the second row (sampling note / feature description / remote path / diagnosis target), with the same background and also draggable; double-clicking a button in the title bar never maximizes the window. The New Connection, Import Sessions and connection diagnostics dialogs use the same bar as the standalone windows: a 15px line icon (accent) + a 13px title at the left + the × at the top right; dialogs are fixed-size, get no maximize, and keep their × on macOS (dialogs show no traffic lights). On macOS, windows that show the traffic lights take the system title bar height (28pt today), so the lights and the title share one centre line. Two exceptions have no window buttons in their header, so it is not a title bar and stays 48: the settings window, whose top-left strip is the navigation header doubling as the drag area (no close / minimize buttons); and the message dialog (notice / confirm / prompt), which has no × at the top right and closes only from its button bar or Esc.
- The row below divides directly into “sidebar ‖ right area”; the right area starts with the tab bar (see §4B).

- The sidebar/right boundary and the horizontal dividers within the right area are draggable (VelaDock / GridSplitter), allowing width/height adjustment and full-section collapsing.
- The terminal area and file area together form “one session view” and switch as a whole when the active tab changes.

---

## 3. Sidebar (Width 260, `bg-sidebar`)

> **Window title bar note**: this application uses a **custom borderless window** (`WindowDecorations="None"`, not the native title bar; on macOS it switches to the system frame but only borrows the native traffic lights — the title bar itself is still self-drawn). `TitleBarView` draws a 28px title bar whose left side already contains the application icon and “VelaShell” name.
> Therefore, **the application icon and name are no longer repeated at the top of the sidebar**. The “Logo bar” in the original design (`tGM2H` for dark, `4RBrb` for light) has been removed to avoid duplicating the icon and name in the custom title bar. The first sidebar row now begins directly with the “toolbar”, and the session resource tree fills the released space above.

From top to bottom:

1. **Toolbar (36px, first sidebar row, `cnUAB`)**: group heading “Resource Explorer” (`126Cj`) on the left; two buttons on the right:
   - `plus` (`2mkdr`): New connection (opens the New Connection dialog in §13).
   - `ellipsis` (`oPsSE`): More (2026-10-09, joesdu/VelaShell#571). Top to bottom: Import connections from file… /
     Export all connections… / Save CSV import template… / Import sessions from another tool… ([session import](session-import.md))
     — divider — Sort groups by name / Collapse all groups / Expand all groups. The first three, together with
     "Import into this group / Export group / Export selected", are described in [session import](session-import.md) section 7;
     "Export all connections" is disabled when there are no connections. "Sort groups by name" is a one-off reorder
     (natural order: "Rack 2" before "Rack 10"; the current locale's collation, so Chinese sorts by pinyin), and the order can
     still be adjusted by hand afterwards.
2. **Session resource tree (`fill_container`, scrollable)**: collapsible group list occupying most of the sidebar.
   - Group row: collapsible triangle + group name, such as “Production” or “Testing”, + count.
   - Host row: status dot (green/yellow/red) + hostname + optional label, such as “jump host” on an `accent-dim` background.
     - **The dot is the merge of every live session under that profile**, not whatever changed state last. One profile
       can have several terminal tabs and several documents open at once (standalone SFTP / FTP / plugin file systems
       such as S3 / workbenches such as Redis) while the tree has a single node for it: the merge priority is
       `Connected > Connecting > Error > Disconnected`, and it returns to “disconnected” only once all of them are gone.
     - Counter-examples (both were shipped bugs): with last-change-wins, opening a second tab on an already-connected
       session and closing it mid-handshake left the node stuck on “connecting” forever (#321); and opening two tabs on
       one FTP profile and closing either of them turned the node “disconnected” while the other was still alive.
   - The current session row is highlighted with `bg-active` and a vertical accent bar on the left.
   - **Hovering shows the notes** (#549): resting the pointer on a connection that has notes (§13.1 “Organize”) pops
     them up; anywhere on the row works. Connections without notes and group rows show nothing. The tooltip wraps at
     its maximum width, and notes longer than 12 lines or 500 characters are cut and end with `…` — a tooltip limits
     its width but not its height, so a long pasted note would otherwise cover the very rows you are about to click;
     the full text is in the edit dialog.
   - Interaction: click selects; **double-click connects and activates the tab**; right-click opens the session context menu in §12; groups can be reassigned by dragging;
     **dragging a group row reorders the groups** (see below).
   - **Multi-select** (2026-10-09, joesdu/VelaShell#571; grown out of the #524 Ctrl pair selection; ⌘ works too on macOS):
     - **Ctrl+click** adds or removes one row, with no upper limit (under #524 only two could be picked and a third pushed the
       oldest out; now rows keep being added); with one left it falls back to an ordinary single selection. **Shift+click** selects
       every row between the "anchor" and this one — the anchor is the row last plain-clicked or Ctrl-added; **Ctrl+Shift+click**
       adds that range to the existing selection. The range counts the rows you can see: connections inside collapsed groups stay
       out, and group rows are skipped.
     - The selected rows use the same highlight as the selected row (`bg-active` + accent name). Right-clicking any of them opens
       the "multi-select menu" (§12), which acts on all of them; right-clicking a row outside the selection ends it and opens that
       row's own menu.
     - The same profile can never be picked twice (a profile has exactly one node in the tree); to copy within one machine use
       "Copy to" in the single-pane file browser.
     - Group rows do not take part. A plain click or moving the arrow keys outside the selection ends it; collapsing a group drops
       the rows it hides from the selection, and if the selected row itself is hidden (or fewer than two remain) the selection ends.
     - With **exactly two**, the multi-select menu gains "Open in dual-pane SFTP" (§6.2), **the first one picked goes left**.
       SSH / SFTP / FTP and plugin **file protocols** (S3, WebDAV…) can go into the dual pane; plugin workspaces (Redis…) cannot —
       the item is disabled, with a line below explaining why. A terminal plugin protocol (Telnet…) is only recognised once its
       plugin is activated, so opening reports that it is not a file protocol. With three or more the item is not shown.
     - The list control itself stays single-select: the tree tracks the selection on its own, **the row's existing context menu is
       unchanged**, and the multi-select menu is a separate one.
   - **Group order** (#571): group rows can be dragged directly. A drag starts after moving 5px (otherwise a click still expands /
     collapses). Over the upper half of a group row = insert before it, the lower half = insert after it; over a connection row
     inside a group = after that group; over a root-level connection, a pinned connection or empty space = move to the end.
     A 2px accent insertion line marks the spot (not drawn when that row has scrolled out of view), and the ghost label following
     the pointer says "group → move before "X" / move to the end"; dropping back in place shows the "no drop" cursor and no line.
     "Move up / Move down" in the group context menu do the same for those who would rather not drag. The order is saved at once:
     every group's sort order is renumbered 0, 1, 2… (orders left by deleted groups or by numbering-by-count could have gaps and
     even duplicates), and a group created from the New Connection dialog goes after the highest order.
   - By default, follows the active terminal tab: automatically expands the parent group, selects the corresponding connection, and scrolls to it without taking terminal keyboard focus. This can be disabled under “Settings → General → Behavior”.
   - **Pinned connections** (`pin` icon, “Pin to Top / Unpin” in the context menu): a pinned connection is hoisted to
     **the very top of the tree**, sorted by name, with an accent-colored `pin` badge and the same indent as a group row.
     - Hoisting to the top rather than floating within its group is what makes it work together with “collapse groups on
       startup” below: **once every group is collapsed, an item that only floats inside its group is still invisible**,
       which is no pin at all.
     - Pinning changes **where the row is drawn**, not the data: the connection still belongs to its group (`GroupId` is
       untouched) and the group row still counts its members (so expanding a group that has a pinned member shows fewer
       rows than the count — that is correct); unpinning returns it to its group. Drop-target resolution and “delete the
       group once it is empty” are unaffected.
   - **Collapse groups on startup** (“Settings → General”, off by default): groups load collapsed.
     - It only decides the initial state **the first time a group is seen in this run**. Whatever the user expands or
       collapses by hand is remembered in-process, so the full tree rebuilds triggered by creating / editing / deleting a
       connection or by a cloud-sync write-back do **not** fold it back — otherwise every new connection would mean
       re-expanding the group you just opened. The memory is not persisted: a restart returns to what the setting says.
3. **Session list / Quick Connect area (320px, top divider)**:
   - Header (36px): “Quick Connect” title + collapse button.
   - Quick Connect input (`bg-input`, 32px): enter `user@host[:port]` directly and press Enter to connect.
   - “Recent Connections” group heading + the 3 most recent rows (`user@host` + time). Click fills the input; double-click connects directly.
   - The quick command and recent connection areas can be collapsed and resized vertically. The collapsed state and last expanded height are stored locally and restored at the next startup.
4. **Bottom user bar (40px, top divider)**: left = current identity (avatar + the active session's `user@host`, or the local user name for a local terminal; hidden entirely when no tab is active); right, in order: `bell` message center, `plug` plugin manager, `settings` gear (opens §14 Settings).
   - **Current identity**: the name only gets the width left of the buttons (with an 8px gap) and is trimmed with a trailing ellipsis when it does not fit; the tooltip shows the full name. Even at the narrowest sidebar width (180px) it never runs into the buttons on the right.
   - **Message center (`bell`)**: a 6px accent dot appears at the bell's top-right when unread messages exist (24px leaves no room for a number, and a dot says "there is something new" well enough). Clicking toggles a 360px non-modal overlay **anchored bottom-left, next to the bell**, which can be dragged anywhere in the window by its header (the position is remembered across sessions). It holds messages worth keeping and revisiting (available updates, announcements and security items from a subscribed feed) and deliberately **excludes runtime alerts** — fingerprint changes and session drops each have their own immediate feedback, and mixing them in would bury what needs reading. See [message-center-and-feed.md](message-center-and-feed.md).

---

## 4A. Menu Bar (Full-width 36px, `bg-sidebar` background + bottom divider) ★Refactored

> ⚠️ **Implementation change (2026-07, plan §6)**: **The left text menu (Session/Edit/Action/Search/Tools/Help, `DaZfB` in §4A.1 below) has been removed entirely** because it duplicated the command palette (`Ctrl+P`) by product decision. The “Show menu bar” setting has also been removed (`ShowMenuBar` is retained for compatibility). This row now retains only the **right global feature button group** (§4A.2). §4A.1 remains as historical design reference.

> A separate row below the native title bar and above the sidebar/right area. ~~**Left = text menu**~~ (removed), **right = global feature button group (formerly `GQQwj`, moved here from the terminal toolbar)**. Design nodes: menu bar `TSiDh`, right-side feature container `Menu Bar Actions` (containing `GQQwj`).

### 4A.1 Left Text Menu (`DaZfB`)

Order: **Session / Edit / Actions / Search / Tools / Help** (Inter 12, `text-secondary`, `padding[0,10]` per item, hover `bg-hover` + corner radius 3). Click opens a drop-down menu. `Alt` provides keyboard access, and `Alt+first letter` opens an item quickly. Suggested menu contents:

| Menu | Typical items |
|---|---|
| **Session** | New SSH Connection, New SFTP, Import/Export Configuration, Recent Sessions, Close Current Session, Exit |
| **Edit** | Copy, Paste, Select All, Find, Clear Screen, Preferences (Settings) |
| **Actions** | Connect/Disconnect, Reconnect, Split Pane, Duplicate Session, Group Sync Toggle, Broadcast Input, Record Session |
| **Search** | Find in Terminal, Find in Files, Go to Line, Command Palette (Ctrl+P) |
| **Tools** | Tunnel Manager, SFTP File Manager, Operations Orchestration Center, Connection Diagnostics, Host Trust Center, Snippets, Key Management, Shared Credentials |
| **Help** | Keyboard Shortcuts, Documentation, Check for Updates, About VelaShell |

> Menu items, the command palette (§8), and keyboard shortcuts (§16) should share one “command registry” to keep entry points consistent and names uniform.

### 4A.2 Right Global Feature Button Group (formerly `GQQwj`)

Moved from the terminal toolbar to the far right of the menu bar as **quick access to common functions** (24×22 icon buttons, `gap:4`, hover `bg-hover`, active `bg-active`; applies to the “current active session/global”):

| Icon | Name | Behavior |
|---|---|---|
| `search` | Terminal Search | Search the current terminal buffer (open the search bar, see §5.3) |
| `copy` | Copy | Copy the current selection (disabled when there is no selection) |
| `columns-2` | Split Pane | Split the current session horizontally/vertically (VelaDock split) |
| `route` | **Tunnel** | Open the tunnel management panel in §10 (★user specified) |
| `zap` | **Quick Commands** | Open the command palette (§8), or the quick command menu |
| `app-window` | **X Server** | Start / stop the local X server: by default the **built-in X server** (ships with the app, works on every platform, one native window per X window); on Windows the “X Server” page in §14 can switch it to launching the VcXsrv the user installed. The icon turns accent-colored while running and the tooltip shows the display (following the listeners actually open: e.g. `localhost:0.0` on Windows, and `:0` on Linux / macOS, where only the Unix socket is open by default); the button is disabled while starting so a double click cannot start two. When it starts but local programs cannot connect (on Linux / macOS the Unix socket could not be created and TCP is off), a separate warning toast says that SSH forwarding works as usual and that turning on "Also listen on a TCP port" lets local programs connect. **While the built-in engine runs, clicking the button opens a flyout** (anchored below the menu bar on the right, the same spot as the tunnel panel — opening one closes the other; 340 px, styled like §10): the header shows X Server, the display and the program count; each row is one connected X program — its name (the first window's `WM_CLASS`, else the window title, else “X program #number”), a source tag (`user@host:port` for programs forwarded through an SSH session, “This computer” for local ones), its window count and accounted memory, and the window title; one holding the whole X Server gets an orange “Exclusive” badge, and one that disconnected in Retain mode and left only resources says so. Hovering a row reveals a disconnect button, which first asks with a warning-style “Disconnect X program” confirmation (all its windows close, unsaved work is lost) and a danger “Disconnect” button. At the bottom, “Unstick” (releases every X program's mouse and keyboard grabs and its hold on the X Server without stopping it — for when no X window responds, e.g. a remote menu was open when the network dropped) and “Stop X Server” (the stop confirmation below). The flyout refreshes every 2 s while open and closes when the X Server stops. VcXsrv cannot list its programs, so there the button still starts and stops it. When a program has held the X Server exclusively for more than 10 seconds while other X programs wait for it, a warning toast says “X program “name” (source) has held the X Server exclusively for N s; every other X program is waiting for it.” with a “Disconnect it” button; it is updated in place every minute after that. **Shown on every platform**, last in this group. Failures such as the display number being taken or VcXsrv not being found raise an error toast with a “Settings” button that lands on the X Server page; with the display number on “auto”, a number found taken only at the moment of starting is retried with the next free one (up to 4 times), and any exception during start is cleaned up (the server released, back to not running) before the error is reported, so it never gets stuck in “starting”. When stopping while X programs are still connected (only the built-in engine can count them; clients that disconnected in Retain mode and left only resources do not count), a warning-style confirmation “Stop X Server” comes first — it says how many X program connections will be dropped, that all their windows will close and that unsaved work will be lost, with a “Stop” button; with no programs connected it stops right away, and stopping because the main window really closes asks nothing. Same command in the palette: `tools.xserver` |

> Multi-terminal synchronized input no longer occupies a title bar icon. It has been refactored into the “Sync Input” channel in the tab context menu (see §6.1).

### 4A.3 X windows (native windows of the built-in X Server)

Every X top-level window of the built-in X Server is an Avalonia native window (library and interface: [X Server architecture](../xserver/design/architecture.md) §6). The host is the window manager and behaves as follows:

- **Windows that get no native window**: desktop windows (`_NET_WM_WINDOW_TYPE_DESKTOP`: the desktops drawn by xfdesktop, caja, pcmanfm, as large as
  the bounding box of all monitors) — in rootless mode there is no "desktop layer" to put them in, and as a native window one would cover every
  local program as soon as it was activated; a window that becomes a desktop after mapping has its native window closed, and it reappears if the type
  changes back. InputOnly top-levels (GTK's GtkInvisible holding selections and receiving drag-and-drop) are invisible and get none either (they used
  to show up as a black window).
- **Placement**: the native window's content area lines up with the X window's inside (X's border is not drawn; the native frame stands in for it).
  When the client gave a position, the native window's **frame** is aligned with the requested position by ICCCM window gravity (with NorthWest the
  frame's top-left corner sits on the requested coordinates), and once the window is shown and the frame measured, the X window's position is reported
  back to the server; only a request for (0, 0) that is not user-specified (no USPosition) gets centred — dialogs over their parent's content area,
  everything else in the primary monitor's work area, centring the frame; `xterm -geometry +0+0` carries USPosition and stays at (0, 0). System frame
  sizes can only be measured once a window is open, so before showing, the frame measured on the most recent decorated window is used as an estimate,
  and the first frame no longer starts at 0 and then jumps. When a monitor layout change moves the root origin (plugging a secondary screen on the left
  or above, a dock, waking from sleep), every window's X coordinates are reported again from its current native position. The work area is handed to the
  X Server from the system's `Screen.WorkingArea`, so menus, maximized windows and dialogs avoid the taskbar.
- **Mapping honours the states the client set**: a window that set maximized / full screen before mapping is shown maximized / full screen; one with a
  `WM_HINTS` initial_state of IconicState (`xterm -iconic`) is minimized as soon as it maps and does not take the foreground. These states came from the
  client and are not reported back. Later, when the user maximizes / minimizes, only the states the native window manages (maximized, full screen,
  minimized, always on top) change; "skip taskbar", "modal", "sticky" and the like set by the client stay. When a maximized / full-screen window's
  client resizes itself, the native window's current size is pushed back, and the client redraws at the size the system gives it. Whether a native
  size change came from the user is no longer judged by the `Resized` reason (Avalonia's X11 backend always reports `Unspecified`): it is reported
  when it differs by more than a pixel from the last size set from the server's geometry.
- **Buttons follow the Motif hints**: resize / minimize / maximize missing from the `_MOTIF_WM_HINTS` functions disable the corresponding buttons of the
  native window (a dialog that cannot be resized, a splash screen that must not be minimized). Undecorated windows (GTK's HeaderBar, Electron) get a
  borderless native window, and GTK's windows start a native edge resize when pressed within about 4 pixels of their inside edge.
- **Activation and staying on top (focus-stealing prevention)**: when a remote program asks for its window to be activated (`_NET_ACTIVE_WINDOW`), it comes
  to the front only if the request was caused by what the user just did in an X window and the user is using an X window right now; otherwise the
  taskbar entry just flashes (FlashWindowEx on Windows; nothing on other platforms) — so the keys of a sudo password typed in a terminal cannot be taken
  by a remote window that jumped to the front. A client raising its own window (XRaiseWindow, Java's toFront) is honoured only while another X window
  is active; when a client moves the keyboard focus to another top-level itself, that native window is activated, provided the user is using X windows,
  so the user can see where the keyboard went. Menus, tooltips (override-redirect) and "always on top" windows stay on top only while an X window is
  active and drop behind when the user returns to a local window. When `WM_HINTS` urgency or `_NET_WM_STATE_DEMANDS_ATTENTION` turns on and the window
  is not the active one, the taskbar entry flashes.
- **Shapes and transparency**: outside its shape a non-rectangular window (SHAPE: xeyes, irregular popups) is drawn fully transparent; on Windows,
  undecorated windows of this kind also get a window region that trims hit testing to the bounding shape, so clicks outside the shape go through to
  the window underneath (other platforms have no equivalent, and the area outside the shape still catches the mouse). A window with
  `_NET_WM_WINDOW_OPACITY` below 1 gets a transparent background, so the windows behind it show through. Windows with alpha (ARGB visuals) are
  per-pixel transparent only while "Transparent X windows" is on; while it is off (the default) they are shown opaque, as X does without a
  compositing manager.
- **X window titles, taskbar and placement**: titles of windows forwarded over SSH read "user@host:22 — title" ("Show where X windows come
  from", on by default and applied immediately — open windows are retitled on the spot; programs connected locally are not labelled), with the
  source and the title each bidi-isolated (a U+202E in the source cannot reverse the title after it), so windows from several hosts can be told
  apart and a remote program posing as a local credential prompt is easier to spot; windows that draw their own title bar (GTK client-side
  decorations, Electron) have no system title bar, so the source shows only in the taskbar. On Windows the windows of one remote program (by
  WM_CLASS class name) form one taskbar group, which cannot be pinned (a remote program cannot be launched locally). Windows whose program gives
  no position go back to where and how large that program's window was when last closed (remembered by class, instance and WM_WINDOW_ROLE, for
  this run only and never written to disk, kept across X Server restarts; stored in desktop coordinates, so plugging a monitor in on the left
  does not shift them all; not restored when the left part of the title bar is no longer in any monitor's work area; centred as usual while
  another window of the same program is open; client-specified sizes are not overridden).
- **Screenshots**: on Windows the system menu of an X window (right-click on the title bar / Alt+Space) gains "Copy Screenshot" and
  "Save Screenshot…": the window's own content (no system frame, transparent outside a non-rectangular shape), saved by default under the window
  title. Keys and mouse inside X windows belong to the X program, so the entry is in the system menu rather than a shortcut; macOS / Linux have no
  entry yet.
- **Input method, clipboard, cursor and screen saver**: with "Use this computer's input method in X windows" on (the default) the X window is a
  client of the local input method and committed text is typed into the X program (committed text and keys pressed afterwards arrive in order;
  text committed while Shift is held is still typed; switching away mid-composition clears the preedit). When the remote program connects to the
  built-in X Server over XIM (xterm, Emacs, Java, Tk, Motif, GTK 2/3 — they must start with `XMODIFIERS=@im=velashell`, which VelaShell sets in
  the shell after connecting when the session forwards X11 with no display address set and the engine is built-in; a different input method
  already set on the remote side is left alone), the candidate window follows the insertion point the program reports; programs supporting
  on-the-spot (Swing text fields, GTK) draw the composing preedit in their own input box, underlined, and for over-the-spot programs (xterm,
  Emacs) VelaShell overlays the preedit on the insertion point's line. Programs not on XIM (Qt 5 / 6, GTK 4, or without that variable) behave as
  before: the preedit is drawn over the window and the candidate window sits at the last click position. Besides text the clipboard carries HTML and images (images
  and rich text copied from screenshot tools, GIMP or LibreOffice paste locally, and local screenshots paste into remote programs). When a remote
  program holding the pointer pulls it back somewhere (continuous dragging, CAD rotation) the local cursor moves along; when it asks to confine the
  pointer to a window, the cursor cannot leave while an X window is active and is released when switching to a local window (Windows and Linux X11
  desktops; not macOS or Wayland). While a remote video player suspends the screen saver, the local screen saver and display sleep are held off
  too, until it resumes or the X Server stops (Windows only for now). On resize the host waits for programs that declare `_NET_WM_SYNC_REQUEST` to
  finish redrawing before showing them, so dragging an edge does not flicker.
- **One-window mode**: when the "Window mode" setting is one large window / no title bar / fullscreen, the built-in X Server starts in
  one-window mode: at start it opens an "X Desktop :N" window (initially 80% of the primary monitor's work area, centered; without system
  decorations for no title bar, fullscreen right away for fullscreen) holding the whole X screen. Remote programs no longer get a window each —
  a remote window manager (xfwm4, metacity, twm…) manages them, so running `startxfce4` or similar in an SSH session gives the whole remote
  desktop. Resizing that window resizes the screen (the remote desktop re-lays itself out); the picture is shown one-to-one in physical
  pixels. Its close button counts as stopping the X Server (confirmed first, as usual, when programs are connected); with "One display per SSH
  session" on, each session gets an "X Desktop user@host:22" window and closing it closes only that session's display — when programs are connected
  it first asks "Closing the X desktop of user@host:22 disconnects its N X program connection(s)… X programs of other sessions are not affected"
  (a danger-styled "Close"), and the other sessions and the shared display carry on. In one-window mode there is no tray icon forwarding (the
  tray belongs to the remote desktop's panel).
- **Tray icons**: the built-in X Server acts as the system tray, so X programs' tray icons (GtkStatusIcon, Java's SystemTray, Qt, Electron's
  XEmbed fallback) appear in the local tray next to VelaShell's own tray icon (which is still governed by "minimize to tray on close"; neither
  replaces the other). The icon looks as the X program draws it (following redraws after about 0.2 s) and its tooltip is the icon's name, prefixed
  with `user@host:22` for programs forwarded over SSH per "Show where X windows come from" (the same switch as window titles); a
  click sends a left click, and the context menu's "Open (click)" sends a left click and "Menu (right-click)" a right click — the program then
  pops up its own menu (an X popup, near the tray at the bottom-right of the screen). When the program exits or its icon window goes away the
  tray icon disappears; stopping the X Server removes them all.
- **Dragging into X windows**: local text and files can be dragged onto an X program's window (the program must support XDND — GTK, Qt,
  Java and browsers do; xterm does not), and the cursor shows whether the target accepts. Text is handed over directly. Files depend on where
  the program runs: a program forwarded through an SSH session cannot open local paths, so the files (directories with their contents) are first
  uploaded through **that same session's SFTP** to a new `/tmp/velashell-drop-random/` (mode 0700) and the program gets the remote paths
  (`text/uri-list` and plain-text paths), with a toast “Uploading N item(s) to user@host:port for the X program…” while it uploads; if the session
  is gone the files are not accepted, and an upload failure shows an error toast. Uploaded files stay on the remote host — the program still needs
  them — and are not deleted for it. Programs that connected locally get local paths.
- **Dragging out of X programs** (multi-window mode): pressing and dragging in an X program to anywhere outside every X window (the local
  desktop, Explorer / Finder, a local program's window) is taken over by VelaShell as a local drag-and-drop; dropping it is always a **copy** (the
  remote original is untouched). Text is handed over directly. Files depend on where the program runs: a program forwarded through an SSH session
  gives remote paths, so the files (directories with their contents) are first fetched through **that same session's SFTP** into a new directory
  under the `velashell-xdrag` temporary directory, with a toast “Fetching N item(s) dragged out of user@host:port; keep holding the mouse
  button…” — only then does it become a local drag, so the mouse button has to stay down; releasing before that drops nothing and says so; a
  disconnected session or a transfer failure shows an error toast. Fetched files stay in the temporary directory (the target may still be
  copying) and are cleared on a later drag-out after a day; characters not allowed in local file names become `_`. Programs that connected
  locally give local paths directly. Things dragged out and back onto an X window are not accepted (that would only send freshly fetched files
  back). Single-window mode does not take drags out (whatever is dragged outside the X desktop window belongs to the remote desktop).
- **Closing**: the native close button asks the client to close itself (`WM_DELETE_WINDOW`); menus and tooltips do not pass a close on. When a program
  is stuck — it advertises `_NET_WM_PING` and did not reply within 5 seconds — a warning-style confirmation “Not Responding” appears: "“title” is not
  responding. Force quit this X program? Its other windows will close too, and unsaved work will be lost.", with a danger button “Force Quit” that
  disconnects the program. Ownership between native windows comes only from `WM_TRANSIENT_FOR` (popups no longer borrow "the currently active X window"
  as their owner, which could make closing A's window take B's along): when the host closes a window it closes the windows it owns too, and those still
  mapped in X are shown again without an owner; when the user clicks the owner's close button its owned windows do not close with it — instead the
  owner's program is asked to close; application exit and system logoff / shutdown always go through. When the main window really closes (not just to
  the tray), the built-in X Server is stopped.
- **Keyboard and mouse**: X keycodes are injected by physical key, with the layout following the system (or the one chosen in the settings); before
  every press in an X window the system layout is checked and a layout switch takes effect right away (at most once a second on platforms other than
  Windows). Another press of a key that is already down is passed on as auto-repeat, at the system's cadence. When an X window gets the focus, the
  system's real CapsLock / NumLock state is pushed to X (on Linux the desktop's keymap and lock keys are read on a background thread, so activating a
  window no longer stalls the UI). On macOS, if key combinations with Command never get a KeyUp, releasing Command releases them too. Pointer
  coordinates are clamped to X's 16-bit range before injection. Each window keeps the 16 most recently used image cursors, releasing the least recently
  used beyond that and all of them when the window closes. Typing and clicking in the host's own UI are reported to the X Server too, so the idle time
  remote programs see no longer counts X input alone.
- **Restarting right after a stop**: callbacks the old server delivers while shutting down are ignored (windows are recognised by handle, not by XID),
  so they neither create windows from old handles nor close the new server's windows.

The clipboard exchanging only with the focused session and "Restrict programs from SSH sessions" are covered under the “X Server” page in §14 and
in [settings-audit.md](settings-audit.md), eleventh batch; what users run into and what to do is in [X Server troubleshooting](../xserver/troubleshooting.md).

---

## 4B. Tab Bar (Single 36px Row by Default, Multiple Rows Optional, `bg-page` background) ★Refactored

> Located at the top of the right area. **It carries only tabs and overflow controls, and no longer contains global feature buttons** (those have moved up to the menu bar in §4A.2). Design nodes: `nunbT`, overflow control group `Tab Overflow Controls` (`pZGS4`).

### 4B.1 Layout

```
┌───────────────────────────────────────────────────────────────────────┐
│[Tab1][Tab2][Tab3 …scroll…] [+]   ……flexible gap……     [◀][▶][▾]       │
└───────────────────────────────────────────────────────────────────────┘
     ← tab scroll container (clip) →    new tab   spacer       scroll left  scroll right  list
```

- **Tab scroll container** (`clip:true`, `fill_container` taking the middle space): lays out all tabs horizontally; overflow is clipped and can be scrolled horizontally with the wheel; the active tab automatically uses `ScrollIntoView`.
- **`+` New Tab**: follows the last tab.
- **Flexible spacer**: pushes overflow controls to the far right.
- **Overflow control group `◀ ▶ ▾`** on the right, with 24×24 icon buttons:
  1. `◀` chevron-left: scroll left by one screen/one tab; disabled when at the far left.
  2. `▶` chevron-right: scroll right; disabled when at the far right.
  3. `▾` chevron-down: **tab drop-down list** (VS Code style), listing all session names + status dots. Clicking an item activates it and scrolls it into view. This is especially useful during overflow. `◀ ▶` are available only during overflow; `▾` is always available.

### 4B.2 Individual Tab Content

- Structure: status dot (`status-connected/connecting/disconnected`, 7px) + session name (JetBrains Mono 11) + close `x` (12px).
- Active state: `bg-terminal` background + 2px accent bar at the top + `text-primary` text.
- Inactive state: `tab-inactive-bg` background + `text-tertiary` text + `text-muted` close icon, which becomes prominent only on hover.
- Connecting tabs use a yellow dot; disconnected tabs use a red dot.
- Width follows the title (#521): the full title when there is room; when one row is too narrow the tab shrinks as described in §4B.3, and **only the title shrinks** (with an ellipsis) — the dot, icons and the button at the end stay as they are.
- A pinned tab (§4B.5) shows a pin instead of the `x`.
- Interaction:
  - Click = activate; middle-click/click `x` = close, with a second confirmation when there are unsaved changes or active transfers. A pinned tab ignores middle-click.
  - Double-click empty tab space = rename (inline editing).
  - Drag = reorder; drag to an edge = trigger Dock splitting/floating window. A pinned tab only moves within the pinned block, and an unpinned tab cannot be dragged into it.
  - Right-click = tab context menu. Current state (2026-09-27): sync input (terminal tabs only) | Close, Close Other Tabs, Close All Tabs, Close Tabs to the Left, Close Tabs to the Right | Pin Tab / Unpin Tab | Split Horizontally, Split Vertically, Maximize / restore pane | Tab Position, Show Tabs in Multiple Rows (check item). Close Others / All / Left / Right always **skip pinned tabs**. Duplicate session, rename and move to new window are not implemented yet.

### 4B.3 Tab Overflow Logic (VS Code style) ★User specified

- When total tab width ≤ container width: lay out normally, hide/disable `◀ ▶`, and keep `▾` available.
- When the tabs don't fit, **shrink first, then overflow** (#521, following VS Code's `tabSizing: shrink`): only **the widest tabs** are trimmed, down to one common width; tabs narrower than that keep their width (a short tab such as "FTP" doesn't narrow along with the long ones). Overflow below only starts once they are down to the **120px** floor and still don't fit.
- On overflow:
  1. Clip the container and automatically use `ScrollIntoView` to keep the active tab visible.
  2. Allow `◀ / ▶` to scroll; disable the corresponding button at each boundary.
  3. Allow horizontal wheel scrolling over the container.
  4. Always list all tabs in the `▾` drop-down as a fallback for quick navigation during overflow.
- Drop-down items: status dot + session name + a check mark on the right for the current item. Support keyboard up/down + Enter selection. Tabs can also be closed here, with `x` appearing on hover.
- **Multiple rows** (#521, matching Visual Studio's "Show tabs in multiple rows"): when on, tabs that don't fit wrap onto the next row instead of shrinking or overflowing; the strip grows with the number of rows (35px per row, the 1px separator stays below the last row), and the active tab's accent line moves to its row. Two entry points change the same value: "Show Tabs in Multiple Rows" (check item) in any tab's right-click menu, and the switch of the same name under Settings → Appearance → Window (`AppearanceOptions.MultiRowTabs`, off by default). One switch for the whole workspace, so every pane changes together; it only applies to tab strips at the top — a strip docked left or right already has one tab per row, and the menu item is disabled there.

### 4B.4 Resource Monitor on Tab Hover ★User specified (see §11)

- Move the mouse over **any tab name** and **hover stationary for >400ms** to show the “System Resource Monitor” at the **current mouse position**.
- Move the mouse out of the tab, or into another tab, to make the panel **disappear automatically**. A 150ms fade-out debounce is recommended. Moving into the panel itself keeps it visible.
- The panel shows resource data for the **session associated with that tab**, not the active session, so it can quickly inspect other sessions.

### 4B.5 Pinned Tabs ★User specified (#521)

> Matches Visual Studio's "Pin Tab". This is **not** the Pin of the old Dock.Avalonia (pinning a panel to the side as auto-hide, see the product red line in [dock-replacement-plan.md](dock-replacement-plan.md)); that one is still out of scope.

- Entry points: "Pin Tab" / "Unpin Tab" in the tab's right-click menu (only the one that applies is shown); the pin at the end of a pinned tab — one click unpins it.
- Position: pinned tabs always come first in their pane. Pinning moves the tab to the end of the pinned block, unpinning moves it to the start of the unpinned ones (the nearest valid position either way). Dragging and moving between panes keep the same order: a pinned tab lands in the target pane's pinned block, an unpinned tab can only land after it, and the insertion line is drawn where the tab will actually land.
- Protection: Close Others / All / Left / Right skip pinned tabs, and middle-click does not close them. Closing the tab itself still works: right-click → Close, or `Ctrl+W`.
- A "connecting" placeholder tab can be pinned too; the real tab that replaces it once connected keeps the pin.
- The pinned state lasts for the current run only and, like the split layout, is not saved (layout persistence is a separate item in the host repo's `feature-plan.md`).

---

## 5. Main Terminal Area (`bg-terminal`)

### 5.1 In-Terminal Toolbar (28px, below the tab bar, `BdPtF`)

> `GQQwj` has moved to the menu bar (§4A.2). This row retains only **read-only information for the current session** on the left, plus a small number of optional actions on the right:
- Left (`termInfo`): `root@web-prod-01:~` (accent) + `uptime: 42d 7h 23m` (muted) + `|` + `latency: 12ms` (green).
- Right: optionally retain secondary buttons strongly tied to the current terminal, such as “Clear Screen / Full Screen / More”. Global functions have moved to the menu bar.

### 5.2 Terminal Canvas

- Render using the VT engine: draw monospaced text line by line, with support for xterm-256color, true color, cursor, selection, and hyperlink detection.
- Support mouse selection and copy, right-click paste, Ctrl+wheel font zoom, and scrolling back through the buffer.
- **Alt+left-drag = rectangular block selection** (#128, aligned with Windows Terminal behavior): whether Alt is held at mouse-down determines whether the operation is block selection or a normal linear selection. Changing Alt during the drag does not switch modes. Copy takes the same column range from each line, always inserting line breaks between lines. When the application enables mouse tracking (htop/vim/tmux), mouse events are given to the application; use Shift+Alt+drag to force block selection.
- **Shift+left-click = extend the selection** (#266, aligned with Windows Terminal / xterm): when a selection already exists, Shift+click keeps the anchor in place and moves only the far end to the clicked cell (before or after the anchor — the selection flips direction accordingly); keep the button held to keep dragging and fine-tune it. To grab a long log spanning more than one screen, select the start, scroll back, then Shift+click the end. The extension reuses the linear/block mode fixed at the original mouse-down. With no selection yet, Shift+click still starts a new one, preserving the existing "hold Shift to bypass application mouse reporting and select text" semantics.
- **Ctrl+Shift+left-drag = append a discontiguous region**: commits the in-progress region and starts another one, repeatable. Copy concatenates the regions in **document order, top to bottom**, with a line break between them — so "select line 1, Ctrl+Shift-select line 3, copy once and get both lines" holds regardless of the order they were picked. Each region remembers its own mode, so Ctrl+Shift+Alt+drag appends a rectangular region that coexists with linear ones, and Ctrl+Shift+double-click appends another word. A plain drag without Ctrl+Shift starts over (dropping every appended region); so does a search-hit highlight. Ctrl+**Shift**+click on a URL does not open the browser (opening links is Ctrl+click without Shift). Terminals have no precedent for this (Windows Terminal / iTerm2 / xterm all have single-region selection only), so the binding is ours.
- **Banners the server sends during authentication** (`SSH_MSG_USERAUTH_BANNER`: legal notices, PAM's "your password will expire in 3 days") are written at the top of the terminal as grey `●` notice lines when the first shell opens. Banners from every hop of a jump chain are collected, in arrival order. They are shown once; further shells on the same connection do not repeat them. Banners come from an **unauthenticated** peer, so in each line control and bidi control characters are replaced with `?` before it reaches the terminal (which defuses escape sequences; the rule is the SSH librarys public `PeerText.Sanitize`, which the keyboard-interactive prompt dialog uses too), with at most 64 lines (the rest fold into a single `…` line) and 512 characters per line. The host used to leave this callback unset, and banners were silently dropped.
  **Lines sent before the identification string** (some bastion hosts and network devices print a legal notice or a maintenance announcement before the SSH identification string, SSH library spec 02 §3) are handled the same way: as one banner, placed before that hop's authentication banners (since 2026-10-05; before that the SSH library collected them but the host did not take them).
- See §7 “Disconnected State” for the idle/disconnected overlay, and §7 “Connecting State” for the connecting one.

### 5.3 In-Terminal Search Bar

- Triggered by the menu bar `search` button (§4A.2) or Ctrl+F. A search field slides in from the top of the terminal: input + previous/next + match count + close. Matches are highlighted and Enter jumps to the next match.

---

## 6. File Browser / SFTP (Lower Right Area, Default 220px, Collapsible/Resizable)

- **Header (36px)**: left = clickable current-path breadcrumb for level-by-level navigation, then a pencil button (or Ctrl+L) that switches to manual path entry (Enter navigates, Esc cancels; `~` and `~/…` expand to the login home directory, while `~user` and `~user/…` are expanded by the server (SFTP's `expand-path` / `home-directory`, since 2026-10-05), and anything that cannot be expanded is passed to the server as is so it reports the real error), then a `copy` button that **copies the full remote path of the current directory in one click** (the same command as “Copy Current Folder Path” in the empty-area context menu — previously you had to enter edit mode before you could select and copy it), and finally refresh; right = icons for view switching, upload, new folder, hidden-file toggle, and so on. To the left of the upload button is the free space of the file system holding the current directory, “12 GB free of 50 GB” (since 2026-10-05, `VelaTextMuted`, monospace 10px; from SFTP's `statvfs@openssh.com`, refreshed on entering a directory and on refresh; not shown when it cannot be queried: servers without the extension, FTP, plugin protocols).
- **Space precheck before uploading** (since 2026-10-05): when the files of a batch uploaded from this machine (minus resume offsets) add up to more than a normal user can still write on the target file system, a yellow confirmation “Not enough space?” comes first (“The target file system has X free, but these files take Y. They may not fit — running out of space midway leaves an incomplete file on the server. Upload anyway?”, buttons “Upload anyway” / Cancel); Cancel uploads nothing. When usage cannot be queried, nothing is held back.
- **Column header (26px, `bg-surface`)**: `Name(280) | Size(100) | Permissions(120) | Modified`. Columns can be sorted by clicking.
- **File list (`fill_container`, scrollable)**: 28px per row. Icon (folder/file type) + name + size + permissions (`drwxr-xr-x`) + time. The first row may be `..` to return to the parent directory.
- Interaction:
  - Double-click folder = enter; double-click file = download and open with the default application, or preview.
  - Drag a local file here = **upload** (triggers the file transfer component in §9); drag a list item locally = **download**.
  - **Name conflicts**: when an upload/download encounters an existing file with the same name, use the setting in §14 File Transfer “When a file already exists”: ask (confirmation dialog: overwrite or skip) / overwrite / skip / rename (`file (1).txt`).
  - Right-click = file context menu (download, upload, rename, delete, properties, copy path, new). “New” includes **New Symbolic Link**: first ask for the link target path (prefilled with the row’s path when right-clicking a row), then ask for the link name (prefilled with the last segment of the target). The target is written verbatim; relative paths resolve against the link’s directory (`ln -s` semantics). On FTP, only servers that support `SITE SYMLINK` can create links; plugin protocols report that the operation is not supported.
  - **Properties dialog**: basic information (type, path, link target, size, modified time) + owner / group + the rwx permission matrix and octal input (kept in sync both ways, with a `chmod 640` echo beside them).
    - On SFTP sessions, Owner and Group are **editable drop-downs**: the suggestions are the user and group names from the remote passwd / group databases (the same lookup that turns ids into names in the list), and a numeric UID / GID can be typed directly. Input is read in `chown`’s order: as a name first, and as a number only if no such name exists. On SFTP-only accounts where the lookup is unavailable, the suggestions are empty and only numbers work.
    - Owner / group names in the list: sessions with exec read the whole table once (`getent`); ids missing from it (on SFTP-only accounts the whole table is missing) are filled in through SFTP's `users-groups-by-id@openssh.com` (since 2026-10-06, each id asked once per session), and only ids still unknown show as numbers.
    - Clearing a field means “leave unchanged”; a name the remote host doesn’t know is reported in the error bar on the spot and nothing is sent.
    - OK applies only what changed, **chown before chmod**: the owner change is the one most likely to be refused (a non-root user can only switch the group to one they belong to), so when it is refused nothing has changed yet; if chown succeeds and chmod is refused, the list is still refreshed.
    - FTP and plugin protocols cannot change ownership, so both rows are read-only there. No recursion or batch operation; for a symbolic link, the change applies to the target.
  - **Symbolic links**: the icon is `folder-symlink` (points to a directory, amber, double-click to enter) or `file-symlink` (points to a file, or the link is broken); hovering the name shows “→ target”; the Type column reads “Symbolic Link”; the permission string starts with `l`; the properties dialog adds a “Link Target” row.
    - **Delete** removes only the link itself and never touches the target; recursive deletes also remove nested links as links. Before descending into a subdirectory, a non-following stat confirms it is a real directory —
      the SFTP draft does not say whether listing attributes come from lstat or stat, and when a server reports followed attributes, a link to a directory looks like a real directory in the listing.
    - **Copy** (remote → remote) produces a link (`cp -P`).
  - **Copies on the same server** (since 2026-10-06): when the server supports `copy-data` (OpenSSH's sftp-server does), the copy happens inside the server and the data never leaves it —
    it used to always download to a local temporary file and upload again, minutes for a file of several GB. Progress works as usual; with preserve timestamps on, the destination gets the source's modification time. Servers without it still download then upload.
    - **Downloading a folder** does not descend into nested directory links (`rsync -r` semantics, avoiding links to `/` or back to an ancestor); an explicitly selected link is still followed, and links to files download the file content.
  - **Directory comparison and synchronization** (dual-pane SFTP / FTP / FTPS documents only; the terminal sidebar's file browser has no local pane and does not offer it): a 32px toolbar at the top of the document holds two icon-plus-text buttons, "Compare Directories" (`git-compare`) and "Synchronize…" (`folder-sync`), with the verdict of the last comparison on the right.
    - **Compare Directories**: compares the current level of both panes and selects the differing items in each pane (newer or unique items; different-size items and conflicts are selected in both). The verdict is cleared as soon as either pane changes directory.
    - **Synchronize…** opens a separate window (same window spec as route tracing; one per document, clicking again brings it to the front). Top to bottom: options (local directory + Browse, remote directory, direction segments, mode segments, compare-by criteria (modification time / file size / SHA-256 checksum, the last checked by default with a tooltip explaining the fallback), Delete files, Existing files only, file mask) → preview list (checkbox | action | path | local size and time | remote size and time; delete rows use the danger color) → summary / notes (unhandled conflicts, directory links not followed, server cannot set modification times) / errors → status bar (progress and status text; Select all, Select none, Keep remote up to date, Compare, Cancel, Synchronize).
    - Changing any option discards the preview; a plan containing deletions asks for confirmation before running; after running, the window re-checks automatically and the status bar states whether any differences remain.
    - **Keep remote up to date**: the list area becomes an activity log (newest first) and the button reads "Stop watching"; closing the window or the document stops watching.
    - Option meanings, execution order, and comparison rules are in section VII of [sftp-dual-pane-winscp-gap-analysis.md](sftp-dual-pane-winscp-gap-analysis.md).
  - Selected state `bg-hover`; multi-select (Ctrl/Shift) for batch operations.
  - Navigation uses “load first, commit later”: on failure, retain the original path, list, selection, and scroll position; after entering a new directory, clear the selection and return to the top.
  - Refreshing the current directory preserves selected items that still exist and the scroll position. Do not rebuild the list when content has not changed. For concurrent navigation, accept only the latest request result.

### 6.1 Multi-Terminal Sync Input Channel (replaces the original Broadcast Input Bar)

- Entry is in the tab context menu under “Sync Input”: four fixed channels A/B/C/D, distinguished by color (pink/blue/orange/green), plus “Leave All Channels”.
- Peer model: any user input in a channel (keyboard/IME/paste) is copied in real time to the other tabs in that channel. Switching to any tab in the channel preserves the same shared input behavior. Each tab can belong to only one channel at a time.
- Tabs in a channel show the channel letter (A/B/C/D, in the channel color) before the status dot in the tab header. A channel bar appears above the terminal: color swatch + “Input synchronized in channel X” + [Pause] [Leave Channel] [Close Channel].
  - **Pause**: this tab temporarily stops sending and receiving channel input; click again to resume.
  - **Leave Channel**: only this tab leaves. The close button on the right side of the bar has the same effect.
  - **Close Channel**: all tabs in the channel leave together.
- Forward directly to each target PTY (the bridge’s SendRaw), without passing through input events on the receiving terminal control. This prevents forwarding loops and does not trigger command-completion popups in non-focused tabs. Smart suggestions appear only in the tab where the user is typing.
- Disconnected tabs do not receive forwarded input, gated by `IsConnected`. Closing a tab automatically leaves its channel. Multi-target dispatch of quick commands no longer goes through a second channel forwarding step, preventing duplicate injection.

### 6.2 Remote + remote dual pane (#524)

Opened from the multi-select in §3 (exactly two connections): one file tab with each pane connected to a different machine. The tab title reads "left ⇄ right", its color bar and icon come from the left profile, and its status dot shows the worse of the two connections.

- **Connecting**: the two connect one after the other (each with its usual credential prompt and host-key / certificate confirmation), under a placeholder tab titled "left ⇄ right". **If either one does not connect, the whole thing is rolled back**: the one already connected is disconnected at once, a failure stays on the placeholder's failure card, and a cancel removes the placeholder; there is never a document with only one pane. Both connections belong to the document and are disconnected together when the tab closes; sessions already open in other tabs are not reused. Any combination of SSH / SFTP / FTP / FTPS / plugin file protocols works.
- **A plugin file-protocol pane** carries the protocol's own right-click actions (e.g. S3's "Copy share link"), and the tab icon is the one the plugin supplies (when the left pane is the plugin). The pane can only **receive** files from the other pane if the plugin also implements the SDK's `IProtocolStreamUpload`; otherwise it is a source only — the copy button toward it is disabled and drops onto it are refused.
- **The panes**: each is a full remote file browser (everything in §6). The server badge in each header carries that machine's identity color (the same one as the tab color bar) — with both sides remote, the path alone does not tell which pane is which machine.
- **Moving files between the panes**:
  - Toolbar "Copy to right ›" and "‹ Copy to left": moves the files and folders selected in one pane into the other pane's current folder;
  - dragging rows from one pane onto the other (only rows from the other pane of the same document are accepted; dragging within one pane is still not supported, see #474).
  - The data is **streamed through this computer's memory and never written to local disk**: the source is read sequentially while the target is written, so it takes about as long as the slower leg. The protocols have no server-to-server copy primitive, and running `scp` / `rsync` on A to reach B directly is deliberately not done (it needs B's credentials or a forwarded agent on A, and A must be able to reach B).
  - It shares the upload pipeline: conflict policy, concurrency limit, bandwidth limit (counted as upload), preserve timestamps, transfer log (direction logged as `RELAY`), and retry from the transfer popup after a failure. The transfer popup shows the direction as `⇄`.
  - **Resume**: same rules as uploads (Settings → File Transfer → Resume). After a cancel or failure the partial target is kept; on retry, or the next time the same file is transferred, the target is verified to be a prefix of the source (the target's current length, backed off by one in-flight write window, tail compared) and writing continues from there. Resuming needs to seek within the source, so it **only happens when the source is SFTP**; when the source is FTP or a plugin protocol (sequential reads only), or the target is a plugin protocol (the SDK's streamed upload does not resume), the resume point cannot be verified and the file is **handled by the same-name conflict policy** (the same path as a verified mismatch) rather than silently sent again in full — which would overwrite an unrelated file of the same name without asking. With resume turned off, a failure midway follows the "clean up partial files" setting. If the source cannot be read before writing starts (deleted / renamed), the target is untouched and an existing file of the same name is **never** deleted as if it were a partial file. A cancel is reported as a cancel, not as "transfer interrupted, can be resumed".
  - Folders are recursive; links to directories nested inside them are not followed (same rule as downloads).
- **Compare directories**: compares the current level of both panes (by size and modification time, using the inferred time precision when either side is FTP), selects the newer, unique or different entries in each pane, and shows the summary on the right of the toolbar.
- **Not provided**: the synchronize window and "keep remote up to date" (they are designed around local ↔ remote); restore-sessions-on-startup does not reopen these tabs.

---

## 7. Status Bar (Full-width 24px, `bg-sidebar`)

- **Left**: `wifi` icon (connection color) + `SSH • web-prod-01:22` + `｜` + `Latency: 12ms` (accent) + `｜` + `↑ 2h 34m` (online duration).
  Latency is measured every third sample: SSH sessions use the SSH-level round trip (a keepalive request, accurate through proxies and jump hosts, since 2026-10-06); other sessions fall back to an ICMP ping of the host;
  nothing is shown when it cannot be measured (ICMP blocked, the SSH measurement taking over 2 seconds). It used to be ICMP for everything: for machines reached through a jump host or proxy it measured the direct path from this machine to the target, and servers that block ICMP never showed a latency.
- **Right**: `xterm-256color` ｜ `120×36` (terminal size) ｜ `cpu 23%` ｜ `memory 1.2G` ｜ `net 4.2 MB/s` ｜ `UTF-8`.
- CPU, memory, and network are lightweight real-time metrics for the current session. They share a source with the resource panel in §11; here they are a compact persistent version.
- Each field is clickable. For example, clicking the size triggers one resize synchronization, and clicking the encoding opens the encoding menu.

### Disconnected State (`ZufZw`)

When a session disconnects, dim the terminal canvas and center a disconnection notice (red status + “Connection disconnected” + “Reconnect” button + reason/time). The corresponding tab dot turns red, the status bar connection icon turns red, and the host dot in the sidebar turns red. Support an automatic reconnect toggle with exponential backoff.

**Automatic reconnect only rescues disconnects nobody asked for** — dropped links, timeouts, a server that
died. The four cases below never reconnect on their own; the notice still shows, and coming back is the
user’s call via “Reconnect” (the same principle as “a tunnel the user stopped by hand is never restarted”):

- The user clicked “Disconnect” in the toolbar.
- The user typed `exit` / `logout` on the remote, i.e. the remote shell exited on its own (on the wire: the
  peer closed the channel normally).
- A local terminal’s shell exited.
- The very first connection never succeeded — authentication failures above all, where retrying just throws
  the same wrong password at the server a few more times.

### Connecting State

**The tab comes first, the session second.** The moment the user asks for a connection the tab is there,
in a connecting state; once the session exists it is replaced in place by the real content. The four
document-style connections (SFTP / FTP / plugin file systems S3… / plugin workspaces Redis…) used to
**only get a tab once connected**, so on a slow link there was no tab, no dot and no spinner — the click
looked dead.

While connecting, three surfaces answer at once:

| Surface | What |
| --- | --- |
| Tab content | Centred card: an **indeterminate** progress ring in a 48px circle + “Connecting to &lt;name&gt;” + the connection type (SFTP / FTP / S3 / Redis…) + one way out |
| Tab strip | Amber status dot + a 12px ring in place of the type icon, the arc in that session’s accent colour |
| Status bar (bottom right) | The background-activity ring + a one-line summary; hover lists every activity in flight |

Easy things to get wrong:

- **No spinning while waiting on a human.** Credential and certificate prompts are people-time, not
  network time: close out the activity before raising the prompt and open a new one when it returns.
- Progress is **always indeterminate**: connecting has no countable stages, and a smooth fake bar is
  just decoration.
- The card’s “Cancel”, the tab’s ×, `Ctrl+W` and the context-menu closes are one and the same: closing a
  tab that is still connecting means “don’t”, and the cancellation propagates all the way into the handshake.
- The profile’s dot in the session tree turns amber for the duration too.

### Connection Failures for Document-Style Connections

With the placeholder tab from the previous section, a failure **lands in the tab that owns it**: the same
card in its error state (red icon + the reason + “Reconnect” + “Close Tab”), identical to the terminal’s
in-tab disconnect overlay. **No modal dialog** — a modal blocks work that has nothing to do with this
failure. Document-style connections used to raise one only because “no connection meant no tab, and a
failure had nowhere to go”; now it has somewhere.

- **A deliberate cancel is not a failure**: cancelling the credential prompt raises nothing and takes the
  placeholder tab with it (leaving an empty shell behind makes the cancel pointless). Cancelling also clears
  `LastConnectionError`, which is how the plugin-opened-session path tells “could not connect” apart from
  “the user said no”. Declining a certificate likewise raises no dialog, but its reason does go on the failure
  card — that is how this connection ended, and it belongs in its own tab.
- When all three credential attempts fail and the loop exits, the last reason lands on the card as well;
  otherwise typing a password three times ends in silence.
- Messages from the plugin protocol exception family (`ProtocolConnectionException` and friends) are, per the
  SDK contract, already user-facing and already carry the endpoint, so the host presents them verbatim rather
  than wrapping them in another “Failed to connect to X:” that prints the same address twice.
- The credential prompt still runs behind its serialization gate: restoring several sessions at startup fires
  them concurrently, and two modal dialogs over the same owner deadlock each other.

### Background Activity Ring (status bar, bottom right)

Every piece of background work that can outlast a moment registers here; the ring shows it and hovering
lists them. Registered today: plugin loading / verification / prewarm, settings sync, and **every connection
path** — first SSH connect and reconnect, plugin terminals (Telnet / serial) on open and reconnect, the four
document-style connections, and opening a plugin panel.

The boundary: a plugin’s lazy activation is registered by the plugin manager itself; whatever the plugin does
to load its own data after the panel opens is invisible to the host and stays unspun — no fabricated spinner
for work the host cannot see.

---

## 8. Command Palette ★User specified (`Ctrl+P`)

**Trigger**: global `Ctrl+P` opens the palette. The `zap` button does the same. It appears above center, is 560px wide, uses `bg-surface` + a large shadow, and has an overlay. Click the overlay or press `Esc` to close.

**Structure, from top to bottom**:
1. **Search field (48px)**: `search` icon + input (JetBrains Mono 14) + blinking caret + `Ctrl+P` mode hint on the right. Filter with fuzzy matching as the user types.
2. **“Sessions” group**: 10px muted uppercase group heading + session result rows, each with status dot + session name + environment label (`Production` on `accent-dim`) + `Enter Connect` hint on the right. The first item is highlighted with `bg-active` by default.
3. **Divider**.
4. **“Commands” group**: command result rows with icon + command name (Inter 12) + keyboard shortcut on the right (it follows the current bindings: a changed shortcut shows the new keys, an unbound one shows nothing). Examples:
   - `New SSH Connection` … `Ctrl+N`
   - `Open SFTP File Manager` … `Ctrl+Shift+F`
   - `Open Settings` … `Ctrl+,`
   - (Extensions) `Open Tunnel Manager`, `Split Pane`, `Switch Theme`, `Record Session`, `Connection Diagnostics`, `Operations Orchestration`…
   - `Send Break` (session category, since 2026-10-05): available when the active tab is a connected SSH session (local and plugin terminals cannot use it); it sends a BREAK (RFC 4335),
     which is how serial console servers and network device consoles get into ROMMON / the boot menu. Whether the server performed it is reported in a toast: an ordinary toast when it did,
     a yellow one, “The server did not perform the Break (it may not support it, or the session has no terminal)”, when it did not.
5. **Footer (32px, `bg-input`)**: left = key hints `↑↓ Navigate` / `↵ Confirm` / `Esc Close`; right = result count `6 results`.

**Interaction logic**:
- A prefix switches modes: `>` commands, `ssh ` sessions, `@` files, `#` command history, `:` line navigation (optional extension).
- Move between results with `↑/↓` or `Ctrl+P/N`, execute the highlighted item with `Enter`, complete with `Tab`, and close with `Esc`.
- `Enter` on a session item connects and activates its tab. `Enter` on a command item executes the corresponding function, equivalent to clicking its entry point.
- Support recently used ordering and fuzzy scoring, with matched characters highlighted for subsequence matches.

---

## 9. File Transfer Notification Component ★User specified (Floating / Draggable / Auto-Hiding)

**Appearance condition**: **only when an active file transfer task exists**, show a floating panel in the **upper-right corner of the main window** (280px wide, `bg-surface` + small shadow + corner radius 6). It does not exist when there are no tasks.

**Structure**:
- **Header (32px, bottom divider)**: left = `arrow-up-down` (accent) + “Transfers” + count badge (`accent` background, such as `2`); right = `x` close, which only hides the panel and does not cancel tasks. **The entire header is a drag handle.**
- **Transfer entries, stacked vertically**:
  - First row: filename, such as `backup-2025.tar.gz`, + percentage on the right (`67%` accent) or status (`Complete` in green).
  - Progress bar (3px, `border-primary` background + proportionally filled `accent`).
  - Description row (9px muted): `142 MB / 212 MB • 4.2 MB/s • ↑ Uploading` / `1.2 KB • Complete • ↓ Downloaded`.
  - On hover, actions appear on the right: pause/resume, cancel, open containing folder.

**Interaction logic (★critical)**:
1. **Draggable**: hold the header to move freely within the main window. Remember the position for the next appearance and keep it within the visible window bounds.
2. **Automatic appearance**: when a new transfer starts, fade in from the upper-right if the panel does not exist.
3. **Real-time progress**: stack multiple tasks vertically; the badge count equals the number of active tasks.
4. **Automatic disappearance**: **when all transfer tasks complete, or all are canceled/failed and acknowledged, the panel disappears automatically**. After all tasks complete, it is recommended to remain for about 3 seconds showing the “Complete” state before fading out. If a new task is added during that period, reset the timer.
5. Mark failed tasks in red and show “Retry”. Clicking `x` manually only collapses the panel; tasks continue in the background and can be reopened from the status bar or command palette.

---

## 10. Tunnel Management Panel ★User specified (opened by clicking “Tunnel”)

**Trigger**: click the menu bar `route` (Tunnel) button (§4A.2), select Tools → Tunnel Manager, or choose Open Tunnel Manager in the command palette. Show it as a floating panel near the button (380px wide, `bg-surface` + small shadow).

**Structure**:
- **Header (40px, bottom divider)**: left = `route` (accent) + “Tunnels” + tunnel count badge (same colouring as the Create button: `accent-dim` background + `accent` text; never a solid `accent` fill, where accent-coloured text disappears); right = `plus` (expand new form) + `x` (close panel). The header doubles as a drag handle: drag the panel out of the way and it remembers the spot on release, reopening there next time (the same mechanism as the transfer panel and the message centre: an offset from the default anchor, clamped back into view when the window shrinks).
- **Tunnel list, one row per tunnel**:
  - First row: status dot (green = active) + tunnel name, such as `MySQL Forwarding` + type label on the right (`Local` with `accent-dim` / `Remote` with an `info` background / `Dynamic`, extensible) + an outlined `AUTO` badge when automatic reconnect is on, and an outlined `STARTUP` badge when “Set up automatically when VelaShell starts” is on. When the row runs out of room the tunnel name is truncated; the badges always stay whole.
  - Detail row (11px monospaced; the route line in `text-secondary`, the other descriptions in `text-tertiary`): `L 3306 → db-prod-01:3306` / `R 6379 → localhost:6379`.
  - Status row: uptime while active, otherwise the status text.
  - **Statistics row** (11px `text-tertiary` monospaced; the description row the `fuXS7` draft reserved): `3 conns · 1.4 MB`, calling out the live count when connections are transferring (`3 conns (2 live) · 1.4 MB`), and “No connections yet” before anything has connected. The byte figure combines both directions.
  - On hover, show actions: enable/disable toggle, edit, delete. Starting and stopping go over the network and take a moment: while one is in flight the start/stop button is replaced by a spinning progress ring (not a button, so it cannot be clicked again), edit and delete are disabled, and the status row reads “Starting tunnel…” / “Stopping tunnel…”; when it finishes the matching button comes back. The same applies while automatic reconnect is rebuilding a tunnel, so a manual start in the meantime does not create a second copy.
- **New Tunnel form (`plus` expands, `tunNewForm`)**:
  - Title: `circle-plus` + “New Tunnel”.
  - One row with three fields: **Type** drop-down (Local L / Remote R / Dynamic D, 80px) + **Local Port** input + **Remote Address** input (`host:port`).
  - Check boxes: “Forward to the server itself” (locks the target to 127.0.0.1), “Reconnect automatically after a dropout” (**off by default**) and “Set up automatically when VelaShell starts” (**off by default**).
  - Button row, right aligned: `Cancel` (outlined) / `Create` (`accent-dim` pill: light background + accent outline/icon/text, `plus` + “Create”).
- **Interaction logic**:
  - Attempt to establish forwarding immediately after creation. Success = green dot + increment badge; failure = red dot + error message.
  - Local and dynamic forwards run a **local port-occupancy precheck** before binding, reporting “local port 27017 is already in use by another program” instead of a low-level socket error.
  - A tunnel does **not** follow the lifecycle of a terminal tab: creating or starting one while disconnected opens a background connection dedicated to tunnels, and that connection is dropped once the server's last tunnel is removed.
  - When the host session drops, its tunnels are marked stopped; those with “reconnect automatically after a dropout” are redialed and rebuilt on their own (failures back off from 10s to 5min), and the rest wait for a one-click start. **A tunnel the user stopped by hand is never brought back up automatically.**
  - **Running state is never persisted**: only the tunnel configuration is saved, so after a restart every tunnel is stopped and waits for a manual start. Only tunnels with “Set up automatically when VelaShell starts” are connected and set up in the background at launch (without holding up startup; a missing credential still brings up the login prompt). A failure during that automatic setup (the server is unreachable, the local port is taken, the credential prompt was cancelled) raises an error toast in the bottom-right corner naming the server and the tunnel; one server failing does not stop the others, one tunnel failing does not stop the rest on the same server, and if none came up the background connection opened for them is dropped again.
  - Type descriptions: local forwarding (`-L`), remote forwarding (`-R`), dynamic SOCKS (`-D`).

---

## 11. System Resource Monitor Panel ★User specified (shown after 400ms tab hover)

**Trigger**: hover over any **tab name** for **>400ms** to show the panel at the **mouse position** (280px wide, `bg-surface` + small shadow + `padding:12` + `gap:8`). **It disappears automatically when the mouse leaves** (see the debounce in §4B.4).

**Content, from top to bottom**:
1. **Header**: hostname (Inter 13 bold, such as `Ubuntu-Prod-WEB`) + latency on the right (`9ms` in green).
2. **CPU**: `CPU (8 cores)` + percentage on the right, `12%`; progress bar below (6px, `bg-active` background + `term-blue` fill).
3. **RAM**: `RAM` + `4.2 / 16 GB`; progress bar filled with `term-yellow`.
4. **Disk**: `Disk` + `120 / 512 GB (23%)`; progress bar filled with `term-green` (`#55EFC4`).
5. **System information (10px, two label/value columns)**: `OS Version：Ubuntu 22.04.4 LTS`, `Kernel：Linux 6.8.0-40-generic`. Load average, process count, and network throughput can be added.

**Logic**:
- Show a real-time snapshot for the session represented by the tab. Fetch once when opened and poll every second.
- Change progress bar colors by threshold: normal green/blue, >70% yellow, >90% red.
- Avoid screen boundaries when positioning the panel. Flip left/up near the right/bottom edge.
- If the session is disconnected, show a “Data unavailable” placeholder.

---

## 12. Session Context Menu (`e6klM`)

Right-click a session row in the sidebar to open it (200px, `bg-surface`, corner radius 6, vertical `padding:4`). Items use an icon + text, `bg-hover` on hover, and red text for dangerous actions:

- Connect / Disconnect
- Open in New Tab / Open in New Window
- Edit Connection (opens the §13 form with fields prefilled)
- Duplicate Session / Copy Address
- Pin to Top / Unpin (the label follows the current state; a pinned row is hoisted to the very top of the tree, see §4)
- Join Sync Group
- Open SFTP
- Move to Group ▸ (submenu)
- Divider
- Delete (red)

**Multi-select menu** (joesdu/VelaShell#571): after a Ctrl / Shift multi-select (§3), right-clicking one of the selected rows opens not the menu above but one that acts on every selected connection:

- Open N connections (`plug`, accent primary action) — started in tree order from top to bottom with at most 6 handshakes at a
  time, so the tabs appear one after another; above 10 it asks first ("This will open N connections at once"). A failure only
  shows in its own tab
- Open in dual-pane SFTP (`arrow-right-left`, §6.2) — only with exactly two; if one of them cannot go into the dual pane the item
  is disabled, with a line below explaining why
- Divider
- Move to Group ▸ (submenu, as above; moved one by one, so "delete the group once it is empty" still applies)
- Batch edit… (`sliders-horizontal`, [batch edit dialog](#121-batch-edit-dialog))
- Export selected… (`file-down`, [session import](session-import.md) section 7; their jump hosts come along automatically)
- Divider
- Delete N connections (red) — asks first and cannot be undone; same rules as deleting one (an emptied group stays, jump hosts
  pointing at it are not rewritten)

Right-clicking a row outside the selection opens the menu above as usual and ends the selection.

**Group menu** (right-click a group row; the group is selected first):

- Open all connections (`plug`, accent primary action) — same concurrency and confirmation as "Open N connections"; nothing
  happens on an empty group
- Divider
- Move up / Move down (`arrow-up` / `arrow-down`) — disabled at the top / bottom
- Divider
- Export group… (`file-down`) / Import into this group… (`file-up`; the import dialog's "Put into" preselects this group)
- Divider
- Delete Group (red, confirmed; the connections inside go with it)

### 12.1 Batch edit dialog

Opened from "Batch edit…" in the multi-select menu (540 wide, the same shell as the shared credential editor: 28px title bar +
52px button bar, action buttons from the two shared themes in DESIGN.md §5.1). Every item has a checkbox in front:
**only ticked items change; everything else stays as it is on each connection**:

| Item | Notes |
| --- | --- |
| Username | prefilled when all selected connections share one, empty otherwise |
| Port | 1–65535; prefilled when shared, 22 otherwise |
| Authentication | segmented: shared credential (pick one, #550) / password (may be left empty to be asked when connecting) / key (file + optional passphrase) / agent. FTP and plugin protocols only take passwords; those rows are skipped for anything else |
| Jump host | direct, or one of this computer's SSH / SFTP connections (excluding the ones being edited); only applies to SSH / SFTP, and the whole batch is skipped if it would close a loop |

The group is not here: "Move to Group" in the multi-select menu is how the group changes, and it carries the "delete the group
once it is empty" rule — a second entry point here would drift from it sooner or later. Before "Apply" is pressed, the line at the
bottom already says something like "2 connections will change; 1 keeps its authentication (FTP and plugin protocols only support
passwords)" — with FTP mixed in, switching to key authentication should not come as a surprise afterwards. "Apply" is disabled
when nothing is ticked, the port is out of range, the shared credential option has none picked, or the key option has no file.
After applying, the explorer refreshes and a toast says how many changed (warning level when some were skipped).

Tabs and file rows likewise have their own context menus (see the corresponding sections).

---

## 13. New Connection and Password Verification Flow

### 13.1 New Connection Dialog (design “New Connection v2”, 8 frames; card 760×664)

Since 2026-09-30 the dialog is a “protocol rail on the left + paged form on the right” instead of “one long form + an
‘Advanced options’ fold in the footer”: expanded, the SSH form ran to some thirty rows and could only scroll inside the
768 cap, and the horizontal protocol tabs no longer fit a 500-wide window once a few plugin protocols were installed.

- **Title bar** (28, as in §2): a 15px plug line icon (accent) + the 13px title “New Connection” + a 27×27 × at the top right (close hovers system red, clipped to the card's rounded corner).
  The dialog has a **fixed size** (card 760×664; the XAML says 792×696 in Windows terms, including the margin outside the card),
  so it no longer grows and shrinks as you switch pages; no maximize.
- **Protocol rail on the left (188, `bg-sidebar`)**: a “Built-in” group with `SSH` / `SFTP` / `FTP`, each with a small
  kind label on the right (terminal / files); a “Plugins” group (with its count next to the heading) listing the
  plugin-contributed protocols (`S3`, `Telnet`, `Serial`, `Redis`, …) declared in each plugin's
  `contributes.protocols` — drawn **without loading the plugin assembly**. The manifest does not say what kind a
  protocol is (file / terminal / workspace is only known after activation), so plugin items carry no kind label and
  all use the plug icon. With no plugin protocols the group is not shown. The rail only grows downwards and scrolls on
  its own when it runs out of room.
  - The selected item is `bg-active` with an accent icon and name; hover applies `bg-hover` directly, with no brush
    transition: the rest background is `Transparent` (`#00FFFFFF`, fully transparent *white*), and Avalonia
    interpolates unpremultiplied ARGB per channel, so a fade passes through half-transparent white and flashes bright
    grey on every pointer enter and exit on dark themes (joesdu/VelaShell#577 — nothing to do with display scaling;
    the rule is in the host's [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) §6.1).
    Keyboard focus draws an accent border only under `:focus-visible` — a mouse click never leaves a stuck outline.
  - “Get more protocols…” at the bottom opens the plugin manager (modeless; a protocol enabled there arrives in the
    rail right away through the registry's `Changed` event, no need to reopen the dialog). It only appears when the
    main window supplied the callback.
  - The disabled placeholders are all gone — the last one (Serial) was taken over by the `velashell.serial` plugin;
    the host no longer knows any concrete protocol.
- **Section tabs on the right (40 high)**: which pages exist depends on the protocol —

  | Page | Shown for | Contents |
  |---|---|---|
  | General | always | destination, authentication, the plugin protocol's everyday fields, organize |
  | Terminal | SSH only | run command after authentication + delay, session-level terminal overrides |
  | SSH options | SSH and SFTP (same SSH connection) | compression, allow legacy algorithms, custom algorithm lists |
  | Forwarding | SSH only (interactive shell) | ssh-agent forwarding (and its two restrictions), X11 forwarding |
  | Advanced | FTP, or a plugin protocol that declares `IsAdvanced` fields | FTP default remote path; plugin tuning fields |

  - The selected tab's foreground turns `text-primary`; hover applies directly for the same reason (the rest background
    is `Transparent` here too), and a 2px accent underline slides between tabs (180ms, no motion on first placement).
  - **An accent dot next to a tab = that page has non-default values.** When editing a saved profile, what was filled
    in must not hide behind another tab and look lost — this replaces the old “auto-expand Advanced options on edit”;
    a new connection is all defaults and shows no dots.
  - The Advanced tab shows the **number** of plugin advanced fields (only those that currently apply: a field whose
    visibility condition is false cannot be seen on the page, so it is not counted).
  - Switching protocols falls back to General when the current page no longer exists (SSH's Forwarding page → FTP), and
    stays put when it does (SSH ↔ SFTP on SSH options).
- **The General page** is split into sections, each headed by a 10px `text-tertiary` title followed by a hairline:
  - **Destination**: the Host / Port row, then the jump host (its explanation lives in the ⓘ tooltip next to the label —
    the jump host decides whether you can connect at all, so it stays on General rather than on some other page).
  - **Authentication** (hidden entirely for protocols declaring `NoCredentials`): username + an authentication
    **segmented control** (Password / Key / Certificate / Agent, in `AuthMethod` enum order — all four visible at once,
    replacing the dropdown); FTP puts its encryption mode in that slot, plugin protocols leave it empty.
    Below come FTP's anonymous login / passive mode, the plaintext-FTP warning, the password or certificate + private
    key + passphrase, the Agent explanation, and remember password.
  - **Credential source** (#550, since 2026-10-03; the first row of the Authentication section; hidden for anonymous FTP
    and for protocols without credentials): a dropdown whose first item is “Enter in this connection”, followed by every
    shared credential (“name (username)”); next to it a “New…” outline button that carries the username and password /
    private key already in the form into the shared-credential editor (without the “Connections using this credential”
    list) and selects the new credential once it is created.
    - **With a shared credential selected**: the authentication segmented control, the password / key / certificate fields,
      the Agent explanation and “Remember password” all collapse; a `text-tertiary` summary under the dropdown reads
      “Shared credential: user X · method” and points to Settings → Shared Credentials. **The username stays editable**:
      filled in, it overrides the credential's; left empty, the connection follows the credential, and the placeholder says
      which name will be used (“Leave empty to use admin from the credential”); when the credential carries a username the
      field is no longer required. If the selected credential's username equals the one in the form, the form's copy is
      cleared — otherwise the connection would stay pinned to the old name when the credential is renamed later.
    - **Saving**: the profile stores only the reference (`CredentialSource`), never the authentication material; leftovers in
      the collapsed fields are cleared as well, so “Test” does not connect with an invisible old password.
    - **Two errors block saving**: the referenced credential no longer exists (the dropdown shows and selects
      “(shared credential no longer exists)” so the user sees it and picks again, rather than silently falling back to
      “Enter in this connection”); FTP or a plugin protocol was given a non-password credential (they only take passwords).
    - **Switching back to “Enter in this connection”** copies the credential's username (if the form's is empty), method and
      password / key into the form instead of leaving a row of empty boxes.
  - **Plugin protocol fields**: headed “&lt;protocol&gt; settings”; only fields without `IsAdvanced` are drawn here.
  - **Organize**: display name / session group / tags side by side, with a full-width **Notes** row below
    (#549, `SessionProfile.Notes`).
    - A multi-line text box: Enter starts a new line (the dialog has no default button, so Enter never saves by
      accident), Tab still moves to the next control; at most about five lines tall, longer text scrolls inside the
      box instead of stretching the page.
    - Every connection type has it. Saving trims leading/trailing blank lines and spaces and stores whitespace-only
      text as null; line breaks inside the text are kept as typed.
    - Stored as plain text (it does not go through `ISecretProtector`) and uploaded as-is by cloud sync when no
      end-to-end passphrase is set — the hint under the box says so; passwords belong in Authentication.
    - Shown when you hover the connection in the explorer (§4).
  - Only the form area **scrolls** (with the app-wide two-state scrollbar: a 2px sliver while idle, never pinned open);
    title bar, rail, tabs, feedback strip, and footer stay put. Long forms (S3, Redis, certificate auth) scroll within the
    page; on a short screen the window height is clamped to `min(768 design cap, screen working area − 48)`
    (`ApplyScreenBounds`; 768 matches the 948×768 settings window), so the footer buttons always stay reachable.
  - Plugin fields marked `IsAdvanced` live on the Advanced page and General only draws the everyday ones;
    when editing a saved profile whose advanced fields differ from their declared defaults, the Advanced tab shows the dot.
  - **Terminal page (SSH only): “Run command after authentication” + a delay (0–60 s)**:
    - It is **per profile** (`SessionProfile.PostAuthCommand` / `PostAuthCommandDelaySeconds`), and is a
      different thing from the “Run command after connect” under Settings → Terminal → Session, which applies to
      **every** terminal: what you run after logging in differs per machine (`sudo su -` on a bastion, `tmux attach`
      on a dev box), so a single global box forces you to pick one. When both are set, the **global one runs first,
      then the per-profile one** — the same order the user sees across the two screens.
    - **Shown for SSH only**: the command is injected into a shell channel, and SFTP / FTP / object storage have no
      terminal at all. Switching to those protocols hides the row and saves the value as null — keeping it would store
      a command that never runs, and that then fires once out of nowhere when you switch back to SSH.
    - **Why a delay**: PTY input is kernel-buffered, so waiting for a prompt is not required; the delay exists because
      the peer keeps writing to the terminal after login (motd scripts, corporate login banners, banners that also
      consume stdin), and an immediate injection gets buried or swallowed by that output. `0` = send as soon as the
      handshake completes. The 60 s ceiling is **clamped in the model's setter**, not only in the UI — the config file
      can be hand-edited, and a `99999` would make the command look like it never runs.
    - **Runs on reconnect too** (in-place reconnect reuses the same tab), because it describes “what to do every time
      I log into this machine”. When the delay is > 0 it is sent via `DispatcherTimer.RunOnce` rather than blocking the
      handshake, and the callback re-checks identity by session id + connection status — within those seconds the tab
      may have disconnected, been closed, or reconnected as a different session.
  - **Session-level terminal items on the Terminal page (SSH only)**: encoding, terminal type, color scheme, tab
    color (color picker, see §14.3; no color = automatic, with a one-click reset at the bottom of the flyout), startup directory, keep-alive interval, and **anti-idle (s)**. There can only be one global setting, but
    machines are not alike — the bastion host speaks GBK while the dev box speaks UTF-8, and production tabs should be
    red at a glance. Those differences follow the machine; folded into one global switch they become an either/or.
    Every item except anti-idle means “empty / `-1` = follow the global setting”.
    - **Anti-idle and keep-alive do not guard the same thing; they complement each other and cannot substitute for one
      another**: keep-alive is the SSH **protocol-level** heartbeat (`KeepAliveSeconds` →
      `SshConnectionOptions.KeepAlive`) and guards against NAT and firewalls quietly reclaiming an idle TCP
      connection; anti-idle guards against the **server** dropping you for being idle (bash's `TMOUT`, a bastion host's
      session timeout), and those all count “has anyone typed into the tty” — a protocol heartbeat is invisible to them.
      With two “seconds” fields side by side, the hint text has to nail that distinction, or users will simply assume
      they set the wrong one.
    - **It sends `NUL` (0x00), not a space**: a space is taken as user input and stays on the command line, and
      full-screen programs (vim / less / htop) swallow it as a keypress — you come back after an hour to thirty spaces
      in front of your command, or to less already scrolled to the end of the file. `NUL` is discarded by the line
      discipline yet still refreshes the tty's read activity: there was input, and nothing happened.
    - **Per session only, with no global switch** — unlike every other item in the group, its `null` means “off”, not
      “follow the global setting”: only a specific few machines kick you for idling, and the injected byte does end up
      in the peer's tty, so spraying it over every session only makes the rest carry a risk they never asked for. `0`
      in the dialog = off, and it is normalized back to `null` on save so that a profile which never touched this item
      is not left with a false “I set it, and I set it to off” trace. The 3600 s ceiling is **clamped in the model's
      setter**, the same discipline as keep-alive.
    - **It only fires when the session is genuinely idle**: the clock runs from the **last outbound write** (keystrokes,
      programmatic injection, and its own byte), not from the last injection. When the timer wakes up and finds the last
      keystroke too recent, it moves the alarm to the moment a full interval will have passed and goes back to sleep —
      so while the user is typing, not one extra byte goes out.
    - **Nothing is sent during a ZMODEM session**: that stream carries protocol frames, and one stray byte means a CRC
      error and a retransmit at best, a failed transfer at worst — the same reason keystrokes are held back during a
      transfer; and a transfer is traffic in itself, so the server does not consider the session idle anyway. The
      withheld injection is not dropped: it goes out as soon as sending is possible again.
    - **Applies on reconnect too** (the same discipline as “Post-authentication command”): it is set from the session
      profile when the transport is attached, so both the first connection and every reconnect get it — missing the
      reconnect path shows up as “it starts kicking me again after a dropped connection”, and nobody connects that to
      reconnecting. Changing the profile takes effect on the **next connection**, the same as the keep-alive field next
      to it; a local terminal has no session profile, so not a single byte is ever sent there.
  - **The SSH options and Forwarding pages** (`SessionProfile.Ssh` → `SshSessionOptions`): compression, ssh-agent
    forwarding and X11 forwarding, **all off by default** (same as OpenSSH); plus the two algorithm negotiation options
    (allow legacy algorithms, custom algorithm lists — since 2026-09-29, see below). Each has one line of muted text
    saying *when* to turn it on — getting compression wrong only costs CPU, getting a forwarding wrong lends part of
    this machine to the remote side. “Allow legacy algorithms” and “ssh-agent forwarding” each carry a warning-colored
    outline tag next to their title (“Insecure” / “Trusted hosts only”). With everything at its default the whole
    object is stored as `null`, so old profiles need no migration.
    - **Visibility**: compression and the two algorithm options are on the SSH options page and show for both SSH and
      SFTP (they ride the same SSH connection); the two forwardings are on the Forwarding page, hang off the interactive
      shell, so they **show for SSH only** and are saved as false on SFTP profiles — the same reasoning as
      “Post-authentication command”.
    - **Compression (high-latency / low-bandwidth links)**: recommended when latency is high (cross-border or
      intercontinental links, round trips above ~100 ms) or bandwidth is scarce — terminal output and text files
      shrink a lot. Not recommended on a LAN or when mostly moving already-compressed data (images, archives): it only
      costs CPU. Negotiates `zlib@openssh.com` (compression starts after authentication) and **never offers** legacy
      `zlib` (compresses before authentication — the door for CRIME-style attacks); `none` always stays in the list
      after it, so a server with `Compression no` simply falls back to no compression instead of failing. Each hop of
      a jump chain negotiates with its own profile.
    - **ssh-agent forwarding (`-A`)**: lets you ssh onward from this server with the keys in your local agent, without
      copying private keys to it. The muted text states the cost: while the session is open, root on that server can
      use your agent to sign — only enable it for servers you trust. It is requested only for the interactive shell of
      the final hop (jump hosts open tunnels, not shells). When it is on, two tightening options appear, indented:
      - **Only forward the selected keys** (`SshSessionOptions.AgentForwardKeys`, stored as OpenSSH public key lines;
        `null` = the whole agent is visible): when checked, a candidate list appears, one row per key with name, type,
        `SHA256:` fingerprint and source. Candidates are merged and de-duplicated by fingerprint from three places: keys
        currently in the local agent, public keys under `~/.ssh`, and keys already saved in this connection — the last
        are listed and kept checked even if they are no longer found elsewhere, so opening and re-saving never silently
        drops one. An empty list (agent not running, no `.pub` under `~/.ssh`) gets a warning line saying how to fix it.
        Keys that are not selected are simply invisible to `ssh-add -l` on the server.
        - **Checked with nothing selected cannot be saved** (error in the footer): that would be a connection that
          never forwards while the user believes it does.
        - **At run time, if none of the saved keys can be parsed, nothing is forwarded** and a warning line is written at
          the top of the terminal — it never falls back to "whole agent visible" (the SSH library treats an empty
          `AllowedKeys` as unrestricted, which would turn the strictest setting into the widest).
      - **Ask before every signature** (`AgentForwardConfirm`): every time the server uses the agent to sign, a modal
        confirmation dialog appears (see below).
      - The muted line written once the shell is open includes both, e.g. "ssh-agent forwarding on · only 1 key(s) ·
        every signature needs your approval".
      - For SFTP connections, and when forwarding itself is off, neither option is saved.
    - **Agent signature dialog** (`AgentSignPromptView`, same shell as the host key dialog): the warning banner says "A
      remote session wants to sign with a key in your local ssh-agent", and the advice names the legitimate cause (you
      are running ssh / git on that server) and when to deny (you did nothing of the sort). The info block lists the
      **session** (`user@host:port`), **purpose**, **destination** (sign-ins only), key type, fingerprint and the agent
      comment (usually the private key path); a footer line reads "If you do not answer within 60 seconds, the request is
      denied."
      - **Purpose** and **destination** are what lets the user tell "the `git pull` I just ran" from "someone using this key
        to sign in elsewhere". They come from the context the SSH library recognizes in the data to be signed
        ([spec/07 §7.2.1](../ssh/spec/07-forwarding.md#721-signature-confirmation-must-say-what-the-signature-is-for)): the
        purpose reads 'Sign in to an SSH server as "git"', "Data signature (namespace git)" or "Unrecognized data — not an
        SSH sign-in"; the destination shows the matching known-host names (with the port when it is not 22), "Not in your
        known hosts (fingerprint)" when it does not match, or "Cannot be verified — the remote side did not prove which
        host it is signing in to", **the last two in the warning color**. If the known hosts cannot be read, the names are
        simply missing and the dialog still opens.
      - Three buttons: **Deny** (outline) / **Allow for this session** (outline; until the session closes, this key is
        not asked about again **for the same purpose** — the same user signing in to the same destination, or a data
        signature in the same namespace; approving a `git pull` to github does not approve using the key to sign in
        elsewhere) / **Allow once** (accent pill).
      - **Deny is both the default and the cancel button, and has focus when the dialog opens**: the dialog can pop up
        while the user is typing in the terminal, so a stray Enter or Esc can only land on Deny; no "allow" button is
        ever the default.
      - **Fail-closed**: no answer within 60 seconds, the channel closing, no main window or a dialog error all deny,
        and a dialog that is still open is closed on the spot; an implementation that returns "allow" after the deadline
        is still denied. The remote ssh sees "agent refused to sign".
      - When several sessions ask at once the dialogs **queue and appear one at a time** instead of stacking modals —
        otherwise you could not tell which dialog belongs to which session. The deadline keeps running while queued.
    - **X11 forwarding**: shows remote GUI windows on the local X server. The host **has a built-in X server** (the
      X Server title-bar button and the “X Server” page in §14) that works out of the box on every platform; on Windows
      it can use the VcXsrv the user installed instead, and Xming / X410 users keep bringing their own (no third-party
      X server binaries are bundled; see “Won't do” in `feature-plan.md`). Turning it on reveals two more fields, laid
      out in two rows: the “Local X display” label on a row of its own, then the input and the “Trusted (`-Y`)” checkbox
      side by side on the second row, each vertically centred:
      - **Local X display**: when empty, tried in order: ① the local X server managed by VelaShell (its display if it
        is running; if not, and “Start automatically for X11 forwarding” is on, it is started first — unless another X
        server is already in use (on Windows a process in the current user's session listens on `localhost:0` — another user's on a
        terminal server does not count; elsewhere `DISPLAY` is set) or VcXsrv
        is selected but not installed, in which case it stays out of the way silently) ② the `DISPLAY` environment
        variable ③ `localhost:0.0` (almost nobody sets `DISPLAY` on Windows, and those X servers listen on TCP 6000 by
        default). The placeholder shows the last two. A failed auto-start only writes a yellow line and never blocks the
        shell. **A filled-in value is used as is**; the local X server stays out of it. When the display comes from the
        built-in X server and the mode is trusted, x11 channels **go straight into the in-process server** without a
        local port, and the session's `user@host:port` is handed to the server as the connection label (the log, the client list
        and "the clipboard follows the focused session" all recognise sessions by it). If the built-in engine is stopped later
        (say, after switching to VcXsrv), X programs started from this session go over local TCP to whichever X server is running now
        (on Windows only; Linux / macOS have no other engine, so there is no fallback). On Linux / macOS the built-in X server does not
        open the TCP port by default, so **entering** `localhost:N` here fails — leave it empty (through the connector) or enter `:N`.
      - **Trusted (`-Y`)**, **checked by default**. Unticked means untrusted (like `ssh -X`): remote programs can only touch the
        windows of untrusted programs — they cannot capture other programs, read their keyboard input or fake input. The built-in X
        server supports both (since 2026-10-10; an untrusted channel goes through the connector straight in as an untrusted client,
        without running `xauth`); with an external X server, untrusted mode needs its SECURITY extension and a local `xauth`. The
        tooltip explains what each mode means, and that a few old programs do not work correctly when untrusted. (Before, the
        built-in X server had no SECURITY extension: unticked meant no forwarding, with a yellow line at the top of the terminal.)
      - Forwarding **lasts for the whole session** with no expiry — “new windows stop opening after half an hour” only
        makes people think forwarding is broken.
    - **A refusal does not take the session down**: `X11Forwarding no`, a missing xauth, no local agent running and
      `AllowAgentForwarding no` are all common, and failing the whole session over an add-on would be backwards. Both X11
      and agent forwarding are requested as “skip it if it can't be set up” (the SSH library's
      `ForwardFailureMode.Continue`): when setup fails the library does not throw, the shell opens in one go, and the
      reason comes back on `SshShell.X11SetupFailure` / `SshShell.AgentSetupFailure`; each gets a **yellow** line at the
      top of the terminal saying why (when no local agent is running, a localized hint is chosen by reason code, and on
      Windows it names the service to start); on success a **grey** line
      (“X11 forwarding on → local display localhost:0.0”; on Linux / macOS the built-in X server's display looks like `:0`) in the same style as the jump-chain line. Both the first
      connection and in-place reconnects write it. 〔History〕Agent forwarding used to be dropped after a refusal, and
      the whole shell reopened.
    - **Allow legacy algorithms (for old devices)** (`SshSessionOptions.LegacyAlgorithms`, off by default): old network
      gear and old systems are often down to `diffie-hellman-group14-sha1`, SHA-1 `ssh-rsa` and `hmac-sha1`, and the
      default lists cannot agree with them. Turning it on **appends these three after the defaults** — a server that
      also speaks the newer ones still gets those first; the muted text names exactly these three. CBC and the like are
      not implemented in this version, so a device that only speaks CBC still cannot connect.
    - **Custom algorithm lists** (four inputs when expanded: key exchange `KexAlgorithms`, host key `HostKeyAlgorithms`,
      cipher `Ciphers`, MAC `Macs`; monospace, placeholder “default”): written the OpenSSH `ssh_config` way — `+a,b`
      appends after the defaults, `-a,b` removes from them (`*` / `?` wildcards allowed), `^a,b` moves to the front, and a
      list without a prefix replaces the defaults. “Defaults” means the list after legacy algorithms are applied, so a
      line copied from `~/.ssh/config` works as is. Hovering an input shows the current default list and what else can
      be added (it refreshes with the legacy toggle).
      - Which names are accepted follows what the SSH library actually implements. An unknown name is reported as
        unknown; a name OpenSSH knows but this version does not implement (CBC, 3des, group1, group-exchange-sha1, ssh-dss,
        hmac-md5, umac…) is reported as “not implemented in this version” — CBC is the most common thing in a copied
        config, and the user needs to hear “enabling it will not help”, not “you misspelled it”. Removal entries must be
        known names too (a misspelled removal removes nothing while the user believes it is off); removing everything, or
        writing only a prefix, is reported on the spot.
      - An invalid list shows an error below the field naming the kind and the name, and greys out Save / Connect /
        Test; switching to a protocol that does not use SSH, such as FTP, drops the check. Collapsed lists are not saved,
        for the same reason the display address is not saved while X11 is off.
    - **A negotiation failure gets one more line saying what to do next**: when the server offers algorithms this
      version implements but has not enabled, the message names them and points here (“Allow legacy algorithms” or the
      custom lists); when there are none, it says plainly that enabling more will not help (typically an old device
      that only speaks CBC).
  - **Authentication method “Agent”** (`AuthMethod.Agent`, the last segment of the segmented control — the enum is persisted by
    ordinal, so new values can only be appended): signs with the keys in the local ssh-agent and **stores no
    credentials** in the profile; selecting it collapses the password and key fields, leaving one muted line saying
    where the agent comes from. On Windows that is the named pipe of the “OpenSSH Authentication Agent” service;
    `SSH_AUTH_SOCK` is honored only when it is itself a `\\.\pipe\…` path (1Password and KeePassXC set it that way) and
    ignored when it points at a Git Bash / WSL Unix socket — that is a different agent and cannot be reached.
    - Every key in the agent is tried in turn (probe first, then sign, so the agent's confirmation prompt is never
      triggered for a key the server would not accept anyway).
    - If the agent is not running, the error comes within **3 seconds** (“Cannot reach the local ssh-agent”, with a
      `Start-Service ssh-agent` hint) — a named-pipe connect without a timeout keeps waiting for the pipe to appear, and
      all the user would see is a spinner until the whole connection times out. An agent with no keys gets its own
      message (“load one first with ssh-add”) instead of a generic “authentication methods exhausted”.
    - **The agent is only touched when this method is explicitly chosen**; no other method falls back to it implicitly.
  - **FTP's Advanced page: “Default remote path”** (`FtpSettings.InitialRemotePath`, shown only
    when `FTP` is selected in the rail). After connecting, the remote pane opens that directory instead of the login working directory —
    the upload target is the same `/var/www/html` or `/pub/incoming` year after year, while the login directory an FTP
    server hands you is often just the root, so clicking down four or five levels on every connection is pure busywork.
    - **It is the first candidate, not a hard requirement**: if it cannot be opened (typo, directory removed, account
      chrooted) it falls back to the login working directory and then to the root — the same discipline as falling back
      to the root when the home directory cannot be opened. A mistyped path must not strand the user on a blank error page.
    - **Normalization lives in the `FtpSettings` setter** (trim, backslashes to forward slashes, add the leading `/`,
      strip the trailing `/`, empty and bare `/` become null), so the dialog, the importers, and a hand-edited config
      file all share one rule: users type `\pub` out of Windows habit and paste paths with trailing slashes, and FTP's
      `CWD` is not uniformly forgiving about either.
    - It lives on `FtpSettings` rather than as a flat `SessionProfile` field: the latter is copied field by field in four
      places across the repo, so every new flat field costs four edits, whereas protocol-specific settings go into their
      own nullable nested object and only need the matching `Clone()` update. SSH / SFTP still start at the login home
      directory, and plugin protocols (S3…) declare their own fields in their descriptor.
- **Feedback strip**: the result of Test / Save (one accent line on success; on failure the error text + a copy button)
  is pinned above the footer, outside the scrolling form — when the user presses Test their eyes are on the footer, and
  the answer must not land somewhere scrolled out of view.
- **Footer**: left = a destination preview (monospace 11px `text-tertiary`, after the protocol icon): `user@host:port`
  for SSH, `sftp://` / `ftp://` / `ftps://` for SFTP / FTP (explicit and implicit FTPS both count as `ftps`), IPv6
  literals in brackets, no user for anonymous FTP or credential-less plugin protocols, no port for protocols declaring
  `NoEndpoint`; hidden while the host is empty. It updates as you type, so you can check at a glance where you are about
  to connect. Right = `Test` / `Save` (outline) / `Connect` (accent pill, DESIGN.md §5.1), all 28 high.

### 13.2 Password Verification Dialog (Two Steps)

- **Step 1 (`oNZIM`, 420px)**: header title + session information bar (`bg-input`: host icon + `user@host`) + password input with visibility toggle + “Remember password” checkbox + `Cancel`/`Connect` footer.
- **Step 2 (`twD13`)**: second-factor verification (2FA / OTP / key passphrase / host fingerprint verification). Information bar + input + `Previous`/`Verify`.
- **One-time code dialog (keyboard-interactive, since 2026-09-29, `KeyboardInteractivePromptDialog`)**: shown when a
  two-step verification server that only allows keyboard-interactive (PAM + TOTP, Duo, bastion MFA) asks for a code.
  - **When it appears**: a single non-echoed prompt that looks like a password prompt (the word "password" in English,
    Chinese, Japanese or Korean, and nothing like verification code / OTP / token — those usually want "password + code"
    typed together) is **answered once** with the saved password, without a dialog; if the password is asked again (a
    password-change flow, or the previous one was wrong) the user answers. Everything else opens the dialog. An
    information-only round (the server sends instructions but no prompts) does not open it; its text is carried into
    the next dialog.
  - **Private key / certificate / SSH Agent never fall back to the password**: a plain password prompt is answered with
    an empty string so the server rejects it, and only code-style prompts open the dialog — that is the second step of
    `AuthenticationMethods publickey,keyboard-interactive` (key + code); a password box popping up after the key was
    rejected would contradict "only the authentication method the user chose".
  - **What it looks like**: the title is the name the server sends, or “Two-step verification” when there is none; the
    first line is the connection target (`user@host:port`, so you can tell hosts apart when several sessions want a code
    at once), then the server's instructions and one input per prompt, masked when `echo = false`; the first input has
    focus when it opens. Server text has control characters and bidirectional-text controls removed, line breaks
    normalized and lengths capped before it reaches the screen. Only one dialog is shown at a time.
  - **Cancel = do not connect**: no error, and no credential dialog afterwards. On a first connection the tab is
    removed; on a reconnect it counts as a user disconnect (otherwise auto-reconnect would bring the same dialog back a
    few seconds later); an SFTP document connection removes its placeholder. Closing the connecting tab or an
    authentication timeout closes the dialog on the spot and is handled as the usual cancel / timeout.
  - **"Change password" dialog (since 2026-10-05)**: when the server requires a password change during password
    authentication (an expired password, `PASSWD_CHANGEREQ`), the same dialog asks for it: the title is “Change
    password”, the instructions are “The server requires the password to be changed before you sign in. A saved password
    is not updated automatically.” plus the server's own words (sanitized), and there are two masked inputs, “New
    password” and “Confirm new password”. Entries that differ or are empty are not handed over; the dialog asks again
    with “The two entries do not match, or are empty” — the protocol has no confirmation step, so a mistyped one would
    become the account's password as is. When the server rejects the previous new password the instructions change to
    “Choose a different one” (at most 3 attempts). Cancel works as above and reports “Password change was cancelled”.
    Once changed, the password saved with the connection is stale: the next rejection goes through the credential dialog
    to re-enter and save it. Private-key / certificate / agent authentication never shows this dialog; without a dialog
    service the password is not changed and the error says the server requires a change and it was not changed.
- **Connections that use a shared credential (#550, since 2026-10-03)**: they **never prompt up front** — the username and
  authentication material come from the credential. Only when the credential cannot be resolved (deleted, no password on
  this device) or the server rejects it does the flow fall back to the credential dialog, with a notice under the
  information bar (`VelaBgActive` background + `triangle-alert` icon + `text-secondary` text):
  - **Rejected**: “Signing in with the shared credential “X” was rejected. If the password was just changed…”;
    **incomplete**: the credential lacks a username, password or key on this device (what a credential pulled by cloud sync
    without an end-to-end passphrase looks like); **missing**: what is entered is used for this sign-in only — to keep it,
    edit the connection and choose another source. When the user's own one-off entry is rejected again, no reason is repeated.
  - Step 2 gains a checkbox “Also update the shared credential “X” (used by N connections)”: ticked, the entry is saved back
    to the credential and every connection referencing it picks it up on its next connect — the one step that turns
    “change the server password” into a single entry. **Ticked by default when incomplete, unticked when rejected** (it may
    be just this one host with a different password, and a wrong tick breaks the other few dozen); absent when the
    credential no longer exists.
  - “Remember password” and the AES badge are hidden: such a connection stores no password of its own; to keep one, save it
    back to the credential.
  - The username is prefilled with the connection's own, or else the credential's; for a connection that follows the
    credential's username, an entry equal to the credential's (or one saved back to it) is not written onto the connection —
    otherwise saving the profile after connecting would silently turn “follow the credential” into an override.
  - **The dialog returns a copy** (for every connection, not only these): the instance passed in is the one cached by the
    session tree or the command palette, and editing it in place let a password the user chose not to remember be persisted
    by later saves such as moving the session to another group or duplicating it. “Reconnect” on a failure card uses the
    copy from the latest attempt, so a retry does not ask again.
  - A jump host whose credential cannot be resolved does **not** prompt (the dialog asks for the target's credentials); the
    connection ends with a “Jump host X: reason” error.
- **Host fingerprint confirmation**: on the first connection to an unknown host, or when a recorded fingerprint changes, show a “Host Trust” confirmation (fingerprint + Accept and Save/Trust once/Reject; a change also shows the recorded fingerprint from known_hosts alongside), linked to §15 Host Trust Center. A change raises this dialog by default rather than being refused outright (#476); strict fail-closed behaviour is the “Block and alert on fingerprint change” switch on the Security Audit page.
  **Each key type of a host is recorded separately** (servers often have RSA, ECDSA and Ed25519 keys at once): a key matching any recorded one is accepted; accepting a key of a new type **adds a record** instead of overwriting the existing one,
  so whichever type is negotiated later is recognised; only a different key of the same type replaces it. The change dialog shows the recorded fingerprint of the same type, or the most recently seen one when that type was never recorded.
  When connecting, the recorded types go to the front of the host key algorithm list, so a normal server negotiates the type already recorded.
  **Host key rotation is on** (OpenSSH's `UpdateHostKeys` is on by default too): after connecting to a host that is already permanently trusted, the other host keys the server proves it holds are recorded as trusted, per type,
  and a security alert is raised ("now trusted") —— after an administrator adds an Ed25519 key or rotates out an old RSA key, the user no longer sees "fingerprint changed". Old keys the server no longer presents are removed from the trust store by type, with a security alert as well ("removed from the trusted records") —— a type is removed only when every record of that type matches a fingerprint to be forgotten; nothing is done for "trust once". The “Trusted hosts” list has one row per key type, and removing a row removes that type.
  〔History〕Only one key per host:port used to be recorded: a server with an additional key of another type, or a changed host key algorithm for the connection, raised “fingerprint changed”; accepting it overwrote the old key, and switching back raised it again.
  If “Accept and Save” is chosen but the trust store cannot be written, **this connection goes ahead**, the fingerprint is not asked about again for the rest of this run, and a security alert is raised (the audit log records “Host fingerprint could not be saved”) —
  without it, the user would be asked again after a restart and would not know why.

**Connection state machine**: `Idle → Connecting (yellow) → Authenticating → Connected (green)` / any failure `→ Disconnected (red)` with reason and retry.

---

## 14. Settings Panel (implemented as an 840×740 dialog, 200px left navigation + right content)

The left side contains navigation sections and the right side shows the corresponding content. **The current implementation has 14 pages** (Shared Credentials, Proxy, X Server, Cloud Sync and Support and Donations were added beyond the design; see the itemized remediation ledger in [settings-audit.md](settings-audit.md)):

| Page | Frame | Implementation status (2026-07-12) |
|---|---|---|
| General | `2BIRD` | Startup/tray/language/connection defaults/session logs/import/export/behavior and automatic reconnect; the unimplemented “Updates” and “Master Password” groups are hidden |
| Appearance | `ZAbb9` | Theme (twelve named themes + follow system, live preview), accent color (follows the theme out of the box: “Follow theme” in front of the swatches = no override, use the current theme's own accent — since 2026-09-29; the previous factory pink `#E91E63` covered every theme's accent, and saved configurations are not migrated; any other color comes from the color picker, see 14.3), UI font/size, opacity, terminal colors (color picker, offering the scheme's 16 ANSI colors as swatches) and color schemes (defaults to the active theme's paired scheme, see below) |
| Terminal | `08FpM` | Font/line height/TERM/encoding/cursor/three-state bell/scrolling/copy and paste/IME/commands run after connection |
| Key Management | `UBP59` | Enumerates `~/.ssh` (type + SHA256 fingerprint), generates Ed25519 / ECDSA / RSA keys (OpenSSH format, generated and written by the SSH library), imports/deletes/copies public keys, default authentication key; an “Add keys to the agent automatically” toggle in the “SSH Agent” section (off by default): after a successful private-key-file authentication the key is added to the local ssh-agent in the background, skipped if the agent already holds it; an absent or refusing agent is only logged and never affects the connection; certificate authentication does not trigger it |
| Shared Credentials | — (new, 2026-10-03, #550) | One set of username + password / private key / certificate / Agent shared by many connections; change it here once and every connection referencing it uses the new value on its next connect. Toolbar: search (name, username, notes) + “New Credential”; table columns: name (with a warning icon when this device has no password or key for it) / username (“From connection” when it carries none) / method / in use (N connections) / actions (edit, delete). **Editor** (`SharedCredentialEditorView`): a “Copy from a connection” dropdown, name, username (optional), the method segmented control and its fields, notes; below, a checklist “Connections using this credential” (filterable; connections the current method cannot serve are greyed out — FTP and plugin protocols only take passwords) and “Select matching connections” (ticks the connections that still store exactly the same username and material on their own, for moving existing connections over). Saving takes effect at once, without the settings window's Save: newly ticked connections switch to it (a username equal to the credential's is cleared so it follows the credential), unticked ones get the credential copied back onto themselves. **Delete** asks first; every connection using the credential keeps its own copy of it and still connects afterwards. Footer note: passwords and key passphrases are stored AES-256-encrypted on this device; cloud sync carries names and usernames, and passwords and passphrases only with an end-to-end passphrase |
| Keyboard Shortcuts | `YQvri` | Every key binding; the global and tab bindings can be changed, unbound or reset (2026-10, joesdu/VelaShell#551; rules under “Customizing shortcuts” in [Keyboard Shortcuts](keyboard-shortcuts.md)). Entries are checked one by one against actual bindings |
| File Transfer | `HGwa7` | Paths/editor/concurrency/conflict policy/hidden files/bandwidth limits/transfer logs (conditionally visible); unimplemented resume features are hidden |
| Security Audit | `glqQE` | Session recording toggle + playback center entry, host trust policy, trusted host management (address redaction), alert channels (in-app/sound/Webhook), audit log (“View” opens the audit log window, see §15; retention days, default 180) |
| Proxy | — (new) | One proxy shared by every outbound connection: none / follow system (default) / HTTP / SOCKS5, plus “resolve DNS through the proxy” |
| X Server | — (new, 2026-09-23) | The local X server. The first section is the **Engine**: built-in (default, ships with the app, every platform) or VcXsrv (installed by the user on Windows; **not bundled**); the section only appears on Windows, other platforms only have the built-in one. With built-in selected a note below explains that each X window is a native window and SSH X11 forwarding connects straight into the built-in server without a local port; how local programs connect is written per platform (`DISPLAY=localhost:N` on Windows, `DISPLAY=:N` on Linux / macOS — the Unix socket, with TCP off by default). Shared by both: **Display** (display number: auto or :0–:15, with the description likewise per platform; window mode: multiple windows / one large window / no title bar / fullscreen / rootless, and on the built-in engine one large window / no title bar / fullscreen start it in one-window mode) / **Clipboard** (enable — the built-in engine exchanges only with the session that has the keyboard focus, as the description says; copy on selection — off by default, with the description saying that when on, whatever is copied locally can be pasted into X programs with a middle click) / **Keyboard layout** (auto = follow the system's current layout, or pick one of 27 common XKB layouts) / **Startup** (start with VelaShell, start automatically for X11 forwarding). Shown only with the built-in engine: in the "Display" section below window mode, **Transparent X windows** (since 2026-10-10, off by default; the description says windows with alpha are shown opaque while it is off and that it has no effect in the single-window modes) and **Show where X windows come from** (since 2026-10-10, on by default, **applies immediately**; titles, taskbar entries and tray tooltips of windows forwarded over SSH start with `user@host:port`, and the description says windows that draw their own title bar show it only in the taskbar); in the "Keyboard" section below the layout, **Use this computer's input method in X windows** (since 2026-10-10, on by default); and a **Security** section (its own section since 2026-10-10; these used to sit under "Keyboard"): **Restrict programs from SSH sessions** (off by default; when on, programs forwarded over SSH cannot simulate input, listen to every key typed in X windows or change input devices, and the description names the cost: tools such as xdotool on those servers stop working), **One display per SSH session** (since 2026-10-10, off by default; when on, X programs from different SSH sessions get separate displays — they cannot see each other's windows, read each other's keyboard input or clipboard, or send each other input — while X programs on this computer keep using the shared display; the description names the cost of a little memory and a thread per session) and **Also listen on a TCP port** (since 2026-10-10, Linux / macOS only, off by default; the port listens on 127.0.0.1 only, and the description says containers on a bridge network cannot reach it and should mount `/tmp/.X11-unix` on Linux). VcXsrv-only, shown only when VcXsrv is selected: **Program** (VcXsrv location, empty = auto-detect; offers `winget install marha.VcXsrv` when not found), **Keyboard** (model, capture special Windows keys), **Extensions** (native OpenGL, disable access control), **Advanced** (additional arguments + a “Help” dialog documenting every VcXsrv command-line argument in five languages, filterable; the command line for the next start is previewed live below), show tray icon. With the keyboard layout on “auto” the built-in engine follows the system's current layout (Windows, macOS, and Linux with a desktop X display; including the AltGr level, which is the Option level on macOS); a chosen layout uses the keymap tables shipped with the app. Changes apply the next time the X server starts (except "Show where X windows come from") |
| Snippets | `HBNhv` | Common command snippet library with collapsible groups (shared by command palette/completion suggestions through SonnetDB `quick_commands/commands` v2); supports adding, editing, deleting, and changing group assignment, built-in commands included (with restore to default); groups and commands can be reordered by dragging; command text may contain variable placeholders (see §14.2) and span several lines (see §14.4) |
| Cloud Sync | — (new) | GitHub Gist multi-device sync: token/Gist binding, end-to-end encryption passphrase, sync scope, version history, and restore (see plan.md §13.C) |
| About | `Nwoks` | Version/runtime environment/dependencies/contributors (clickable GitHub avatars)/dual licensing and authenticity statement; Check for updates: Linux opens the releases page for manual download; Windows / macOS download into writable installations and offer a restart; Microsoft Store manages updates for the Store build (see [Message center and feed](message-center-and-feed.md)) |
| Support and Donations | — (new) | Alipay/WeChat/Wise donation and contribution guidance |

Common interaction: click on the left to switch; scroll on the right; Appearance provides live preview (save persists changes, cancel rolls them back); “Restore Defaults” and “Clear History” show confirmation dialogs; `Ctrl+,` opens Settings.

### 14.1 Named themes (2026-08-31)

A theme is no longer a dark/light switch — it is a complete palette. Twelve ship in the box — seven dark, five light — plus a
“follow system” pseudo-theme that resolves to VelaDark / VelaLight by OS appearance. Every theme
carries a **paired terminal color scheme** whose background equals the UI's terminal-canvas color:
the terminal and the chrome around it are one plane, and a mismatch shows up as a visible seam.

| Theme | Base | Lineage | Paired terminal scheme |
| --- | --- | --- | --- |
| VelaDark (factory default, id `dark`) | dark | Dracula | Dracula |
| One Dark (`one-dark`) | dark | One Dark | One Dark |
| Tokyo Night (`tokyo-night`) | dark | Tokyo Night | Tokyo Night |
| Nord (`nord`) | dark | Nord | Nord |
| Everforest (`everforest`) | dark | Everforest | Everforest Dark |
| Obsidian (`obsidian`) | dark | neutral near-black (OLED) | Obsidian |
| Gruvbox (`gruvbox`) | dark | Gruvbox | Gruvbox Bright |
| VelaLight (`light`) | light | Alucard | Alucard |
| One Light (`one-light`) | light | One Light | One Light |
| Rosé Pine Dawn (`rose-pine-dawn`) | light | Rosé Pine Dawn | Rosé Pine Dawn |
| GitHub Light (`github-light`) | light | GitHub Light | GitHub Light |
| Sakura (`sakura`) | light | pink, no upstream palette | Sakura |

- The persisted value is the id above; `dark` / `light` keep their historical values, so existing
  configurations need no migration.
- The **first entry** in the terminal color-scheme dropdown is “Follow theme (<paired scheme>)”:
  picking it follows the theme (no color overrides at all, so the terminal changes with the theme).
  Every entry below it pins that scheme; the paired one still carries a “(default)” suffix to show
  where it comes from. Changing the foreground/background/cursor/selection color while following
  (with the color picker, see 14.3) counts as choosing your own colors, and leaves the following state.
- Following is recorded **explicitly** in `Appearance.TerminalColorsFollowTheme`, never re-derived by
  comparing colors against the factory values — that comparison could not tell “picked Dracula” from
  “following the theme”, which made picking Dracula under a non-Dracula theme a no-op.
- Palettes are defined by the host's `UiThemeCatalog` (seed colors) and `ThemeTokenApplier`
  (derivation rules); the token list and rules live in `DESIGN.md` §2 of the VelaShell repository.
- Plugins still see only `dark` / `light` / `system` — named themes are not part of the plugin contract.

### 14.2 Quick command variable placeholders (2026-09-29)

The command text of a quick command (the sidebar quick commands panel, Settings → Snippets) can contain placeholders,
and the user is asked what to fill in before it is sent. Commands that differ by one or two arguments each time —
`kubectl logs -f <pod>`, `journalctl -u <service> -n 200` — no longer have to be saved half-finished.

- **Syntax**: `{{name}}` or `{{name=default}}`. A name used several times is asked once, with the first non-empty
  default; an empty answer takes the default (an empty string when there is none). The quick command data structure is
  unchanged, so cloud sync and import / export keep working.
- **Which double braces are not placeholders**: ops commands are full of `{{…}}` already (`docker inspect -f
  '{{.State.Status}}'`, kubectl go-templates, Ansible's `{{ inventory_hostname }}`), so placeholders are deliberately
  narrow: the name must start with a letter or underscore and consist of letters, digits, underscores and hyphens, and
  **no whitespace is allowed inside the braces** (so anything starting with `.` or containing spaces does not count);
  the argument-less Go template actions `end` / `else` / `break` / `continue` do not count; a default may not contain
  braces or line breaks. Anything that is not a placeholder is sent as is, and a command without placeholders is sent
  straight away with no dialog.
- **The dialog**: one row per variable (name + input pre-filled with the default, read out by the variable name), with
  a live “Will send:” preview of the whole command below; the first input has focus, Enter sends, Esc cancels. Cancel
  sends nothing at all.
- **Line breaks in a value become spaces**: a quick command sends its text without Enter, so a line break inside a
  value would press Enter for the user.
- Target tabs may disconnect while the dialog is open: targets are picked again from the current tabs when sending,
  and only those still connected receive it.
- In Settings → Snippets, the new / edit area has a one-line syntax hint under the command input.
- When terminal command completion offers a quick command as a candidate, it still inserts the raw text with
  placeholders — completion continues what has already been typed, and substituted text would no longer match it.

### 14.3 Color picker (2026-10-01)

Everywhere a user picks a color is a color picker, no longer a text box for typing `#RRGGBB`: the accent color and the
terminal foreground / background / cursor / selection on the Appearance page, and the tab color on the Terminal page of
the new / edit connection dialog (§13.1). The 16 ANSI colors on the Appearance page stay read-only swatches.

- **The field** looks like an input (`bg-input`, 1px border, radius 4, at least 32 high): a 14px chip + the value in
  monospace + a chevron. With no color it shows an empty chip frame and a line of explanation (“Follow theme” for the
  accent, “Automatic” for the tab color) in a dimmer foreground and the UI font. The border turns accent while the
  flyout is open or under keyboard focus.
- **The flyout** (236 wide, global flyout chrome), top to bottom: saturation / value pad (148 high), hue strip, an old |
  new comparison chip (left half = the color when it opened) + hex box, swatches, and a clear button.
  - Pad and strip can be dragged, clicked, or driven by arrow keys: on the pad left / right change saturation and up /
    down change value, 1% per press; the strip moves 1 degree per press, Home / End jump to the ends; Shift makes every
    step ten times larger.
  - The hex box is for precise values (a code copied from a design or another terminal's config): it applies on Enter or
    blur; an unparsable value is **never written** and the box falls back to the current color — terminal colors that do
    not parse would drop the whole set back to factory colors. Only hex is accepted, not color names.
  - Swatches: the terminal colors offer **the current scheme's 16 ANSI colors** (picking from the scheme's own colors
    keeps a changed foreground or selection in harmony with it); everything else offers **the current theme's palette**
    (`VelaAccentPalette0..7`, the same family as the automatic tab colors). The swatch matching the current color gets a
    heavier outline; its tooltip and accessible name are the hex value itself.
  - The clear button only appears where the value may be empty: “Automatic” for the tab color. The accent goes back to
    following through the “Follow theme” button at the start of its row, so the flyout does not repeat it; the terminal
    colors are required and have no clear button.
- **Written back on release**: while dragging only the flyout's preview changes; the setting is written on release, on
  each arrow-key step, on a swatch click, and on Enter or blur in the hex box. Changing the accent re-derives the whole
  token set and changing a terminal color repaints every terminal — writing on every pixel of mouse movement is waste.
- Output is always uppercase `#RRGGBB`, which all three consumers (accent, terminal colors, tab color) accept; on input
  `#RGB` and eight-digit values are also understood, so hand-edited configs still display.
- State is kept as H / S / V: dragging saturation on a gray does not snap the hue back to red, and picking a gray swatch
  does not send the hue strip wandering.

### 14.4 Quick commands: editable built-ins, multiline commands, drag to reorder (2026-10-03, joesdu/VelaShell#555)

Applies to Settings → Snippets and the sidebar quick commands panel (both share one set of data, so the order arranged on
the settings page shows up in the sidebar right away).

**Built-in commands**

- Built-in commands have “Edit” and “Delete” just like the user's own. The built-in catalog itself never changes; edits
  are a layer on top of it: a deleted one is recorded as “no longer shown”, an edited one is stored as a custom command
  with the same id that takes the catalog entry's place. Untouched built-ins still follow the UI language for their
  descriptions, and built-ins added in later versions still appear.
- Deleting an edited built-in deletes it; it does not fall back to the original. Opening the editor and saving without
  any change leaves it an untouched built-in.
- An edited row (moving it to another group counts) gets a “Restore Defaults” button: the catalog original comes back to
  its original group and position.
- “Restore built-in commands” above the list only appears once a built-in has been deleted or edited, and asks first:
  deleted ones come back, edited ones are reverted, the user's own commands are not affected.

**Multiline commands**

- The command input accepts Enter for new lines (wraps, monospace, scrolls inside the box beyond roughly ten lines).
  Line breaks are saved as `\n`, with leading and trailing whitespace trimmed.
- The sidebar shows only the first line followed by “+N lines”; the whole command is in the tooltip.
- A multiline command is sent as a bracketed paste: when the remote side has bracketed paste on (bash 5.1+ by default,
  zsh, fish) the shell puts the whole block on the command line and waits for Enter instead of running it line by line.
  If any target terminal does not have bracketed paste on (sh, older bash, network device CLIs and the like), each line
  would run there as soon as it arrives, so a “Send multiline command” confirmation (previewing the first five lines)
  comes first. It follows the “Confirm multi-line paste” switch under Settings → Terminal; with that switch off there is
  no question.
- A command whose only line break is at the end is not multiline and is still typed in as before.
- Multiline commands are not offered by terminal command completion: completion continues the typed line, and accepting
  a multiline candidate would press Enter for the user.
- Variable placeholders (§14.2) can be combined with multiple lines: the values are asked for first, then the rendered
  block is sent by the rules above.
- A quick command is **not** executed automatically on click (maintainer decision, #555): many commands need their
  arguments adjusted in the terminal first, and pressing Enter yourself is a confirmation in its own right.

**Drag to reorder** (Settings → Snippets)

- Group headers and every row have a six-dot handle on the left; press and drag it, a drag starts after 5px of movement.
  The row / group being dragged fades, and an accent-colored insertion line marks where it will land (a command's line
  starts after the handles, a group's line spans the full width).
- The drop position is the insertion point closest to the pointer: a command can go between any two rows, right under a
  group header (= the top of that group), or into another group; dragging onto the header of a collapsed group puts it
  at the top of that group. Dragging a built-in into another group counts as an edit and can be restored.
- Groups go between groups. “Ungrouped” always stays last: it has no handle, and no group can be placed after it.
- Dropping it back where it was, or releasing more than 40px away from the list, cancels. Dragging to the top or bottom
  edge of the settings page's scroll area scrolls automatically, faster the closer to the edge.
- Reordering is mouse-only for now; there is no keyboard alternative. The sidebar panel has no drag handles (a click
  there sends the command).

**Sync and older versions**: deleted built-ins, each group's command order and the order of the built-in groups travel
with cloud sync as added fields, without a schema bump. A device that has not been updated ignores them; if it pushes
again, deleted built-ins reappear and the order falls back to the default, but none of the user's own commands are lost
(an edited built-in shows next to its original there).

### 14.5 Splash screen (2026-10-10)

**Why**: a measured cold start takes about 8 seconds to the first frame (over 3 seconds of it Avalonia platform initialisation and XAML
loading, mostly Defender's first scan of unsigned assemblies), and nothing is on screen meanwhile, so users think the app failed to open.
Windows only offers a busy cursor that disappears after a few seconds; the Dock bounce on macOS and the launch notification on Linux desktops
already say "opening", so the splash screen is Windows only.

**Behaviour**

- Shown as soon as the process holds the single-instance lock, about 50 ms after `Main` (about 0.2 s after process creation on a warm start).
  A second double-click only brings the existing window forward, without flashing a splash screen.
- Fades out (160 ms) once the main window has painted its first frame. If startup fails part-way, the splash screen is closed before the error
  box appears so the box is not buried under it; if the first frame has not come after 60 seconds, it closes itself.
- Does not take focus: it is shown without activation, so switching to another window during startup is not undone; it has a taskbar button,
  and the pointer over it is the "working in background" cursor. Centred on the monitor under the pointer.
- A 600 × 340 card, radius 8, self-drawn shadow, 120 ms fade-in. Follows the theme, accent and UI language from the settings.
- Progress comes from the real startup marks, in five stages: runtime → database → interface → sessions and settings → main window. The running
  stage is estimated as `1 − e^(−t/τ)`, capped at 0.95, so the progress never fills up while the screen is still waiting.

**Five styles** (Settings → Appearance → Window → Splash screen; Classic out of the box)

| Style | What it shows |
| --- | --- |
| Classic | Closest to Visual Studio: a large logo and the product name top left, the current stage bottom left, a row of accent dots sliding left to right along the bottom (fast at the ends, slow in the middle) |
| Terminal boot | A mini terminal: the prompt types `velashell`, then one line per stage with its real duration (`[ ok ]` and a spinner); the second line is the SSH protocol version string; a thin line along the bottom is the total progress. If startup hangs, you can see where |
| Vela constellation | The Vela in the product name is the southern constellation of the sails: stars twinkle, each stage lights a main star and draws one more segment, and the constellation closes just as the main window appears; lit stars carry their Bayer designations |
| Prompt | The `>_` from the logo types the product name letter by letter; below it a status line and five squares, the running one breathing |
| Mascot | The chibi mascot with her laptop (the original art, multiplied onto a light panel), a speech bubble saying the current stage in her voice, five paw prints lighting up. The one exception to DESIGN.md's "no illustrations in the UI": it has to be chosen by the user |
| Off | — |

**Preview**: a live preview under the dropdown, drawn by the same code as the real splash screen and looping at the "typical" speed; the theme
and accent come from what is being edited on the page (saved or not). Hidden when Off is selected. Changes **take effect on the next launch**.

**Implementation constraints** (do not break when changing it)

- It cannot be an Avalonia window, which could only appear after Avalonia has initialised. It is a native layered window with its own
  message loop on its own thread, its frames drawn by SkiaSharp into a DIB.
- Theme, accent, language and style are read from `~/.velashell/startup.appearance` (the database is not open yet); see the thirteenth batch
  of the [settings audit](settings-audit.md).
- Fonts are system fonts (Segoe UI; Cascadia Mono, or Consolas without it): the bundled Inter and Cascadia Mono are Avalonia resources and
  cannot be read yet. CJK text and the Braille spinner fall back per character to a system font that has them.
- The startup timeline in the diagnostic log gains a `SplashShown` mark; when the splash screen closes it logs
  `[Splash] style=… scale=… open=… ms frames=… render avg=… ms max=… ms`, to tell whether the splash itself slowed startup down.
- `VELASHELL_NO_SPLASH=1` turns the splash screen off.

---

## 15. Advanced Feature Panels (Large Standalone Panels)

Open these as tabs or standalone floating windows. All can also be entered from the command palette:

- **Operations Orchestration Center (`bR5c4`, 920×760)**: header + content area + results panel. Execute scripts/playbooks on multiple hosts in batches: choose a target group → configure steps → execute → summarize live results (success/failure/output).
- **Host Trust Center (`gPWeC`)**: policy row (trust policy: strict/TOFU/lenient) + host fingerprint table (host, fingerprint, algorithm, first-seen time, status, delete/reset actions). ✅ **Implemented in simplified form (2026-07-12)**: Settings → Security Audit includes “Host Trust Policy” (three first-confirmation options / block changes) + a “Trusted Hosts” list (view/delete, screenshot-safe address redaction); the standalone large panel is not implemented separately for now.
- **Session Recording and Playback (`NceE6`)**: recording list + player (timeline, speed control, search), asciinema-style playback, and export. ✅ **Implemented (2026-07-12)**: `RecordingPlayerView` is a standalone window, entered through Settings → Security Audit → Playback Center. The left column contains the recording list (name/time/duration/size, delete); the right column contains a read-only terminal + timeline seek + 1x/2x/4x speed + skip idle segments. The title bar's action group (§2) provides export recording (asciicast v2 `.cast`), refresh, clean up (deletes recordings and actually frees the disk space) and the “Auto Recording” toggle (= `Security.RecordProductionSessions`, its icon turns accent while on). Storage is described in Architecture Design §4.11 (SonnetDB time series). The “Search” feature in the design has not been implemented.
- **Audit log (2026-09-29)**: `AuditLogView` is a standalone window (frame, title bar and resize grip as in the recording player), entered through Settings → Security Audit → Audit log → “View”. It lists the event kinds in `audit_log` — connection success / failure, host fingerprint decisions, externally launched logins — with the columns time, category, event, session and detail: events are put into words (Connected / Connection failed / Host fingerprint rejected / Host fingerprint trusted once / Changed host fingerprint accepted / External login; unknown ones are shown as stored); session names are looked up from the profile id when loading, and a deleted profile shows — (the `user@host:port` in the detail still tells which machine it was). Connection failures, rejections and accepted fingerprint changes count as “problems”, and their event names are red. The filter bar combines three filters: category (All categories / Connection / Security), “Problems only”, and a keyword (event, category, session name, detail; case-insensitive). Only the latest 2000 entries are loaded, and the summary says so when it is full; a database read failure is reported in the summary. Esc clears the keyword first, then closes the window.
  - **Retention**: `Security.AuditLogRetentionDays` (default 180 days, 1–3650), shared by the audit log and the connection history (`conn_history`, the ledger behind “Recent connections”) — keeping connection history longer than the audit makes no sense, and keeping it shorter would leave sessions in the audit that the sidebar no longer knows. Older entries are pruned **at startup only**, at the same time as session logs and recordings expire; pruning is a time-based `DELETE`, not the recordings' “stage → rebuild → restore” (a process dying half-way through that would lose entries still within retention, a risk the audit log should take least of all).
- **Connection Diagnostics Center (`RGXg1`, 920×640)**: run ping / DNS / port probes / traceroute / SSH handshake analysis against a target, and output step-by-step diagnostic conclusions and recommendations.

---

## 16. Global Keyboard Shortcuts (factory defaults; the global and tab bindings can be changed or unbound in Settings → Shortcuts)

> Current state after checking against actual bindings on 2026-07-12. Items marked “Not implemented” were concepts in the initial design and have no current binding.
>
> **This section is a design-level excerpt, not the complete list.** Every bound keyboard shortcut and mouse gesture
> (terminal selection gestures, the completion popup and per-dialog keys included) lives in
> [Keyboard Shortcuts](keyboard-shortcuts.md) — that table shares its source with `ShortcutCatalog` and is test-guarded,
> so treat it as authoritative when adding shortcuts.

| Shortcut | Function | Status |
|---|---|---|
| `Ctrl+P` | Command palette | ✅ |
| `Ctrl+N` | New SSH connection | ✅ |
| `Ctrl+T` | New tab (same as New Connection) | ✅ |
| `Ctrl+Shift+N` | Clone current session | ✅ |
| `Ctrl+Shift+F` | Toggle file browser (SFTP) | ✅ |
| `Ctrl+,` | Settings | ✅ |
| `Ctrl+W` | Close current tab | ✅ |
| `Ctrl+Tab` / `Ctrl+Shift+Tab` | Next/previous tab | ✅ |
| `Ctrl+Shift+T` | Open Tunnel Manager | ✅ |
| `Ctrl+F` | Search in terminal | ✅ |
| `Ctrl+Shift+C` / `Ctrl+Shift+V` | Copy/paste in terminal | ✅ |
| `Ctrl+Backspace` | Delete the previous word before the cursor in the terminal (equivalent to `Alt+Backspace`) | ✅ |
| `Alt+Enter` | Open command completion | ✅ |
| `Enter` / `Ctrl+R` | Reconnect after disconnection | ✅ |
| `Ctrl+S` | Save in the remote editor | ✅ |
| `Esc` | Close the current floating panel/panel | ✅ |
| `Ctrl+Alt+1..8` / `Ctrl+Alt+9` | Jump to the Nth / last tab (`Ctrl+digit` would take control characters, hence `Ctrl+Alt`) | ✅ |
| `Ctrl+\` | Split pane | Not implemented (splitting is completed by dragging) |

---

## 17. Floating Panel Management and Boundary Rules (General Constraints for the Agent)

1. **Floating panel levels**: Settings/New Connection/Password are **modal dialogs** (centered + overlay); the command palette is **quasi-modal** (overlay, close with Esc); file transfer/Tunnel/Resource Monitor are **non-modal floating panels** (no overlay, can coexist with the main interface).
2. **Singletons**: the command palette, tunnel panel, resource monitor panel, and file transfer component are each global singletons. Repeated triggers focus the existing instance instead of creating another.
3. **Screen/window boundary avoidance**: all mouse-following or anchor-following floating panels automatically flip direction at an edge and remain fully visible.
4. **Position memory**: persist the drag positions of the file transfer component, the message center, and the tunnel panel.
5. **Theme linkage**: all floating panels follow the global dark/light theme in real time.
6. **Resource panel hover timing**: start the 400ms timer on entering a tab name and cancel it when the pointer leaves. Close only after leaving the combined panel-and-tab area, with a debounce of ~150ms, to avoid flashing out while moving toward the panel.
7. **Transfer/tunnel and session lifecycle**: when a session disconnects, pause its tunnels and transfers and show a notice. If an active transfer exists before closing the session, ask for confirmation.
8. **Empty states**: when there are no sessions, show guidance in the terminal area (New Connection/Recent Connections); when there are no transfers, do not render the transfer component; when there are no tunnels, show “No tunnels + New” in the tunnel panel.

---

## 18. Suggested Implementation Priority (Iteration Order for the Agent)

1. **P0 Layout skeleton**: three-area layout + Dock splitters + status bar; connect theme tokens.
2. **P0 Top-level structure refactor**: menu bar (§4A, including the text menu + global feature button `GQQwj`) + tab bar (§4B, overflow scrolling `◀▶`, `▾` drop-down, drag reordering, splitting).
3. **P0 Terminal engine integration**: VT rendering + input + selection/copy/search.
4. **P1 Command palette (§8)** and **New Connection/Authentication flow (§13)**.
5. **P1 File browser + transfer component (§6/§9)**: drag-and-drop upload/download + floating progress + automatic disappearance.
6. **P1 Tunnel panel (§10)** and **Resource Monitor hover panel (§11)**.
7. **P2 Full Settings pages (§14)** and **Advanced panels (§15)**.
8. Follow the floating panel and boundary rules in §17 throughout.

---

### Appendix: Design File Frame ID Reference (for Pixel Comparison)

Main interface dark `CsTjc` / light `mkrrg`; sidebar `aMaSq` (dark) / `Cw1Yt` (light), sidebar top Logo bar `tGM2H`/`4RBrb` **deleted, replaced by the system native title bar**; **menu bar `TSiDh`** (left text menu container `DaZfB`, right feature container `Menu Bar Actions`, containing the global feature button group `GQQwj`, moved from the terminal toolbar); tab bar `nunbT` (including overflow control group `Tab Overflow Controls`=`pZGS4`, `◀◁ tabScrollLeft` / `▶ tabScrollRight` / `▾ tabListDrop`); terminal toolbar `BdPtF` (now session information only); terminal canvas `QzoMC`; file area `dyuii`; status bar `gzmsb`; command palette `FN5dM`; file transfer component `9Ralg`; tunnel panel `fuXS7`; resource monitor panel `EP3Gd`; context menu `e6klM`; disconnected state `ZufZw`; new connection `oAHna`; password verification `oNZIM`/`twD13`; session settings `ZNjAC`; Settings pages are listed in §14; advanced panels `bR5c4`/`gPWeC`/`NceE6`/`RGXg1`.
