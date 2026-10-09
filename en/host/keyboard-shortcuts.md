# Keyboard Shortcuts

Every keyboard shortcut and mouse gesture VelaShell actually binds, grouped by where it applies.

The tables show the **factory defaults**. The "Global" and "Tabs & Panels" groups (except pane focus) and search, clear
screen and zoom in the "Terminal" group can be changed or unbound in Settings → Shortcuts; see
[Customizing shortcuts](#customizing-shortcuts).

## Maintenance rules (read this first)

**This document is not a hand-copied list — it is a projection of the code.**

- The single source of truth is `src/VelaShell/ViewModels/ShortcutCatalog.cs`. Both Settings → Shortcuts and this document read from it;
  for the customizable ones, the factory table is `ShortcutBindings` in `src/VelaShell/Services/ShortcutKeymap.cs`.
- **Every new or changed shortcut must be registered in `ShortcutCatalog` and added to the table below.** This is enforced, not left to memory:
  - `ShortcutCatalogTests.EveryBinding_AppearsInCatalogExactlyOnce` requires every entry of the factory table to appear in the catalog exactly once;
    `MainWindowAxaml_HasNoHardcodedKeyBindings` stops a `KeyBinding` from being hardcoded in `MainWindow.axaml` (users could not change it);
  - `ShortcutCatalogTests.DefaultBindings_DoNotTakeNewTerminalKeys` stops a new factory binding from taking a key away from the terminal (`Ctrl+letter` is a control character);
  - `ShortcutCatalogTests.Doc_ListsEveryCatalogEntry` compares the catalog against this document row by row, and prints ready-to-paste Markdown for anything missing;
  - `ShortcutCatalogTests.Catalog_HasNoUnresolvedResourceKeys` makes sure every string resolves to a translation instead of showing a raw `Sc_Xxx` key in the UI.
- New resource keys use the `Sc_` prefix and must be added to **all five `resx` files** (`Strings` / `zh-Hans` / `zh-Hant` / `ja` / `ko`), otherwise `AllCultures_HaveIdenticalKeySets` fails. Actions that also exist in the command palette reuse its `Cmd_` key so both places read identically.
- Only **real bindings** belong here. A key mentioned in a menu or tooltip but never bound must not be listed (this happened with `Ctrl+N` — hint only, no binding; see settings audit C-10).

## Where bindings live

Find the right place by scope, then come back and update the table:

| Scope | Implementation |
| --- | --- |
| Global (window level, customizable) | `ShortcutBindings` in `src/VelaShell/Services/ShortcutKeymap.cs` (factory table); the main window registers `Window.KeyBindings` from the current keymap, nothing is hardcoded in `MainWindow.axaml` |
| Terminal context + macOS Command aliases | `src/VelaShell/Services/KeyboardShortcutService.cs` |
| Clipboard / paging / key encoding inside the terminal control | `src/VelaShell.Terminal/Input/TerminalKeyRouter.cs`, `src/VelaShell.Terminal/Emulation/InputEncoder.cs` |
| Terminal mouse gestures (selection, zoom, links) | `src/VelaShell.Terminal/Rendering/VelaTerminalControl.cs` |
| Search bar (matched against the keymap), completion popup, disconnected-state keys | `src/VelaShell/Views/TerminalTabView.axaml.cs` |
| Command palette / file manager / process manager / editor / dialogs | each view's own `OnKeyDown` |
| Esc cancelling a dock drag | `src/VelaShell/Docking/Controls/DockDragController.cs` |
| Explorer multi-select and dragging | `src/VelaShell/Views/SessionTreeView.axaml.cs` (rules in `src/VelaShell.Presentation/ViewModels/SessionTreeViewModel.cs`) |
| AI assistant panel | `plugins/VelaShell.Plugin.Ai/Ui/ChatPanelView*.cs` |

## Platform differences

Global shortcuts default to `Ctrl` on Windows, Linux and macOS alike, and can be changed in Settings; a new binding may
also use Win / Cmd (`Meta`).

On macOS the terminal also has a set of **fixed** Command aliases (`KeyboardShortcutService`): `Cmd+C` / `Cmd+V` to copy and
paste, `Cmd+T` / `Cmd+W` / `Cmd+,` for new tab, close tab and Settings. They do not follow your changes, and `Cmd+C` /
`Cmd+V` cannot be taken by a custom binding. `Ctrl+C` keeps its "send interrupt" meaning on all three platforms and is
never stolen by copy.

## Reading the table

- An "Applies when" of `—` means the shortcut is unconditional.
- `Left Click` / `Right Click` / `Middle Click` / `Double Click` / `Drag` / `Wheel` / `Gutter` / `Command mark` / `Title Bar` / `Tab` / `Tab strip` / `Splitter` / `Connection row` / `Group row` in the key column are mouse gestures, not keyboard keys. `Command mark` is the narrow gutter column that shows one marker per command; it only appears once the remote shell emits OSC 133 marks.
- An action listed on several rows has several equivalent bindings, or variants under different conditions (scrollback paging, for instance, differs between the main and the alternate screen).

## Customizing shortcuts

Window-level bindings **consume keys before the terminal control sees them** (Avalonia matches `KeyBindings` before it
dispatches the key event): a bound combination never reaches the remote program. And `Ctrl+letter` is exactly a terminal
control character, so any set of factory bindings collides with some remote program —
[joesdu/VelaShell#551](https://github.com/joesdu/VelaShell/issues/551) was nano's `^K` being swallowed by the command
palette. That is why these bindings can be changed, or unbound.

**What can be changed**: everything in the "Global" and "Tabs & Panels" groups except "Move focus between panes", plus
search, clear screen and zoom in / out / reset in the "Terminal" group (full list in the table at the end of this
section). Everything else is fixed.

**How**: Settings → Shortcuts; every changeable row has three buttons on the right —

- **Pencil**: click it, then press the new key combination; `Esc` cancels, and so does clicking elsewhere. Modifiers on
  their own do not count; the box shows the modifiers you are holding until you press a real key.
- **×**: unbind. The key no longer triggers anything and goes straight to the terminal.
- **↺** (only when changed): reset to the factory default.

"Reset all" next to the search box drops every change at once. Changes are staged and take effect when you **save
settings**; changed keycaps are shown in the accent color, unbound rows say "Unbound".

**Rules**:

- It needs `Ctrl` or `Alt` (or Win / Cmd); without a modifier only `F1`–`F24` are allowed — otherwise you could no longer
  type that character.
- Fixed shortcuts cannot be taken: `Ctrl+Shift+C` / `Ctrl+Shift+V` (copy, paste), `Ctrl+C` (interrupt),
  `Ctrl+Shift+Up` / `Ctrl+Shift+Down` (jump between prompts), `Ctrl+Backspace` (delete word), `Ctrl+R` (reconnect),
  `Alt+Enter` (command completion), `Alt+arrows` (pane focus), `Ctrl+L` (SFTP path bar); on macOS also `Cmd+C` / `Cmd+V`.
- If another action already has the combination you are asked first: "Replace" leaves that one unbound, "Cancel"
  changes nothing. Resetting a binding whose factory default is currently taken asks the same way.
- Recording the factory default again counts as no change and leaves no entry.

**Terminal conflict hints**: when the current binding takes a key away from the terminal, a line in the warning color
says so under the row. For the factory `Ctrl+P`: "In the terminal this key sends ^P; to send ^P to the remote side,
press Ctrl+Shift+P" — the terminal sends the same control character for `Ctrl+Shift+letter`, so that combination is the
alternative while nobody has it. The alternative for `Ctrl+W` is taken by "Close all tabs", so that row only says
remote programs never receive it while it is bound. The factory bindings that take a terminal key are `Ctrl+N` /
`Ctrl+T` / `Ctrl+W` / `Ctrl+P` / `Ctrl+B` / `Ctrl+F` / `Ctrl+-`.

**Where it is stored**: `shortcuts.overrides` in the settings, holding only the changed entries — binding id → gesture,
an empty string meaning unbound. When a factory default changes later, users who never touched it follow along.
Unrecognized gestures and gestures that break the rules above (only possible by hand-editing the settings file) fall back
to the factory default. Settings export / import carries it, and cloud sync includes it when app settings are synced.

```json
"shortcuts": {
  "overrides": {
    "app.palette": "Ctrl+Shift+P",
    "session.close": ""
  }
}
```

Gestures use Avalonia's key names: modifiers are `Ctrl` / `Shift` / `Alt` / `Meta`, digits are `D0`…`D9`, `,` is
`OemComma`, `=` is `OemPlus`, `-` is `OemMinus`. The easiest way is to record one in the settings page and look at the
exported settings.

| Action | Binding id | Factory default |
| --- | --- | --- |
| New SSH Connection | `session.new` | `Ctrl+N` |
| New Tab (same as New Connection) | `session.new.tab` | `Ctrl+T` |
| Clone Current Session | `session.clone` | `Ctrl+Shift+N` |
| Open Settings | `app.settings` | `Ctrl+,` |
| Command Palette | `app.palette` | `Ctrl+P` |
| Close Tab | `session.close` | `Ctrl+W` |
| Close all tabs | `session.close.all` | `Ctrl+Shift+W` |
| Next Tab | `tab.next` | `Ctrl+Tab` |
| Previous Tab | `tab.prev` | `Ctrl+Shift+Tab` |
| Go to tab 1 … 8 | `tab.goto.1` … `tab.goto.8` | `Ctrl+Alt+1` … `Ctrl+Alt+8` |
| Go to last tab | `tab.goto.last` | `Ctrl+Alt+9` |
| Split Horizontally | `split.horizontal` | `Ctrl+Shift+D` |
| Split Vertically | `split.vertical` | `Ctrl+Shift+S` |
| Maximize / restore pane | `pane.maximize` | `Ctrl+Shift+X` |
| Toggle Explorer Sidebar | `view.sidebar` | `Ctrl+B` |
| Toggle File Browser | `tools.files` | `Ctrl+Shift+F` |
| Tunnel Manager | `tools.tunnel` | `Ctrl+Shift+T` |
| Toggle line number &amp; time gutter | `terminal.linegutter` | `Ctrl+Shift+L` |
| Search Terminal Content (only while a terminal tab has focus) | `search.terminal` | `Ctrl+F` |
| Clear Screen | `edit.clear` | `Ctrl+Shift+K` |
| Zoom in | `view.zoom.in` | `Ctrl+=` |
| Zoom out | `view.zoom.out` | `Ctrl+-` |
| Reset zoom | `view.zoom.reset` | `Ctrl+0` |

## Full table

### Global

| Action | Keys | Applies when |
| --- | --- | --- |
| New SSH Connection | `Ctrl+N` | — |
| New Tab (same as New Connection) | `Ctrl+T` | — |
| Clone Current Session | `Ctrl+Shift+N` | — |
| Open Settings | `Ctrl+,` | — |
| Command Palette | `Ctrl+P` | — |

> `Ctrl+K` is not bound ([joesdu/VelaShell#551](https://github.com/joesdu/VelaShell/issues/551)). Window-level bindings consume the key before the terminal control sees it, and `^K` is kill-to-end-of-line in bash / zsh and cut-line in nano, so the old command-palette alias was dropped; only `Ctrl+P` remains.

### Tabs &amp; Panels

| Action | Keys | Applies when |
| --- | --- | --- |
| Close Tab | `Ctrl+W` | — |
| Close all tabs | `Ctrl+Shift+W` | — |
| Next Tab | `Ctrl+Tab` | — |
| Previous Tab | `Ctrl+Shift+Tab` | — |
| Go to tab 1 | `Ctrl+Alt+1` | — |
| Go to tab 2 | `Ctrl+Alt+2` | — |
| Go to tab 3 | `Ctrl+Alt+3` | — |
| Go to tab 4 | `Ctrl+Alt+4` | — |
| Go to tab 5 | `Ctrl+Alt+5` | — |
| Go to tab 6 | `Ctrl+Alt+6` | — |
| Go to tab 7 | `Ctrl+Alt+7` | — |
| Go to tab 8 | `Ctrl+Alt+8` | — |
| Go to last tab | `Ctrl+Alt+9` | — |
| Split Horizontally | `Ctrl+Shift+D` | — |
| Split Vertically | `Ctrl+Shift+S` | — |
| Maximize / restore pane | `Ctrl+Shift+X` | Only when the window is split |
| Move focus between panes | `Alt+←+→+↑+↓` | Only when the window is split; otherwise the key goes to the remote shell |
| Toggle Explorer Sidebar | `Ctrl+B` | — |
| Toggle File Browser | `Ctrl+Shift+F` | — |
| Tunnel Manager | `Ctrl+Shift+T` | — |
| Toggle line number &amp; time gutter | `Ctrl+Shift+L` | — |

### Terminal

| Action | Keys | Applies when |
| --- | --- | --- |
| Copy | `Ctrl+Shift+C` | — |
| Paste | `Ctrl+Shift+V` | — |
| Paste (classic X11 binding) | `Shift+Insert` | — |
| Send Interrupt ^C | `Ctrl+C` | Copies instead when the selection-copy option is on and text is selected |
| Search Terminal Content | `Ctrl+F` | — |
| Jump to the next match | `Enter` | Only while the search bar is open |
| Jump to the previous match | `Shift+Enter` | Only while the search bar is open |
| Close the search bar | `Esc` | Only while the search bar is open |
| Scroll back one page | `PageUp` | Main screen with scrollback only |
| Scroll forward one page | `PageDown` | Main screen with scrollback only |
| Scroll back one page | `Shift+PageUp` | Works on the alternate screen too |
| Scroll forward one page | `Shift+PageDown` | Works on the alternate screen too |
| Jump to the previous prompt | `Ctrl+Shift+Up` | requires shell integration (OSC 133) |
| Jump to the next prompt | `Ctrl+Shift+Down` | requires shell integration (OSC 133) |
| Delete Previous Word | `Ctrl+Backspace` | — |
| Move the cursor to the line start | `Shift+Home` | — |
| Move the cursor to the line end | `Shift+End` | — |
| Reconnect After Disconnect | `Enter` | Only while the session is disconnected |
| Reconnect After Disconnect (alternate) | `Ctrl+R` | Only while the session is disconnected |
| Close a disconnected tab | `Esc` | Only while the session is disconnected |
| Clear Screen | `Ctrl+Shift+K` | — |
| Zoom in | `Ctrl+=` | — |
| Zoom out | `Ctrl+-` | Takes ^_; to send it, use Ctrl+Shift+- |
| Reset zoom | `Ctrl+0` | — |

### Command Completion

| Action | Keys | Applies when |
| --- | --- | --- |
| Show Command Completion | `Alt+Enter` | — |
| Select the next suggestion | `Down` | Only while the suggestion popup is open |
| Select the previous suggestion | `Up` | Only while the suggestion popup is open |
| Insert the selected suggestion | `Enter` | Only while the suggestion popup is open |
| Dismiss the suggestion popup | `Esc` | Only while the suggestion popup is open |
| Dismiss the suggestion popup | `Ctrl+C` | Only while the suggestion popup is open |
| Dismiss the suggestion popup | `Left Click` | Only while the suggestion popup is open |
| Fall through to the shell native completion | `Tab` | — |
| Accept the inline (ghost) suggestion | `Right` | — |
| Accept the inline (ghost) suggestion | `End` | — |

### Terminal Mouse Gestures

| Action | Keys | Applies when |
| --- | --- | --- |
| Open the link under the pointer | `Ctrl+Left Click` | — |
| Select the word | `Double Click` | Requires the matching option in Settings |
| Add another word to the selection | `Ctrl+Shift+Double Click` | — |
| Extend the selection from its anchor | `Shift+Left Click` | — |
| Rectangular (block) selection | `Alt+Drag` | — |
| Add a separate selection range | `Ctrl+Shift+Drag` | — |
| Add a separate block selection | `Ctrl+Shift+Alt+Drag` | — |
| Select text despite mouse reporting | `Shift+Drag` | Needed when the app enables mouse reporting |
| Paste with the right button | `Right Click` | Can be turned off in Settings |
| Zoom the terminal font | `Ctrl+Wheel` | — |
| Scroll the buffer five times faster | `Alt+Wheel` | — |
| Close Tab | `Tab+Middle Click` | Not on pinned tabs |
| Scroll the tab strip | `Tab strip+Wheel` | — |
| Even out panes | `Splitter+Double Click` | Only when the window is split |
| Open the gutter settings menu | `Gutter+Right Click` | — |
| Collapse or expand an output block | `Gutter+Left Click` | — |
| Select this command's output | `Command mark+Left Click` | requires shell integration (OSC 133) |

### Explorer

| Action | Keys | Applies when |
| --- | --- | --- |
| Add or remove a connection from the selection | `Ctrl+Left Click` | — |
| Select a range of connections | `Shift+Left Click` | — |
| Move a connection to another group | `Connection row+Drag` | — |
| Reorder groups | `Group row+Drag` | — |

### Command Palette

| Action | Keys | Applies when |
| --- | --- | --- |
| Select the next command | `Down` | — |
| Select the previous command | `Up` | — |
| Run the selected command | `Enter` | — |
| Close the command palette | `Esc` | — |

### SFTP File Manager

| Action | Keys | Applies when |
| --- | --- | --- |
| Edit the current path | `Ctrl+L` | — |
| Go to the typed path | `Enter` | — |
| Cancel path editing | `Esc` | — |
| Open the selected file or folder | `Double Click` | — |

### File Operations

| Action | Keys | Applies when |
| --- | --- | --- |
| Save in Remote Editor | `Ctrl+S` | — |
| Close the editor | `Esc` | Remote file editor |

### Task Manager

| Action | Keys | Applies when |
| --- | --- | --- |
| Refresh the process list | `F5` | — |
| End the selected process | `Delete` | Only while the list has focus |
| Close the window | `Esc` | — |

### Dialogs and Windows

| Action | Keys | Applies when |
| --- | --- | --- |
| Cancel and close the current dialog | `Esc` | Applies to every dialog and secondary window |
| Trigger the default button | `Enter` | — |
| Maximize or restore the window | `Title Bar+Double Click` | — |
| Cancel the dock drag in progress | `Esc` | Only while a dock drag is in progress |
| Paste into a password field | `Ctrl+V` | — |
| Copy is blocked in password fields | `Ctrl+C` | Ctrl+X is blocked as well |

### AI Assistant Plugin

| Action | Keys | Applies when |
| --- | --- | --- |
| Send the message | `Enter` | — |
| Insert a line break | `Shift+Enter` | — |
| Recall the previous input | `Up` | Only when the caret is on the first line |
| Recall the next input | `Down` | Only when the caret is on the last line |
| Select the next file candidate | `Down` | Only while the @ file picker is open |
| Select the previous file candidate | `Up` | Only while the @ file picker is open |
| Insert the file reference | `Enter` | Only while the @ file picker is open |
| Insert the file reference | `Tab` | Only while the @ file picker is open |
| Close the file picker | `Esc` | Only while the @ file picker is open |
| Delete a whole file reference | `Backspace` | — |
| Commit the session rename | `Enter` | — |
| Cancel the session rename | `Esc` | — |
| Close the AI secondary window | `Esc` | — |
