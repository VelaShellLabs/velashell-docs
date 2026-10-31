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

The main window is 1440×900 and can be freely resized. **〔2026-07 final, user decision〕The window uses a custom borderless frame**: `WindowDecorations="None"` (Avalonia 12.x `BorderOnly`/`ExtendClientArea` managed decorations intercept title bar input on Win32, preventing button clicks and window dragging; this was abandoned in testing). It uses custom drawing + the native `BeginMoveDrag` movement loop + a WndProc hook for Windows 11 snap support. **〔Since 2026-09〕macOS is the exception**: the main window uses the system frame and the native traffic lights (close / minimize / full screen, on the left of the title bar); the self-drawn minimize/maximize/close buttons are hidden and the logo makes room for the traffic lights (not in full screen), and the title bar takes the system title bar height (28pt today — the traffic lights cannot be moved, so the title bar aligns to them instead). Windows and Linux are unchanged. How each platform's frame works, and why, is in [`architecture.md`](architecture.md) §5 "Window shell". The first row is a 28px custom title bar (`TitleBarView`: left = product logo + name; right = global feature button group `GQQwj` + minimize/maximize/close buttons). **The text menu has been removed entirely** (its functions duplicate the command palette). From top to bottom, the client area is: **custom title bar → (resource sidebar ‖ main workspace ‖ optional dedicated AI region) → status bar**. The resource sidebar can move to either side of the main workspace; the AI region always stays at the far right of the window. The diagram below shows the layout with AI hidden.

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
- **Modeless tool windows remember size and position** (since 2026-10-24): Task Manager, Resource Monitor, Route Trace, Plugin Manager, plugin log, Layout templates, SSH connection info, Audit log and the Playback center record their normal-state size and position when closed (in the document store's `windowLayout`, one entry per kind of window, e.g. `pluginManager`, `layoutTemplates`) and reopen where they were; when that position is no longer within any screen's work area (a secondary screen unplugged or rearranged), the window is centered on its owner as usual. A window closed while maximized or minimized keeps the last normal-state values. Modal dialogs (such as the upload picker) only remember their size. The directory bookmark and directory sync windows opened from an SFTP document are not included.
- **Every window's title bar is 28px** (main window, standalone windows and dialogs, since 2026-09-26; previously the main window was 36, dialogs 48 and standalone windows 34–56): the title bar holds the icon, the title, the window-level actions and the window buttons (the × at the top right for dialogs). Window buttons are always 27×27 squares flush with the top-right corner, close hovers system red; the card clips to its rounded corner, so a square's hover fill follows the corner instead of poking out of the window. **Window-level actions live in the title bar** (since 2026-09-29): an icon-only group just left of the window buttons, styled like the main window's global feature button group `GQQwj` — each has a tooltip saying what it does and an automation name, and its icon turns accent while a toggle is on or while something is waiting to be done. Resource monitor: the host badge (green dot when connected / grey when not + host name) and pause / resume (while paused it becomes an accent "resume"); recording player: export / refresh / clean up / auto-record (accent while on); remote editor: save (accent while there are unsaved changes — it used to be an always-accent solid button); connection diagnostics: export report (disabled until there is a report) / re-run. The two-line headers of these four windows now keep only the subtitle in the second row (sampling note / feature description / remote path / diagnosis target), with the same background and also draggable; double-clicking a button in the title bar never maximizes the window. The New Connection, Import Sessions and connection diagnostics dialogs use the same bar as the standalone windows: a 15px line icon (accent) + a 13px title at the left + the × at the top right; dialogs are fixed-size, get no maximize, and keep their × on macOS (dialogs show no traffic lights). On macOS, windows that show the traffic lights take the system title bar height (28pt today), so the lights and the title share one centre line. Two exceptions have no window buttons in their header, so it is not a title bar and stays 48: the settings window, whose top-left strip is the navigation header doubling as the drag area (no close / minimize buttons); and the message dialog (notice / confirm / prompt), which has no × at the top right and closes only from its button bar or Esc.
- Below the title bar, the main workspace starts with the tab bar (see §4B) and the resource sidebar can change sides. An optional dedicated AI region sits at the far right, outside the document tab tree.
- **Dedicated AI region**: The title bar's AI assistant button or the palette's **AI: Open Assistant** command expands the same chat control without adding a tab or switching the active terminal. Default width is 380px; drag its left edge to resize (300px minimum, 1200px maximum, also constrained by available window space). Pinned mode reserves actual workspace width. Unpinned mode reserves only a 32px rail: hovering over its AI icon button expands an overlay; the blank rail area does not trigger expansion, leaving collapses it after 300ms, with open plugin dropdowns/menus delaying collapse. Collapse keeps the rail; × hides it too. Both preserve the conversation, draft and tasks without closing the plugin panel. The host saves width/pin preferences in `Appearance.AiPanelWidth` / `AiPanelPinned` on normal exit, independently of window-position memory or session restoration. These local layout values are excluded from settings exports and settings-change audits. Startup restores preferences without opening AI. A narrow window temporarily uses an overlay to preserve the main workspace's minimum space and returns to the pin preference when enlarged. A Sparkles icon accompanies the title, and the displayed content width stays at least 300px. The server selector is left-aligned and limits its width and truncates long names to leave room for the four action buttons.

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
     "Export all connections" is disabled when there are no connections. The export dialog's format segments are VelaShell
     JSON / CSV spreadsheet / OpenSSH config (the third since 2026-10-24: `~/.ssh/config` Host blocks, SSH / SFTP only, no
     passwords). "Sort groups by name" is a one-off reorder
     (natural order: "Rack 2" before "Rack 10"; the current locale's collation, so Chinese sorts by pinyin), and the order can
     still be adjusted by hand afterwards.
2. **Session resource tree (`fill_container`, scrollable)**: collapsible group list occupying most of the sidebar.
   - **Filter box** (since 2026-10-24): pinned above the tree and not scrolled with it (one in the expanded sidebar and one in the collapsed rail's flyout, bound to the same text), placeholder “Filter by name, host, tag or note”.
     - Filters by name / host / tag / note / the name of the containing group: the text is split on whitespace, **every word must match**, and different words may match different fields; case-insensitive, contiguous substrings. The username is not used (typing `root` would keep almost everything). Typing a group name lays out the whole group.
     - Groups with a match expand automatically, groups without one are hidden entirely; pinned connections still come first, and those without a match are hidden too. When nothing matches, the tree area reads “No connections match the filter.”.
     - Expansion is temporary: expanding / collapsing while filtering is not written to the expansion memory, and clearing the filter (Esc in the box, or the × at its right end) returns every group to how it was before filtering. When exactly one connection is left, Enter connects it.
     - A selection that gets filtered out is cleared (the context menu must not act on another row), and “follow the active terminal tab” does not select hidden rows; groups cannot be dragged while filtering (the insertion line can only be drawn between visible rows, so the drop target would lie).
     - Filtering only changes the display: the group menu (open all / export / delete) still acts on the whole group; to act on the filtered results, multi-select with Shift / Ctrl (ranges already follow the visible rows).
   - Group row: collapsible triangle + group name, such as “Production” or “Testing”, + count.
   - Host row: status dot (green/yellow/red) + hostname + optional label, such as “jump host” on an `accent-dim` background.
     - A connection that picked a tab icon (§13.1 “Organize”) shows the same 12px icon between the status dot and the name (colored like the name, accent when selected); rows without one draw nothing and keep no space for it.
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
| `layout-template` | **Layout templates** | Opens a drop-down menu (since 2026-10-24): one item per template, written “name · N tabs · M panes”; clicking one opens that set of connections in the template's split layout (§4C). With no templates the first item is a disabled “No layout templates yet”. After a separator come “Save current layout as template…” (disabled when the workspace has no tabs) and “Manage layout templates…”. The command palette has two commands of the same names, `layout.save` / `layout.manage` |
| `zap` | **Quick Commands** | Open the command palette (§8), or the quick command menu |
| `sparkles` | **AI Assistant** | Run `velashell.ai.chat` to open the right-side AI chat dock panel, or activate it if already open. The AI plugin's chat tab, collaboration window, and model configuration window share the Lucide `sparkles` icon |
| `app-window` | **X Server** | Start / stop the local X server: by default the **built-in X server** (ships with the app, works on every platform, one native window per X window); on Windows the “X Server” page in §14 can switch it to launching the VcXsrv the user installed. The icon turns accent-colored while running and the tooltip shows the display (following the listeners actually open: e.g. `localhost:0.0` on Windows, and `:0` on Linux / macOS, where only the Unix socket is open by default); the button is disabled while starting so a double click cannot start two. When it starts but local programs cannot connect (on Linux / macOS the Unix socket could not be created and TCP is off), a separate warning toast says that SSH forwarding works as usual and that turning on "Also listen on a TCP port" lets local programs connect. **While the built-in engine runs, clicking the button opens a flyout** (anchored below the menu bar on the right, the same spot as the tunnel panel — opening one closes the other; 340 px, styled like §10): the header shows X Server, the display and the program count; each row is one connected X program — its name (the first window's `WM_CLASS`, else the window title, else “X program #number”), a source tag (`user@host:port` for programs forwarded through an SSH session, “This computer” for local ones), its window count and accounted memory, and the window title; the second line starts with the trust level (since 2026-10-24) — “Trusted” (a grey outlined badge: full access to the whole X Server; local programs and `ssh -Y`) or “Untrusted” (an info-colored outlined badge: `ssh -X`, reaches only the windows of untrusted programs), with a tooltip explaining each; one holding the whole X Server gets an orange “Exclusive” badge, and one that disconnected in Retain mode and left only resources says so. Hovering a row reveals a disconnect button, which first asks with a warning-style “Disconnect X program” confirmation (all its windows close, unsaved work is lost) and a danger “Disconnect” button. At the bottom, “Unstick” (releases every X program's mouse and keyboard grabs and its hold on the X Server without stopping it — for when no X window responds, e.g. a remote menu was open when the network dropped) and “Stop X Server” (the stop confirmation below). The flyout refreshes every 2 s while open and closes when the X Server stops. VcXsrv cannot list its programs, so there the button still starts and stops it. When a program has held the X Server exclusively for more than 10 seconds while other X programs wait for it, a warning toast says “X program “name” (source) has held the X Server exclusively for N s; every other X program is waiting for it.” with a “Disconnect it” button; it is updated in place every minute after that. **Shown on every platform**, last in this group. Failures such as the display number being taken or VcXsrv not being found raise an error toast with a “Settings” button that lands on the X Server page; with the display number on “auto”, a number found taken only at the moment of starting is retried with the next free one (up to 4 times), and any exception during start is cleaned up (the server released, back to not running) before the error is reported, so it never gets stuck in “starting”. When stopping while X programs are still connected (only the built-in engine can count them; clients that disconnected in Retain mode and left only resources do not count), a warning-style confirmation “Stop X Server” comes first — it says how many X program connections will be dropped, that all their windows will close and that unsaved work will be lost, with a “Stop” button; with no programs connected it stops right away, and stopping because the main window really closes asks nothing. Same command in the palette: `tools.xserver` |

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
  X Server from the system's `Screen.WorkingArea`, so menus, maximized windows and dialogs avoid the taskbar. While a native window is dragged,
  the X side's position follows at the pace the program processes it (the next position goes out once the previous move has been handled), so on
  a slow link the program no longer freezes for a while after the button is released.
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
  compositing manager — except windows whose clients declare their edges not part of the window with an input shape (the shadow margin around
  GTK 4 drop-downs and popup menus), which are shown with their transparency instead of a black border.
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
- **Remote GTK 4 programs render with cairo**: in sessions with X11 forwarding (whichever X server the display goes to), the silent post-connect
  injection that sets `XMODIFIERS` also sets `GSK_RENDERER=cairo` (an existing value in the remote shell is kept). GTK 4's default GL can only fall
  back to software rendering on the remote side over SSH and sends the whole window's pixels every frame (29 MB for one button click in
  `gnome-calculator`, 0.15 MB with cairo). Only programs started from that terminal get it.
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
  is gone the files are not accepted, and an upload failure shows an error toast.
  Uploaded files are not deleted when the drop ends (the program may read them later, an editor saves back in place, and a path dropped into a
  terminal is only used once the user types the command): once every session to that host (the same connection name `user@host:port`) that was
  connected at the time of the drop has ended, they are removed in the background the next time that connection name connects or something is
  dropped on that host again; a failed or canceled upload, or one where nothing was uploaded, is removed in the background right away (the original
  error is still reported). Leftovers from a crash or an earlier run are cleared on the next drop to that host once they are a day old (by the remote
  host's own clock; only real directories under `/tmp` — not symbolic links — named exactly `velashell-drop-` plus 32 lowercase hex digits and owned
  by the same user as the directory just created). Failures to delete are silent and retried at most three times. Programs that connected locally get
  local paths.
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

**Tab bar position** (since 2026-10-24, Settings → Appearance → Window → “Tab bar position”, `AppearanceOptions.TabBarPosition`: Top (default) / Bottom): with “Bottom”, every pane's horizontal tab strip moves below the content together with its 1px separator, which still sits between the strip and the content. The tabs themselves look the same, and the active tab's accent line stays on the tab's top edge — at the bottom that is exactly the side facing the content, for the same reason the accent line sits on the inner edge of a vertical strip. It is one value for the whole workspace, so every pane changes together; changing it on the settings page previews immediately and Cancel rolls it back. A pane whose tabs were moved left or right from the tab right-click menu “Tab Position” was chosen per pane and is not affected; switching it back to horizontal puts it on the side set here.

### 4B.1 Layout

```
┌───────────────────────────────────────────────────────────────────────┐
│[Tab1][Tab2][Tab3 …scroll…] [+]   ……flexible gap……     [◀][▶][▾]       │
└───────────────────────────────────────────────────────────────────────┘
     ← tab scroll container (clip) →    new tab   spacer       scroll left  scroll right  list
```

- **Tab scroll container** (`clip:true`, `fill_container` taking the middle space): lays out all tabs horizontally; overflow is clipped and can be scrolled horizontally with the wheel; the active tab automatically uses `ScrollIntoView`.
- **`+` New Tab** (since 2026-10-24): a 24px icon button in the same style as the three overflow buttons; its tooltip and screen-reader name are both “New Tab”, and it can be reached with Tab and pressed with Enter.
  - Position: right after the last tab — on the last row when multiple rows are on; below the last tab, aligned with the tabs' content edge, when the strip is vertical (docked left / right); at the start of the strip in an empty pane. When the tabs don't fit and scroll, it stays pinned at the end of the viewport.
  - The tab viewport gives up 28px at the end of its axis for it, and both single-row shrinking and multi-row wrapping use the width left after that, so the `+` itself never pushes the strip into overflow.
  - Clicking it first makes that pane the active one, then opens the same new-connection dialog as `Ctrl+T` (`session.new`); the tab it connects **lands in the pane whose `+` was pressed**, even when the layout has another, empty pane (since 2026-10-25, the maintainer's call: the pressed pane wins over an empty one).
    The first tab the connection flow creates (a terminal tab, or the "connecting" placeholder of a document-style connection) takes that pane; once the flow is over (connected, failed, cancelled, or only "Save" was pressed) the claim expires and later tabs follow the default rule. If that pane is closed during the flow, the default rule applies. `Ctrl+T` and the other entry points still follow the new-document placement rule (empty panes first).
- **Flexible spacer**: pushes overflow controls to the far right.
- **Overflow control group `◀ ▶ ▾`** on the right, with 24×24 icon buttons:
  1. `◀` chevron-left: scroll left by one screen/one tab; disabled when at the far left.
  2. `▶` chevron-right: scroll right; disabled when at the far right.
  3. `▾` chevron-down: **tab drop-down list** (VS Code style), listing all session names + status dots. Clicking an item activates it and scrolls it into view. This is especially useful during overflow. `◀ ▶` are available only during overflow; `▾` is always available.
  - **On a vertical strip** (since 2026-10-24): overflow is judged by height, and the three buttons move to the bottom of the strip, still in one row; `◀ ▶` turn into up / down, and both the wheel and the two scroll buttons scroll vertically. `▾` points to the side the list opens on, toward the content: right for a strip on the left, left for a strip on the right; for a horizontal strip at the bottom it points up and the list opens upward.

### 4B.2 Individual Tab Content

- Structure: status dot (`status-connected/connecting/disconnected`, 7px) + session name (JetBrains Mono 11) + close `x` (12px).
- Active state: `bg-terminal` background + 2px accent bar at the top + `text-primary` text.
- Inactive state: `tab-inactive-bg` background + `text-tertiary` text + `text-muted` close icon, which becomes prominent only on hover.
- Connecting tabs use a yellow dot; disconnected tabs use a red dot.
- **Protocol icon** (since 2026-09-10): after a session tab's status dot comes a 3×12 connection color strip, then a 13px icon. **The color answers “which machine”, the icon answers “which kind”**, and the two are not mixed:
  the strip and the icon both use the connection's identity color (`ConnectionAccent`: picked automatically from the connection id, or the tab color chosen on the Terminal page), while the glyph follows the connection type —
  - **SSH terminal**: `square-terminal` (the framed terminal glyph).
  - **File protocols** (SFTP / FTP document tabs; a dual-pane document uses the left pane's connection): `hard-drive` — the tab holds a file browser, not a terminal.
  - **Plugin protocols** (a plugin's terminal protocol such as serial / Telnet, file protocol such as S3, or workspace such as Redis): the icon the plugin reports in its descriptor
    (`ProtocolDescriptor.Icon` / `WorkspaceDescriptor.Icon`, see [SDK §3.6](../sdk/sdk-reference.md#36-tab-icons-pluginicon-sdk-204));
    if it reports none, or its path fails to parse, the generic plug `plug`. A filled plugin icon is filled with the same identity color.
  - **Local terminals get no icon**: there is no connection profile, so the icon and its slot are dropped rather than leaving a gap; the strip is transparent.
  - When the connection picks a **tab icon** in the “Organize” group of its General page, that icon overrides both the type glyph and the plugin's icon; a stored id that is not recognized (added by a newer version, edited by hand) counts as no choice.
  - Plugin panel tabs (`PanelOptions.Icon`): the icon the plugin reports, `plug` when none, stroked and filled in the accent color `VelaAccent`; when the panel opens as a window, its title bar uses the same icon.
  The host only knows its own built-in protocols and keeps no “plugin id → icon” table — third-party plugins could never get into such a table, so a plugin hands over its own icon.
- Width follows the title (#521): the full title when there is room; when one row is too narrow the tab shrinks as described in §4B.3, and **only the title shrinks** (with an ellipsis) — the dot, icons and the button at the end stay as they are.
- A pinned tab (§4B.5) shows a pin instead of the `x`.
- Interaction:
  - Click = activate; middle-click/click `x` = close, with a second confirmation when there are unsaved changes or active transfers. A pinned tab ignores middle-click.
  - Double-click empty tab space = rename (inline editing).
  - Drag = reorder; drag to an edge = trigger Dock splitting/floating window. A pinned tab only moves within the pinned block, and an unpinned tab cannot be dragged into it.
  - Right-click = tab context menu. Current state (2026-10-24): Connection info… (SSH session tabs only, disabled while not connected, see §11B) | sync input (terminal tabs only) | Close, Close Other Tabs, Close All Tabs, Close Tabs to the Left, Close Tabs to the Right | Pin Tab / Unpin Tab | Split Horizontally, Split Vertically, Maximize / restore pane | Tab Position (submenu: Top / Left / Right; the first item reads “Bottom” when “Tab bar position” is set to the bottom), Show Tabs in Multiple Rows (check item). Close Others / All / Left / Right always **skip pinned tabs**. Duplicate session, rename and move to new window are not implemented yet.

### 4B.3 Tab Overflow Logic (VS Code style) ★User specified

- When total tab width ≤ container width: lay out normally, hide/disable `◀ ▶`, and keep `▾` available.
- When the tabs don't fit, **shrink first, then overflow** (#521, following VS Code's `tabSizing: shrink`): only **the widest tabs** are trimmed, down to one common width; tabs narrower than that keep their width (a short tab such as "FTP" doesn't narrow along with the long ones). Overflow below only starts once they are down to the **120px** floor and still don't fit.
- On overflow:
  1. Clip the container and automatically use `ScrollIntoView` to keep the active tab visible.
  2. Allow `◀ / ▶` to scroll; disable the corresponding button at each boundary.
  3. Allow horizontal wheel scrolling over the container.
  4. Always list all tabs in the `▾` drop-down as a fallback for quick navigation during overflow.
- Drop-down items: status dot + session name + a check mark on the right for the current item. Support keyboard up/down + Enter selection. Tabs can also be closed here, with `x` appearing on hover.
- **Multiple rows** (#521, matching Visual Studio's "Show tabs in multiple rows"): when on, tabs that don't fit wrap onto the next row instead of shrinking or overflowing; the strip grows with the number of rows (35px per row, the 1px separator stays between the rows and the content: below the last row when the strip is at the top, above the first row when it is at the bottom), and the active tab's accent line moves to its row. Two entry points change the same value: "Show Tabs in Multiple Rows" (check item) in any tab's right-click menu, and the switch of the same name under Settings → Appearance → Window (`AppearanceOptions.MultiRowTabs`, off by default). One switch for the whole workspace, so every pane changes together; it only applies to horizontal tab strips (top or bottom) — a strip docked left or right already has one tab per row, and the menu item is disabled there.

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
- The pinned state is saved with layout templates (§4C) and with “Restore split layout”; when neither is used it lasts for the current run only.

---

## 4C. Layout templates (since 2026-10-24)

Save the current split panes and tabs as a named template and reopen the same set of connections in the same arrangement with one click. Entry points: the `layout-template` drop-down in the title bar (§4A.2), the command palette (the “Layout templates” group in §8, plus the two commands `layout.save` / `layout.manage`), and the “Manage layout templates” window. There is no keyboard shortcut.

- **What is saved**: split orientations and proportions; for each pane its tab order, selected tab, pinned states (§4B.5) and tab strip position (top / left / right); the active tab of the whole workspace. Each tab records the saved connection it points to (profile Id) and how it was opened: terminal, standalone SFTP / FTP / plugin file tab (a terminal and a standalone SFTP tab from the same SSH profile are recorded separately), dual-remote file tab (two profiles), plugin workspace, local terminal (by shell type). A “connecting” placeholder or a failure card is recorded as what it would be once connected. No passwords are stored. Pane maximization, and workspace-wide display switches such as multiple rows or a bottom tab strip, are not part of a template.
- **Saving** (“Save current layout as template…”):
  - A name box opens with “Layout N”, where N is the first unused number; the name is trimmed and limited to 100 characters.
  - If a template with the same name (case-insensitive) exists, it first asks “Replace layout template” (button “Replace”); the replaced template keeps its Id and creation time.
  - Success is an info toast, “Saved layout template "X": N tabs, M panes.” If some tabs could not be saved it becomes a warning naming at most 5 of them, with an ellipsis for the rest — what cannot be saved is plugin panels (AI chat and the like, which have no contract for reopening them), temporary connections whose profile is not in the database (quick connect, external launch), and terminals that have neither a profile nor a shell.
  - When nothing can be saved, it only warns “Nothing in the current layout can go into a template…” and shows no name box.
- **Opening**:
  - The latest profiles are read from the database first; tabs whose profile was deleted, whose shell this computer lacks, or whose type this version doesn't know are skipped and listed with name and reason in a warning toast, and the rest open. When none can be opened it says so (“…can be opened any more”) and leaves the layout untouched.
  - With no tabs in the workspace, the template becomes the whole layout; with existing tabs, the template is placed as one block to the right of the current layout (a left/right split, half each), and the existing tabs stay where they are; a maximized pane is restored first.
  - More than 10 tabs asks for confirmation first (the same dialog as “Open N connections” in the explorer).
  - Every tab goes through its existing open flow, at most 6 handshakes at a time, with login prompts as usual when credentials are missing; a tab that arrives late (because a credential prompt came first) is still put back in its place in the template.
  - Only the template's active tab takes focus; the other tabs are selected in their panes as the template says.
  - A pane that never received a tab (the credential prompt was canceled) is closed when the opening finishes.
- **Manage window** “Layout templates”: a 640×460 (minimum 480×320) modeless tool window, only one at a time — opening it again brings it to the front; its size and position are remembered like the other tool windows (the `layoutTemplates` entry in `windowLayout`, see §2).
  - 28px title bar; a line of explanation at the top (“Arrange tabs and split panes, then save them as a template… contain no passwords.”), and the bottom action bar has “Export…” (`file-down`) and “Import…” (`file-up`) on the left and “Save current layout as template…” on the right.
  - Each row shows the name and “N tabs · M panes”, and on the right, in order: open (`play`, accent), overwrite with current layout (`save`, asks first), rename (`pencil`; a name taken by another template is refused with “Choose another name.”), delete (`trash-2`, danger confirmation; deletes the template only, never the connections it refers to). Double-clicking a row also opens it; Esc closes the window.
  - While the window is open, the dialogs of these actions are owned by it.
- **Export / import** (since 2026-10-25):
  - Export writes every template into one JSON file, default name `velashell-layout-templates.json`, starting in the downloads folder, file type “VelaShell layout templates”. It holds the export header `velaShellExport`
    (`format: velashell-layout-templates`, `version: 1`, `exportedAtUtc`, `count` — the same key as the settings export) plus a `templates` array whose items have the same shape as in the store, ids included, so importing on another device with sync on yields the same template.
    No passwords or host addresses, though tab titles are often `user@host`. With no templates it says “There are no layout templates to export yet.”; on success “Exported N layout template(s) to <path>. The file holds connection IDs and how each tab opens, no passwords.”
  - Import: a file over 4 MB, not JSON, of the wrong format or without a `templates` array gives “This is not a VelaShell layout template file.”; one without a single usable template gives “The file contains no usable layout templates.”.
    A file whose format matches is read on a best-effort basis even when its version is newer, keeping unknown fields; items with an empty name or not a single usable tab are dropped, an all-zero id gets a new one, and duplicate ids within the file keep the later item.
  - Conflicts: when templates have the same id as, or share a name (case-insensitive) with, local ones, “Some templates already exist” names up to 5 of them with three buttons, “Replace / Save as new / Cancel”, asked once for all:
    Replace keeps the local template's id and creation time and takes the file's name and layout; Save as new gets a new id and “ (2)” after the name. Conflicts are judged only against templates that existed before the import; duplicate names within the file are taken as they are.
  - Imported templates get “now” as their modification time (so they win over the tombstone left when one was deleted locally and over the cloud copy). Afterwards it says “Imported N layout template(s).”, or, as a warning when some refer to connections this device does not have,
    “Imported N layout template(s). M of them refer to connections that do not exist on this device; those tabs will be skipped when the template is opened.”
- **Storage**: the local SonnetDB collection `layout_templates`, one document per template; a deleted template leaves a tombstone in `layout_template_tombstones` (id plus deletion time, kept 180 days, after which it is removed together with the template it covers).
  It is not in the settings file and does not travel with settings export; with cloud sync on and “Layout templates” ticked in the sync scope it syncs through the Gist (next item).
- **Cloud sync** (since 2026-10-25): `velashell-layouts.json` next to `velashell-sync.json` in the Gist (`format: velashell-layout-templates-sync`, `schemaVersion: 1`, holding `templates` + `tombstones` in `data`;
  `cipherText` with an end-to-end passphrase, encrypted the same way as the main payload). A push writes both files in one commit, and a read fetches both at once.
  - Merging is per template id: live templates compare their modification time and deletions their tombstone's deletion time, the later wins; at the same instant the tombstone wins; equal times count as the same copy, without comparing content. Different ids with the same name are neither merged nor renamed; both stay.
  - It merges before pushing, after pulling, and when the main payloads turn out identical on both sides (the result does not depend on order or repetition); when the cloud lacks local changes it pushes once more, while “Pull only” never pushes.
  - Older versions read and write only the main file and never touch this one — which is why templates are not in the main payload (an old client pushing the whole payload back would wipe a new section, tombstones included).
  - A file written by a newer version, or encrypted while this device has no passphrase or a different one, is skipped (neither read nor written) with a note after the sync result; a broken file (not JSON, wrong format marker, truncated ciphertext) counts as absent and is rewritten with the merged result on the next push (the old content stays in the Gist history).
  - Restoring a past version writes back all of that version's templates with the time set to now, leaving extra local ones alone. The “layout on exit” (next item) is not synced.
- **Restore split layout** (Settings → General → Startup & Window, `General.RestoreLayoutOnStartup`, off by default): a sub-option of “Restore sessions”, disabled while that is off. When on, exiting also records the layout (`layout_session/last`) with only the live tabs (the same rule as restore sessions, plus local terminals and dual-remote file tabs); the next start reconnects them and arranges the split panes, falling back to reconnecting them one by one into a single pane when no layout was recorded. When off, exiting clears the old record.

---

## 5. Main Terminal Area (`bg-terminal`)

### 5.1 In-Terminal Toolbar (28px, below the tab bar, `BdPtF`)

> `GQQwj` has moved to the menu bar (§4A.2). This row retains only **read-only information for the current session** on the left, plus a small number of optional actions on the right:
- Left (`termInfo`): `root@web-prod-01:~` (accent) + `uptime: 42d 7h 23m` (muted) + `|` + `latency: 12ms` (green).
- Right: optionally retain secondary buttons strongly tied to the current terminal, such as “Clear Screen / Full Screen / More”. Global functions have moved to the menu bar.

### 5.2 Terminal Canvas

- Render using the VT engine: draw monospaced text line by line, with support for xterm-256color, true color, cursor, selection, and hyperlink detection.
- Support mouse selection and copy, right-click paste, Ctrl+wheel font zoom, and scrolling back through the buffer.
- **Highlight visible matches of selected text** (#628, 2026-10-15): double-click a word or finish a linear drag selection to highlight all case-insensitive plain-text substring matches in the current viewport. When browsing history, only the screen being viewed is searched; folded-out rows are excluded. The query must be at least 2 graphemes (a letter plus a combining mark counts as one), so a stray one-cell drag does not light up every copy of a letter on screen; a single character activates only when it is a whole word by itself (no word character on either side, such as a double-clicked variable `i`), and then matches whole words only. Adjacent rows in the viewport that auto-wrap split are joined before searching, and a match that crosses a wrap is drawn per row; a selection that itself crosses an auto-wrap (double-clicking a long wrapped URL) activates matching as well. The real selection and copied text stay unchanged, and match backgrounds do not cover the real selection. A new selection replaces the old matches; input, clearing the selection, resizing the grid, or switching between main and alternate screens clears them. Output, scrolling, and folding refresh matches against the latest visible text, without rescanning on every drag movement or cursor blink. Blank, rectangular, disjoint, and multi-line selections across hard line breaks do not activate matching. When a drag runs past the end of a line, the never-written cells it covers are left out of the query (trimmed to the last written cell, the same way copy reads text), while trailing spaces the program actually printed are still matched literally regardless of copy-trimming settings. State is separate from Ctrl+F: closing the search bar preserves selection matches; Ctrl+F's search rules are unchanged, and navigating a search result still selects that result as before and ends the previous selection matching. Enabled by default, with no new setting or shortcut.
- **Alt+left-drag = rectangular block selection** (#128, aligned with Windows Terminal behavior): whether Alt is held at mouse-down determines whether the operation is block selection or a normal linear selection. Changing Alt during the drag does not switch modes. Copy takes the same column range from each line, always inserting line breaks between lines. When the application enables mouse tracking (htop/vim/tmux), mouse events are given to the application; use Shift+Alt+drag to force block selection.
- **Shift+left-click = extend the selection** (#266, aligned with Windows Terminal / xterm): when a selection already exists, Shift+click keeps the anchor in place and moves only the far end to the clicked cell (before or after the anchor — the selection flips direction accordingly); keep the button held to keep dragging and fine-tune it. To grab a long log spanning more than one screen, select the start, scroll back, then Shift+click the end. The extension reuses the linear/block mode fixed at the original mouse-down. With no selection yet, Shift+click still starts a new one, preserving the existing "hold Shift to bypass application mouse reporting and select text" semantics.
- **Ctrl+Shift+left-drag = append a discontiguous region**: commits the in-progress region and starts another one, repeatable. Copy concatenates the regions in **document order, top to bottom**, with a line break between them — so "select line 1, Ctrl+Shift-select line 3, copy once and get both lines" holds regardless of the order they were picked. Each region remembers its own mode, so Ctrl+Shift+Alt+drag appends a rectangular region that coexists with linear ones, and Ctrl+Shift+double-click appends another word. A plain drag without Ctrl+Shift starts over (dropping every appended region); so does a search-hit highlight. Ctrl+**Shift**+click on a URL does not open the browser (opening links is Ctrl+click without Shift). Terminals have no precedent for this (Windows Terminal / iTerm2 / xterm all have single-region selection only), so the binding is ours.
- **Banners the server sends during authentication** (`SSH_MSG_USERAUTH_BANNER`: legal notices, PAM's "your password will expire in 3 days") are written at the top of the terminal as grey `●` notice lines when the first shell opens. Banners from every hop of a jump chain are collected, in arrival order. They are shown once; further shells on the same connection do not repeat them. Banners come from an **unauthenticated** peer, so in each line control and bidi control characters are replaced with `?` before it reaches the terminal (which defuses escape sequences; the rule is the SSH librarys public `PeerText.Sanitize`, which the keyboard-interactive prompt dialog uses too), with at most 64 lines (the rest fold into a single `…` line) and 512 characters per line. The host used to leave this callback unset, and banners were silently dropped.
  **Lines sent before the identification string** (some bastion hosts and network devices print a legal notice or a maintenance announcement before the SSH identification string, SSH library spec 02 §3) are handled the same way: as one banner, placed before that hop's authentication banners (since 2026-10-05; before that the SSH library collected them but the host did not take them).
- **Highlights from the user's output rules** (since 2026-10-25): highlight rules from Settings → Output Rules (§14.6) sit above the built-in semantic highlighting and combine property by property — a text color set by a rule wins (even over colors a program set explicitly with SGR, since those are the characters the user asked to mark);
  cells without a rule text color keep the built-in semantic coloring, and URL / IP underlines are still drawn; background and bold change only where set; selection, search and same-text-as-selection backgrounds stay on top. Among user rules the first match per column wins, so rules higher in the list win.
  Colors come only from the current color scheme's 16 ANSI colors (rules store palette indices); like the built-in highlighting, main screen only. Rules run only on new or changed lines — lines unchanged between frames run no rule at all. Auto-answer is in §5.2.3.
- See §7 “Disconnected State” for the idle/disconnected overlay, and §7 “Connecting State” for the connecting one.

### 5.2.1 Fill a shared password (#628, 2026-10-15; entry shape redone 2026-10-16)

- There is no persistent entry point. When the current cursor line on the live primary screen is a conservative pattern such as `Password:`, `Enter password:`, `[sudo] password for …:`, sudo-rs’s `[sudo: authenticate] Password:` (sudo-rs is the default sudo since Ubuntu 25.10; the leading bracketed sudo / su / doas tag is accepted in all its forms, but only for those names, so a shell prompt such as `[joe@host ~]` does not count), localized sudo’s `[sudo] joe 的密码：` / `[sudo] joe のパスワード:` / `[sudo] joe의 암호:` (since 2026-10-31 the localized “X 的密码 / X のパスワード / X의 암호” form is recognised only after a `[sudo…]`-family tag; passwd’s new-password prompts such as 「新的密码：」「重新输入新的密码：」 and Japanese 「現在のパスワード:」 are not recognised, just like English `New password:`), doas’s `doas (joe@host) password:`, git/svn’s `Password for 'https://host:8000':` (the object after `for` may contain colons when it is quoted as a whole, #638), SSH’s `…'s password:`, or the supported Chinese password prompts, the shared-password list pops up directly against the bottom edge of the cursor row (WindTerm style); when there is no room below (for example, the prompt is on the last row of a maximized window), it flips above the cursor row's top edge, so the prompt line is covered in neither case. A 2px gap separates it from the cursor row on either side (the gap is built into the anchor rectangle rather than `VerticalOffset`, because the offset is not mirrored on flip and would push a flipped list into the prompt line; the command completion popup does the same since 2026-10-24 — before, it used `VerticalOffset` and a flipped popup overlapped the cursor line by 2px). A long prompt that auto-wrap folded over several rows in a narrow pane is matched as the whole logical line (looking back only through wrapped rows on the active screen); a cursor row that is a complete prompt by itself still matches. The command-palette action “Fill shared password” is the manual version of the same entry and does not depend on detection; it first hands keyboard focus back to the terminal, so Up/Down/Enter/Esc reach the list. Opened on a line that is not a recognised password prompt (an ordinary shell prompt, an editor, another program), the manual version fills without pressing Enter, and the warning at the top switches to the “fills only, no Enter” line (see the Enter condition below). No default shortcut is assigned; Ctrl+Shift+K still clears the screen. Right-click paste is unchanged; no plugin-owned terminal contract is added.
- The list shows only password-authentication shared credentials that have a stored password, labelled with name and username. The risk notice is permanently shown above the list (the separate confirmation step is gone); a status line above the list reports “no usable credentials” or an error. When listing fails, it says so truthfully (“Unable to read or decrypt the credential. Nothing was sent.”) and writes a Trace entry, instead of claiming there are no credentials. Listing reads metadata without decryption. **Clicking a row reads, fills and submits it immediately** (an Enter follows the password; since 2026-10-24 — before that only the password was sent and the user had to press Enter, so keys typed right after picking ended up in the password). **Enter condition** (since 2026-10-31): the cursor line (the whole logical line or the cursor row itself, the same test that opens the list automatically) is a recognised password prompt, and OSC 133 shell integration does not report the shell idle at its own prompt (`IsAtShellPrompt`: no command running and the cursor's logical line carries an A / B mark); unless both hold, the password is filled without Enter — it stays on the command line, so Ctrl+U can still take it back if it went to the wrong place. There is no confirm button and no search box, because keystrokes typed at a password prompt must reach the PTY unchanged (they may well be the user typing the password by hand); long lists are scrolled.
- Keyboard (#638, 2026-10-19): while the list is open, Up/Down only move a highlight (Down starts at the first row, Up at the last, wrapping around) and neither read nor send anything; Enter fills the highlighted row the same way, submitting it under the Enter condition above, then the list closes and focus returns to the terminal; any later key lands after the submit. No row is pre-highlighted when the list opens: Enter pressed before any arrow key goes to the terminal unchanged (the user’s own submit, even of an empty password) and never turns into “send the first password”. While a read is in flight (list disabled), Enter is swallowed and Up/Down do nothing — a quickly repeated second Enter would otherwise overtake the password to the remote end, submit an empty password, and the keystroke would also invalidate the pending fill. Esc closes the list as before. At a password prompt an arrow key would only push an escape sequence into the password, so Up/Down always belong to the open list; typed text still goes straight to the terminal.
- The popup must not be focusable and must not take keyboard focus away from the terminal; doing so would let the terminal-lost-focus rule close the popup before the click lands. The keys above are intercepted by the terminal tab in the tunnel phase, just like the command-completion popup’s. Detection is heuristic and cannot reliably distinguish every log from a real question; a false positive only costs one ignorable popup.
- Nothing pops up on an alternate screen (vim/htop), in a history viewport (scrolled back), or when there is no focus, no active tab, an inactive window, or a view that is not in effect. OTPs and private-key passphrases are not detected. The popup closes on tab switch, focus loss, scrolling, window deactivation, disconnect, rebinding of the completion wiring to another tab, and detachment from the visual tree. Closing it (or pressing Esc) invalidates the request; once the user has started typing at that prompt line (or has dismissed it with Esc, or already filled it), it does not keep reopening — only a new prompt line offers the entry again. Keys that invalidate the request without changing the tracked input (Backspace or Enter at an empty prompt, and so on) close the list right away without counting as a dismissal, so no list is left behind where every row only answers “expired”. Outbound input that does not come from the keyboard (bytes forwarded from another tab in the synced input channel, the NUL sent by anti-idle, program injection) likewise invalidates the request and closes the list without counting as a dismissal (since 2026-10-24); the one exception is while a row has been picked and its password is being read — the list then stays, and the fill answers “expired”, saying that nothing was sent. When a wrong password brings the prompt back in place (the prompt line under the cursor is a new one), the list reopens for the new request.
- The request binds the original tab, bridge, connection generation, cursor line/column, screen generation, and session encoding. Switching active tabs (even away and back), reconnecting/disconnecting, closing, input, prompt-context changes, or closing the popup permanently invalidates it. Late results are not sent.
- The password bytes, encoded for that session and followed by one Enter (the same bytes the Enter key would send right now, CR LF when the remote enabled newline mode), enter the bridge’s ordered queue as a single write. That Enter is appended only under the Enter condition above: it is decided when the list opens (`SharedPasswordFillRequest.SubmitsWithEnter`), and whether the shell is idle at its own prompt is checked again right before sending. The outbound-input generation is checked atomically with queueing; intervening sync input or other injection rejects the request. The operation bypasses the clipboard, TypedInput, sync-input broadcasting, completion tracking, and input diagnostics. Control characters, unencodable text, missing/inapplicable credentials, and decryption failures are rejected; retained ciphertext from a failed decrypt is never sent as a password. Queueing is not a remote-delivery guarantee.
- **Security limit:** neither the SSH channel nor prompt text reliably proves that remote echo is disabled, and EchoSuppressor is not such proof. A remote echo may expose the password in the screen, session log, or recording — which is why the risk notice stays visible above the list. This feature does not change at-rest encryption/keychain storage or handle pre-login structured SSH authentication callbacks.

### 5.2.2 Command blocks: copy / rerun (OSC 133, 2026-10-24)

- **Prerequisite**: the remote shell has the snippet from Settings → Terminal → “Sessions” → “Shell integration (command blocks)” installed (bash 4.4+ / zsh; fish not verified). The snippets changed on 2026-10-24 — PS2 now also ends with B; zsh's A moved from `precmd` to the start of `PS1`, and a stray backslash is gone — so **anyone who installed the old snippet has to copy it again**: with the old zsh snippet no command block ever forms and every prompt is followed by an extra `\`; with the old bash snippet single-line commands work, but multi-line ones can only be copied, not rerun.
- **Entry point**: right-click the terminal's left gutter (line numbers / timestamps / command markers / folds — any column) on any row of a command block (prompt row, command row, output row); the menu starts with “Copy this command” and “Rerun this command”, and below a divider come the gutter toggles as before. Right-clicking above the first prompt, or in a session without the integration, shows the menu as before. Right-clicking the text area still pastes. There is no shortcut and no new setting.
- **“This command”** is the on-screen text between the end of the prompt (B) and the start of execution (C): text broken by auto-wrap is joined back into one line; multi-line commands keep their lines, with continuation prompts (`> `, `for> `) cut off when the snippet marks them with B; trailing whitespace is removed from each line, leading whitespace kept (bash's `HISTCONTROL=ignorespace` relies on it); control characters are stripped (and when any are, both items are disabled, see the next item). The start and end columns move along with reflow on resize, so after narrowing or widening the window the extracted text is still exact.
- **Both items are disabled** (the block is there, but no command can be cut from it) when: an old integration script sends no B; the command has not been submitted yet (still being typed), or was canceled half-way with Ctrl+C; Enter was pressed on an empty prompt; the text between B and C contains control characters or non-space text concealed with SGR 8 (`CommandInput.HasHiddenText`, usually a block forged in output; since 2026-10-31) — what would be copied or rerun is not what is on screen. Syntax-highlight colours and concealed spaces do not count.
- **Rerun** sends the command text plus one Enter to **the same terminal**, and is only available when the remote side sits at an **empty new prompt**; otherwise it is disabled: the previous command is still running (no end marker yet), a full-screen program (vim / less / htop and other alternate-screen programs) is active, something has already been typed on the command line, or it is inside a multi-line command's continuation; whatever cannot be decided (no B, a prompt exactly filling a row) counts as not idle. It is checked when the menu opens and again on click. A multi-line command whose continuation lines carry no B cannot be rerun (the continuation prompt would end up in the command, and `> filename` would truncate the file), though it can still be copied.
- **Multi-line commands** follow the quick-command rules (§14.4): the whole text is sent as a bracketed paste followed by Enter, and the shell runs it only once it has all of it; when the remote side has bracketed paste off, a “Rerun a multi-line command” confirmation comes first (first five lines previewed, button “Rerun this command”), governed by Settings → Terminal → “Confirm multi-line paste”.
- A rerun goes to this tab only, never through the synced input channel; it opens no command completion; it is recorded in the completion history as usual. It runs at the **current** prompt: after a `cd`, or an `ssh` into another machine in between, it runs in that directory / on that machine.

### 5.2.3 Output rules: auto-answer (2026-10-25)

Rules from Settings → Output Rules (§14.6) whose action is "Auto-answer" send a fixed text (or just Enter) for the user when a prompt appears. There are none out of the box. The rules:

- **Remote output only**: only what the bridge delivers is checked (the control's `FeedRemote`); prompts the host writes into the terminal, local echo and recording playback do not count.
- **Only the logical line the cursor is on, main screen** (wrapped lines joined): when a program stops for input, the prompt is on the cursor's line; output that has scrolled past (`cat` of a file that happens to contain "Continue? [y/N]") is not answered. The cost is therefore one line per batch, regardless of how much output arrives. The alternate screen is never answered. A logical line any of whose physical rows carries an OSC 133 A / B mark (a shell prompt) is not answered (since 2026-10-31): when the last line of `cat` / `printf` output has no trailing newline, bash draws its prompt on that same line and the text still looks like a question; the check is repeated when the timer fires, so a prompt arriving in a later batch cancels an answer already scheduled. Without shell integration this cannot be detected, so anchor the rule's pattern at the end of the prompt (e.g. `\[y/N\] ?$`).
- **Not while typing locally**: if there has been local input since the last Enter (typing, pasting, local echo, quick commands, characters forwarded by a sync channel) that line is not checked; it is checked again after Enter — the echo of `echo "(yes/no)"` typed at a shell prompt is not a question, and a prompt the user has started answering is not answered again.
- **Wait for 300 ms of quiet** before answering, restarting the timer on new output: when a batch happens to end mid-line, the cursor can briefly sit on a line that looks like a prompt.
- **One answer per logical line**: a line erased completely counts as a new one (a new question on the same line after a clear is answered); on a column change the answered flag moves with its logical line to the reflowed head row (since 2026-10-31): answered lines are not answered again, and answers still waiting for quiet or cooldown are still sent.
- **Cooldown and per-minute limit** (counted per rule): the cooldown is 2 seconds by default, and a new prompt during it is postponed until it ends, not dropped; the per-minute limit is 6 by default — when reached, that line is not answered, a notice appears once, and answering resumes after the one-minute window, so a remote that floods prompts cannot make it flood answers back.
- When one line matches several auto-answer rules, only the first one in the list answers.
- **Sending** takes the same programmatic injection path as quick commands: encoded with the session's charset, Enter encoded per the terminal mode, marked as programmatic so no completion pops up; **not forwarded to sync input channels by default**, only when the rule ticks "Also forward to sync channels"; nothing is sent while disconnected. The answer goes through the terminal's background input path (`WriteBackgroundInput`, since 2026-10-31): typed-input events, line tracking, sync forwarding and invalidating an open shared-password list all happen as before, but the user's selection is not cleared and a scrolled-back view is not pulled to the bottom (quick commands still do both).
- **Password guard**: an auto-answer rule whose pattern literally mentions password-like words, or that would match one of the common password prompts (sudo, sudo-rs, ssh, git, key passphrases, PINs, CJK sudo), cannot be set to auto-answer — passwords go through shared passwords (§5.2.1).
- **Notice**: a floating bar centered at the top of the terminal area (`bg-surface` fill, 1px `border-secondary`, radius 4, `VelaShadowWindow` shadow, a 12px `zap` icon in the accent color on the left, 11pt `text-secondary` text)
  reading “Auto-answered: {rule name}” / “"{rule name}" hit its per-minute limit; auto-answer paused”, disappearing after 4 seconds and ignoring the pointer.
- A plugin's own terminal view takes only global highlights and runs no triggers; plugin protocol tabs going through the bridge (serial / Telnet) get both.

### 5.3 In-Terminal Search Bar

- Triggered by the menu bar `search` button (§4A.2) or Ctrl+F. A search field slides in from the top of the terminal: input + previous/next + match count + close. Matches are highlighted and Enter jumps to the next match.

---

## 6. File Browser / SFTP (Lower Right Area, Default 220px, Collapsible/Resizable)

- **Header (36px)**: left = clickable current-path breadcrumb for level-by-level navigation, then a pencil button (or Ctrl+L) that switches to manual path entry (Enter navigates, Esc cancels; `~` and `~/…` expand to the login home directory, while `~user` and `~user/…` are expanded by the server (SFTP's `expand-path` / `home-directory`, since 2026-10-05), and anything that cannot be expanded is passed to the server as is so it reports the real error), then a `copy` button that **copies the full remote path of the current directory in one click** (the same command as “Copy Current Folder Path” in the empty-area context menu — previously you had to enter edit mode before you could select and copy it), and finally refresh; right = icons for view switching, upload, new folder, hidden-file toggle, search (`search`, since 2026-10-24, only on panels of SSH sessions; see “Remote search” below), go up (`corner-left-up`, since 2026-10-25; next to Refresh, does nothing at `/`; tooltip “Go up (remote pane)”, while the dual-pane document's local pane has “Go up (local pane)”), and so on. To the left of the upload button is the free space of the file system holding the current directory, “12 GB free of 50 GB” (since 2026-10-05, `VelaTextMuted`, monospace 10px; from SFTP's `statvfs@openssh.com`, refreshed on entering a directory and on refresh; not shown when it cannot be queried: servers without the extension, FTP, plugin protocols).
- **Follow terminal directory** (the `terminal` toggle on the right of the header, tooltip “Follow terminal directory (sync to the terminal's current path)”, accent-coloured when on): while on, the list jumps to wherever the terminal `cd`s. The source is the working directory the shell reports itself (OSC 7 / 633; OSC 1337 and OSC 9;9 configured by the user in their rc files are recognised too). After an SSH session connects, a reporting hook matching the remote shell (bash / zsh / fish / dash and other POSIX sh) is injected silently; nothing is injected when “Settings → Terminal → Report terminal working directory” is off, when the shell kind cannot be detected, or on Windows peers (cmd / PowerShell).
  - **Keeps following after switching user or shell** (since 2026-10-19, #637): the hook lives only in the login shell, so the new shell started by `su - node`, `sudo -i`, `sudo su -` or just typing `zsh` does not have it — following used to stop there and only came back after `exit`. Now, whenever Enter is followed by about 0.4 s of terminal silence without a directory report, the process table is read once over a separate exec channel on the same connection (`/proc` on Linux, `ps` on macOS / BSD) to see whether this connection's terminal foreground is an interactive shell sitting at its prompt that has not been hooked yet (following the extra terminal that sudo opens with its default `use_pty`); if so, the hook for that shell kind is injected once, again without leaving anything on screen. The login shell is identified from the process tree (the exec channel and the shell channel hang off the same sshd session process), not from anything the shell reports — a report could be forged by anything printed in the terminal.
  - Not injected when the foreground is a password prompt, a running script or command, a REPL or a program on the alternate screen (vim, less, top), `ssh` to another machine or `docker exec` (those directories are not on the machine SFTP sees), or a shell inside tmux / screen; if the user presses a key or the terminal prints anything while waiting or probing, that round is dropped and the next Enter looks again. Hooked shells report at every prompt, so ordinary Enters never reach the probe.
- **Space precheck before uploading** (since 2026-10-05): when the files of a batch uploaded from this machine (minus resume offsets) add up to more than a normal user can still write on the target file system, a yellow confirmation “Not enough space?” comes first (“The target file system has X free, but these files take Y. They may not fit — running out of space midway leaves an incomplete file on the server. Upload anyway?”, buttons “Upload anyway” / Cancel); Cancel uploads nothing. When usage cannot be queried, nothing is held back.
- **Upload picker** (opened by Upload; drawn in-app, so files and folders share one list and can be mixed in one multi-selection, marquee included):
  - The path bar uses the same gestures as the header's path bar above: the pencil button at the end of the breadcrumb, a click on the blank space after the breadcrumb, or `Ctrl+L` switches to a text box (pre-filled with the current path, fully selected); Enter navigates, Esc only closes the box (press it again to close the picker), losing focus cancels; when navigation fails the box stays open with the text untouched and the reason in the red bar. It accepts surrounding double quotes (Explorer's “Copy as path” adds them), `%VAR%` environment variables, `~` / `~/…`, paths relative to the current directory and `..`; a bare drive letter such as `D:` means `D:\`. A path to a file opens its folder and selects the file, so one more Enter uploads it (since 2026-10-12).
  - **Type to locate** (as in Explorer, since 2026-10-12): with focus outside the text box, typing the first letters of a name selects the item starting with them, scrolls it into view and moves focus to it; characters typed within one second build one prefix (typing `fxyc` lands on `fxyc.txt`), and repeating one letter cycles through the items starting with it; case-insensitive, and `..` never matches. Locating replaces any earlier multi-selection.
  - Enter in the list follows the same rules as a double-click (since 2026-10-12): with a single folder (or `..`) selected it opens that folder and focuses the first row of the new list; with files or several items selected it uploads them. Previously the list row swallowed Enter, so it never reached the Upload button.
  - **Window size** (since 2026-10-13): 920×640 by default (previously 672×552), resizable from all four edges and corners, minimum 560×380; when it doesn't fit the screen it is clamped to the work area and re-centred on opening.
  - **List / icon views** (since 2026-10-13): at the top-right of the title bar, just left of the close button, sits a segmented switcher (`list` | `layout-grid` joined into one control; the active segment gets a `VelaAccentDim` fill and a `VelaAccent` icon; tooltips “List view” / “Icon view”).
    - **List**: the existing layout — 28px rows with name + size + modified, with a column header. The default view.
    - **Icons** (Explorer's “Medium icons”): 108×96 tiles laid out left to right, wrapping when a row is full, no column header; a 40px icon above the name, which takes at most two lines (ellipsis beyond). Folders are amber (`VelaFileFolderIcon`); file icons carry the upper-cased extension (at most 4 characters, e.g. `PFX`, `JPEG`; omitted when there is none or it is longer); `..` uses `corner-left-up`; size and modified time move into the tooltip. Selection / hover fills are 4px-rounded blocks with a 2px gap between tiles.
    - In the icon view the marquee selects every tile its rectangle touches, in two dimensions (blank space right of the last column or below the last row selects nothing); ←/→ step tile by tile and continue on the next row, ↑/↓ jump a whole row keeping the column (landing on the last item when the cell straight below is empty but another row exists), PageUp/PageDown move a screenful, Home/End go to the ends; type-to-locate, Enter and double-click follow the same rules as the list. In both views PageUp/PageDown now move the selection (previously the list only scrolled and left the selection in place).
    - Switching views keeps the selection and returns focus to the file list. Size and view are saved on close (one record shared by all sessions, kept in the document store's `windowLayout`, not on the settings page) and restored next time.
- **Column header (26px, `bg-surface`)**: `Name(280) | Size(100) | Permissions(120) | Modified`. Columns can be sorted by clicking. The dual-pane document's local pane (Name / Size / Modified) sorts on click too (since 2026-10-25; before, its headers were plain text). The rules are the same in both panes: folders always come first, clicking the same column again reverses the direction, a new column starts ascending, and the active column shows ▲ / ▼.
- **File list (`fill_container`, scrollable)**: 28px per row. Icon (folder/file type) + name + size + permissions (`drwxr-xr-x`) + time. The first row may be `..` to return to the parent directory.
- Interaction:
  - Double-click folder = enter; double-click file = open it as “Settings → File Transfer → Double-click a file” says (the system default app by default); see the next item.
  - **Remote editing: three ways to open** (#396, since 2026-09-08). The “open” entries look alike, yet one used to upload
    changes back and another did not (double-click never watched the local copy at all); now an entry only decides
    **who opens the file**:
    - **Right-click “Open”** (the menu's primary action): downloads to `%TEMP%/VelaShell/builtin-edit/<8 chars>/` and
      shows the built-in editor (a modeless window). Files over 5 MB are not opened; the message says to download and edit
      locally. Uploading back is the editor's Save / `Ctrl+S` — every save uploads once, with no file watching.
      The temporary copy is deleted when the window closes; copies left behind by a crash or a killed process are swept in
      the background after the next start (since 2026-10-24: only subdirectories of `builtin-edit` named with 8 hex digits,
      deleted once neither the directory nor anything in it has changed for 7 days; what cannot be deleted waits for the next
      start; no message). The two external-program paths (`remote-edit`, see below) are cleaned up separately, by the state
      marker each session leaves (see “Crashes and drafts” below).
    - **Right-click “Open with default editor”**: hands the file to the program configured in “Settings → File Transfer →
      Default editor” (a full path or command name on Windows, a command name on Linux, an app name on macOS, started with
      `open -a`). When none is configured, a “No default editor configured” dialog offers to jump straight to that settings page.
    - **Double-click a file**: one of three per “Double-click a file” — system default app (default) / built-in editor
      (same as right-click “Open”) / configured editor (same as “Open with default editor”). The default stays the system
      default app so existing users' muscle memory is untouched; it uploads changes back as well.
  - **Automatic upload-back** (the system-default-app and configured-editor paths; the built-in editor is covered above):
    - The download goes through the transfer overlay (progress, throttling, cancellable) into an exclusive subdirectory of
      `%TEMP%/VelaShell/remote-edit/<8 chars>/`, then the chosen program opens it.
    - The local copy is watched: writes, size changes and renames all count (many editors' “safe save” writes a temporary
      file and renames it over the original). After a save, a 600 ms debounce, then a wait until the file is readable (up
      to 3 more tries × 300 ms while the editor briefly holds a lock), and the copy is uploaded to its original path
      through the transfer overlay, which shows a row as usual; a save made during the upload is uploaded right after it.
    - Opening the same remote file again (same session, same path) **reuses** the session: if the local copy is clean it
      first re-downloads the latest content; if it has changes not yet uploaded, or the last upload failed, the local copy
      is not overwritten and the message reads “X still has changes that were never uploaded. Kept your local copy at
      (path) instead of overwriting it.”
    - **The editor process is not tracked** (a single-instance editor's launcher hands the path to the running instance
      and exits; whether a process has exited says nothing about whether the file is still open). A session ends only in
      two cases: when the panel it belongs to is torn down — an SFTP document tab (single or dual pane) or a terminal tab
      closes, finishing the last upload before SFTP / SSH is disconnected (waiting up to 2 minutes; terminal tabs since
      2026-10-25 — before, sessions opened from the terminal side panel's file browser had no such exit and stayed until the
      app exited); when the terminal drops, its side panel is torn down and its editing sessions end with it, and since the
      connection is already gone, anything not uploaded is kept as a draft with its location reported; and when the app
      exits (waiting up to 5 seconds).
    - A failed upload: the transfer row turns red, the file panel reports “Remote update failed (file name): reason”, and
      the local copy stays. Anything still not uploaded at wrap-up keeps its copy and temporary directory, with the message
      “X was not uploaded. Your edited copy is kept at (path).”; the temporary directory is deleted only once everything
      is uploaded. There is no permanent “editing” status anywhere in the UI (removed 2026-09-08).
    - **Crashes and drafts** (since 2026-10-25): each session writes `.vela-edit.json` in its own subdirectory (clean / has
      changes not uploaded, plus the file name, remote path, server name and connection profile id; when clean, the length and
      modification time of the copy that matches the remote). It is written clean on opening and on a clean re-download, dirty
      as soon as a save is seen, and clean again once an upload succeeds with nothing new pending. At start-up a background scan
      (only subdirectories named with 8 hex digits, skipping sessions that are open) deletes copies whose marker says clean
      **and** whose length and modification time still match, once untouched for 7 days; those with changes not uploaded,
      without a marker (left by older versions), or whose copy no longer matches a clean marker are all **drafts, never deleted
      automatically**, and each is listed in the message center ("External edit not uploaded", with the file, server and remote
      path, and where the local copy is; see [message-center-and-feed.md](message-center-and-feed.md) §4B) with the action
      "Open folder" (command `files.remoteEditDrafts.open`: with a single draft it opens that subdirectory, otherwise the
      `remote-edit` root); once a draft has been deleted, its message is removed at the next start. A draft cannot be
      re-uploaded from the message center.
  - **The two remote-editing settings** (Settings → File Transfer → “Remote editing” section):
    - **Double-click a file** (`Transfer.DoubleClickAction`: `system` (default) / `builtin` / `editor`; any other stored
      value reads as `system`); its description reads “The built-in editor uploads when you save; the other two upload every
      save while "Upload automatically after editing" is on” (corrected on 2026-10-25; it used to say “All three open a local
      copy and upload it back on every save”, which did not hold with the switch off).
    - **Upload automatically after editing** (`Transfer.AutoUploadOnEdit`, on by default): off = the local copy is not
      watched at all, nothing is sent back, and nothing is uploaded when the tab closes or the app exits either. It only
      governs the two external-program paths; the built-in editor's Save is itself the upload and is not affected. The
      settings page describes it as “Applies to the external editor and the system default app: when off, their local copy is
      not watched and nothing is sent back. Saving in the built-in editor always uploads”.
  - **Remote-editing diagnostic log** `<data root>/logs/remote-edit.log` (by default `~/.velashell/logs/remote-edit.log`;
    it moves with the data root when `--data-root` changes it): always on, one line per event as `time [stage] detail`,
    stages being open / watch / upload / close — enough to see whether the watcher got the event, whether the upload
    started and which step failed (this path fails as “saved, but nothing happened”, with no exception and no UI feedback).
    Past 1 MB the whole file is cleared and restarted; write failures are silent and never affect editing or uploading.
    Only the two external-program paths log here; the built-in editor does not go through it.
  - Drag a local file here = **upload** (triggers the file transfer component in §9); drag a list item locally = **download**.
  - **Type to locate** (since 2026-10-12; in the remote pane, the local pane of dual-pane documents, and the terminal sidebar's file browser): with focus in that pane, typing the first letters of a name follows the upload picker's rules (characters typed within one second build one prefix, repeating one letter cycles, case-insensitive, `..` never matches, the item is scrolled into view and gets focus); entering another folder starts a fresh prefix. Typing does nothing to a pane that doesn't have focus (the terminal, the other pane). When the rows are replaced wholesale (entering a folder, refresh, background auto-refresh), focus stays in that pane's list — on a row fully inside the viewport, without scrolling — so typing keeps working. With a single folder (or `..`) selected, Enter opens it (like a double-click); Enter on a file does nothing (double-clicking a file downloads and opens it, which a stray Enter should not trigger).
  - **Path bar of the dual-pane documents' local pane** (since 2026-10-12, shared with the upload picker): the pencil at the end of the breadcrumb, a click on the blank space after it, or `Ctrl+L` (with focus in the local pane; in the remote pane it opens the remote path bar) switches to a text box; Enter navigates, Esc / losing focus cancels, and a failed navigation keeps the box open. It accepts the same forms as the upload picker (quotes, `%VAR%`, `~`, relative paths, a bare drive letter); a path to a file opens its folder and selects the file.
  - **Hidden files and columns**: the terminal side panel's file browser and the remote panes of single- and dual-pane file tabs share “Show hidden files” and the six column switches under “Settings → File Transfer”: they open laid out per the settings, toggling from any panel's toolbar or header context menu writes the setting back, and saving syncs every open panel (file tabs since 2026-10-25; before, file tabs always started from the factory values and toggles there were not saved). Remote search's “Show in the file pane” on a dot file turns hidden files on for that panel only, without writing it back.
  - **Settings follow along**: after settings are saved, open file tabs (single and dual pane) pick up the new transfer options immediately (conflict policy, concurrency, retry count, double-click action…) without being reopened (since 2026-10-25). A batch that has already started is not revisited: same-name conflicts were resolved with the policy in force before it started and the concurrency limit stays at its starting value; the retry count applies from the next file on; throttling, preserve timestamps and fsync after upload are read fresh as each file starts.
  - **Name conflicts**: when an upload/download encounters an existing file with the same name, use the setting in §14 File Transfer “When a file already exists”: ask (confirmation dialog: overwrite or skip) / overwrite / skip / rename (`file (1).txt`).
  - Right-click = file context menu (download, upload, rename, delete, properties, copy path, new; on panels of SSH sessions the empty area also has “Search remote files…” and a directory row has “Search in this folder…”, see “Remote search” below). “New” includes **New Symbolic Link**: first ask for the link target path (prefilled with the row’s path when right-clicking a row), then ask for the link name (prefilled with the last segment of the target). The target is written verbatim; relative paths resolve against the link’s directory (`ln -s` semantics). On FTP, only servers that support `SITE SYMLINK` can create links; plugin protocols report that the operation is not supported.
  - **Properties dialog**: basic information (type, path, link target, size, modified time) + owner / group + the rwx permission matrix and octal input (kept in sync both ways, with a `chmod 640` echo beside them).
    - On SFTP sessions, Owner and Group are **editable drop-downs**: the suggestions are the user and group names from the remote passwd / group databases (the same lookup that turns ids into names in the list), and a numeric UID / GID can be typed directly. Input is read in `chown`’s order: as a name first, and as a number only if no such name exists. On SFTP-only accounts where the lookup is unavailable, the suggestions are empty and only numbers work.
    - Owner / group names in the list: sessions with exec read the whole table once (`getent`); ids missing from it (on SFTP-only accounts the whole table is missing) are filled in through SFTP's `users-groups-by-id@openssh.com` (since 2026-10-06, each id asked once per session), and only ids still unknown show as numbers.
    - Clearing a field means “leave unchanged”; a name the remote host doesn’t know is reported in the error bar on the spot and nothing is sent.
    - OK applies only what changed, **chown before chmod**: the owner change is the one most likely to be refused (a non-root user can only switch the group to one they belong to), so when it is refused nothing has changed yet; if chown succeeds and chmod is refused, the list is still refreshed.
    - FTP and plugin protocols cannot change ownership, so both rows are read-only there. No recursion or batch operation; for a symbolic link, the change applies to the target.
  - **File name encoding** (since 2026-10-24): SFTP file names of an SSH session are decoded with the session encoding at connect time — the encoding override on the connection dialog's Terminal page, or the global terminal encoding when there is none; a machine whose terminal already shows GBK correctly no longer shows its Chinese file names as a row of replacement characters in the file panel.
    - With a UTF-8 session encoding (the default), nothing changes: undecodable bytes show as replacement characters, but opening, deleting and renaming still hit the original bytes (lossless round trip).
    - With an encoding such as GBK, file names are decoded and encoded in it; bytes that are invalid in that encoding are not guaranteed to round-trip. An unrecognized encoding name falls back to UTF-8's lossless round trip.
    - The encoding is taken when connecting (a reconnect takes it again); switching the encoding temporarily from the status bar only affects the terminal, not a file panel already open. FTP and plugin protocols are not affected.
    - Remote search (below) follows it too: results are decoded with it, and the search location, file name and containing text are sent to the server encoded in it (since 2026-10-25).
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
    - When an FTP connection has a server time zone set (§13.1, FTP “Advanced” page, since 2026-10-24), the synchronize window (keep remote up to date included) and Compare Directories convert the times LIST reports to UTC with it, and by-day comparison truncates in the server's calendar; MLSD times are UTC already and are not shifted. The file panel's time column still shows the server's own clock.
    - **Keep remote up to date**: the list area becomes an activity log (newest first) and the button reads "Stop watching"; closing the window or the document stops watching.
    - Option meanings, execution order, and comparison rules are in section VII of [sftp-dual-pane-winscp-gap-analysis.md](sftp-dual-pane-winscp-gap-analysis.md).
  - **Directory bookmarks** (since 2026-10-14): the document toolbar provides "Save current directories" and "Bookmarks". Each bookmark stores a name, an absolute local directory, and an absolute remote directory for the current connection; the list applies both paths together and supports editing and deletion. "Manage bookmarks" opens a dedicated management window.
    - Saving normalizes the local path and remote POSIX path and de-duplicates by the normalized local + remote pair; saving does not check whether either directory currently exists. Leading and trailing spaces in the remote path are kept as typed (a POSIX directory name may begin or end with a space; the name is still trimmed). A relative path resolves against the remote pane's current directory; `~` and `~/x` expand with the remote pane's home directory and `~user` is expanded by the server. If expansion fails, the bookmark is rejected ("Could not expand “~”: the remote home directory is unknown. Enter an absolute path.") instead of being stored as "current directory/~/x".
    - Applying navigates both panes in parallel. A failed side keeps its previous path and shows that pane's error; a successful side is not rolled back. A bookmark is not deleted when a directory is unavailable.
    - Bookmarks are stored with the connection profile and included in connection export. For saved profiles they are persisted field by field: add, edit and delete run against the latest bookmark list in the store, and the on-screen list changes only after the write succeeds. Operations that save the whole profile (editing the connection, the write-back after connecting, pinning, dragging between groups, trusting a certificate, and so on) always keep the stored bookmarks, so they can no longer clear or roll them back; only a Gist sync pull and a JSON backup import (overwriting an existing connection) replace the whole profile: a Gist payload that carries the bookmarks field (an empty list included) brings its bookmarks, while a payload pushed by an older version that has no such field at all keeps the local bookmarks (since 2026-10-24; before, they were cleared); an overwriting JSON backup import follows the same rule: the file's bookmarks are taken only when it carries the bookmarks field (an empty list included), while a backup exported before directory bookmarks existed, without the field, keeps the local bookmarks (since 2026-10-25; before, they were cleared); CSV cannot carry bookmarks, so an overwriting CSV import keeps the local ones (as stored at that moment). The explorer's “Duplicate Connection” takes the latest bookmarks from the store (including ones added in a document after the tree loaded). Opening a document first loads the latest bookmarks from the store, so bookmarks just added from another document show up. Temporary profiles created from recent connections are valid only for the current document. Remote + remote documents, access history, grouping, and shortcuts are not offered.
    - The management window's add, edit and delete dialogs are owned by the management window itself (they work even when the document is not the active tab), and failures are shown in the window's bottom bar; the management window closes together with the document. Bookmark names and paths in the toolbar drop-down are shown literally, so `_` is no longer swallowed as an access key.
  - **Remote search** (since 2026-10-24, find / grep): only file panels of SSH sessions have it — the terminal side panel, the single-pane SFTP document, and the SSH pane of a dual-remote document; FTP and plugin protocols have no exec, so the entry points do not appear.
    - Entry points: `search` in the header, “Search remote files…” in the empty-area context menu (starting from the current directory), and “Search in this folder…” on a directory row (starting from that directory). One modeless “Search remote files” window per panel (760×560, minimum 560×380); when the panel goes away (tab closed, session disconnected) the window closes with it.
    - Fields: search location (defaults to the panel's current directory, must be an absolute path starting with `/`), file name (e.g. `*.log`; without wildcards it means “name contains”), containing text (optional, matched literally, files only), ignore case (on by default); at least one of file name and text is required. Enter starts the search.
    - Results arrive one by one; double-click, Enter or the row's “Show in the file pane” button: a directory is entered, a file has its directory opened and is selected and scrolled into view, a deleted one is not selected; a hidden dot file turns on “show hidden files” for that panel only, without writing it back to the settings.
    - At most 500 items and 2 minutes per search (fixed, not settings); hitting the limit or the time-out says plainly that the results are incomplete. “Stop” is always available, and closing the window stops it. Esc stops a running search first, and closes on the next press.
    - The remote login shell is checked first: bash, zsh, POSIX sh and fish can search; Windows cmd / PowerShell and csh-like shells get “not supported”, and when the shell cannot be determined (command execution disabled, SFTP-only account, time-out) it says so — in both cases **not a single command is sent**. User input is passed only as single-quoted positional parameters to a fixed script, never spliced into the command; searching from `/` skips `/proc`, `/sys` and `/dev`, and links are not followed.
    - **Session encoding** (since 2026-10-25): when the session encoding is not UTF-8 (e.g. GBK), results are shown in the session encoding and clicking one selects that row; the three inputs are encoded into the session encoding's bytes (octal escapes restored with `printf`, since the exec request itself can only be UTF-8),
      and the whole search matches bytes in the C locale — musl systems such as Alpine match nothing at all for a non-UTF-8 pattern in a UTF-8 locale; the price is that `?` matches one byte rather than one Chinese character, and ignoring case works only for ASCII.
      When the input has characters the session encoding cannot represent, that search is sent as UTF-8 as is (and most likely finds nothing).
    - **Names containing a newline** (since 2026-10-25): each counts as one result instead of two. `find -exec … {} +` hands the paths to a small batch script that ends each record with `\0`; `tr` swaps newlines and `\0`, and `head` then counts records
      (not `-print0`: BusyBox's `head` has no `-z` and its `grep` no `-Z`); names containing `\r` are shown as they are, too. On a very old BusyBox whose `find` lacks `-exec … +`, a probe detects it and the old line-based output is used
      (newline names still split in two, and a content search finds nothing).
    - The status line reports each outcome on its own: N items found / limit reached, incomplete / timed out, incomplete / stopped / session disconnected / not a POSIX shell / shell could not be determined / the server is missing find, head or tr (searching by content also needs grep) / the failure text.
  - Selected state `bg-hover`; multi-select (Ctrl/Shift) for batch operations.
  - Navigation uses “load first, commit later”: on failure, retain the original path, list, selection, and scroll position; after entering a new directory, clear the selection and return to the top.
  - Refreshing the current directory preserves selected items that still exist and the scroll position. Do not rebuild the list when content has not changed. For concurrent navigation, accept only the latest request result.

### 6.1 Multi-Terminal Sync Input Channel (replaces the original Broadcast Input Bar)

- Entry is in the tab context menu under “Sync Input”: four fixed channels A/B/C/D, distinguished by color (pink/blue/orange/green), plus “Leave All Channels”.
- A whole group or a multi-selection can also be opened from the explorer with synced input (since 2026-10-24): the new terminal tabs join the first free channel together; see §12 for the rules.
- Peer model: any user input in a channel (keyboard/IME/paste) is copied in real time to the other tabs in that channel. Switching to any tab in the channel preserves the same shared input behavior. Each tab can belong to only one channel at a time.
- Tabs in a channel show the channel letter (A/B/C/D, in the channel color) before the status dot in the tab header. A channel bar appears above the terminal: color swatch + “Input synchronized in channel X” + [Pause] [Leave Channel] [Close Channel].
  - **Pause**: this tab temporarily stops sending and receiving channel input; click again to resume.
  - **Leave Channel**: only this tab leaves. The close button on the right side of the bar has the same effect.
  - **Close Channel**: all tabs in the channel leave together.
- Forward directly to each target PTY (the bridge’s SendRaw), without passing through input events on the receiving terminal control. This prevents forwarding loops and does not trigger command-completion popups in non-focused tabs. Smart suggestions appear only in the tab where the user is typing.
- Disconnected tabs do not receive forwarded input, gated by `IsConnected`. Closing a tab automatically leaves its channel. Multi-target dispatch of quick commands no longer goes through a second channel forwarding step, preventing duplicate injection.
- Auto-answers from output rules (§5.2.3) are not forwarded through the channel by default (a rule can opt in); input forwarded by the channel counts as "typing locally" on the receiving side, so the receiving tab's triggers do not answer that line.

### 6.2 Remote + remote dual pane (#524)

Opened from the multi-select in §3 (exactly two connections): one file tab with each pane connected to a different machine. The tab title reads "left ⇄ right", its color bar and icon come from the left profile, and its status dot shows the worse of the two connections.

- **Connecting**: the two connect one after the other (each with its usual credential prompt and host-key / certificate confirmation), under a placeholder tab titled "left ⇄ right". **If either one does not connect, the whole thing is rolled back**: the one already connected is disconnected at once, a failure stays on the placeholder's failure card, and a cancel removes the placeholder; there is never a document with only one pane. Both connections belong to the document and are disconnected together when the tab closes; sessions already open in other tabs are not reused. Any combination of SSH / SFTP / FTP / FTPS / plugin file protocols works.
- **A plugin file-protocol pane** carries the protocol's own right-click actions (e.g. S3's "Copy share link"), and the tab icon is the one the plugin supplies (when the left pane is the plugin). The pane can only **receive** files from the other pane if the plugin also implements the SDK's `IProtocolStreamUpload`; otherwise it is a source only — the copy button toward it is disabled and drops onto it are refused.
- **The panes**: each is a full remote file browser (everything in §6). The server badge in each header carries that machine's identity color (the same one as the tab color bar) — with both sides remote, the path alone does not tell which pane is which machine.
- **Moving files between the panes**:
  - Toolbar "Copy to right ›" and "‹ Copy to left": moves the files and folders selected in one pane into the other pane's current folder;
  - dragging rows from one pane onto the other (only rows from the other pane of the same document are accepted; dragging within one pane is still not supported, see #474).
  - The data is **streamed through this computer's memory and never written to local disk**: the source is read sequentially while the target is written, so it takes about as long as the slower leg. The protocols have no server-to-server copy primitive, and running `scp` / `rsync` on A to reach B directly is deliberately not done (it needs B's credentials or a forwarded agent on A, and A must be able to reach B).
  - It shares the upload pipeline: conflict policy, concurrency limit, bandwidth limit (counted as upload), preserve timestamps, transfer log (direction logged as `RELAY`), and retry from the transfer popup after a failure (temporary failures are retried automatically first, see §9; relays do not go into the transfer queue, so a dropped connection still ends in a failure + “Retry”). The transfer popup shows the direction as `⇄`.
  - **Resume**: same rules as uploads (Settings → File Transfer → Resume). After a cancel or failure the partial target is kept; on retry, or the next time the same file is transferred, the target is verified to be a prefix of the source (the target's current length, backed off by one in-flight write window, tail compared) and writing continues from there. Resuming needs to seek within the source, so it **only happens when the source is SFTP**; when the source is FTP or a plugin protocol (sequential reads only), or the target is a plugin protocol (the SDK's streamed upload does not resume), the resume point cannot be verified and the file is **handled by the same-name conflict policy** (the same path as a verified mismatch) rather than silently sent again in full — which would overwrite an unrelated file of the same name without asking. With resume turned off, a failure midway follows the "clean up partial files" setting. If the source cannot be read before writing starts (deleted / renamed), the target is untouched and an existing file of the same name is **never** deleted as if it were a partial file. A cancel is reported as a cancel, not as "transfer interrupted, can be resumed".
  - Folders are recursive; links to directories nested inside them are not followed (same rule as downloads).
- **Compare directories**: compares the current level of both panes (by size and modification time, using the inferred time precision when either side is FTP; an FTP side with a server time zone converts its LIST times with it, and by-day comparison uses the left pane's calendar), selects the newer, unique or different entries in each pane, and shows the summary on the right of the toolbar.
- **Not provided**: the synchronize window and "keep remote up to date" (they are designed around local ↔ remote). With only “Restore sessions” on, startup does not reopen these tabs; with “Restore split layout” (§4C) on, they come back with the layout (as long as both profiles are still in the database).

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
died. The five cases below never reconnect on their own; the notice still shows, and coming back is the
user’s call via “Reconnect” (the same principle as “a tunnel the user stopped by hand is never restarted”):

- The user clicked “Disconnect” in the toolbar.
- The user typed `exit` / `logout` on the remote, i.e. the remote shell exited on its own (on the wire: the
  peer closed the channel normally).
- A local terminal’s shell exited.
- The very first connection never succeeded — authentication failures above all, where retrying just throws
  the same wrong password at the server a few more times.
- A reconnect failed on the host fingerprint: the user rejected it in the confirmation dialog, “Block and alert on
  fingerprint change” stopped it, or an automatic reconnect hit a fingerprint that needs confirming (see “Host
  fingerprint confirmation”: automatic reconnects never show that dialog). Retrying gives the same answer every time;
  the dialog comes up when the user clicks “Reconnect”, and once that connects, automatic reconnect works as usual again.

**Allow the algorithms with one click after a negotiation failure** (since 2026-10-24): when algorithm negotiation fails and the peer offers algorithms this version implements but has not enabled (the ones named by the “one more line on what to do” of §13.1), the terminal tab's failure card gets an outlined button between “Copy” and “Reconnect”: “Allow these algorithms and reconnect” (warning-colored icon; tooltip: “Appends … to the custom (category) list in the connection's SSH options (after the defaults), then reconnects.”).
- It first asks “Allow these algorithms?”: the text names the connection, the category and the algorithms line by line, and says only these are added, the other legacy algorithms stay off, and they are weaker than the defaults; buttons “Allow and reconnect” / Cancel. Cancel leaves the profile untouched and does not reconnect.
- After confirmation, **only the algorithms the diagnosis named** are appended with `+` to that category's custom list in the profile of **the hop that failed** (the destination, or the jump host found by following “Jump host”); the “Allow legacy algorithms” switch is not touched. A list that already replaces the defaults or starts with `+` gets them appended; one written with `-` / `^` is rewritten as the full list in effect right now with them appended (freezing the defaults at this moment). What changes is the stored profile (saved through the workflow, without the password typed this time), and for the destination hop the tab's own copy as well; then it reconnects in place. A temporary profile rebuilt from “Recent connections” is not in the database, so only this attempt changes; when the jump host's profile cannot be found, you are told to edit its SSH options by hand and nothing reconnects.
- No button when this version implements none of the peer's algorithms (typically an old device that only speaks CBC), or when the failure was not a negotiation failure.
- **Three more entry points** (since 2026-10-25; before, only terminal tabs had it):
  - **The failure card of document-style connections** (standalone SFTP documents, and the SSH / SFTP pane of a dual-remote document): the same “Allow these algorithms and reconnect” above “Reconnect”; after confirming, the stored profile is changed (for the destination hop also the copy the card's “Reconnect” uses), then the card's own retry runs, and the button goes away on retry.
    Only SSH / SFTP profiles get it — a plugin workbench reaching out through an SSH tunnel is not connected with this profile, so it does not. In a dual-remote document the failing pane is the one changed.
  - **The connection diagnostics window**'s issue card (§15): “Allow these algorithms and diagnose again” below the description (tooltip “… then runs the diagnosis again.”), confirmed with “Allow and diagnose again”; after confirming, the stored profile is changed and the diagnosis runs once more with a changed **copy** for the destination hop
    (the copy passed in by the caller is left alone); when a jump host's profile cannot be found or written, the reason appears below the button. If the store was changed, the explorer refreshes after the window closes. The confirmation is owned by the diagnostics window.
  - **The profile dialog's “Test”** failing (§13.1 feedback bar): “Allow these algorithms and test again” below the error (tooltip “Appends … to the custom (category) list in the SSH options (after the defaults), then tests again.”), confirmed with “Allow and test again”.
    The destination hop is the profile being edited, so **only the form changes**: the custom algorithm lists expand and that category gets the remedied value, and the confirmation ends with “This changes the list in this form only; it is stored when you click Save or Connect.” — other unsaved edits in the form are not saved behind the user's back;
    when the failing hop is a jump host, that host's own profile is changed and saved on the spot. When the error changes (a new test, a failed save) the button goes away.
  - All four share the confirmation title “Allow these algorithms?” and the same way of naming them; only the last sentence of the body (then reconnects / diagnoses again / tests again) and the confirm button differ.
  - Jump hosts are found the same way the connection chain is built: the first hop comes from the "Jump host" of the profile the connection uses (in the profile dialog, the one chosen in the form), the rest from the store — before, counting started from the stored copy, so a temporary profile rebuilt from "Recent connections" could not find its jump host.

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
- When the failure is an algorithm negotiation failure and the peer offers algorithms this version implements but has not enabled, the card gets
  “Allow these algorithms and reconnect” above “Reconnect” (since 2026-10-25; rules in “Allow the algorithms with one click after a negotiation failure” above).

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
   - After “Sessions” comes the **“Layout templates” group** (since 2026-10-24): one row per template with an `Enter to open` hint on the right; Enter opens that set of connections in the template's layout (§4C). Usage is tracked as `layout:{template Id}`.
3. **Divider**.
4. **“Commands” group**: command result rows with icon + command name (Inter 12) + keyboard shortcut on the right (it follows the current bindings: a changed shortcut shows the new keys, an unbound one shows nothing). Examples:
   - `New SSH Connection` … `Ctrl+N`
   - `Open SFTP File Manager` … `Ctrl+Shift+F`
   - `Open Settings` … `Ctrl+,`
   - (Extensions) `Open Tunnel Manager`, `Split Pane`, `Switch Theme`, `Record Session`, `Connection Diagnostics`, `SSH connection info` (since 2026-10-24, available when the active tab is a connected SSH session, §11B), `Session Health Check (All Connections)` (since 2026-10-25, `tools.healthCheck`, `activity` icon, every stored connection sorted by name, §11C), `Operations Orchestration`…
   - `Send Break` (session category, since 2026-10-05): available when the active tab is a connected SSH session (local and plugin terminals cannot use it); it sends a BREAK (RFC 4335),
     which is how serial console servers and network device consoles get into ROMMON / the boot menu. Whether the server performed it is reported in a toast: an ordinary toast when it did,
     a yellow one, “The server did not perform the Break (it may not support it, or the session has no terminal)”, when it did not.
5. **Footer (32px, `bg-input`)**: left = key hints `↑↓ Navigate` / `↵ Confirm` / `Esc Close`; right = result count `6 results`.

**Interaction logic**:
- A prefix switches modes: `>` commands, `ssh ` sessions, `@` files, `#` command history, `:` line navigation (optional extension).
- Move between results with `↑/↓` or `Ctrl+P/N`, execute the highlighted item with `Enter`, complete with `Tab`, and close with `Esc`.
- `Enter` on a session item connects and activates its tab. `Enter` on a command item executes the corresponding function, equivalent to clicking its entry point.
- Support recently used ordering and fuzzy scoring, with matched characters highlighted for subsequence matches.
- Besides the title, session items also match their tags → host → notes (since 2026-10-24; these keywords are neither shown nor highlighted): keywords only match contiguous substrings, a keyword starting with the query ranks above one containing it further in, tags outrank the host and the host outranks notes; overall such hits rank after contiguous-substring hits on the title and before subsequence hits on the title. An entry under “Recent connections” that matches a saved profile carries the same keywords.

---

## 9. File Transfer Notification Component ★User specified (Floating / Draggable / Auto-Hiding)

**Appearance condition**: **when an active file transfer task exists, or a batch has stopped and is waiting (see “Paused batches” below)**, show a floating panel in the **upper-right corner of the main window** (280px wide, `bg-surface` + small shadow + corner radius 6). It does not exist when there are neither.

**Structure**:
- **Header (32px, bottom divider)**: left = `arrow-up-down` (accent) + “Transfers” + count badge (`accent` background, such as `2`); right = `x` close, which only hides the panel and does not cancel tasks. **The entire header is a drag handle.**
- **Transfer entries, stacked vertically**:
  - First row: filename, such as `backup-2025.tar.gz`, + percentage on the right (`67%` accent) or status (`Complete` in green).
  - Progress bar (3px, `border-primary` background + proportionally filled `accent`).
  - Description row (9px muted): `142 MB / 212 MB • 4.2 MB/s • ↑ Uploading` / `1.2 KB • Complete • ↓ Downloaded`.
  - While waiting for an automatic retry (since 2026-10-24), the right side shows “Retry n/N” (muted) and the description row “Interrupted, retrying shortly”.
  - On hover, actions appear on the right: pause/resume, cancel, open containing folder.
- **Paused batches** (since 2026-10-24, above the transfer entries, in a scroll area of its own up to 160px high, one row per batch):
  - First line: the file name for a single file, “N files” for several + “Resume” (accent) and “Discard” (tertiary) on the right.
  - Second line (9px muted): direction `↑` / `↓` + server name + state — “connection lost, paused” / “not finished when VelaShell last closed” / “resuming…” / “connection deleted, cannot resume” / “could not resume: reason”.
  - While resuming, neither button is shown; when the profile was deleted, only “Discard” is.

**Interaction logic (★critical)**:
1. **Draggable**: hold the header to move freely within the main window. Remember the position for the next appearance and keep it within the visible window bounds.
2. **Automatic appearance**: when a new transfer starts, fade in from the upper-right if the panel does not exist.
3. **Real-time progress**: stack multiple tasks vertically; the badge count equals the number of active tasks.
   - **SFTP uploads** count bytes from the continuous-confirmation waterline (`DurableLength`) after each `WRITE` receives `STATUS OK`, not bytes merely enqueued into the pipeline. While `Flush` drains in-flight writes, sample every 200 ms and throttle UI reports to 100 ms. Force an update when confirmed bytes reach the total, and report the full frame only once (when both a `Flush` sample and the last write callback hit the total, only the first passes, and successful finalization does not add another), but **100% is not `Completed`**: completion still waits for optional `fsync` and successful `CLOSE`; a failure during finalization remains a failure. For uploads / downloads a plugin starts through the remote file capability, the full frame (which plugins treat as "done") is not forwarded with the host's progress; it is sent once after the transfer task succeeds, and never when it fails.
   - For **resumed SFTP uploads/downloads**, the percentage includes the verified prefix. Speed is `(cumulative transferred bytes − verified prefix) / elapsed time since verification`, still the **whole-transfer average for this attempt**, not an instantaneous rate; neither the existing prefix nor verification time contributes.
   - **Sub-byte rates** (`0 < rate < 1 B/s`) must format safely as `B/s`; they must never index outside the units array and block UI updates.
4. **Automatic disappearance**: **when all transfer tasks complete, or all are canceled/failed and acknowledged, the panel disappears automatically**. After all tasks complete, it is recommended to remain for about 3 seconds showing the “Complete” state before fading out. If a new task is added during that period, reset the timer. While paused batches exist the panel does not hide by itself (`x` still collapses it).
5. **Failures and retries** (since 2026-10-24): temporary failures (dropped connection, time-out, channel closed) are retried automatically on the same connection as many times as Settings → File Transfer → “Retry failed transfers” says (0–10, default 3), waiting 1 s and doubling up to 30 s; meanwhile the entry goes back to waiting and keeps its transfer slot. With resume on and bytes actually sent in that attempt, the retry continues from the partial file on the target; if not a byte moved it starts again from the original offset; with resume off the whole file is sent again. Permission denied, disk full, missing paths, a server refusing the write and user cancellation are not retried; an entry that still fails after retrying reads “Still failing after N retries: reason”. Same-session copies are not retried, and nothing is retried on the original session once its panel is gone.
   - When the retries run out or the connection is gone, an upload / download batch started from a file panel **stops as a whole** and becomes one “paused batch” row; the remaining files do not start (instead of failing one red row after another). A permanently failing item is still marked red and the batch goes on. Other transfers (dual-pane relays, directory sync, panels without a connection profile) still end in a red failure + “Retry”.
   - Mark failed tasks in red and show “Retry”. Clicking `x` manually only collapses the panel; tasks continue in the background and can be reopened from the status bar or command palette.
6. **Resume / restart** (since 2026-10-24): paused batches are kept in the local SonnetDB collection `transfer_queue` (only the files not yet settled, with direction and paths, not the bytes transferred); batches unfinished at exit are restored as paused on the next start (“not finished when VelaShell last closed”) and the panel shows up, but **no server is connected at startup**.
   - “Resume” looks, by the batch's connection profile, for a file tab already connected to it, then for a terminal tab that has opened its file panel; failing both it opens a new file tab as usual (login prompt, fingerprint check and failure card as usual), and if that does not connect the row stays for another try. Each file is probed for resume first: a partial file continues where it stopped, a file not started is sent whole; the conflict policy is not asked again (it was settled when the batch was planned; when the start offset does not verify, “Ask” counts as overwrite). An item whose local or remote file disappeared or lost its permissions in the meantime is marked red as a permanent failure.
   - When the profile was deleted, the batch can only be discarded — the host is never guessed from the recorded server name.
   - “Discard”: the remaining files are not transferred, the finished ones are unaffected; partial files are not deleted (with resume on they are what resuming needs, with resume off they were handled by “Clean up partial files” at the moment of failure).
   - Not in this queue: dual-pane relays (they need both connections at once), same-session copies, directory sync (its plan has to be confirmed in the preview again), remote-edit uploads, and transfers started by the AI plugin (they keep their own task ledger with per-transfer approval and promise no recovery across restarts).

---

## 10. Tunnel Management Panel ★User specified (opened by clicking “Tunnel”)

**Trigger**: click the menu bar `route` (Tunnel) button (§4A.2), select Tools → Tunnel Manager, or choose Open Tunnel Manager in the command palette. Show it as a floating panel near the button (380px wide, `bg-surface` + small shadow).

**Structure**:
- **Header (40px, bottom divider)**: left = `route` (accent) + “Tunnels” + tunnel count badge (same colouring as the Create button: `accent-dim` background + `accent` text; never a solid `accent` fill, where accent-coloured text disappears); right = `plus` (expand new form) + `x` (close panel). The header doubles as a drag handle: drag the panel out of the way and it remembers the spot on release, reopening there next time (the same mechanism as the transfer panel and the message centre: an offset from the default anchor, clamped back into view when the window shrinks).
- **Tunnel list, one row per tunnel**:
  - First row: status dot (green = active) + tunnel name, such as `MySQL Forwarding` + type label on the right (`Local` with `accent-dim` / `Remote` with an `info` background / `Dynamic` / `Remote SOCKS`) + an outlined `AUTO` badge when automatic reconnect is on, and an outlined `STARTUP` badge when “Set up automatically when VelaShell starts” is on. When the row runs out of room the tunnel name is truncated; the badges always stay whole.
  - Detail row (11px monospaced; the route line in `text-secondary`, the other descriptions in `text-tertiary`): `L 3306 → db-prod-01:3306` / `R 6379 → localhost:6379`.
    - An end that uses a Unix domain socket shows its path (since 2026-10-24): “This computer listens on 127.0.0.1:2375 → server-side target /var/run/docker.sock”, “SSH server listens on /tmp/app.sock → this computer's /run/user/1000/app.sock”.
    - A remote SOCKS tunnel's route reads “SSH server provides SOCKS5 at 127.0.0.1:1080 → this computer connects to the allowed destinations”, with one more line for the allow list, “Allowed: localhost:8080 *.corp.example:443” (a lone `any` reads “Allowed: everything (the remote side can reach whatever this computer can)”).
  - Status row: uptime while active, otherwise the status text.
  - **Statistics row** (11px `text-tertiary` monospaced; the description row the `fuXS7` draft reserved): `3 conns · 1.4 MB`, calling out the live count when connections are transferring (`3 conns (2 live) · 1.4 MB`), and “No connections yet” before anything has connected. The byte figure combines both directions.
  - **Rate row** (since 2026-10-24, 11px `text-tertiary`): while running, `↑ 12.5 KB/s  ↓ 468.8 KB/s` (the average of the last three whole seconds; ↑ is what this computer sends into the tunnel, ↓ what comes back), followed by “· Limit 512.0 KB/s” when a limit is set; when stopped, only “Limit 512.0 KB/s”, and only if a limit is set. With a limit set, the tooltip says “Each direction is limited separately, and all connections on this tunnel share the allowance. Changes take effect the next time the tunnel starts.”
  - **Open connections** (since 2026-10-24): while running with connections in flight, “› Open connections: N” (`text-secondary`, collapsed by default); expanding it lists “source → target” and “open 3m 05s · ↑ 1.5 KB ↓ 2.0 MB” for each, at most 20, the rest summed up as “N more not shown”; a source that cannot be named (a client that came in on a Unix socket) reads “unknown source”.
  - While the panel is open, rates and open connections refresh every second; liveness checks and automatic recovery still tick every 5 seconds.
  - On hover, show actions: enable/disable toggle, edit, delete. Starting and stopping go over the network and take a moment: while one is in flight the start/stop button is replaced by a spinning progress ring (not a button, so it cannot be clicked again), edit and delete are disabled, and the status row reads “Starting tunnel…” / “Stopping tunnel…”; when it finishes the matching button comes back. The same applies while automatic reconnect is rebuilding a tunnel, so a manual start in the meantime does not create a second copy.
- **New Tunnel form (`plus` expands, `tunNewForm`)**:
  - Title: `circle-plus` + “New Tunnel”.
  - One row with three fields: **Type** drop-down (Local L / Remote R / Dynamic D, 80px) + **Local Port** input + **Remote Address** input (`host:port`).
    - Since 2026-10-24 the type drop-down has 4 entries: Local Forward / Remote Forward / Dynamic SOCKS / Remote SOCKS; the endpoint row shows what the type needs: dynamic has only the local port, remote SOCKS has no local port.
    - For remote forwards and remote SOCKS the host field is “Listen address (on the server)” and the check box below it “Listen only on the server itself (127.0.0.1)” (ticked by default, binding the server's loopback); a local forward keeps “Target host (from the server's view)” and “Forward to the server itself (127.0.0.1)”.
    - **Use Unix domain sockets** (check box, local and remote forwards only): ticking it shows two path fields — for a local forward “Local socket to listen on (blank = local port)” and “Socket on the server (blank = target host:port)”, each end optional; for a remote forward “Local socket to connect to” and “Socket the server listens on”, both required. The port / host / loopback check box of an end that has a path are disabled. A local path must be a full path of at most 107 bytes in UTF-8 (103 on macOS), a server path must start with `/`; mistakes are pointed out in red on the spot. Dynamic and remote SOCKS have no socket form.
    - **Allowed destinations** (remote SOCKS only): “Allowed destinations (host:port the remote side may reach)” — “Create” stays disabled while it is empty, and a malformed rule is named in red. The syntax is that of ssh_config's `PermitRemoteOpen`: separated by spaces or commas, `*` `?` wildcards in the host, `*` allowed as the port, IPv6 in brackets; a lone `any` allows everything (combined with others it is an error); names are compared as the remote side gives them, without resolving, and a rule written as an IP matches only IP literals.
    - **Rate limit** (every type): “Rate limit, KB/s per direction (0 = none)”, default 0; each direction is limited separately and all connections on the tunnel share the allowance; a change takes effect the next time the tunnel starts (a running tunnel cannot be edited anyway).
  - Check boxes, top to bottom: “Forward to the server itself” (locks the target to 127.0.0.1; “Listen only on the server itself” for remote / remote SOCKS), “Use Unix domain sockets”, “Reconnect automatically after a dropout” (**off by default**) and “Set up automatically when VelaShell starts” (**off by default**).
  - Button row, right aligned: `Cancel` (outlined) / `Create` (`accent-dim` pill: light background + accent outline/icon/text, `plus` + “Create”).
- **Interaction logic**:
  - Attempt to establish forwarding immediately after creation. Success = green dot + increment badge; failure = red dot + error message.
  - Local and dynamic forwards run a **local port-occupancy precheck** before binding, reporting “local port 27017 is already in use by another program” instead of a low-level socket error.
    - When this computer listens on a Unix socket (since 2026-10-24) there is no port precheck; the path is checked instead: if a file already exists there, nothing is created and **nothing is deleted for the user** — “… already exists — probably left over from last time, or another program's socket. Delete it once you are sure it is unused, then start again.”; stopping the tunnel deletes the socket file it created.
    - Other reasons this computer cannot listen (an address that is not this computer's, a port reserved by the system, a socket directory that is not writable…) are reported from the tunnel configuration — where (`address:port` or the socket path) and the socket error code — instead of the SSH library's raw text.
  - When the target cannot be reached, the message says which end: for local / dynamic forwards the target is reached from the server (“Target refused the connection: the target address is resolved from the server's perspective…”); for remote / remote SOCKS forwards it is reached from this computer (“The target on this computer refused the connection / cannot be reached from this computer…”).
  - Remote SOCKS: a request for a destination outside the list gets the SOCKS reply “not allowed by ruleset”, and the tunnel row records “The remote side asked for a destination that is not on the allow list, and it was refused. Add it to this tunnel's allow list to permit it.”. An empty list or a malformed rule that arrives by hand-editing or sync is refused as well, never taken to mean allow-all or deny-all.
  - A remote forward configured with a socket on only one end is not created (either both ends use ports or both use sockets); it never silently falls back to a port.
  - A tunnel does **not** follow the lifecycle of a terminal tab: creating or starting one while disconnected opens a background connection dedicated to tunnels, and that connection is dropped once the server's last tunnel is removed.
  - When the host session drops, its tunnels are marked stopped; those with “reconnect automatically after a dropout” are redialed and rebuilt on their own (failures back off from 10s to 5min), and the rest wait for a one-click start. **A tunnel the user stopped by hand is never brought back up automatically.**
    Automatic redials never show the host fingerprint confirmation (same as terminal tabs, see “Host fingerprint confirmation”); a server whose redial failed on the fingerprint is not redialed again and does not enter the backoff — an error toast says so, and its tunnels wait for the user to start one by hand (that start asks), after which automatic reconnect works as usual.
  - **Running state is never persisted**: only the tunnel configuration is saved, so after a restart every tunnel is stopped and waits for a manual start. Only tunnels with “Set up automatically when VelaShell starts” are connected and set up in the background at launch (without holding up startup; a missing credential still brings up the login prompt). A failure during that automatic setup (the server is unreachable, the local port is taken, the credential prompt was cancelled) raises an error toast in the bottom-right corner naming the server and the tunnel; one server failing does not stop the others, one tunnel failing does not stop the rest on the same server, and if none came up the background connection opened for them is dropped again.
    - That toast carries “**Retry**” (since 2026-10-24): for an unreachable server it reconnects and sets up every tunnel on it marked for startup that is not up yet; for one tunnel that failed it retries just that one; another failure reports again, again with Retry. A retry is a person's click: the host fingerprint may be asked about as usual, and a server whose automatic recovery was paused because authentication needed a person is authenticated again; a tunnel the user started, edited or deleted in the meantime is left alone. Success is announced with an info toast, “Tunnel “x” on srv is up.”. The two toasts for automatic reconnects that failed on the fingerprint / Tailscale authentication carry no Retry (they need a manual start from the panel and a check in the dialog).
  - Type descriptions: local forwarding (`-L`), remote forwarding (`-R`), dynamic SOCKS (`-D`), remote SOCKS (`-R` without a target, since 2026-10-24); local and remote forwards can use Unix domain sockets instead.

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

**GPU measurement semantics**: NVIDIA uses `nvidia-smi` and AMD uses DRM sysfs. Intel `i915` / `xe` reads the DRM clients' cumulative per-engine-class counters and resident memory from `/proc/<pid>/fdinfo`. The scan runs only when `/sys/bus/pci/drivers/i915` or `xe` actually has a bound device; it first uses `ls -ln /proc/[0-9]*/fd` to pick the fds that point into `/dev/dri/`, reads only those fds' fdinfo, once per tick, buckets them per card by `drm-pdev`, and de-duplicates by `drm-client-id`. i915 engine counters are nanoseconds, divided by the delta of the remote `/proc/uptime` (falling back to the local sampling interval only when that is missing); xe only reports cycle counters, so it uses Δcycles / Δtotal-cycles. Deltas are paired per (client, engine) and only count clients present in both samples — a new client is only a baseline this tick, and a client that exited drops out; engine classes with several instances are normalized by `engine-capacity`, and utilization is the busiest engine class (the same semantics as `intel_gpu_top`), not a sum across classes. Memory: `local*` / `vram*` regions are the on-board VRAM of Intel discrete cards such as Arc, labeled “VRAM”; sysfs gives no total for it, so only the used amount is shown. `system*` / `stolen-system*` / `gtt` regions are system memory shared by integrated graphics, labeled “Memory”, with total system memory as the denominator. The GPU process table comes only from `nvidia-smi`, so its memory column header is always “VRAM” and does not switch to “Memory” with the selected card. A non-root user can see only clients they are permitted to read, so this must not be presented as whole-device utilization. Missing permissions or driver interfaces show “unavailable”, never a fabricated 0%. When listing cards, DRM devices without a PCI vendor ID (Raspberry Pi vc4 / v3d, Rockchip, simpledrm) yield no metrics at all and are not listed as cards; a PCI display device with no bound driver shows an empty driver field.

---

## 11A. Plugin Manager Window (opened from `plug` at the bottom of the sidebar)

**Trigger**: `plug` in the sidebar's bottom user bar (§3); “Get more protocols…” at the bottom of the new-connection dialog's protocol rail (§13.1). A modeless standalone window owned by the main window; **only one is ever open** — triggering it again just brings the open one to the front.

**Window**: 676×556 by default, minimum 516×396, centered on its owner; resizable from all four edges and corners, with the per-platform frame (see §2). On close it records its normal-state size and position (the `pluginManager` entry of `windowLayout` in the document store) and opens there next time.

**Structure, from top to bottom**:
- **Title bar (28px, bottom divider)**: left = `plug` (accent) + “Plugin Manager”; right = minimize / maximize / close. Double-clicking the title bar toggles maximize; dragging it moves the window.
- **Toolbar (bottom divider, 14×10 padding)**:
  - **Security notice** (first row; `VelaBgInput` fill with a border, 5 radius, a warning-colored `shield` on the left, 10pt `text-secondary`, wrapping): “Plugins run with your account permissions and can reach your files, sessions and network. Before installing, confirm where the package came from and check its signature fingerprint…”. It sits **before** the Install button: the signature dialogs only appear for unsigned packages or unknown publishers, by which time the user has already picked the file — the risk has to be visible before the button is pressed.
  - Second row, left: a button group (outlined, 28 high, 4 radius, 11pt): `download` “Install .vpx…”, `refresh-cw` “Check for updates”, “Update all”; middle: a one-line notice (`text-muted` 10pt, ellipsized) where the outcome of installing / checking / updating and “Plugin host for "X" is waiting for a debugger — attach to process N.” appear; right: a “Plugin store ↗” link (`text-tertiary`, accent with underline on hover, tooltip “Browse reviewed plugins at market.easilynet.top”, opened in the system browser).
  - Visibility: the security notice and Install appear only when there is a writable user plugin directory; Check for updates additionally needs a marketplace client; Update all appears only when at least one row can be updated directly.
- **List** (scrollable, 14×12 padding, with 6 more on the right so the overlay scrollbar does not cover the buttons when it expands): sorted by display name (case-insensitive); with no plugins it shows the empty state (30px `plug` + “No plugins installed.”). It refreshes by itself whenever the plugin runtime changes (install, uninstall, enable/disable, crash).

**List row** (card: `VelaBgInput` fill, border, 6 radius, 14×11 padding, 8 between rows):
- **Left: status dot** (8px): green = active (`VelaStatusConnected`); yellow = restarting (the isolated process exited unexpectedly and waits for its backoff restart); red = failed / invalid / incompatible; grey = everything else (idle, disabled, deactivated).
- **Middle, top to bottom**:
  1. Display name (12pt Medium) + version `v1.2.3` (monospace 10pt `text-muted`) + a host-mode chip (`InProcess` / `Isolated`, monospace 9pt); plugins mounted for development (`plugins.dev.txt` / `VELA_PLUGIN_DEV_ROOT`) also get a warning-colored `DEV` chip.
  2. Status text (10pt `text-secondary`) + plugin id (monospace 9pt) + “by X” (the manifest's `author`, falling back to `publisher`; hidden when both are missing). An active plugin with no open panel / document / protocol session reads “Running in background” — an in-process plugin stays resident once activated, so closing its tab does not stop it.
  3. **Publisher fingerprint** “publisher SHA256:…” (monospace 9pt `text-muted`): the signer's public-key fingerprint pinned at install time (publisher continuity; see step 4 of “Installing” below). The manifest's `author` is a line the plugin writes about itself, which anyone can write; this line is the cryptographic identity. It is **truncated only on screen** (one line, ellipsized), with the full fingerprint in the tooltip — a shortened fingerprint cannot be checked, yet looks as if it could. Built-in plugins, side-loaded ones (installed from the command line or dropped into the directory) and plugins installed from an unsigned package have no pinned identity, and the line is hidden rather than showing “unknown”.
  4. Error / reason (red, wrapping, only when present).
  5. **“Host too old” notice** (warning color, wrapping): “Version X is available, but it needs a newer VelaShell (Y).” Y is whatever is missing — `apiLevel N`, `VelaShell x.y.z` or `SDK x.y.z` — and the tooltip says the same. In this case there is **a notice but no update button**: a button that is bound to fail is worse than none.
- **Right: row actions** (outlined, 26 high, 4 radius, left to right; an action whose condition is not met does not appear at all):

| Button | Shown when | What it does |
| --- | --- | --- |
| View log | The manifest is valid (an invalid plugin never ran; the reason is already in its error line) | Opens this plugin's log window, see below |
| Revoke permissions | It holds a `terminal.write` / `session.open` grant (always or for this session), or an active upload / download lease; rechecked every 500 ms while the window is open and whenever it is activated | Revokes them at once and cancels pending and running upload / download leases; no confirmation |
| Reload | Mounted for development, manifest valid, and not invalid / incompatible / disabled | Deactivate → reread the manifest → load again, so a rebuild runs without restarting VelaShell |
| Update to x.y.z | The store has a newer version that this host can install, and the user installed the plugin | See “Updating” below |
| Enable / Disable | The manifest is valid and the plugin is not invalid / incompatible | Takes effect at once |
| Uninstall | The user installed it (built-in plugins are read-only and cannot be uninstalled) | Danger style; asks “Uninstall "X"? Its data will be removed.” first, then deletes the directory and its KV, secrets, time-series data and log |

**Checking for updates**:
- Opening the window asks the store once; the Check for updates button is there to retry after an offline attempt. **Only the ids of plugins the user installed leave the machine** (the store has no built-in or development-mounted plugins); the plugin manager makes no other outbound requests.
- The notice goes “Checking for plugin updates…” → “All plugins are up to date.” or “N plugin(s) have a newer version.” (plugins the host is too old for count towards N too).
- Versions are compared the way the app's own updater compares them (prerelease suffixes count: with `1.2.0-beta.1` installed, `1.2.0` in the store is newer). Built-in, development-mounted and plugins the store does not know have no “update”.

**Updating / Update all**: the row's “Update to x.y.z” first downloads that version to a temporary directory (notice “Updating “X”…”, or “Update failed: reason”), then checks whether the publisher fingerprint the store reports matches the one pinned on this machine. If it matches, it installs directly (that key is the one the user accepted in the first place); if it does not match, the version is unsigned, or nothing was ever pinned here, it goes through the full install confirmation below. “Update all” does the same for every row that can be updated directly at that moment. An update replaces files only; the plugin's KV, secrets and time-series data are kept. The source is checked before downloading (since 2026-10-25): if the marketplace address or the download URL in the ticket is not https, or the marketplace gives no 64-hex-digit `fileSha256`, the download is refused before any request is sent (nothing is left on disk); after downloading the digest is checked, and installing still verifies the signature.

**Installing a `.vpx`** (“Install .vpx…” picks a file; updates go through here too) may show these danger-style dialogs, in this order:
1. **Unreadable trust store** (“Plugin trust store is unreadable”, usually because the local encryption key changed or could not be obtained this time — another system account, a data folder copied from elsewhere, or the system keychain could not be opened, with the message center saying which; when the original key can be restored or the keychain comes back, it advises against resetting): explains the cause and the cost; “Reset trust store” creates an empty store (the unreadable one is kept as a backup), trusted publishers have to be trusted again and installed plugins lose their tamper-evident receipts. Without a reset nothing can be installed.
2. **Unsigned** (“Untrusted plugin package”): “"X" is not signed. Plugins can execute code with your account permissions. Install only if you trust its source.” → “Install anyway” allows it once.
3. **Valid signature, unknown publisher** (same title): shows the public-key fingerprint and asks the user to verify it through the author's official channel → “Trust publisher and install”. This “trust” is written into the local trust store **only after the install succeeds**: the package may be taking over an installed plugin's identity, and that question can only be asked once the id has been read out of the package; a failure to write it is reported but does not roll the install back.
4. **Publisher changed** (“Plugin publisher changed”): a plugin with the same id is installed and the signer pinned back then does not match this package (or this package is unsigned, shown as “(this package is not signed at all)”). The dialog shows both full fingerprints side by side and states that “a routine key rotation and a hijacked publishing identity look exactly the same from here” → only “Install anyway” replaces it, pinning the new key as the new baseline. When refused, the installed version is left untouched (the check runs before the old version is removed). This only protects plugins installed from the plugin manager: side-loaded plugins cannot pin a publisher, and the host has no way to stop an impersonating overwrite of them (see [Developer Guide §12.2](../templates/dev-guide.md#122-signing)).
- The outcome goes to the toolbar notice: “Installed "X".” / “Install failed: reason”.

**Plugin log window** (the row's “View log”, since 2026-10-24):
- A modeless standalone window owned by the plugin manager (closed together with it); **at most one per plugin** — clicking the same row again brings it to the front and refreshes it. 820×560 by default, minimum 520×320; size and position are remembered like the other tool windows, one entry shared by all plugins (`pluginLog`).
- **Title bar**: `file-text` + “Plugin log · display name”; an action group with `refresh-cw` Refresh (disabled while reading) and `folder-open` Open containing folder (the log directory `logs/plugins/`, created first if it does not exist yet); then minimize / maximize / close.
- **Subtitle line**: the log file's path (monospace, full text in the tooltip); when only the tail was read it reads “Last 256 KB only · path”.
- **Body**: a monospace, read-only, non-wrapping text box; after opening and after every refresh the caret goes to the last line. There is no live follow mode; reproduce the problem, then click Refresh.
- **Empty state** (`file-text` icon + text): “This plugin hasn't logged anything yet. What it writes through its logger, the standard output and error of its isolated process, and its activation, failures and crashes all end up here. Click refresh after reproducing the problem.” When the plugin id cannot be used as a file name, it points to the lines starting with `[Plugin:<id>]` in the application log instead.
- **Log file**: `<data root>/logs/plugins/<plugin id>.log` (by default `~/.velashell/logs/plugins/`), rotated to `<id>.log.1` at 1 MB (only that one old file is kept; not `<id>.1.log`, which is the current log of a plugin whose id is `<id>.1`), so at most about 2 MB per plugin. The window reads the last 256 KB, topping up from the rotated file when the current one is shorter, and drops the line cut in the middle. Every source is on the host side: what the plugin writes through `IPluginLogger` (called directly in process, relayed over RPC from an isolated process), the isolated process's stdout / stderr (where unhandled-exception stacks land), and what the host records for it — activation, activation failure (with the full exception) and crashes; the copy in the host's own log stays. It is deleted when the plugin is uninstalled; one whose folder was deleted while VelaShell was not running is removed on the next start (an older version's `<id>.1.log` is renamed into place); when exporting a diagnostic bundle it can be ticked per plugin in the selection dialog (unticked by default), and a ticked one is masked the same way (see “Export diagnostics…” on the About page in §14).

---

## 11B. SSH Connection Info Window (since 2026-10-24)

**Trigger**: “Connection info…” in a session tab's right-click menu (§4B.2; shown only for SSH session tabs, disabled while not connected), or “SSH connection info” in the command palette (Tools, available when the active tab is a connected SSH session). Local terminals and plugin terminal protocols (Telnet, serial) do not have it.

**Window**: a modeless tool window, 760×680 by default, minimum 560×420, resizable and maximizable, remembering its size and position like the other tool windows (the `connectionInfo` entry in the document store's `windowLayout`, shared by all tabs, see §2). At most one per tab; opening it again brings it to the front and refreshes it. When the tab closes the window stays (its information can still be copied), and a refresh then says “This session is not connected right now. Reconnect it, then refresh.”. Esc closes it.
- **Title bar**: a 15px accent `info` icon + “Connection info” + the session label (green dot when connected / red when not; names wider than 240px are truncated, with the full name on hover) + icon buttons “Copy as text” (turning into an accent check mark for 1.5 seconds after copying) and “Refresh” + maximize / close.
- **Subtitle line**: `user@host:port · as of HH:mm:ss`.
- **Body**, in blocks, with every value monospaced and selectable:
  - **Peer**: state, version string, host key (“type · bits”, noting a host certificate when one was presented), SHA256 fingerprint, session ID (the exchange hash in hex), whether the PING extension (`ping@openssh.com`) is supported.
  - **Negotiated algorithms**: key exchange, host key, cipher, integrity MAC, compression — one value when both directions agree, “out a · in b” when they differ; with an AEAD cipher the MAC reads “built into the AEAD cipher”; strict KEX (Terrapin mitigation), extension negotiation (RFC 8308).
  - **Authentication**: the method that succeeded, server-sig-algs (“not announced” when the server sent none).
  - **Timing**: dialing (TCP, proxy or jump host), version exchange, key exchange (incl. confirming the host key), authentication (incl. waiting for input), total, round trip (last measured; “not measured yet” before any measurement).
  - **Traffic and rekeying**: sent and received (bytes · packets); the rekey count, and when there was one, the cause of the latest (peer-initiated / requested locally / byte count, packet count or interval reached its threshold / packet count reached the hard limit, with observed value and threshold) and how long it took; obscured keystroke timing (off / on / on but batching only, see §13.1).
  - **Channels (N)**: a table — number, type (the protocol name, such as `session`, `direct-tcpip`), state (opening / open / half-closed (EOF) / closing), age, sent, received; “No channels are open.” when there are none.
  - **Route (proxy and jump hosts)**: numbered from near to far; an outbound proxy as one line (e.g. `socks5 host:port`); each jump host as one line with its address and one with “version · key exchange · host key · cipher · authentication”, marked “down” when that hop has dropped. The block is hidden for direct connections.
- **Rules**: everything comes from read-only properties the SSH library already exposes; **opening and refreshing never send anything on the wire** (the round trip is the one the status bar's latency measurement and keep-alive already took). The jump hosts passed on a successful connection are recorded hop by hop by the host while it assembles the jump connections (the library's dialer only records per-hop results on failure), starting afresh on every connection. Refresh is manual, with no automatic polling (so the content does not jump while you select text to copy); the data is fetched from the tab's current session, so after a reconnect a refresh follows the new connection.

## 11C. Session Health Check Window (since 2026-10-25)

Runs the four steps of connection diagnostics on a batch of saved connections, one by one, and produces a table of "which ones cannot be reached".

**Triggers**: "Health-Check This Group" on an explorer group's context menu, "Health-Check N Connections" on the multi-select menu (§12), and the command palette's "Session Health Check (All Connections)" (`tools.healthCheck`, every stored connection sorted by name).
**It runs only when you start it, never on a schedule**; one batch at a time — starting another while the previous one is still running only brings the window to the front with "The previous health check is still running; stop it or wait for it to finish first.", without interrupting it.

**Window**: at most one modeless window at any time (`SessionHealthCheckWindow`), with the same frame as §11B (28px title bar, resize grip in the bottom-right corner, double-click to maximize); size and position are stored under `healthCheck` in the document store's `windowLayout`. Closing the window stops the run.
- **Title bar actions** (left to right): show problems only (`circle-alert`, accent when on), verify login (`key-round`, accent when on; the tooltip has two lines: the state + "On: log in once with the saved credentials and disconnect right away. Off: check only up to the SSH handshake, without logging in to the server. Takes effect on the next run."),
  stop (`square`, disabled when nothing is running), run again (`refresh-cw`, disabled while running), copy as text (`copy`, tab-separated, all rows, briefly turns into a check mark after copying).
- **Subtitle**: the summary "N connections · OK a · need attention b · unreachable c", prefixed with "Checked x / N · " while running.
- **Table**: 26px header, 28px rows; columns: result (colored by status: OK / Reachable green, Needs credentials / Needs you yellow, Unreachable / Timed out red, Waiting / Not checked / Stopped gray, Checking… accent), name, target (monospace), failed at, details, time,
  row actions (connection diagnostics `stethoscope`, SSH / SFTP only, opening §15's diagnostics owned by the health check window; edit connection `pencil` — after saving, the latest copy is taken from the store and that one host is checked again).
  With "show problems only" on and nothing wrong, "No connection needs attention" is centered. One line of gray text at the bottom explains the rules (concurrency and time limit formatted from the actual values).

**What each host gets**:
- Only SSH / SFTP profiles are checked; FTP and plugin protocols are listed as "Not checked" (a plugin probe opens a real session and may show the plugin's own dialogs). Up to 6 hosts at once (the same as opening a whole group), at most 30 seconds each — then "Timed out", without holding up the batch;
  on "Stop", whatever is unfinished or queued becomes "Stopped". Both numbers are constants, not settings.
- The four steps of connection diagnostics are reused (DNS → TCP → SSH identification → authentication; with a jump chain the first three target the first hop and authentication goes through the whole chain). Results: all four pass → "OK"; with "verify login" off → "Reachable"; no saved credentials → "Needs credentials"; needs a human answer → "Needs you";
  any other failure → "Unreachable" — the first failed step is "failed at" ("Jump host settings" when the chain is misconfigured), and the details are the issue title plus that step's reason.
- **The authentication step runs, but never shows a dialog** (the connection is assembled as "unattended", every hop of a jump chain marked):
  - Without saved credentials the step is skipped as in diagnostics and recorded as "Needs credentials" (a manual connection would show the login dialog; the health check does not).
  - A host fingerprint that needs a human decision (first seen with first-connection confirmation on, or changed without "block on change") is rejected the way automatic reconnects do → "Needs you", with "The health check did not accept host key {fingerprint} of {host}: it needs your confirmation, and the health check never asks. Connect to it manually once to check the fingerprint.";
    a first-seen key raises no security alert (the user picked this machine to check; they just have not connected to it themselves yet), a changed one still does. With first-connection confirmation off, the key is recorded on first use as usual, the same as a manual connection.
  - keyboard-interactive: recognizable password prompts are still answered with the saved password; a one-time code or password change is declined → "Needs you" (§13.2 one-time code dialog). A Tailscale check-mode re-check is rejected the same way → "Needs you".
  - When an external agent asks for its own per-use confirmation (`ssh-add -c`, the 1Password / KeePassXC approval prompt), that is the agent's UI, which the health check cannot suppress.
  - Cancellations such as Stop or a host's time limit propagate as cancellations and are not counted as "Unreachable".
- **Verify login** switch (on by default): when off, only the SSH handshake is checked and nothing logs in to the server (some environments alert on every login); takes effect on the next run; remembered only for this launch — **it is not a setting** and is not written to the settings file.
- Health check logins are not written to the audit log or connection history (the same as the profile dialog's "Test"), and "Recent connections" does not change.

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
- Open N Connections with Synced Input (`link-2`, since 2026-10-24) — opens only what yields a terminal (SSH and plugin
  terminal protocols; SFTP / FTP / workspaces are skipped), always as new tabs, and puts them into the first free synced
  input channel (A → D, §6.1; free = no tab in it and not reserved by another batch that is still opening); when all four
  channels are in use it opens nothing and asks you to close one first. Tabs of the same profiles that are already open are
  left alone and not pulled into the channel (one may be sitting in vim, or be in another channel). Concurrency and
  confirmation are the same as "Open N connections". Each one joins the channel as soon as it finishes (connected, or failed
  with its overlay kept), so one that reconnects later is still in the channel; at the end one toast says "N terminals joined
  synced input channel X", with a clause each for any that did not connect and any non-terminals skipped, and then at warning
  level
- Health-Check N Connections (`activity`, since 2026-10-25) — runs the four diagnostic steps on the selected connections one by one and opens the session health check window (§11C); no terminals, no dialogs
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
- Open All with Synced Input (`link-2`) — same rules as "Open N Connections with Synced Input" in the multi-select menu,
  applied to the whole group
- Health-Check This Group (`activity`, since 2026-10-25) — same as "Health-Check N Connections" in the multi-select menu, applied to the whole group
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
| Authentication | full-width segments below the heading: shared credential (pick one, #550) / password (may be left empty to be asked when connecting) / key (file + optional passphrase) / agent / Tailscale SSH. FTP and plugin protocols only take passwords; those rows are skipped for anything else |
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
  | SSH options | SSH and SFTP (same SSH connection) | compression, obscure keystroke timing (SSH only), allow legacy algorithms, custom algorithm lists, agent endpoint |
  | Forwarding | SSH only (interactive shell) | ssh-agent forwarding (and its two restrictions), X11 forwarding |
  | Advanced | FTP, or a plugin protocol that declares `IsAdvanced` fields | FTP default remote path and server time zone; plugin tuning fields |

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
    - **IPv6** (#611): type the bare address into Host (`2001:db8::1`, or `fe80::1%en0` with a zone id). A `[2001:db8::1]` or
      `[2001:db8::1]:2222` pasted from elsewhere works too — Save and Test drop the brackets, and the port after them replaces the Port
      field (the footer preview shows exactly that). Malformed brackets (unclosed, or not followed by `:port`) get a hint under the field
      and the three buttons stay disabled. Profiles store only the bare address: with brackets it would go to the server verbatim through
      a jump host, and its host keys would be recorded as a different machine from the bare address. Everywhere `host:port` is shown,
      IPv6 is bracketed.
  - **Authentication** (hidden for protocols declaring `NoCredentials`; a username and password typed before it was hidden are not saved with the profile and never handed to the plugin when connecting, nor are those left in an older profile): a full-width single-row segmented control below the username (Password / Key / Certificate / Agent / Tailscale SSH / Key store, in `AuthMethod` enum order; the key store segment is shown only on Windows, see below). FTP keeps its encryption mode to the right of the username; plugin protocols leave that slot empty.
    Below come FTP's anonymous login / passive mode, the plaintext-FTP warning, the password or certificate + private
    key + passphrase, the Agent explanation, the key store picker, and remember password.
  - **Credential source** (#550, since 2026-10-03; first row of Authentication; hidden for anonymous FTP, credential-less protocols, Tailscale SSH and the key store): a dropdown whose first item is “Enter in this connection”, followed by every
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
  - **Organize**: display name / session group / tags side by side, a **tab icon** on the second row, and a full-width
    **Notes** row below (#549, `SessionProfile.Notes`).
    - **Tab icon** (since 2026-10-24, `TerminalOverrides.TabIcon`, stores a lucide glyph name): pick one of 23 icons the way
      you pick a color (server, database, cloud, globe, terminal, shield, key… — the names say what is drawn, not what it
      means). The field looks like the color picker's (§14.3): icon + name + drop-down arrow; the flyout is a grid of icon
      cells, the current one with a bold border and an accent icon, and “Follow protocol” at the bottom clears it.
      No choice = follow the protocol, and the field shows that protocol's default icon one step dimmer. Every connection
      type has it (the Terminal page, where the color lives, exists only for SSH, while the icon is drawn for SFTP / FTP /
      plugin tabs too). Tabs and the explorer both draw it (§4B.2, §3); tabs already open keep their icon (like the tab
      color, it is read once when the tab is created), while the explorer updates as soon as the profile is saved.
      It travels with JSON import / export and cloud sync, not with CSV (neither does the tab color).
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
    color (color picker, see §14.3; no color = automatic, with a one-click reset at the bottom of the flyout; the tab icon is not here but under General → Organize), startup directory, keep-alive interval, and **anti-idle (s)**. There can only be one global setting, but
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
    forwarding and X11 forwarding, **all off by default** (same as OpenSSH); obscure keystroke timing (since 2026-10-24, off
    by default, see below); plus the two algorithm negotiation options
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
    - **Obscure keystroke timing** (since 2026-10-24, `SshSessionOptions.ObscureKeystrokeTiming`, below compression, **off by
      default** — unlike OpenSSH, where it is on, because of the bandwidth cost): normally every keystroke is a packet of its
      own, so the gaps between keys are visible on the wire and reveal what was typed (passwords included). When on, input
      is sent on a fixed 20 ms beat, and while typing, empty beats carry cover packets of the same size
      (`ping@openssh.com`). The muted text states the cost: about 50 packets per second in each direction while typing, each
      keystroke up to 20 ms late; nothing extra while idle, pasting unaffected; the cover packets need a server that supports
      `ping@openssh.com` (OpenSSH 9.5 or later). The beat is not adjustable. Shown for SSH only (it needs an interactive
      shell), not saved on SFTP profiles. When the shell opens, a muted line at the top of the terminal says “Keystroke
      timing obscured (20 ms beat with cover packets)”; if the server does not understand PING it becomes a yellow line,
      “…input is only batched on a 20 ms beat without cover packets; the gaps between keys can still be seen.”. The
      connection info window (§11B) shows its state in the “Traffic and rekeying” block too.
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
      that only speaks CBC). The terminal tab's failure card also offers an “Allow these algorithms and reconnect” button
      that appends the named ones to the custom list in one click — so do the failure card of document-style connections, the connection
      diagnostics window and this dialog's “Test” feedback bar, as “… and reconnect / diagnose again / test again” (terminal tabs since 2026-10-24,
      the others since 2026-10-25; see “Allow the algorithms with one click after a negotiation failure” in §7).
    - **Agent endpoint** (since 2026-10-25, `SshSessionOptions.AgentEndpoint`, a new "SSH Agent" section at the bottom of the "SSH options" page, for SSH and SFTP): a single-line input,
      with the placeholder “System default — or a named pipe `\\.\pipe\name`, or a socket's full path” on Windows / “System default (SSH_AUTH_SOCK) — or a socket path such as `~/.1password/agent.sock`” elsewhere.
      Grey text: “Leave empty for the system default agent (SSH_AUTH_SOCK; on Windows the OpenSSH Authentication Agent service, or Pageant when that service is not running). When set, this connection talks only to that agent: SSH agent authentication, agent forwarding and "Add keys to agent" all use it, like IdentityAgent in ssh_config.”
      - Errors under the box (a pipe on a non-Windows system, no pipe name after `\\.\pipe\`, not a full path, longer than `sun_path`'s 107 / macOS 103 bytes) **only warn and never block saving** — the profile may sync to another platform.
        What the user wrote is stored as is (surrounding whitespace and one level of quotes removed), and `~` is expanded only when connecting. A value counts as a non-default on this page (tab dot).
      - Each hop of a jump chain uses its own endpoint. When it cannot be reached the message names the endpoint: “Cannot reach the ssh-agent this connection names (endpoint): It does not exist or nothing is listening on it. Make sure the program that provides it (1Password, KeePassXC, gpg-agent…) is running and the path is right;
        to use the system default agent, clear Agent endpoint in the connection's SSH options.” (no longer telling the user to start the Windows agent service); with no keys at all: “The ssh-agent this connection names (endpoint) holds no keys.”
      - The candidates of “Forward only selected keys” are listed from this agent (with the endpoint as it was when the dialog first opened; changing the endpoint afterwards does not re-list them until the dialog is reopened); the grey line shown once agent forwarding is up adds “ · agent: endpoint” when one is set.
  - **Authentication method “Certificate”** (`AuthMethod.Certificate=2`, an OpenSSH user certificate): no separate form —
    it is key authentication plus one more field, “Certificate File”. The signature always comes from the private key;
    the certificate is only the CA's endorsement, so the private-key and passphrase fields are used as before. The
    certificate field sits **above** the private-key field, because the key can be derived from the certificate but not
    the other way round.
    - **Picking a certificate fills in the key**: after a certificate is chosen with the certificate field's “Browse...”,
      if the private-key field is still empty, the key path is derived by ssh-keygen's naming convention — strip the
      trailing `-cert.pub` (case-insensitive) — and filled in only if that file actually exists. If the key field
      already has something, the name does not follow the convention, or the derived file does not exist, nothing
      happens and the user picks the key. Only a certificate chosen through the file picker triggers this; a typed path
      does not. The login dialog (§13.2) and the shared-credential editor follow the same rule. Without it the user
      browses the same directory twice and can easily pick the neighbouring, non-matching key, which at connect time
      only yields a vague “publickey” rejection.
    - **Leaving either field empty is a hard error**: key authentication allows an empty key field; certificate
      authentication does not. An empty certificate field shows “Certificate file is required” below it (there is no
      default location to fall back to); an empty key field shows “Certificate authentication also needs the matching
      private key”; a path whose file is missing shows “Certificate file not found” / “Private key file not found”.
      These are field hints and do not block saving. On connect, a missing file first brings up the credential
      dialog (§13.2) to collect both; the connection service then checks again and reports the two cases separately —
      “Certificate authentication requires a certificate file.” and “Certificate authentication requires the private
      key matching the certificate.” — so the user knows which one to supply.
    - Two things are checked on the spot at connect time: that it is a user certificate (a host certificate, signed by
      ssh-keygen with `-h`, cannot log in), and that the public key in the certificate pairs with the private key. A
      mismatch reports both sides' type and SHA256 fingerprint instead of leaving the server to answer a Permission
      denied that cannot be told apart from “CA not trusted” or “principal mismatch”.
  - **Authentication method “Agent”** (`AuthMethod.Agent=3`; the enum is persisted by ordinal, so new values can only be appended): signs with the keys in the local ssh-agent and **stores no
    credentials** in the profile; selecting it collapses the password and key fields, leaving one muted line saying
    where the agent comes from. On Windows that is the named pipe of the “OpenSSH Authentication Agent” service;
    `SSH_AUTH_SOCK` is honored only when it is itself a `\\.\pipe\…` path (1Password and KeePassXC set it that way) and
    ignored when it points at a Git Bash / WSL Unix socket — that is a different agent and cannot be reached. To use another agent (1Password, KeePassXC, gpg-agent…),
    set Agent endpoint under SSH options (since 2026-10-25, see above), which the grey text says as well; with an endpoint set, only that agent is used.
    - Every key in the agent is tried in turn (probe first, then sign, so the agent's confirmation prompt is never
      triggered for a key the server would not accept anyway).
    - If the agent is not running, the error comes within **3 seconds** (“Cannot reach the local ssh-agent”, with a
      `Start-Service ssh-agent` hint) — a named-pipe connect without a timeout keeps waiting for the pipe to appear, and
      all the user would see is a spinner until the whole connection times out. An agent with no keys gets its own
      message (“load one first with ssh-add”) instead of a generic “authentication methods exhausted”.
    - **The agent is only touched when this method is explicitly chosen**; no other method falls back to it implicitly.
  - **Tailscale SSH** (`AuthMethod.Tailscale=4`): uses the system's signed-in Tailscale network. Enter a MagicDNS name or tailnet IP, normally port 22, and the remote system username. Existing host-key, proxy and jump-host policy still applies; no SSH password, key, certificate, agent or keyboard-interactive credential is prepared. Switching, saving and importing clears old secrets, shared references and Remember password. A hidden shared credential cannot supply the username, and this method cannot be a shared credential.
    - Login retries explicitly identify external Tailscale shared credentials as unsupported. Their username and authentication method are not inherited, and the entry is never updated. Users may still explicitly select Tailscale SSH with the system username they entered themselves.
    - `accept` / `check` are server authorization policy, not client configuration. Every actual handshake uses the current server response; panel changes work with one profile and one running application. No approval, checkPeriod or link cache and no management API polling.
    - Without a challenge, continue directly (including a valid checkPeriod). A USERAUTH_BANNER containing an allowed official HTTPS link opens one owned, modeless window per actual authentication hop. The browser opens only on a user click; the URL is selectable/copyable. Showing/updating never waits for the window to close, and there is no “I have verified” button. The link box shows exactly the string that is opened / copied, the full `AbsoluteUri` (escaped as is, not a decoded version), and below it a warning-colored line reads “Before approving, make sure the page asks you to approve SSH access to this server. If it asks you to sign in or to add a new device or machine to your tailnet instead, do not approve it; cancel this connection.” — device-login links and SSH check links both live under `login.tailscale.com/a/`, so the link alone cannot tell them apart.
    - **Wait limit**: this hop does not use the library's fixed authentication timer; `TailscaleAuthDeadline` manages it. A **2-minute** guard starts only after the host-key decision (time spent reviewing a fingerprint does not count), so a silent server still fails quickly; the first valid check link extends it to **30 minutes** (about as long as the server waits for approval, enough for SSO plus MFA), once only, and later links keep the same deadline. The window states “Waiting up to N minutes. If the server has not approved by HH:mm, this connection is cancelled.”
    - A new URL updates the existing window. SUCCESS continues the original connection and closes it even without a browser click. Cancel/Esc/close cancels only this attempt; rejection/timeouts report the failing hop safely, with no target-password fallback. Verification links never enter ordinary banners, logs, exception chains, history or saved configuration. The window intercepts only a direct close of itself (turned into Cancel), not the main window's close: one click on the main window's close button closes it, and the check window closes with it and cancels this authentication; when the main window vetoes its own close (minimize to tray, confirm before closing), the check keeps waiting.
    - When the server ends authentication with an explanatory banner followed by DISCONNECT, it is reported as **rejected**, and the failure message gets “Server message: …” appended (all links removed, control characters sanitized, one line, at most 300 characters); a banner that came before the check link (a legal notice, for example) does not count as a reason.
    - Background terminal/tunnel recovery pauses only after a check link has appeared or the server has explicitly rejected, including queued application-start tunnel tasks. Explicit reconnect/start remains allowed and successful authentication clears the pause. Transient failures, such as the connection being reset during authentication or a timeout before any check link arrived, back off and retry like other authentication methods; a first connection stays on the failure overlay without a credential prompt.
    - A password / key / certificate / agent profile that reaches a check-mode node fails immediately once a check link is recognized in the banner, with “This host requires Tailscale SSH browser verification. Change the session's authentication method to Tailscale SSH and connect again.” The password is never sent, and the user no longer waits 2 minutes for a generic authentication timeout; this failure also pauses background reconnects.
    - Connection diagnostics need only the remote username for a Tailscale profile: with a username the authentication test runs (check mode still opens the check window); without one the test is skipped with a hint to fill it in the edit dialog. An authentication failure gets Tailscale-specific advice (this device is signed in to Tailscale, the remote username is correct, and the tailnet SSH policy allows this user) instead of asking the user to save credentials or check a password.
    - The server decides whether established SSH/SFTP/tunnels survive. No guarantee is made that a plain mode switch releases a pending check or terminates an established connection. Real policy propagation/revocation needs a dedicated-node acceptance run; protocol regressions are not real Tailscale policy validation.
  - **Authentication method “Key store”** (`AuthMethod.KeyStore=5`, since 2026-10-25, shown only on Windows): signs with a non-exportable RSA / ECDSA key from the Windows certificate store, the TPM or a smart card; the private key never leaves the key store, and the profile stores only which key to use
    (`SessionProfile.KeyStoreKey`: key store name, key name, machine-level or not, and the public key line recorded when the key was picked; not a secret, stored in plain text and synced; the public key is only for display and copying, and the key actually in the key store is what counts when connecting).
    - The sixth segment. Hidden on other platforms unless the profile already uses it (synced from Windows) — then it is selected anyway and the picker says “This profile signs with a key from a Windows key store, which this computer does not have. To connect from here, choose another authentication method.”
      The `UniformGrid` leaves no room for the hidden segment, so the other five still share the width evenly.
    - When selected: the label “Key in the system key store” → a drop-down (first line the certificate subject or key name, second line in small monospace “location · type bits · SHA256 fingerprint · key name”, the location being Software key store / TPM / Smart card / another KSP's name, with “(machine)” for machine-level keys)
      + a 32×32 refresh button (tooltip “List the keys again (after inserting a smart card or creating a key)”) + an outlined “Copy public key” button (tooltip: add it to the server's `~/.ssh/authorized_keys` first); a small monospace line under the drop-down repeats the selected key's details;
      with no keys at all, a grey line explains how to create one (RSA / ECDSA P-256 / 384 / 521 in the software key store, the TPM or a smart card, e.g. `New-SelfSignedCertificate -Provider "Microsoft Platform Crypto Provider"`);
      last a grey line “The private key never leaves the key store; this profile stores only which key to use. The TPM or a smart card may ask for a PIN when signing. Windows only.”
    - The list is built the first time the segment is selected and again only on refresh; `NCryptEnumKeys` lists the user- and machine-level key names of the software key store, TPM (Platform Crypto Provider) and smart card KSPs, always silently (no prompt even without a card),
      skipping keys that cannot be opened or are not RSA / ECDSA; certificate subjects come from the key location recorded on certificates in the "Personal" store, without retrieving the certificate's private key. A saved key that cannot be found is listed first, selected, and flagged in the warning color as “not found on this computer”, so opening and saving again does not lose it.
    - Saving without a key is refused (“Key store authentication needs a key chosen from the system key store.”); switching to another method does not keep the reference. Like Tailscale, **no shared credentials**: selecting it clears the credential source and hides that row; the shared-credential editor treats such connections as incompatible and does not offer them as a source to copy from.
    - Connecting: the Windows key store signs (the library asks the server whether it accepts the public key before signing), and Windows asks for a TPM / smart card PIN at signing time; a key that cannot be opened reports “The key cannot be opened from the system key store, or it cannot be used.” without contacting the server;
      on other platforms: “This connection signs with a key from a Windows key store (…), and this computer has no system key store. Choose another authentication method.” It is never added to the agent (the private key cannot be exported) and cannot be combined with certificate authentication.
      Connection diagnostics count a reference as a saved credential, with the same advice as a private key on an authentication failure. How CSV / JSON and the ssh_config export write it is in section 7 of [session-import.md](session-import.md).
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
  - **FTP's Advanced page: “Server time zone”** (since 2026-10-24, `FtpSettings.ServerUtcOffsetMinutes`, below “Default
    remote path”): the drop-down offers “Same as this computer” (the default, stored as null, i.e. the old behavior) plus the
    real-world offsets from UTC−12:00 to UTC+14:00 (including :30 / :45).
    - It means **the server clock's UTC offset**, not “how many hours ahead of this computer”: daylight saving time on this
      computer or traveling with the laptop needs no change; when the server itself switches to daylight saving time it has
      to be changed by hand for that half of the year (a fixed offset can do no better; WinSCP is the same).
    - It only affects how LIST times are converted in directory comparison and synchronization (§6 “Directory comparison and
      synchronization”): LIST gives the server's wall-clock time without a time zone; MLSD / MLST times are UTC already and
      unaffected; the file panel's time column still shows the server's clock.
    - An offset edited in by hand that is not in the list is inserted in order rather than silently changed; values outside
      UTC−12:00 … UTC+14:00 become null. A chosen value puts the dot on the “Advanced” tab.
- **Feedback strip**: the result of Test / Save (one accent line on success; on failure the error text + a copy button)
  is pinned above the footer, outside the scrolling form — when the user presses Test their eyes are on the footer, and
  the answer must not land somewhere scrolled out of view. When “Test” fails on algorithm negotiation, “Allow these algorithms and test again”
  appears below the error (since 2026-10-25; rules in §7).
- **Footer**: left = a destination preview (monospace 11px `text-tertiary`, after the protocol icon): `user@host:port`
  for SSH, `sftp://` / `ftp://` / `ftps://` for SFTP / FTP (explicit and implicit FTPS both count as `ftps`), IPv6
  literals in brackets, no user for anonymous FTP or credential-less plugin protocols, no port for protocols declaring
  `NoEndpoint`; hidden while the host is empty. It updates as you type, so you can check at a glance where you are about
  to connect. Right = `Test` / `Save` (outline) / `Connect` (accent pill, DESIGN.md §5.1), all 28 high.

- **Workspace variants** (a plugin workspace's `VariantKey` / `Variants`; SDK contract in [Development Guide §5.14.1](../templates/dev-guide.md)):
  once a type is chosen the dialog is reshaped by that type's variant; whatever the variant does not declare is inherited from the descriptor.

  | Follows the variant | Rule |
  | --- | --- |
  | Port | The variant's `DefaultPort`. **Followed only while the current port "is a default value"** (the descriptor's default or any variant's default); a port the user typed (e.g. 13306) is left alone. Switching to another plugin tab takes that tab's default port, not the previous tab's variant port. Coming back to the same plugin tab from a built-in protocol (SSH, …) restores the port it had before you left (the profile's own port) instead of the tab's default; a port typed by hand on the built-in protocol is left alone |
  | Host label | The variant's host label |
  | Credentials | Shown or hidden by the variant's `Features` (`NoCredentials` / `AnonymousAccess`); the credential-source picker is collapsed with them. In the dialog the values are only hidden, not cleared (they are back when you switch to a type with credentials); saving does not store the username and password, and opening the session never prompts for sign-in and hands the plugin empty strings (even if an older profile still holds them) |
  | SSH tunnel, certificate trust | Follow the variant's `Features`; the pre-open check and the dialog read the same result |
  | `Features` | **Replaced as a whole**, not merged; with a missing key the variant is looked up by the field's `DefaultValue`, and with no matching variant the descriptor's `Features` apply |

  The variant field's row is **never shown** in the dialog (it only carries the value); if it is the only plugin field on the "General" page, that page's section heading is omitted too.
- **Type-picker step** (the first step when creating a plugin workspace connection; for a plugin without variants it does not exist and the dialog is pixel-identical to before):
  - **When it appears** (decided by the host, not an SDK switch): `VariantKey` is set and the field it names is a `Choice`, not hidden,
    not free-text (`AllowsCustomValue` is `false`), with at least 2 candidates; and this is a **new** connection (editing an existing
    one always opens the form) whose type has not yet been confirmed. Dynamic choices, free-text, a single candidate, a hidden field
    and no `VariantKey` all show no cards.
  - **Stage 1 (pick a type)**: the right column hides the page-tab strip and the form and shows the heading "Choose a connection type"
    plus wrapping type cards (name from the choice `Label`; icon from the variant `Icon`, falling back to the descriptor `Icon`, then
    to the generic plugin icon). The default value is preselected. The destination preview in the footer's left is hidden (there is no
    endpoint yet); on the right `Cancel` (outline) / `Next` (accent pill, `IsDefault`).
  - **Interaction**: a single click only selects, without leaving; a double click selects and goes "Next"; `Enter` = "Next" (the dialog's
    default button); cards are buttons, `Tab` order = candidate order; with a card focused, `Enter` selects it and goes "Next" (same as a double
    click) and `Space` only selects. On entering stage 1 the focus lands on the selected card, so `Enter` can go straight on. `Cancel` closes the dialog.
  - **Next**: writes the chosen type into the variant field, the port / host label / credentials / tunnel change as in the table above,
    and stage 2 opens on the "General" page. The port follows only while it is a default value; every other value already entered (host,
    user name, fields) is left untouched. Choosing the same type again and pressing Next (after "Change" when editing, or after "Back")
    changes nothing, port included; only for a new connection whose type has not been confirmed yet (for example after switching back
    from another protocol) is the port refreshed from the variant once.
  - **Stage 2 (form)**: the page tabs and form as usual, with one extra header row at the top of the form, "Type MySQL" (label + current type name, no colon); the footer shows
    `Back` (outline, **new connections only**) / `Test` / `Save` / `Connect` (the button set is not changed to "OK").
  - **Back**: returns to stage 1 with the current type preselected; **no entered value is cleared and the port is not touched**, so the
    values are all still there after "Next".
  - **Editing an existing profile**: opens straight on the form with no `Back`; the header reads "Type MySQL · Change" (the dot is drawn), and "Change"
    returns to the cards, after which "Next" switches the type (the workspace id is unchanged, entered values are kept, the port follows
    only if it is a default; pressing Next without changing the type changes nothing). `Cancel` here also closes the dialog.
  - **Switching protocol tabs**: switching to another plugin tab or a built-in protocol resets this step; coming back before the type
    was confirmed shows the cards again, and after it was confirmed stays on the form.

  | Stage | Page tabs / form | Card panel | "Type" header | Footer left | Footer right |
  | --- | --- | --- | --- | --- | --- |
  | Pick type (new) | hidden | shown | — | hidden | `Cancel` `Next`* |
  | Form (new) | shown | hidden | shown (no "Change") | destination preview | `Back` `Test` `Save` `Connect` |
  | Form (edit) | shown | hidden | shown + "Change" | destination preview | `Test` `Save` `Connect` |
  | Workspace without variants | shown | — | — | destination preview | `Test` `Save` `Connect` (as before) |

  \* `IsDefault`.
- **Plugin field pages** (page keys come from the SDK's `Pages` / `Page`, see [Development Guide §5.14.1](../templates/dev-guide.md); available since: 2.1.5):
  - A field's page key **takes precedence over** `IsAdvanced`; an unknown page key is treated as unset and falls back to the default place ("General" or "Advanced").
  - Custom pages have no count badge, no modified-value dot and no auto-expand when editing (only "Advanced" has these three).
  - When all fields of a page are invisible the page tab is hidden, and if the user is on it they fall back to "General".
  - If the descriptor has no custom pages (or all are hidden), the whole group of custom page tabs is absent and the dialog is as before.
  - In the type-card stage the custom page tabs are hidden together with the tab strip.

### 13.2 Password Verification Dialog (Two Steps)

- **Step 1 (`oNZIM`, window 520 DIP)**: username input, target and known-host fingerprint; `Cancel` / `Next`.
- **Step 2 (`twD13`)**: full-width Password / Certificate / Key / Tailscale SSH segments and their material; `Previous` / `Login`. UI indexes are explicitly mapped to the enum. Tailscale requires only a username and never requests secrets, remembers a password or updates a shared credential. The Certificate segment enables `Login` only once both the certificate and the private key are filled in, and picking a certificate fills in the key by `-cert.pub` the same way (§13.1, “Authentication method “Certificate””). FTP and plugin protocols accept only a password, so the segments are hidden and the step shows the password field directly (the other segments mean nothing to them; picking one would only change this attempt's method and clear the password, so an empty password would be sent). OTP and host-key decisions are separate interactions.
- **One-time code dialog (keyboard-interactive, since 2026-09-29, `KeyboardInteractivePromptDialog`)**: shown when a
  two-step verification server that only allows keyboard-interactive (PAM + TOTP, Duo, bastion MFA) asks for a code.
  - **When it appears**: a single non-echoed prompt that looks like a password prompt (the word "password" in English,
    Chinese, Japanese or Korean, and nothing like verification code / OTP / token — those usually want "password + code"
    typed together) is **answered once** with the saved password, without a dialog; if the password is asked again (a
    password-change flow, or the previous one was wrong) the user answers. Everything else opens the dialog. An
    information-only round (the server sends instructions but no prompts) does not open it; its text is carried into
    the next dialog.
  - **Not shown during a session health check** (§11C): password prompts are still answered with the saved password; a one-time code or password change is declined without a dialog and reported as
    "The server asks for interactive verification (a one-time code or similar); the health check does not answer it. Connect to it manually once." / "The server requires a password change; the health check does not do that. Connect to it manually once."
    — a responder that declines everything is attached instead of none at all: without one, the library's built-in "password also answers" would type the password into the code round and burn a one-time-code failure.
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
- **Host fingerprint confirmation**: on the first connection to an unknown host, or when a recorded fingerprint changes, show a “Host Trust” confirmation (fingerprint + Accept and Save/Trust once/Reject; a change also shows the recorded fingerprint from known_hosts alongside), linked to §15 Host Trust Center. A change raises this dialog by default rather than being refused outright (#476); strict fail-closed behaviour is the “Block and alert on fingerprint change” switch on the Security Audit page. “Confirm on first connection” is on by default (since 2026-10-25, matching OpenSSH's `StrictHostKeyChecking=ask`; it used to be off, trusting on first use, and is switched on once on upgrade, after which the user's choice stands). Without a main window to show the dialog on, the answer is reject.
  **Automatic reconnects never show this dialog** (a terminal tab's countdown reconnect, the batch reconnect after waking from sleep, a tunnel's automatic redial): usually nobody is at the screen,
  and the dialog would either pile up unattended modal windows and leave the reconnect hanging, or get clicked away without a look — both defeat the point of having someone check the fingerprint.
  Whatever would have asked is rejected instead; the disconnect notice says that automatic reconnect stopped and to reconnect by hand to check the fingerprint, and a security alert is still recorded
  (audit log: “rejected a host fingerprint”); every hop of a jump chain behaves the same. The two settings that never ask are unchanged: when the user turns “confirm on first connection” off an unseen host is still
  recorded on first use (TOFU), and with “block on change” on a change is still blocked. The login dialog for missing credentials and the one-time-code dialog are not affected and still appear.
  **The session health check** (§11C) does not show this dialog either and rejects outright, saying "The health check did not accept host key {fingerprint} of {host} … Connect to it manually once …"; a first-seen key raises no security alert, a changed one still does.
  **Each key type of a host is recorded separately** (servers often have RSA, ECDSA and Ed25519 keys at once): a key matching any recorded one is accepted; accepting a key of a new type **adds a record** instead of overwriting the existing one,
  so whichever type is negotiated later is recognised; only a different key of the same type replaces it. The change dialog shows the recorded fingerprint of the same type, or the most recently seen one when that type was never recorded.
  When connecting, the recorded types go to the front of the host key algorithm list, so a normal server negotiates the type already recorded.
  **Host key rotation is on** (OpenSSH's `UpdateHostKeys` is on by default too): after connecting to a host that is already permanently trusted, the other host keys the server proves it holds are recorded as trusted, per type,
  and a security alert is raised ("now trusted") —— after an administrator adds an Ed25519 key or rotates out an old RSA key, the user no longer sees "fingerprint changed". Old keys the server no longer presents are removed from the trust store by type, with a security alert as well ("removed from the trusted records") —— a type is removed only when every record of that type matches a fingerprint to be forgotten; nothing is done for "trust once". The “Trusted hosts” list has one row per key type, and removing a row removes that type.
  〔History〕Only one key per host:port used to be recorded: a server with an additional key of another type, or a changed host key algorithm for the connection, raised “fingerprint changed”; accepting it overwrote the old key, and switching back raised it again.
  If “Accept and Save” is chosen but the trust store cannot be written, **this connection goes ahead**, the fingerprint is not asked about again for the rest of this run, and a security alert is raised (the audit log records “Host fingerprint could not be saved”) —
  without it, the user would be asked again after a restart and would not know why.
  **Host certificates and trusted host CAs** (since 2026-10-24, OpenSSH's `@cert-authority` / `@revoked`; the CAs are managed under Settings → Security Audit → “Trusted host CAs”, see §14). The order of decisions:
  - **Revocation overrides everything**: if the presented key (or the CA that signed its certificate) is in a revocation record, the connection is aborted with “…has been revoked… Do not connect.”, without asking.
  - A key recorded individually for this host in the trust store is accepted as usual (individual trust comes before CAs).
  - When the host name matches a trusted CA's patterns and the host presents a valid host certificate signed by that CA (type, signature, validity, principals and critical options all check out), it is **accepted without a prompt** and not recorded in “Trusted hosts”.
  - An invalid certificate is rejected; the host is managed by a CA, so it **does not fall back** to asking as for a new host. Which check failed is decided by the SSH library in the order of [spec 03 §5.5](../ssh/spec/03-key-exchange.md#55-host-certificates--cert-v01opensshcom) and handed back as a structured result (signature does not verify, CA key too short, signature algorithm not accepted (e.g. SHA-1), expired or not yet valid, no principals or this host not among them, unknown critical options…); the host turns that into a message in the UI language, a distinct one for each reason, including the certificate's Key ID.
  - A CA-managed host presenting a key that CA does not vouch for is rejected as well, without asking and without recording — whatever “Confirm on first connect” and “Block on change” say, and on automatic reconnects alike; the message gives the next step (verify the fingerprint and import its key from known_hosts, or adjust the trusted host CAs).
  - When a CA matches, connecting moves the certificate host key algorithms to the front (plain types recorded individually stay first), so a server that has a certificate presents it.
  - When the host presents a certificate but no trusted CA covers it, the usual confirmation for the key inside the certificate appears, with an extra “Certificate CA” row in the info block (the CA fingerprint plus a note: after verifying the CA, add it under trusted host CAs and hosts it signs no longer need confirming one by one). The dialog **deliberately offers no** “trust this CA” button — trusting a CA on the strength of a certificate the server itself presents is blind trust.

**Connection state machine**: `Idle → Connecting (yellow) → Authenticating → Connected (green)` / any failure `→ Disconnected (red)` with reason and retry.

---

## 14. Settings Panel (implemented as an 840×740 dialog, 200px left navigation + right content)

The left side contains navigation sections and the right side shows the corresponding content. **The current implementation has 15 pages** (Output Rules, Shared Credentials, Proxy, X Server, Cloud Sync and Support and Donations were added beyond the design; see the itemized remediation ledger in [settings-audit.md](settings-audit.md)):

| Page | Frame | Implementation status (2026-07-12) |
|---|---|---|
| General | `2BIRD` | Startup/tray/language/connection defaults/session logs/settings import / export (by category, see below the table)/behavior and automatic reconnect; under “Restore sessions” in the “Startup & Window” section sits the sub-option “Restore split layout” (off by default, disabled while restore sessions is off, see §4C); below “Check for updates on startup” sits “Download updates automatically” (since 2026-10-24, off by default, with existing configurations migrated to off once; disabled while the startup check is off; shown only on installs that can update themselves in place — Linux, the Store build and read-only install folders don't have it): when the startup check finds a new version it downloads and verifies it in the background and then offers a restart, never restarting on its own; the update channel is on the About page, and the unimplemented “Master Password” group is hidden |
| Appearance | `ZAbb9` | Theme (twelve named themes + follow system, live preview), accent color (follows the theme out of the box: “Follow theme” in front of the swatches = no override, use the current theme's own accent — since 2026-09-29; the previous factory pink `#E91E63` covered every theme's accent, and saved configurations are not migrated; any other color comes from the color picker, see 14.3), UI font/size, opacity, terminal colors (color picker, offering the scheme's 16 ANSI colors as swatches) and color schemes (defaults to the active theme's paired scheme, see below); the “Window” section has the tab bar position (top / bottom, every pane changes together, previewed immediately, see §4B) and Show Tabs in Multiple Rows (§4B.3) |
| Terminal | `08FpM` | Font/line height/TERM/encoding/cursor/three-state bell/scrolling/copy and paste/IME/commands run after connection; scrollback lines (default 10000, 100–200000) apply to open tabs on save, **lowering the value trims the excess history at once and it cannot be recovered**, and it applies to the main screen only — the alternate screen of full-screen programs never has scrollback (details in [Settings Audit](settings-audit.md), “Terminal · Font and Display”) |
| Output Rules | — (new, 2026-10-25) | The user's highlights and auto-answers, sharing one table (list + editor + test), see §14.6 |
| Key Management | `UBP59` | Enumerates `~/.ssh` (type + SHA256 fingerprint), generates Ed25519 / ECDSA / RSA keys (OpenSSH format, generated and written by the SSH library), imports (since 2026-10-24 the SSH library parses the whole private key: an incomplete or damaged key, or an unsupported one — legacy encrypted PEM, an unknown cipher — is refused; a passphrase-protected key is accepted and the status line says the passphrase is needed when connecting; a same-named `.pub` that is not a public key, or does not belong to that private key, is refused and no file is kept)/deletes (the confirmation lists the files to be removed)/copies public keys/exports (since 2026-10-24, each row's “Export key” menu: “Export public key…” saves `name.pub`; “Export private key…” first explains the consequences in a confirmation — anyone holding an unprotected private key can log in, keep it out of cloud drives, chats and repositories — and only “Export anyway” saves it, copied byte for byte; the file is 0600 on macOS / Linux, and on Windows it inherits the target folder's permissions), default authentication key. **Toolbar**: search box · algorithm dropdown (150 wide, tooltip “Algorithm”) · `Import` (outline) · `Generate key` (primary). The dropdown has five fixed entries in this order: Ed25519 (recommended, the default first entry) / ECDSA 256 / ECDSA 384 / ECDSA 521 / RSA 4096 — **key sizes are deliberately not selectable**: RSA is offered only at 4096 (wherever 2048 works, 4096 works too, not the other way round), and each of the three NIST ECDSA curves is its own entry (SSH recognizes only these three); the chosen entry is not persisted. **The automatic name follows the algorithm**: `velashell_ed25519` / `velashell_ecdsa256` / `velashell_ecdsa384` / `velashell_ecdsa521` / `velashell_rsa`, with `_2`, `_3`… appended while it collides with a key already in the list; no file name is asked for. If the private key or `.pub` file already exists, it reports “Key X already exists.” and overwrites nothing. Both the private key and the public-key line carry the comment `velashell@<machine name>`; afterwards the status line reads “Generated name (type).” and the list refreshes. The `velashell_rsa` that older versions always generated keeps being listed and is never touched. An “Add keys to the agent automatically” toggle in the “SSH Agent” section (off by default): after a successful private-key-file or certificate authentication the key is added to the local ssh-agent in the background (for certificate authentication, the certificate together with its key, as `ssh-add` does when it finds a same-named `-cert.pub`), skipped if the agent already holds it; an absent or refusing agent is only logged and never affects the connection |
| Shared Credentials | — (new, 2026-10-03, #550) | One set of username + password / private key / certificate / Agent shared by many connections; change it here once and every connection referencing it uses the new value on its next connect. Toolbar: search (name, username, notes) + “New Credential”; table columns: name (with a warning icon when this device has no password or key for it) / username (“From connection” when it carries none) / method / in use (N connections) / actions (edit, delete). **Editor** (`SharedCredentialEditorView`): a “Copy from a connection” dropdown, name, username (optional), the method segmented control and its fields, notes; below, a checklist “Connections using this credential” (filterable; connections the current method cannot serve are greyed out — FTP and plugin protocols only take passwords) and “Select matching connections” (ticks the connections that still store exactly the same username and material on their own, for moving existing connections over). Saving takes effect at once, without the settings window's Save: newly ticked connections switch to it (a username equal to the credential's is cleared so it follows the credential), unticked ones get the credential copied back onto themselves. **Delete** asks first; every connection using the credential keeps its own copy of it and still connects afterwards. Footer note: passwords and key passphrases are stored AES-256-encrypted on this device; cloud sync carries names and usernames, and passwords and passphrases only with an end-to-end passphrase |
| Keyboard Shortcuts | `YQvri` | Every key binding; the global and tab bindings can be changed, unbound or reset (2026-10, joesdu/VelaShell#551; rules under “Customizing shortcuts” in [Keyboard Shortcuts](keyboard-shortcuts.md)). Entries are checked one by one against actual bindings |
| File Transfer | `HGwa7` | Paths/editor/remote editing (double-click a file, upload after editing; see §6)/concurrency/conflict handling (when a file exists, resume interrupted transfers, retry failed transfers — 0–10 times, default 3, since 2026-10-24, see §9)/hidden files/bandwidth limits/transfer logs (conditionally visible) |
| Security Audit | `glqQE` | Session recording toggle + toggle for masking passwords / tokens in recordings and session logs (on by default; see the fourteenth batch in [settings-audit.md](settings-audit.md)) + playback center entry, host trust policy (confirm on first connection on by default, block on fingerprint change off by default), trusted host management (address redaction; OpenSSH known_hosts import / export since 2026-10-24), trusted host CAs (since 2026-10-24; both described below the table), alert channels (in-app/sound/Webhook), audit log (“View” opens the audit log window, see §15; besides connections and host fingerprints it also records changes to connection profiles, settings and shared credentials since 2026-10-24; retention days, default 180) |
| Proxy | — (new) | One proxy shared by every outbound connection: none / follow system (default) / HTTP / SOCKS5, plus “resolve DNS through the proxy” |
| X Server | — (new, 2026-09-23) | The local X server. The first section is the **Engine**: built-in (default, ships with the app, every platform) or VcXsrv (installed by the user on Windows; **not bundled**); the section only appears on Windows, other platforms only have the built-in one. With built-in selected a note below explains that each X window is a native window and SSH X11 forwarding connects straight into the built-in server without a local port; how local programs connect is written per platform (`DISPLAY=localhost:N` on Windows, `DISPLAY=:N` on Linux / macOS — the Unix socket, with TCP off by default). Shared by both: **Display** (display number: auto or :0–:15, with the description likewise per platform; window mode: multiple windows / one large window / no title bar / fullscreen / rootless, and on the built-in engine one large window / no title bar / fullscreen start it in one-window mode) / **Clipboard** (enable — the built-in engine exchanges only with the session that has the keyboard focus, as the description says; copy on selection — off by default, with the description saying that when on, whatever is copied locally can be pasted into X programs with a middle click) / **Keyboard layout** (auto = follow the system's current layout, or pick one of 27 common XKB layouts) / **Startup** (start with VelaShell, start automatically for X11 forwarding). Shown only with the built-in engine: in the "Display" section below window mode, **Transparent X windows** (since 2026-10-10, off by default; the description says windows with alpha are shown opaque while it is off and that it has no effect in the single-window modes) and **Show where X windows come from** (since 2026-10-10, on by default, **applies immediately**; titles, taskbar entries and tray tooltips of windows forwarded over SSH start with `user@host:port`, and the description says windows that draw their own title bar show it only in the taskbar); in the "Keyboard" section below the layout, **Use this computer's input method in X windows** (since 2026-10-10, on by default); and a **Security** section (its own section since 2026-10-10; these used to sit under "Keyboard"): **Restrict programs from SSH sessions** (off by default; when on, programs forwarded over SSH cannot simulate input, listen to every key typed in X windows or change input devices, and the description names the cost: tools such as xdotool on those servers stop working), **One display per SSH session** (since 2026-10-10, off by default; when on, X programs from different SSH sessions get separate displays — they cannot see each other's windows, read each other's keyboard input or clipboard, or send each other input — while X programs on this computer keep using the shared display; the description names the cost of a little memory and a thread per session) and **Also listen on a TCP port** (since 2026-10-10, Linux / macOS only, off by default; the port listens on 127.0.0.1 only, and the description says containers on a bridge network cannot reach it and should mount `/tmp/.X11-unix` on Linux). VcXsrv-only, shown only when VcXsrv is selected: **Program** (VcXsrv location, empty = auto-detect; offers `winget install marha.VcXsrv` when not found), **Keyboard** (model, capture special Windows keys), **Extensions** (native OpenGL, disable access control), **Advanced** (additional arguments + a “Help” dialog documenting every VcXsrv command-line argument in five languages, filterable; the command line for the next start is previewed live below), show tray icon. With the keyboard layout on “auto” the built-in engine follows the system's current layout (Windows, macOS, and Linux with a desktop X display; including the AltGr level, which is the Option level on macOS); a chosen layout uses the keymap tables shipped with the app. Changes apply the next time the X server starts (except "Show where X windows come from") |
| Snippets | `HBNhv` | Common command snippet library with collapsible groups (shared by command palette/completion suggestions through SonnetDB `quick_commands/commands` v2); supports adding, editing, deleting, and changing group assignment, built-in commands included (with restore to default); groups and commands can be reordered by dragging; command text may contain variable placeholders (see §14.2) and span several lines (see §14.4) |
| Cloud Sync | — (new) | GitHub Gist multi-device sync: token/Gist binding, end-to-end encryption passphrase, sync scope (app settings / connection profiles / snippets / layout templates — the last since 2026-10-25, on by default, see §4C), version history, and restore (see plan.md §13.C). Without an end-to-end passphrase it does not upload the passwords and key passphrases of connections / shared credentials, nor the proxy password and alert Webhook URL from the settings (the same list as settings export, since 2026-10-25); a pulled payload without them keeps the local values (the local proxy password only when the proxy type, host, port and user name all match; the Webhook URL always); with a passphrase the whole payload is encrypted and these travel with it |
| About | `Nwoks` | Version/runtime environment/dependencies/contributors (clickable GitHub avatars)/dual licensing and authenticity statement; Check for updates: on Linux a found update opens, in the browser, the download link of the package matching how VelaShell is installed (deb / rpm / AppImage / tar.gz portable), with the file name and link in the status line; deb / rpm of a prerelease open that version's release page, and an install type that cannot be recognized opens the releases page (since 2026-10-24; before, it always opened the releases page). Windows / macOS download into writable installations and offer a restart: with “Download updates automatically” on the General page, once the startup check has downloaded the update in the background the About page directly shows “Update X is ready. Restart to finish installing.”, and if the background download failed its status line reads “Update X couldn't be downloaded in the background. It will be retried at the next startup check, or click "Check for Updates" to download it now.”. Microsoft Store manages updates for the Store build (see [Message center and feed](message-center-and-feed.md)) |
| Support and Donations | — (new) | Alipay/WeChat/Wise donation and contribution guidance |

Common interaction: click on the left to switch; scroll on the right; Appearance provides live preview (save persists changes, cancel rolls them back); “Restore Defaults” and “Clear History” show confirmation dialogs; `Ctrl+,` opens Settings.

**Settings import / export** (General → “Data & Storage”, by category since 2026-10-24):
- “Export” first opens a category picker: General, Message center, Appearance, Terminal, Key Management, Shortcuts, File Transfer, Security & Audit, Proxy, X Server (in the order of the left navigation, with Message center after General). **Security & Audit and Proxy are unticked by default and carry a “Sensitive” tag**; ticking one shows in a warning color what it brings along: Proxy = the proxy server's address, port and user name (not the password); Security & Audit = the host-key policy, the external-launch confirmation and trusted targets (user@host) and the alert switches (not the webhook URL). “Export…” needs at least one category, then asks where to save (the Downloads folder by default, file name `velashell-settings.json`). What is exported is the settings page as it is right now (unsaved changes included).
- **The proxy password and the alert webhook URL are never written, ticked or not**; window size and position and the sessions open last time are not exported. The file is the settings JSON with only the chosen blocks, plus a `velaShellExport` header (format `velashell-settings`, version 2, export time, categories). Connections, groups, keys and snippets are not part of the settings export (connections have their own connection file export).
- “Import” reads the file (files over 4 MB are refused) and lists the categories it carries (a whole-file export from an older version = all categories); the sensitive ones are unticked by default here too, and their warnings now say what they would change (Proxy: the proxy every later outbound connection uses; Security & Audit: host-key confirmation, the external-launch confirmation and trusted targets). Only the ticked categories are overwritten, the rest stay; as before it only fills the settings page, and nothing is written until “Save”.
- When the file carries no proxy password, the local password is kept only if the proxy's type, host, port and user name all match the local ones; otherwise it is cleared and you are asked to fill it in. The webhook URL is kept from this machine; if webhook alerts are on but there is no URL, you are asked to fill it in too. A JSON in which no category is recognized is reported as “That file is not a VelaShell settings export. Nothing was changed.”.
- Feedback appears below the buttons: “Settings exported to {path} ({categories}). The proxy password and the alert webhook URL are never written to an export.” / “Imported: {categories}. Review the settings, then save.” / “Imported: {categories}. Exports never carry these — fill them in before saving: {proxy password, alert webhook URL}.”

**known_hosts and trusted host CAs** (Security Audit page, since 2026-10-24; all of these take effect immediately and are not tied to the settings window's “Save”):
- Below the “Trusted hosts” list, an “OpenSSH known_hosts” row with “Import…” and “Export…”, to share one trust baseline with command-line ssh.
  - **Import** (files over 16 MB are refused) only adds, never overwrites: several hosts on a line, `[host]:port` and negation within a line are all understood; hashed host names (`|1|…`) are matched one by one against saved SSH / SFTP connections and trusted hosts, and those that match nothing are counted; plain lines with wildcards are skipped and counted (host keys can only be trusted for concrete hosts); when the host already trusts another key of the same type, nothing is overwritten and it counts as a conflict (the message says to delete the old entry in the list first); `@cert-authority` / `@revoked` lines go into the trusted host CAs below; revoked plain lines are not imported; lines with unknown markers are skipped. The result line says how many host keys, trusted CAs and revocation records were imported, and how many lines were skipped for each reason.
  - **Export** writes OpenSSH format, host keys plus CA / revocation records; old records that only kept a fingerprint and no public key cannot be exported and are counted separately (the key is filled in the next time a trusted connection to that host succeeds).
  - “Hash host names on export” (off by default, **not persisted**): like `HashKnownHosts yes`, host key lines are written as `|1|…`; the patterns of `@cert-authority` / `@revoked` lines are written as they are.
- The “Trusted host CAs” section lists trusted CAs and revocation records, each row with a kind tag (“CA” in accent / “Revoked” in the error color) + host name patterns + key type, fingerprint and comment, and “Delete” at the end (no confirmation). Below it is the “Add a trusted host CA” form: host name patterns (comma-separated, `*` `?` wildcards, `!` excludes, non-22 ports written as `[host]:port` — the host column of known_hosts), the CA's public key line (the content of the CA's `.pub`, or a whole `@cert-authority` / `@revoked` line copied from known_hosts) and an optional comment, then “Add”. Mistakes are reported below the form: no or malformed patterns, only negated patterns, not a valid public key line, a CA key that is itself a certificate, an RSA CA key under 2048 bits, the same patterns already trusting this CA. Revocation records can only be added by import or by pasting a whole `@revoked` line.
- CAs and revocation records live in the host's own trust store; `~/.ssh/known_hosts` is not read on each connection. How a connection is decided is in §13.2 “Host certificates and trusted host CAs”.

“Changelog” on the About page opens an in-app modal dialog and selects notes using the installed version: **first read the latest stable release's** `releases/latest/download/latest.json` and compare it with the installed version. A newer stable release shows that release's notes with an “Update available” badge; otherwise show the installed version's notes with a “Current version” badge. When there is no newer stable release, reuse the latest response if its version equals the installed version; only fetch the installed version's manifest if its notes are needed and that response cannot be reused, using `releases/download/<tag>/latest.json` for the installed version, which may itself be a prerelease. This is neither current-version-first rendering with a parallel update query nor a historical-version picker. A specific-version lookup tries only a bounded set of tags with / without the `v` prefix and three-component version normalization, and only after a 404; it never enumerates releases. The manifest tag must match both the requested tag and expected version. If latest-stable discovery fails, still try the installed version and separately report that the available stable release could not be checked, with a retry action; a failed check must not mean “no update.” An unparseable installed version produces an explicit identification error. Stable discovery in this dialog neither reads nor changes the update-channel setting; the existing preview behavior of About → Check for Updates and startup checks is unchanged.

Notes show the root-level `tag` and complete body in `releaseNotes`. Requests use the attachment download client and are parsed as a stream. They neither query the releases list nor fall back to the anonymous Release API on failure. Results for each URL are cached in-process for 5 minutes, then revalidated with `If-None-Match` when the server provides an ETag; a 304 retains the cached result. Store builds can also read notes; merely opening the dialog does not download or install an update. The header shows the version tag, publication date, and the version badge described above. The `ReleaseNotes_Latest` resource key is retained but now means “Update available”; the compatibility constructor without version selection temporarily uses the same label. “View on GitHub” in the footer opens the selected release's `releaseNotes.htmlUrl` from the manifest, but **only this repository's releases pages are allowed** (since 2026-10-25): https, host exactly `github.com`, no user info or port, no query or fragment,
and a normalized path under `/joesdu/VelaShell/releases` whose segments contain only letters, digits and `. _ -` (no dot segments, empty segments or percent escapes); a link that passes is rebuilt from the checked path and opened. A missing or failing link opens this repository's releases list
`https://github.com/joesdu/VelaShell/releases` instead, and the button stays clickable (before, only https was checked and the button was disabled when there was no link).

**Update actions are bound to the displayed version**: offer an update action only when displaying a newer stable release and the same response contains a usable installation-manifest snapshot for the current platform. Missing or invalid installation assets do not hide readable notes, but must not produce an incomplete update target. Writable Windows / macOS installations show “Update now”; download using the displayed snapshot's tag, version, asset URL, and checksum, without rediscovering latest, switching targets after a background check, or changing preview settings. A finished download only becomes ready; **installation waits for the user to choose restart to update**. Ready means a verified package is actually in hand: a download with no selected target reports a failure rather than readiness. Before swapping versions the downloaded package is checked again against the checksum recorded when it became ready; if it was deleted or altered, it is removed, the ready state is dropped and the restart reports a failure, and the next Check for Updates or Update now downloads it again. Other failures, such as a file in use or an extraction error, keep the ready state so the restart can simply be retried. Closing the notes dialog does not cancel a download already started. The dialog explicitly says the download continues after closing and that the user can restart to update from About once it finishes. About takes over the same task's progress, failure state, and restart action instead of starting another download. Do not replace an update already running or ready: reuse the state for the same target, or report a conflict for a different target and direct the user to About. On Linux or with a non-writable installation directory, the action becomes “Go to release page” and opens **the selected tag's release page** for manual download and installation, not the generic releases list or a new latest release (same rule: a failing manifest link falls back to the releases list). Microsoft Store builds remain Store-managed and offer no in-app download/install action. See [Message center and feed](message-center-and-feed.md) for progress, speed, and stall handling.

The body renders Markdown headings, paragraphs, lists, emphasis, and inline code, plus GitHub-style autolinks (auto-generated release notes list PRs and the Full Changelog as bare URLs), pipe tables, strikethrough, and task lists; it supports selection and copying across paragraphs, and scrolls vertically. Links, autolinks included, open only absolute HTTPS addresses after a user click; any other scheme is downgraded to plain text. All image nodes are removed before controls are generated, retaining their alternative text; HTTP, local file, data, and resource images are never loaded, and raw HTML is not executed. Body, link, and code colors and font sizes follow the application theme and font settings. The dialog immediately shows a loading state. If the installed tag has no manifest attachment (404), explain that notes for the current version are unavailable, possibly because this is a local build or the version does not yet include a release notes manifest; do not substitute another version's notes or fall back to the anonymous API. This is distinct from an existing manifest with an empty body. An older manifest without `releaseNotes` retains the version tag and shows the empty-body state. It offers “Retry” after a network failure or timeout; when the download service answers 403 / 429 it shows “GitHub is limiting requests right now. Please try again later.” instead of pointing the user at their network. Its close buttons or `Esc` close the dialog and cancel that read. Repeated clicks do not open duplicate windows or interfere with an update download in progress.

Release-note publishing contract: release tags accept only `v?major.minor.patch` plus an optional prerelease part and **reject** SemVer build metadata `+…` (the version goes into the artifact names verbatim, and the in-app updater rejects attachments with `+`, so such a release could never be installed); the pipeline reports it on its own in `prepare-release`, saying how to fix it. `latest.json` retains `version`, `tag`, and `assets` and adds an optional `releaseNotes` object. Its `body` is the complete Markdown string, `publishedAt` is the ISO 8601 publication timestamp or null, and `htmlUrl` is the release-page address or null; notes reuse the root-level `tag`. CI reads these fields from the Release with authentication and calculates `SHA256SUMS.txt` after generating the final manifest. Clients need no GitHub login or token. Editing a published Release synchronizes only the notes and manifest checksum, without rebuilding packages. A manual release-workflow run can enable `notes_only` to backfill an existing manifest; the default remains a full release build. When only the original manifest is missing and `SHA256SUMS.txt` is still there (for example after a previous sync stopped between deleting the old attachment and uploading the new one), `latest.json` is rebuilt from the checksum file's hashes and the sizes of the assets currently on the Release; a missing package or checksum record for any platform still fails. Synchronization fails if the checksum file is missing; complete a full release first. When both attachments are absent (most likely because the build is still running, and its attach job will pick up the latest body), the job emits a notice and skips. Publication and synchronization for the same tag are serialized, but the two attachment uploads are not atomic; rerunning can recover a partial failure. Older clients ignore the added fields. Local packaging emits an empty body and null date/link by default, without adding a network requirement. About → Check for Updates and prerelease-channel discovery retain their existing logic; the notes dialog's stable discovery and fixed-target download do not replace them. The workflow must reach the default branch and generate or backfill manifest attachments for the latest stable release or the installed historical tag being viewed before the new fields become available. **Historical attachments are a prerequisite**: `notes_only` can restore a manifest only from the checksum file and packages already on the Release, and cannot create a missing checksum file or package. Missing attachments produce an honest unavailable state, not an anonymous API fallback. Download-service throttling, network failures, and cache propagation can still affect retrieval.

“Export diagnostics…” on the About page (since 2026-10-19) packs a zip saved wherever the user chooses in the Save dialog. **Nothing is uploaded and no network request is made.**

- **Choose what goes in first** (since 2026-10-25): the button first opens an “Export diagnostics” selection dialog (460 wide, a 28px title bar with the `file-down` icon, a 52px button bar “Cancel / Export…”). A line at the top states what is **always included**:
  the diagnostic logs `velashell-*.log`, crash reports `crash-*.txt`, a version and platform summary, and the plugin list (id / version / state / host mode of every discovered plugin). Below it are the optional items **that actually have files**, one checkbox each,
  followed by “N file(s) · size” and a line saying what it carries: transfer records (looked up in the directory set under “File Transfer → Log directory”, so a relocated one is found too), the external-editing log `remote-edit.log`, `vcxsrv.log`,
  and one item per plugin (`logs/plugins/<id>.log` together with its rotated `<id>.log.1`). **Nothing is ticked by default**; with no optional item it shows “There are no other logs to include.”; exporting with nothing ticked works. Cancel exports nothing; OK opens the Save dialog.
- **Not included**: session logs `session-*.log` and any other file in the log directory; settings, connection profiles and credentials. The two default kinds and the ticked ones are each taken newest first and capped at 32 MB of source text each, skipping a file that no longer fits;
  ticked plugin logs sit under `logs/plugins/` in the bundle.
- **Masking** (each file, before it is written): hosts and login names from saved connections, recent connections and shared credentials become `<host-N>` / `<user-N>` (the same value maps to the same placeholder throughout the bundle; common login names such as `root` or `ubuntu` and values of two characters or fewer are left alone).
  Since 2026-10-25 general rules are added, sharing the numbering with the known sources: both parts of a disconnect line `[SSH] user@host:port closed` (a format VelaShell defines itself, so single-label host names and domain-style user names are recognized too);
  IPv4 / IPv6 addresses other than loopback (four numbers are told from version numbers by context: directly next to a letter, digit or underscore, an extra part, or after `Version=` / `version:` / `VelaShell ` means a version; a bare `1.2.3.4` is always treated as an IP);
  domain names that are all lowercase, have at least two labels and end in a common top-level domain or internal suffix (`.local` `.lan` `.internal` `.corp` `.home` `.arpa` and so on) — suffixes that clash with file extensions (`.sh` `.md` `.py` `.zip` …) are left out,
  plugin ids, reversed identifiers of three or more labels starting with `com.` / `org.` / `net.` / `io.`, and the public services VelaShell itself talks to (`github.com`, `githubusercontent.com`, `easilynet.top`, `db-ip.com` and their subdomains) are not replaced, and mixed case is never touched;
  the name part of `X:\Users\name`, `/home/name`, `/var/home/name` and `/Users/name` (public / default profile folders, placeholders such as `%USERNAME%` / `$USER` and common login names are kept, and `/home/` inside a URL or a relative path does not count).
  The local home directory still becomes `~`, and the local login name, machine name, email addresses and the values of fragments such as `password=`, `token:` and `Bearer …` are masked as before. **Not recognized**: single-label host names that are neither from a known source nor in a disconnect line,
  host names written in upper case, user names elsewhere, file names and other paths.
- The bundle's `README.txt`, in the UI language, lists `logs/: N file(s)`, `optional logs:` (the ticked items' identifiers, comma-separated, or `(none)`), what is not included, what was masked by rule and what is not covered, and how many replacements of each kind were made.
  The line under the buttons reads “Diagnostics saved to … (N log file(s), masked by rule). It has not been sent anywhere; open it and check before sending it to anyone.”, or shows why saving failed.
- The diagnostic log gets one English line whenever an SSH connection ends (including every hop of a jump chain): `[SSH] <target> closed (connected since yyyy-MM-dd HH:mm:ss ±hh:mm): <reason>` —
  the moment the connection started (local time with offset, taken from the wall clock when the connection was attached; subtracting it from the line's own timestamp gives the wall-clock duration on every platform; since 2026-10-25, before it recorded how long it lasted, which counted sleep differently per platform),
  `closed locally` for a local close; otherwise the library's reason and phase, plus the reason code and the peer's own text when a DISCONNECT was received — control characters stripped and truncated so it always stays on one line.

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
  - Each input is pre-filled with the value last entered for that quick command (since 2026-10-24; remembered per quick
    command + placeholder name, falling back to the default when nothing was remembered). It lives only in the current
    run: never written to disk, never in cloud sync or import / export; placeholders whose names look like credentials
    (containing pass / pwd / secret / token / otp / credential / apikey, or 密码 / 口令 / 密钥 / 令牌 / 验证码 and the like) are not
    remembered even for the run; an input left empty is not remembered, so next time it is back to the default.
- **Line breaks in a value become spaces**: a quick command sends its text without Enter, so a line break inside a
  value would press Enter for the user.
- Target tabs may disconnect while the dialog is open: targets are picked again from the current tabs when sending,
  and only those still connected receive it.
- In Settings → Snippets, the new / edit area has a one-line syntax hint under the command input.
- Picking a quick command with placeholders in terminal command completion (Enter / click in the panel, or accepting such
  ghost text with Right / End; since 2026-10-24) first opens the same dialog, and after confirmation writes the
  substituted command into the command line by completion's usual continuation rule (only the rest when it shares the
  typed prefix, otherwise erase and rewrite); on cancel, a dialog error, the tab disconnecting while the dialog is open, or
  the command line changing, not a single byte is written. A history entry identical to that quick command asks as well.
  Before this, the raw text with placeholders was inserted.

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

### 14.6 Output Rules (2026-10-25)

The page after "Terminal" in the navigation (`SettingsSectionKey.OutputRules`). Rules live in `TerminalBehavior.OutputRules` (empty out of the box, at most 64), travel with the "Terminal" category in export / import and the audit, and with app settings in cloud sync;
changes are written with the window's "Save settings" and apply to open tabs right away (`TerminalSettingsApplier`; local terminals take only global rules).

- **Page**: title “Output Rules” / subtitle “Highlight terminal output with your own regular expressions, or answer prompts automatically”; an explanation (main screen of SSH and local terminals, above the built-in semantic highlighting, higher in the list wins, travels with the "Terminal" category in export and cloud sync);
  a safety card (`bg-input` fill, `shield` icon in the warning color: auto-answer only for yes / Enter / fixed text, never a password in a rule, passwords go through shared credentials, rules that would match a password prompt cannot auto-answer);
  the “Rules” heading with a primary “New rule” button on the right (disabled at 64 with “At most 64 rules.”); with no rules, “No rules yet. The built-in semantic highlighting is not affected.”.
- **List** (`table`, rounded outline): each row has the enabled checkbox | name + action pill (`accent-dim` fill, `accent-text`: Highlight / Auto-answer) + scope (Global / Group: X / Connection: X / No longer exists) | pattern (monospace, accent) |
  why validation failed (error color) | four 24px icon buttons: move up / move down / edit / delete. A rule failing validation is not deleted; it just does not take effect.
- **Editor** (works on a copy): name, pattern (monospace, at most 512 characters) + “Case sensitive” + a note (linear-time engine, no backreferences or lookarounds), action drop-down (Highlight / Auto-answer), scope drop-down (Global / Group / Connection) + target drop-down;
  for highlights: text color, background (“No change” + the current scheme's 16 colors with swatches), bold — at least one; for auto-answer: text to send (monospace, one line of visible text, at most 128 characters), then press Enter (on by default), cooldown (seconds, 1–3600, default 2),
  at most per minute (1–30, default 6), also forward to sync channels (off by default), each with a note; a validation error line (error color); “Cancel” and “Done” (disabled while invalid). On the window's “Save”, a valid draft in the editor is written back as well and an invalid one is dropped.
- **Test**: a note, a multi-line monospace box (at most 200 high), “{1} of {0} lines matched” / “No matches”; each result line = line number + the line cut at its matches (matches underlined and drawn with the rule's text color / background / bold) + an answer note “Auto-answer "X" → sends yes [Enter]”.
  With the editor open only that rule is tested, otherwise all enabled rules; at most 500 lines. In a real terminal auto-answer acts only on the line the cursor is waiting on (§5.2.3).
- **Validation messages** (per kind, five languages): enter a pattern / at most 512 characters / not a valid pattern (near character N) / backreferences, lookarounds, atomic groups and conditionals are not supported / choose the group or connection /
  pick at least one of text color, background or bold / with nothing to send tick "Then press Enter" / at most 128 characters / a single line of visible text / would match a password prompt, cannot auto-answer / comes from a newer version, not in effect.
- Patterns are always compiled with the linear-time engine (`RegexOptions.NonBacktracking`, with a 100 ms match time-out as a backstop) and cached by (pattern, case) across all tabs; empty matches do not count (`y*` does not react to every line). No new shortcuts.

---

## 15. Advanced Feature Panels (Large Standalone Panels)

Open these as tabs or standalone floating windows. All can also be entered from the command palette:

- **Operations Orchestration Center (`bR5c4`, 920×760)**: header + content area + results panel. Execute scripts/playbooks on multiple hosts in batches: choose a target group → configure steps → execute → summarize live results (success/failure/output).
- **Host Trust Center (`gPWeC`)**: policy row (trust policy: strict/TOFU/lenient) + host fingerprint table (host, fingerprint, algorithm, first-seen time, status, delete/reset actions). ✅ **Implemented in simplified form (2026-07-12)**: Settings → Security Audit includes “Host Trust Policy” (three first-confirmation options / block changes) + a “Trusted Hosts” list (view/delete, screenshot-safe address redaction); since 2026-10-24 also OpenSSH known_hosts import / export and “Trusted host CAs” (`@cert-authority` / `@revoked`, see below the table in §14 and §13.2); the standalone large panel is not implemented separately for now.
- **Session Recording and Playback (`NceE6`)**: recording list + player (timeline, speed control, search), asciinema-style playback, and export. ✅ **Implemented (2026-07-12)**: `RecordingPlayerView` is a standalone window, entered through Settings → Security Audit → Playback Center. The left column contains the recording list (name/time/duration/size, delete); the right column contains a read-only terminal + timeline seek + 1x/2x/4x speed + skip idle segments. The title bar's action group (§2) provides export recording (asciicast v2 `.cast`), refresh, clean up (deletes recordings and actually frees the disk space) and the “Auto Recording” toggle (= `Security.RecordProductionSessions`, its icon turns accent while on). Storage is described in Architecture Design §4.11 (SonnetDB time series). The “Search” feature in the design has not been implemented.
- **Audit log (2026-09-29)**: `AuditLogView` is a standalone window (frame, title bar and resize grip as in the recording player), entered through Settings → Security Audit → Audit log → “View”. It lists the event kinds in `audit_log` — connection success / failure, host fingerprint decisions, externally launched logins, and since 2026-10-24 changes to connection profiles, settings and shared credentials — with the columns time, category, event, source, session and detail (the source column since 2026-10-25, 84 wide): events are put into words (Connected / Connection failed / Host fingerprint rejected / Host fingerprint trusted once / Changed host fingerprint accepted / Host fingerprint could not be saved / External login / Profile created, changed, deleted / Profiles moved to Ungrouped / Settings changed / Shared credential created, changed, deleted; unknown ones are shown as stored); the source is Local edit / Cloud sync (restoring a past version included) / Import (connection files JSON / CSV, Xshell, WinSCP, ssh_config) / Login prompt (“also update the shared credential”), and connection and security entries, as well as entries from before 2026-10-25, show —; session names are looked up from the profile id when loading, and a deleted profile shows — (the `user@host:port` in the detail still tells which machine it was). Connection failures, rejections, accepted fingerprint changes and fingerprints that could not be saved count as “problems”, and their event names are red. The filter bar combines four filters: category (All categories / Connection / Security / Profiles / Settings / Shared credentials), time range, “Problems only”, and a keyword (event, category, source, session name, detail; case-insensitive). Only the latest 2000 entries (within the chosen time range) are loaded, and the summary says so when it is full; a database read failure is reported in the summary. Esc clears the keyword first, then closes the window. The window is 1120 wide by default with a minimum width of 960 (so the filter bar fits with a custom range and the detail column keeps about 200 at the minimum; before, 1040 and 880).
  - **Configuration events** (since 2026-10-24): recorded where things are written to disk, so every save path counts — the edit dialog, dragging groups, certificate trust, cloud sync, import, the login prompt's “also update the shared credential” — and each entry carries its source (since 2026-10-25; a `source` tag on `audit_log`, added to an older database automatically when the first entry with a source is written):
    the part of a cloud sync that applies the payload (restoring a past version included) is “Cloud sync”, connection files and Xshell / WinSCP / ssh_config imports are “Import”, the save of the credential from the login prompt's “also update the shared credential” is “Login prompt”, and everything else is “Local edit” —
    the source follows that one asynchronous flow, so an edit the user makes in the dialog while a sync is applying in the background is still a local edit; external launches, certificate trust, the write-back after connecting and settings-file import (the write is the later “Save”) are local edits. **Only which part changed is recorded, never values**:
    - Connection profile: detail “name · user@host:port”; on a change, “old → new” when the name or address changed, plus “changed: name, address, login credentials, other settings” (part names only); the entry carries the profile Id. Dragging to another group counts as “other settings”.
    - Settings: the detail is the names of the changed categories (the same categories as the settings export, e.g. “Appearance, Proxy”).
    - Shared credential: “name · changed: login credentials · used by N connection(s)” (N is the number of connections referring to it at the moment of saving; omitted when 0).
    - Deleting a group moves the profiles still in it to Ungrouped and records one “Profiles moved to Ungrouped” entry (since 2026-10-25; detail “group “X” deleted, N profile(s) moved to Ungrouped”, with no profile Id). Deleting a group in the session tree deletes its connections one by one first, so the group is usually empty by then and nothing is recorded.
    - The comparison is between the stored copies before and after the write; last-connected time, directory bookmarks, pinning, window size and position, the sessions open last time and one-time migration flags do not count as changes, so the write-back after a successful connection, saving the window size on exit and a cloud sync that pulls identical data leave no entries. When the previous copy cannot be read, nothing is recorded; a failure to write the audit entry never fails the save.
  - **Time range**: All time / Last 24 hours / Last 7 days / Last 30 days / Custom…; Custom shows two date boxes, “From” and “To”, counted in local dates with the whole “To” day included, swapped automatically when entered the wrong way round. Changing the range reads the database again.
  - **Export CSV** (title bar action group): exports the rows currently filtered, UTF-8 with BOM; the columns are time (ISO 8601 with time zone), category, event, event code, source (empty when there is none), session, profile Id, detail, with headers in the UI language; a cell starting with `=`, `+`, `-`, `@`, a tab or a carriage return gets a leading single quote against CSV formula injection. Disabled when there are no rows.
  - **Open profile**: rows that carry a profile Id end with an `external-link` button that opens the connection edit dialog owned by the audit window; saving refreshes the session tree as usual. If the profile was already deleted when loading, the button is disabled with the tooltip “This connection profile has been deleted.”; one deleted while the window is open is reported in the summary when clicked.
  - **Retention**: `Security.AuditLogRetentionDays` (default 180 days, 1–3650), shared by the audit log and the connection history (`conn_history`, the ledger behind “Recent connections”) — keeping connection history longer than the audit makes no sense, and keeping it shorter would leave sessions in the audit that the sidebar no longer knows. Older entries are pruned **at startup only**, at the same time as session logs and recordings expire; pruning is a time-based `DELETE`, not the recordings' “stage → rebuild → restore” (a process dying half-way through that would lose entries still within retention, a risk the audit log should take least of all).
- **Connection Diagnostics Center (`RGXg1`)**: the design was 920×640 with ping / DNS / port probes / traceroute / SSH handshake analysis; the implemented window is 768×488 and runs four steps — DNS resolution, TCP connection, SSH handshake, user authentication (the full jump chain when there are jump hosts) —
  with the "Diagnostic steps" panel (320 wide) on the left giving each step's verdict and the right panel giving the issue and advice; the title bar's action group has "Export diagnostic report" and "Re-run checks".
  - **Protocol trace** (since 2026-10-25): the right panel gets a 34-high header bar (aligned with the left "Diagnostic steps"), with two pages “Results” / “Protocol trace · N” (RadioButton segments, the selected one with a `VelaAccentDim` fill and `VelaAccent` text),
    and a copy button at the top right of the trace page (`VelaTitleActionButtonTheme`, “Copy protocol trace”). The authentication step's connection gets the SSH library's packet tap (`IPacketTap`): each packet records its time, hop, direction, message number, length, sequence number and channel,
    with **no field that could hold contents**; only the first 1000 are kept, the rest counted. **Authentication packets we send (50–79) do not record even their length** — the length of a password request or a keyboard-interactive response minus its fixed parts is the length of the password, and a trace is meant to be copied, exported and pasted into tickets; received authentication packets are recorded as usual.
  - Trace page contents: a line of explanation (direction, type, length and sequence only, never contents; authentication packets we send show no length either) → one line per packet in 10-point monospace: `+0.012s`, `→ out` (accent) / `← in`, `KEXINIT (20)`
    (numbers without a fixed name read “key exchange method (31)”, “authentication method (60)”, “local extension (200)”, “unassigned (9)”, and 192 / 193 read `PING` / `PONG`), `1,180 B` (`—` for authentication packets we send), `#sequence`, `ch N`;
    on a jump chain, a line above names the hop `user@host:port` whenever it changes. Without an SSH connection in this run: “This run did not get as far as an SSH connection (an earlier step failed, or there are no saved credentials), so there is no protocol trace.”; past the limit: “N more packet(s) went past the recording limit and were not recorded.”
  - The exported report ends with the same text (headed “Protocol trace (packet metadata only, no contents; lengths of authentication packets we send are not recorded)”). **The tap is attached only for diagnostics**; the profile page's "Test" and ordinary connections do not get it.
  - When the authentication step fails on algorithm negotiation and the peer offers algorithms this version implements but has not enabled, the issue card has “Allow these algorithms and diagnose again” below its description (since 2026-10-25; rules in “Allow the algorithms with one click after a negotiation failure” in §7).
  - To check a batch of connections at once, use the **session health check** (§11C): it reuses the same four steps, but shows no dialogs and attaches no protocol trace.

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
