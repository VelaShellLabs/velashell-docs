# VelaShell Interaction Logic and UI Specification

> This document is based on the `VelaShell-zh.pen` design file and is intended for delivery to an AI Agent for feature implementation.
> Target technology stack: **Avalonia** (UI) + **custom VelaDock** (draggable/split tab docking, replacing Dock.Avalonia; floating windows are disabled by product decision) + custom VT engine (parser / buffer / emulator / render).
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
- Floating panel shadows: **every overlay and self-drawn window card shares one token**, `VelaShadowWindow` (two layers: a near one that firms the edge, a far one that spreads). Dark `0 1 3 #40000000, 0 4 12 #66000000`; light `0 1 3 #1A000000, 0 4 12 #33000000`. **Reach (offsetY + blur) is capped at 16**, matching the card margin — a self-drawn transparent window can only paint its shadow inside the window rectangle, and anything past the margin is cut into a smear by the window edge. Opacity must differ per theme; the same dense black under a light card looks dirty. Values and reasoning live in `DESIGN.md` §4.5 in the code repository.
- Use **lucide** as the unified icon library, at sizes 11–16.
- Common row heights: toolbar rows 28–36, list rows 28–38, status bar 24.

---

## 2. Main Window Structure Overview

The main window is 1440×900 and can be freely resized. **〔2026-07 final, user decision〕The window uses a custom borderless frame**: `WindowDecorations="None"` (Avalonia 12.x `BorderOnly`/`ExtendClientArea` managed decorations intercept title bar input, preventing button clicks and window dragging; this was abandoned in testing). It uses custom drawing + the native `BeginMoveDrag` movement loop + a WndProc hook for Windows 11 snap support. The first row is a 36px custom title bar (`TitleBarView`: left = product logo + name; right = global feature button group `GQQwj` + minimize/maximize/close buttons). **The text menu has been removed entirely** (its functions duplicate the command palette). From top to bottom, the client area is: **custom title bar → (sidebar ‖ right area) → status bar**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│Custom title bar 36px: ▣ VelaShell …gap… [Feature buttons GQQwj]  ─ □ ✕  │
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

- **The custom title bar spans the full-width first row** (`bg-sidebar` background + bottom divider): left = logo and product name; right = global feature button group `GQQwj` (see §4A; “broadcast” is implemented, “group sync” remains disabled) + minimize/maximize/close window controls (46×35, with system red on close hover).
- The row below divides directly into “sidebar ‖ right area”; the right area starts with the tab bar (see §4B).

- The sidebar/right boundary and the horizontal dividers within the right area are draggable (custom VelaDock / GridSplitter), allowing width/height adjustment and full-section collapsing.
- The terminal area and file area together form “one session view” and switch as a whole when the active tab changes.

---

## 3. Sidebar (Width 260, `bg-sidebar`)

> **Window title bar note**: this application uses a **custom borderless window** (`WindowDecorations="None"`, not the native title bar). `TitleBarView` draws a 36px title bar whose left side already contains the application icon and “VelaShell” name.
> Therefore, **the application icon and name are no longer repeated at the top of the sidebar**. The “Logo bar” in the original design (`tGM2H` for dark, `4RBrb` for light) has been removed to avoid duplicating the icon and name in the custom title bar. The first sidebar row now begins directly with the “toolbar”, and the session resource tree fills the released space above.

From top to bottom:

1. **Toolbar (36px, first sidebar row, `cnUAB`)**: group heading “Resource Explorer” (`126Cj`) on the left; two buttons on the right:
   - `plus` (`2mkdr`): New connection (opens the New Connection dialog in §13).
   - `ellipsis` (`oPsSE`): More (import/export configuration, batch operations, collapse all groups).
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
   - Interaction: click selects; **double-click connects and activates the tab**; right-click opens the session context menu in §12; groups can be reassigned by dragging.
   - By default, follows the active terminal tab: automatically expands the parent group, selects the corresponding connection, and scrolls to it without taking terminal keyboard focus. This can be disabled under “Settings → General → Behavior”.
3. **Session list / Quick Connect area (320px, top divider)**:
   - Header (36px): “Quick Connect” title + collapse button.
   - Quick Connect input (`bg-input`, 32px): enter `user@host[:port]` directly and press Enter to connect.
   - “Recent Connections” group heading + the 3 most recent rows (`user@host` + time). Click fills the input; double-click connects directly.
   - The quick command and recent connection areas can be collapsed and resized vertically. The collapsed state and last expanded height are stored locally and restored at the next startup.
4. **Bottom user bar (40px, top divider)**: left = current user avatar + `root`; right, in order: `bell` message center, `plug` plugin manager, `settings` gear (opens §14 Settings).
   - **Message center (`bell`)**: a 6px accent dot appears at the bell's top-right when unread messages exist (24px leaves no room for a number, and a dot says "there is something new" well enough). Clicking toggles a 360px non-modal overlay **anchored bottom-left, next to the bell**, which can be dragged anywhere in the window by its header (the position is remembered across sessions). It holds messages worth keeping and revisiting (available updates, announcements and security items from a subscribed feed) and deliberately **excludes runtime alerts** — fingerprint changes and session drops each have their own immediate feedback, and mixing them in would bury what needs reading. See [message-center-and-feed.md](message-center-and-feed.md).

---

## 4A. Menu Bar (Full-width 36px, `bg-sidebar` background + bottom divider) ★Refactored

> ⚠️ **Implementation change (2026-07, plan §6)**: **The left text menu (Session/Edit/Action/Search/Tools/Help, `DaZfB` in §4A.1 below) has been removed entirely** because it duplicated the command palette (`Ctrl+P`/`Ctrl+K`) by product decision. The “Show menu bar” setting has also been removed (`ShowMenuBar` is retained for compatibility). This row now retains only the **right global feature button group** (§4A.2). §4A.1 remains as historical design reference.

> A separate row below the native title bar and above the sidebar/right area. ~~**Left = text menu**~~ (removed), **right = global feature button group (formerly `GQQwj`, moved here from the terminal toolbar)**. Design nodes: menu bar `TSiDh`, right-side feature container `Menu Bar Actions` (containing `GQQwj`).

### 4A.1 Left Text Menu (`DaZfB`)

Order: **Session / Edit / Actions / Search / Tools / Help** (Inter 12, `text-secondary`, `padding[0,10]` per item, hover `bg-hover` + corner radius 3). Click opens a drop-down menu. `Alt` provides keyboard access, and `Alt+first letter` opens an item quickly. Suggested menu contents:

| Menu | Typical items |
|---|---|
| **Session** | New SSH Connection, New SFTP, Import/Export Configuration, Recent Sessions, Close Current Session, Exit |
| **Edit** | Copy, Paste, Select All, Find, Clear Screen, Preferences (Settings) |
| **Actions** | Connect/Disconnect, Reconnect, Split Pane, Duplicate Session, Group Sync Toggle, Broadcast Input, Record Session |
| **Search** | Find in Terminal, Find in Files, Go to Line, Command Palette (Ctrl+P) |
| **Tools** | Tunnel Manager, SFTP File Manager, Operations Orchestration Center, Connection Diagnostics, Host Trust Center, Snippets, Key Management |
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

> Multi-terminal synchronized input no longer occupies a title bar icon. It has been refactored into the “Sync Input” channel in the tab context menu (see §6.1).

---

## 4B. Tab Bar (Single 36px Row, `bg-page` background) ★Refactored

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
- Interaction:
  - Click = activate; middle-click/click `x` = close, with a second confirmation when there are unsaved changes or active transfers.
  - Double-click empty tab space = rename (inline editing).
  - Drag = reorder; drag to an edge = trigger Dock splitting/floating window.
  - Right-click = tab context menu (close, close others, close to the right, duplicate session, rename, move to new window, join sync group).

### 4B.3 Tab Overflow Logic (VS Code style) ★User specified

- When total tab width ≤ container width: lay out normally, hide/disable `◀ ▶`, and keep `▾` available.
- On overflow:
  1. Clip the container and automatically use `ScrollIntoView` to keep the active tab visible.
  2. Allow `◀ / ▶` to scroll; disable the corresponding button at each boundary.
  3. Allow horizontal wheel scrolling over the container.
  4. Always list all tabs in the `▾` drop-down as a fallback for quick navigation during overflow.
- Drop-down items: status dot + session name + a check mark on the right for the current item. Support keyboard up/down + Enter selection. Tabs can also be closed here, with `x` appearing on hover.

### 4B.4 Resource Monitor on Tab Hover ★User specified (see §11)

- Move the mouse over **any tab name** and **hover stationary for >400ms** to show the “System Resource Monitor” at the **current mouse position**.
- Move the mouse out of the tab, or into another tab, to make the panel **disappear automatically**. A 150ms fade-out debounce is recommended. Moving into the panel itself keeps it visible.
- The panel shows resource data for the **session associated with that tab**, not the active session, so it can quickly inspect other sessions.

---

## 5. Main Terminal Area (`bg-terminal`)

### 5.1 In-Terminal Toolbar (28px, below the tab bar, `BdPtF`)

> `GQQwj` has moved to the menu bar (§4A.2). This row retains only **read-only information for the current session** on the left, plus a small number of optional actions on the right:
- Left (`termInfo`): `root@web-prod-01:~` (accent) + `uptime: 42d 7h 23m` (muted) + `|` + `latency: 12ms` (green).
- Right: optionally retain secondary buttons strongly tied to the current terminal, such as “Clear Screen / Full Screen / More”. Global functions have moved to the menu bar.

### 5.2 Terminal Canvas

- Render using the custom VT engine: draw monospaced text line by line, with support for xterm-256color, true color, cursor, selection, and hyperlink detection.
- Support mouse selection and copy, right-click paste, Ctrl+wheel font zoom, and scrolling back through the buffer.
- **Alt+left-drag = rectangular block selection** (#128, aligned with Windows Terminal behavior): whether Alt is held at mouse-down determines whether the operation is block selection or a normal linear selection. Changing Alt during the drag does not switch modes. Copy takes the same column range from each line, always inserting line breaks between lines. When the application enables mouse tracking (htop/vim/tmux), mouse events are given to the application; use Shift+Alt+drag to force block selection.
- **Shift+left-click = extend the selection** (#266, aligned with Windows Terminal / xterm): when a selection already exists, Shift+click keeps the anchor in place and moves only the far end to the clicked cell (before or after the anchor — the selection flips direction accordingly); keep the button held to keep dragging and fine-tune it. To grab a long log spanning more than one screen, select the start, scroll back, then Shift+click the end. The extension reuses the linear/block mode fixed at the original mouse-down. With no selection yet, Shift+click still starts a new one, preserving the existing "hold Shift to bypass application mouse reporting and select text" semantics.
- **Ctrl+Shift+left-drag = append a discontiguous region**: commits the in-progress region and starts another one, repeatable. Copy concatenates the regions in **document order, top to bottom**, with a line break between them — so "select line 1, Ctrl+Shift-select line 3, copy once and get both lines" holds regardless of the order they were picked. Each region remembers its own mode, so Ctrl+Shift+Alt+drag appends a rectangular region that coexists with linear ones, and Ctrl+Shift+double-click appends another word. A plain drag without Ctrl+Shift starts over (dropping every appended region); so does a search-hit highlight. Ctrl+**Shift**+click on a URL does not open the browser (opening links is Ctrl+click without Shift). Terminals have no precedent for this (Windows Terminal / iTerm2 / xterm all have single-region selection only), so the binding is ours.
- See §7 “Disconnected State” for the idle/disconnected overlay.

### 5.3 In-Terminal Search Bar

- Triggered by the menu bar `search` button (§4A.2) or Ctrl+F. A search field slides in from the top of the terminal: input + previous/next + match count + close. Matches are highlighted and Enter jumps to the next match.

---

## 6. File Browser / SFTP (Lower Right Area, Default 220px, Collapsible/Resizable)

- **Header (36px)**: left = clickable current-path breadcrumb for level-by-level navigation + refresh; right = icons for view switching, upload, new folder, hidden-file toggle, and so on.
- **Column header (26px, `bg-surface`)**: `Name(280) | Size(100) | Permissions(120) | Modified`. Columns can be sorted by clicking.
- **File list (`fill_container`, scrollable)**: 28px per row. Icon (folder/file type) + name + size + permissions (`drwxr-xr-x`) + time. The first row may be `..` to return to the parent directory.
- Interaction:
  - Double-click folder = enter; double-click file = download and open with the default application, or preview.
  - Drag a local file here = **upload** (triggers the file transfer component in §9); drag a list item locally = **download**.
  - **Name conflicts**: when an upload/download encounters an existing file with the same name, use the setting in §14 File Transfer “When a file already exists”: ask (confirmation dialog: overwrite or skip) / overwrite / skip / rename (`file (1).txt`).
  - Right-click = file context menu (download, upload, rename, delete, chmod permissions, copy path, new).
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

---

## 7. Status Bar (Full-width 24px, `bg-sidebar`)

- **Left**: `wifi` icon (connection color) + `SSH • web-prod-01:22` + `｜` + `Latency: 12ms` (accent) + `｜` + `↑ 2h 34m` (online duration).
- **Right**: `xterm-256color` ｜ `120×36` (terminal size) ｜ `cpu 23%` ｜ `memory 1.2G` ｜ `net 4.2 MB/s` ｜ `UTF-8`.
- CPU, memory, and network are lightweight real-time metrics for the current session. They share a source with the resource panel in §11; here they are a compact persistent version.
- Each field is clickable. For example, clicking the size triggers one resize synchronization, and clicking the encoding opens the encoding menu.

### Disconnected State (`ZufZw`)

When a session disconnects, dim the terminal canvas and center a disconnection notice (red status + “Connection disconnected” + “Reconnect” button + reason/time). The corresponding tab dot turns red, the status bar connection icon turns red, and the host dot in the sidebar turns red. Support an automatic reconnect toggle with exponential backoff.

### Connection Failures for Document-Style Connections

That overlay assumes **the tab already exists** — a terminal tab is created before the handshake, so the
failure can simply be drawn inside it. SFTP / FTP / plugin file systems (S3…) / plugin workspaces (Redis…)
have no such tab: they **only get one once they are connected**, so a failure leaves nothing on screen.
A first-connection failure in those four therefore raises an error dialog (title “Connection Failed”, body =
profile name + the reason) instead of only writing the status bar — with the status bar alone, what the user
sees is “I clicked connect and nothing happened”.

- The dialog shares one serialization gate with the credential prompt: restoring several sessions at startup
  fires them concurrently, and two modal dialogs over the same owner deadlock each other.
- **A deliberate cancel is not a failure**: cancelling the credential prompt or declining a certificate only
  updates the status bar, with no dialog — that would merely restate the decision the user just made.
  Cancelling also clears `LastConnectionError`, which is how the plugin-opened-session path tells
  “could not connect” apart from “the user said no”.
- When all three credential attempts fail and the loop exits, report once as well; otherwise typing a password
  three times ends in silence.
- Messages from the plugin protocol exception family (`ProtocolConnectionException` and friends) are, per the
  SDK contract, already user-facing and already carry the endpoint, so the host presents them verbatim rather
  than wrapping them in another “Failed to connect to X:” that prints the same address twice.

---

## 8. Command Palette ★User specified (`Ctrl+P` / `Ctrl+K`)

**Trigger**: global `Ctrl+P` opens the palette. The `zap` button does the same. It appears above center, is 560px wide, uses `bg-surface` + a large shadow, and has an overlay. Click the overlay or press `Esc` to close.

**Structure, from top to bottom**:
1. **Search field (48px)**: `search` icon + input (JetBrains Mono 14) + blinking caret + `Ctrl+K` mode hint on the right. Filter with fuzzy matching as the user types.
2. **“Sessions” group**: 10px muted uppercase group heading + session result rows, each with status dot + session name + environment label (`Production` on `accent-dim`) + `Enter Connect` hint on the right. The first item is highlighted with `bg-active` by default.
3. **Divider**.
4. **“Commands” group**: command result rows with icon + command name (Inter 12) + keyboard shortcut on the right. Examples:
   - `New SSH Connection` … `Ctrl+N`
   - `Open SFTP File Manager` … `Ctrl+Shift+F`
   - `Open Settings` … `Ctrl+,`
   - (Extensions) `Open Tunnel Manager`, `Split Pane`, `Switch Theme`, `Record Session`, `Connection Diagnostics`, `Operations Orchestration`…
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

**Trigger**: click the menu bar `route` (Tunnel) button (§4A.2), select Tools → Tunnel Manager, or choose Open Tunnel Manager in the command palette. Show it as a floating panel near the button (320px wide, `bg-surface` + small shadow).

**Structure**:
- **Header (32px, bottom divider)**: left = `route` (accent) + “Tunnels” + active tunnel count badge; right = `plus` (expand new form) + `x` (close panel).
- **Tunnel list, one row per tunnel**:
  - First row: status dot (green = active) + tunnel name, such as `MySQL Forwarding` + type label on the right (`Local` with `accent-dim` / `Remote` with an `info` background / `Dynamic`, extensible) + an outlined `AUTO` badge when automatic reconnect is on.
  - Detail row (10px muted monospaced): `L 3306 → db-prod-01:3306` / `R 6379 → localhost:6379`.
  - Status row: uptime while active, otherwise the status text.
  - **Statistics row** (10px muted monospaced; the description row the `fuXS7` draft reserved): `3 conns · 1.4 MB`, calling out the live count when connections are transferring (`3 conns (2 live) · 1.4 MB`), and “No connections yet” before anything has connected. The byte figure combines both directions.
  - On hover, show actions: enable/disable toggle, edit, delete.
- **New Tunnel form (`plus` expands, `tunNewForm`)**:
  - Title: `circle-plus` + “New Tunnel”.
  - One row with three fields: **Type** drop-down (Local L / Remote R / Dynamic D, 80px) + **Local Port** input + **Remote Address** input (`host:port`).
  - Check boxes: “Forward to the server itself” (locks the target to 127.0.0.1) and “Reconnect automatically after a dropout” (**off by default**).
  - Button row, right aligned: `Cancel` (outlined) / `Create` (solid accent, `plus` + “Create”).
- **Interaction logic**:
  - Attempt to establish forwarding immediately after creation. Success = green dot + increment badge; failure = red dot + error message.
  - Local and dynamic forwards run a **local port-occupancy precheck** before binding, reporting “local port 27017 is already in use by another program” instead of a low-level socket error.
  - A tunnel does **not** follow the lifecycle of a terminal tab: creating or starting one while disconnected opens a background connection dedicated to tunnels, and that connection is dropped once the server's last tunnel is removed.
  - When the host session drops, its tunnels are marked stopped; those with “reconnect automatically after a dropout” are redialed and rebuilt on their own (failures back off from 10s to 5min), and the rest wait for a one-click start. **A tunnel the user stopped by hand is never brought back up automatically.**
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
- Join Sync Group
- Open SFTP
- Move to Group ▸ (submenu)
- Divider
- Delete (red)

Tabs and file rows likewise have their own context menus (see the corresponding sections).

---

## 13. New Connection and Password Verification Flow

### 13.1 New Connection Dialog (`oAHna`, 500px)

- **Header**: “New Connection” title + `x`.
- **Tabs (`connTabs`)**: built-in `SSH` / `SFTP` / `FTP` plus plugin-contributed tabs (`S3`, `Telnet`, …)
  declared in each plugin's `contributes.protocols` (drawn without loading the plugin assembly);
  `Serial` remains a disabled placeholder until its own plugin lands.
- **Form body (`connBody`)**: name, Host, Port, username, authentication method (password / key / jump host), group, color/icon marker, and so on.
  - The form area **scrolls** (the scrollbar stays visible instead of auto-hiding); header, tabs, and footer are fixed-height rows outside the scroll.
    The window height is clamped to `min(768 design cap, screen working area − 48)` (`ApplyScreenBounds`; 768 matches the 948×768 settings window).
    The field count is protocol-dependent (a plugin protocol such as S3 declares a dozen), so without the clamp the dialog grows past the screen and the footer buttons become unreachable.
  - Plugin fields marked `IsAdvanced` are folded into “Advanced options” by default; the footer button shows a `+N` badge for the folded count,
    and editing an existing profile auto-expands as soon as any advanced field differs from its declared default.
  - **Built-in items inside “Advanced options”**: tags, jump host, and **“Run command after authentication” + a delay (0–60 s)**.
    - The latter is **per profile** (`SessionProfile.PostAuthCommand` / `PostAuthCommandDelaySeconds`), and is a
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
  - **FTP-only item inside “Advanced options”: “Default remote path”** (`FtpSettings.InitialRemotePath`, shown only
    on the `FTP` tab). After connecting, the remote pane opens that directory instead of the login working directory —
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
- **Footer**: left = “Advanced options” (with the folded-count badge); right = `Test` / `Save` / `Connect` (accent).

### 13.2 Password Verification Dialog (Two Steps)

- **Step 1 (`oNZIM`, 420px)**: header title + session information bar (`bg-input`: host icon + `user@host`) + password input with visibility toggle + “Remember password” checkbox + `Cancel`/`Connect` footer.
- **Step 2 (`twD13`)**: second-factor verification (2FA / OTP / key passphrase / host fingerprint verification). Information bar + input + `Previous`/`Verify`.
- **First host fingerprint confirmation**: on the first connection to an unknown host, show a “Host Trust” confirmation (fingerprint + Accept and Save/Reject), linked to §15 Host Trust Center.

**Connection state machine**: `Idle → Connecting (yellow) → Authenticating → Connected (green)` / any failure `→ Disconnected (red)` with reason and retry.

---

## 14. Settings Panel (implemented as an 840×740 dialog, 200px left navigation + right content)

The left side contains navigation sections and the right side shows the corresponding content. **The current implementation has 11 pages** (Cloud Sync and Support and Donations were added beyond the design; see the itemized remediation ledger in `docs/settings-audit.md`):

| Page | Frame | Implementation status (2026-07-12) |
|---|---|---|
| General | `2BIRD` | Startup/tray/language/connection defaults/session logs/import/export/behavior and automatic reconnect; the unimplemented “Updates” and “Master Password” groups are hidden |
| Appearance | `ZAbb9` | Theme (twelve named themes + follow system, live preview), accent color, UI font/size, opacity, terminal colors and color schemes (defaults to the active theme's paired scheme, see below) |
| Terminal | `08FpM` | Font/line height/TERM/encoding/cursor/three-state bell/scrolling/copy and paste/IME/commands run after connection |
| Key Management | `UBP59` | Enumerates `~/.ssh` (type + SHA256 fingerprint), generates RSA, imports/deletes/copies public keys, default authentication key; unimplemented “Load into Agent” is hidden |
| Keyboard Shortcuts | `YQvri` | **Defined as a read-only “Keyboard Shortcut Reference”** (customization is not supported by product decision), with entries checked one by one against actual bindings |
| File Transfer | `HGwa7` | Paths/editor/concurrency/conflict policy/hidden files/bandwidth limits/transfer logs (conditionally visible); unimplemented resume features are hidden |
| Security Audit | `glqQE` | Session recording toggle + playback center entry, host trust policy, trusted host management (address redaction), alert channels (in-app/sound/Webhook) |
| Snippets | `HBNhv` | Common command snippet library with collapsible groups (shared by command palette/completion suggestions through SonnetDB `quick_commands/commands` v2); supports adding, editing, and changing group assignment |
| Cloud Sync | — (new) | GitHub Gist multi-device sync: token/Gist binding, end-to-end encryption passphrase, sync scope, version history, and restore (see plan.md §13.C) |
| About | `Nwoks` | Version/runtime environment/dependencies/contributors (clickable GitHub avatars)/dual licensing and authenticity statement; Check for Updates is an honest placeholder |
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
| Sakura (`sakura`) | light | pink, in-house | Sakura |

- The persisted value is the id above; `dark` / `light` keep their historical values, so existing
  configurations need no migration.
- The **first entry** in the terminal color-scheme dropdown is “Follow theme (<paired scheme>)”:
  picking it follows the theme (no color overrides at all, so the terminal changes with the theme).
  Every entry below it pins that scheme; the paired one still carries a “(default)” suffix to show
  where it comes from. Editing the foreground/background/cursor/selection color while following
  counts as choosing your own colors, and leaves the following state.
- Following is recorded **explicitly** in `Appearance.TerminalColorsFollowTheme`, never re-derived by
  comparing colors against the factory values — that comparison could not tell “picked Dracula” from
  “following the theme”, which made picking Dracula under a non-Dracula theme a no-op.
- Palettes are defined by the host's `UiThemeCatalog` (seed colors) and `ThemeTokenApplier`
  (derivation rules); the token list and rules live in `DESIGN.md` §2 of the VelaShell repository.
- Plugins still see only `dark` / `light` / `system` — named themes are not part of the plugin contract.

---

## 15. Advanced Feature Panels (Large Standalone Panels)

Open these as tabs or standalone floating windows. All can also be entered from the command palette:

- **Operations Orchestration Center (`bR5c4`, 920×760)**: header + content area + results panel. Execute scripts/playbooks on multiple hosts in batches: choose a target group → configure steps → execute → summarize live results (success/failure/output).
- **Host Trust Center (`gPWeC`)**: policy row (trust policy: strict/TOFU/lenient) + host fingerprint table (host, fingerprint, algorithm, first-seen time, status, delete/reset actions). ✅ **Implemented in simplified form (2026-07-12)**: Settings → Security Audit includes “Host Trust Policy” (three first-confirmation options / block changes) + a “Trusted Hosts” list (view/delete, screenshot-safe address redaction); the standalone large panel is not implemented separately for now.
- **Session Recording and Playback (`NceE6`)**: recording list + player (timeline, speed control, search), asciinema-style playback, and export. ✅ **Implemented (2026-07-12)**: `RecordingPlayerView` is a standalone window, entered through Settings → Security Audit → Playback Center. The left column contains the recording list (name/time/duration/size, delete); the right column contains a read-only terminal + timeline seek + 1x/2x/4x speed + skip idle segments. The top bar provides “Export Recording” (asciicast v2 `.cast`) and an “Auto Recording” toggle (= `Security.RecordProductionSessions`). Storage is described in Architecture Design §4.11 (SonnetDB time series). The “Search” feature in the design has not been implemented.
- **Connection Diagnostics Center (`RGXg1`, 920×640)**: run ping / DNS / port probes / traceroute / SSH handshake analysis against a target, and output step-by-step diagnostic conclusions and recommendations.

---

## 16. Global Keyboard Shortcuts (**Customization not supported**, by product decision; the “Keyboard Shortcut Reference” page in Settings is read-only)

> Current state after checking against actual bindings on 2026-07-12. Items marked “Not implemented” were concepts in the initial design and have no current binding.
>
> **This section is a design-level excerpt, not the complete list.** Every bound keyboard shortcut and mouse gesture
> (terminal selection gestures, the completion popup and per-dialog keys included) lives in
> [Keyboard Shortcuts](keyboard-shortcuts.md) — that table shares its source with `ShortcutCatalog` and is test-guarded,
> so treat it as authoritative when adding shortcuts.

| Shortcut | Function | Status |
|---|---|---|
| `Ctrl+P` / `Ctrl+K` | Command palette | ✅ |
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
| `Ctrl+1..9` | Jump to the Nth tab | Not implemented |
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
3. **P0 Terminal engine integration**: custom VT rendering + input + selection/copy/search.
4. **P1 Command palette (§8)** and **New Connection/Authentication flow (§13)**.
5. **P1 File browser + transfer component (§6/§9)**: drag-and-drop upload/download + floating progress + automatic disappearance.
6. **P1 Tunnel panel (§10)** and **Resource Monitor hover panel (§11)**.
7. **P2 Full Settings pages (§14)** and **Advanced panels (§15)**.
8. Follow the floating panel and boundary rules in §17 throughout.

---

### Appendix: Design File Frame ID Reference (for Pixel Comparison)

Main interface dark `CsTjc` / light `mkrrg`; sidebar `aMaSq` (dark) / `Cw1Yt` (light), sidebar top Logo bar `tGM2H`/`4RBrb` **deleted, replaced by the system native title bar**; **menu bar `TSiDh`** (left text menu container `DaZfB`, right feature container `Menu Bar Actions`, containing the global feature button group `GQQwj`, moved from the terminal toolbar); tab bar `nunbT` (including overflow control group `Tab Overflow Controls`=`pZGS4`, `◀◁ tabScrollLeft` / `▶ tabScrollRight` / `▾ tabListDrop`); terminal toolbar `BdPtF` (now session information only); terminal canvas `QzoMC`; file area `dyuii`; status bar `gzmsb`; command palette `FN5dM`; file transfer component `9Ralg`; tunnel panel `fuXS7`; resource monitor panel `EP3Gd`; context menu `e6klM`; disconnected state `ZufZw`; new connection `oAHna`; password verification `oNZIM`/`twD13`; session settings `ZNjAC`; Settings pages are listed in §14; advanced panels `bR5c4`/`gPWeC`/`NceE6`/`RGXg1`.
