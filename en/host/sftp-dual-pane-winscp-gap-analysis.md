# SFTP Dual-Pane and WinSCP Gap Analysis

> Date written: 2026-07-22　　Baseline code: `tmds-ssh` branch
>
> Status re-checked: 2026-10-25 against the code on dev; finished items keep their original description and are marked ✅ for traceability
>
> Purpose: **decision checklist**. List the gaps between VelaShell's dual-pane SFTP and WinSCP item by item, so features can be selected for future implementation.
> Each item is marked with its current status and implementation cost. Anything not verified is marked "unverified"; no assumptions are made.

---

## Status after the 2026-10-25 merges

`feat/sftp-r2b` adds local/remote directory back and forward history (50 steps per pane instance), current-directory name filtering, F5/F2/Delete, filter Esc/Enter, local empty-file creation and system-default opening on double-click, persistent remote column widths and dual-pane ratios, and “Open terminal here” for SSH file panes. These items in the older gap descriptions below are no longer pending; double-click still does not mean transfer. See [keyboard shortcuts](keyboard-shortcuts.md) for focus conditions. Plugin protocol link fields, modification times and server digests are wired as well; creating symbolic links still has no contract.

The terminal entry is limited to SSH profiles. The directory command follows the login shell: an ASCII-only folder name is single-quoted by the login shell's rules (fish by fish's); a name with control characters is not `cd`-ed into and a notice is shown; a name whose bytes in the session encoding include non-ASCII bytes is sent as printable ASCII only, `cd "$(printf '\ooo…')"` (fish: `cd (printf '…')`; tcsh / csh do not understand `$(…)` and only report a syntax error). Windows servers (cmd.exe / PowerShell) still get the single-quoted form, so the `cd` can fail.

## I. Conclusion First

VelaShell is currently a **quite solid one-way file transfer tool**, but it is not yet a **dual-pane file manager**.

The transfer pipeline itself is clearly better than that of similar small tools: recursive upload/download planning, four conflict strategies, resumable transfers including offset verification, rate limiting, concurrency, a progress popup, transfer logs, cancellation semantics, and `LocalPathSafety` path traversal validation.
The remote pane's table, with 7 columns, sorting, hideable columns, resizable columns, and double-click auto-sizing, also exceeds expectations.

The gaps are concentrated in three layers, and **they must be handled differently and should not be scheduled together**:

| Category | Meaning | User perception | Recommendation |
|---|---|---|---|
| **A. Wiring debt** | The code is written but not connected to the dual-pane path | **Feels like a bug** | Fix first, very low cost |
| **B. Missing capability** | It genuinely has not been built, but the architecture supports it | Fewer features | Schedule based on demand |
| **C. Architectural gap** | The interface layer has no provision for it | Fundamental gap versus WinSCP | Requires dedicated design |

---

## II. Category A: Wiring Debt (Recommended First)

These are **not "not implemented"; they are "implemented but not connected"**. Users will report them as bugs, while the cost to fix them is low.

| # | Status | Problem | Location | Symptom |
|---|---|---|---|---|
| A1 | ✅ Fixed 2026-07-22: single- and dual-pane file tabs pass the default-editor resolver to the remote pane | **The external editor always reports "not configured" in the dual-pane view** | `SftpDocumentViewModel.cs:33-40` does not assign `GetDefaultEditorPath` | The editor configured in settings cannot be used. The terminal-side host is wired (`MainWindowViewModel.cs:846`), but the dual-pane view is missing it |
| A2 | ✅ 2026-10-25: the local pane's three headers sort on click, with ▲ / ▼ on the active column, same rules as the remote pane | **The local pane header cannot sort** | `LocalFilePaneView.axaml:128-138` uses plain `TextBlock` headers | `SortCommand` (`LocalFilePaneViewModel.cs:49`) and `ToggleSort:499` are fully available, but not bound |
| A3 | ⚠️ Retry and "Clear finished" wired on 2026-07-23 (a failed row's Retry really re-probes the resume offset and resends); **per-item cancel is still missing**: the command exists, but `CancelTransfer` only changes the state and notifies the unused `TransferManager`, so the transfer keeps running | **The transfer popup cannot cancel, retry, or clear an individual item** | `FileTransferView.axaml` binds only `CancelAllCommand` and `HidePanelCommand` | Three commands (`FileTransferViewModel.cs:106-112`) have no corresponding controls. **Also, `RetryTransfer:419` only changes the state and does not resend**, so connecting the UI would still be a no-op |
| A4 | ❌ Still zero call sites (pending decision: delete or migrate, see the host's `feature-plan.md`) | **The entire `TransferManager` queue has zero call sites** | `Core/Sftp/TransferManager.cs` | Queuing, concurrency, and cancellation are implemented and DI is registered, but production code never calls `QueueTransferAsync`. The browser uses `SemaphoreSlim` directly |
| A5 | ✅ 2026-10-25: a Go-up button next to Refresh on the remote toolbar; the two panes' buttons are titled "Go up (local pane)" / "Go up (remote pane)" | **The remote pane has no button for the parent directory** | `GoUpCommand` (`FileBrowserViewModel.cs:648`) has no axaml binding | The only option is double-clicking the `..` row. The local pane has a button |
| A6 | ✅ 2026-07-29: the remote pane's close button is hidden in file tabs (shown only in the terminal side pane) | **The remote pane cannot be restored after being closed in the dual-pane view** | `ToggleVisibilityCommand` → `IsVisible=false` | This button was designed for the terminal host, which has a reopen path. In the dual-pane view, clicking it leaves only empty space |
| A7 | ✅ 2026-10-25: after settings are saved, open single- and dual-pane file tabs pick up the new transfer options immediately | **Transfer settings in the dual-pane view are a construction-time snapshot** | `SftpDocumentViewModel.cs:36-37` | Changes to settings do not affect already-open SFTP documents. `OnSettingsSaved` only traverses the terminal-side cache |
| A8 | ✅ 2026-10-25: file-tab remote panes take hidden files and columns from the settings; toggling saves the setting and syncs the other open panes | **The dual-pane view does not read or write column visibility and hidden-file settings** | `SftpDocumentViewModel.cs:33-40` | The terminal side wires `ShowHiddenFiles` and `ColumnVisibilityToggled`; the dual-pane view does not |
| A9 | ❌ Still no entry point (pending decision on how it interacts with the conflict policy) | **`DownloadItemCommand` is not bound** | `FileBrowserViewModel.cs:670` | This also makes `PickSavePathForDownload`, "save as when downloading a single file", unreachable from the UI |
| A10 | ✅ 2026-10-25: resumed downloads honour the bandwidth limit too | **Rate limiting is not applied to resumed downloads** | The resume branch in `SftpService.cs` does not wrap the stream in `ThrottledStream` | Bandwidth limiting silently stops working during resumable transfers |
| A11 | ✅ 2026-09-28: the log records UPLOAD / DOWNLOAD / COPY / RELAY by direction | **The transfer log records Copy as DOWNLOAD** | `TransferLogService.cs:40` uses a ternary that distinguishes only Upload and everything else | The log type is inaccurate |
| A12 | ❌ Still only refreshes, and the cursor still shows the drop as allowed (pending decision) | **Dragging a file from the OS into the local pane refreshes but does not copy** | `LocalFilePaneView.axaml.cs:234-243` | Nothing happens after the drop |

> **Duplicate definitions**: `AppSettings.AutoResume` (:623) and `ResumeEnabled` (:668) overlap semantically; only the latter is consumed (`AutoResume` is now a legacy field kept only so old settings files load; nothing consumes it at runtime — see [settings-audit.md](settings-audit.md) M-02).
> `TransferMaxRetries` (:671) and `AutoCleanTempFiles` (:674) have code comments admitting that they are "planned, with no runtime consumer".
> (Later: `AutoCleanTempFiles` was wired on 2026-07-23 and `TransferMaxRetries` on 2026-10-24 — temporary failures such as a dropped connection or a timeout are retried with back-off, under File Transfer → Conflict handling → "Retry failed transfers".)

---

## III. Category B: Missing Capabilities (Compared by WinSCP Feature Domain)

### 3.1 Browsing and Navigation

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Editable path input | ✅ Implemented | Remote pane #226, local pane 2026-10-12: the pencil button / a click on the empty space after the breadcrumbs / `Ctrl+L` turn it into an input box, Enter navigates; see [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §6 |
| Forward/back history | ❌ None | Neither VM has a history stack |
| Bookmarks/favorites | ✅ Implemented (2026-10-14) | A bookmark entry on the dual-pane document toolbar stores a local-directory ↔ remote-directory pair per connection, with add, apply, edit, delete, and persistence; remote ↔ remote pairs and access history are not included |
| Directory history dropdown | ❌ None | |
| Breadcrumbs | ✅ Implemented on both sides | |
| Hidden-file toggle | ⚠️ Remote only | The local pane always shows hidden files and has no toggle |
| Local drive switching | ✅ Implemented | |

### 3.2 File Operations

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Create folder/file | ✅ Complete on remote | The local pane has **create folder only, not create file** |
| Rename | ✅ Both sides | |
| Delete, recursive + progress + cancellation | ✅ Both sides | Good quality, with weighted progress on the remote side. **Deleting a folder on an SSH session takes an `rm -rf` fast path** (2026-09-20, `Transfer.UseRecursiveDeleteCommand`, on by default): the SFTP walk costs one round trip per entry while `rm -rf` costs one in total; it falls back to the SFTP walk when the command is unavailable or exits non-zero. With a non-UTF-8 session charset, paths containing non-ASCII characters, and names with undecodable bytes in UTF-8 sessions, skip the command (the exec command line is UTF-8, so it would delete a different byte string), and after exit 0 the folder is stat'ed again and the SFTP walk takes over if it is still there (since 2026-10-31; the `sha256sum` behind checksum comparison in directory sync and plugin remote digests skips such paths too, reporting them as "not computed"). The cost is that this path has no per-entry progress (the bar goes indeterminate); the confirmation prompt is unchanged |
| Move/copy, remote → remote | ⚠️ Requires typing an absolute path | No directory picker and no drag-and-drop move |
| chmod | ⚠️ Single file only | Has a 9-cell rwx matrix with octal synchronization, but **no batch operation, recursive application, or setuid/sticky bits** |
| chown, change owner/group | ⚠️ Single file only, SFTP only (implemented 2026-09-30) | Owner and Group in the properties dialog are editable drop-downs: suggestions come from the remote passwd / group databases (the same lookup that turns ids into names in the list), and a numeric UID / GID can be typed directly; changing only one keeps the other as it is; combined with a permission change, chown runs before chmod. FTP has no standard command and the plugin SDK has no such contract, so both rows stay read-only there. Like chmod, **no batch operation or recursion** |
| Create/identify symbolic links | ✅ Implemented (2026-09-12) | Link rows have dedicated icons and a "→ target" tooltip, the permission string starts with `l`, and links to directories can be opened directly; the context menu has "New Symbolic Link". Deleting removes only the link itself, copying produces a link (`cp -P`), and folder downloads do not descend into nested directory links. On FTP, links can be created only on servers that support `SITE SYMLINK`; plugin protocols cannot create them |
| **Modify remote timestamps** | ⚠️ Only written back by transfers | SFTP uploads and downloads both write back the modification time per "Preserve timestamps" (uploads set it on the same handle before closing); `ISftpService.SetLastWriteTimeAsync` exists (used by directory sync, FTP via `MFMT`; plain FTP uploads do not write it back). **What is missing is a UI to change the time by hand** (in the properties dialog) |
| Local file attributes/permissions | ❌ None | Local pane context menu: Refresh, Open, Upload, Rename, New folder, Delete |

### 3.3 Transfers

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Drag and drop, between panes and OS → remote | ✅ Implemented | |
| **Drop onto a specific target-folder row** | ❌ None | Drop currently applies only to the "current directory" and does not resolve the drop target |
| **Double-click transfer** | ❌ Different semantics | Double-clicking remote files downloads them and opens them with an OS program; double-clicking a local file **does nothing** |
| Conflict handling, overwrite/skip/rename/ask | ✅ All four | Very good quality |
| Resumable transfers | ✅ Implemented | Includes offset verification and a safe fallback window |
| Rate limiting/concurrency | ✅ Implemented | |
| **Transfer queue, visualizable, pausable, reorderable** | ⚠️ Partial (2026-10-24) | WinSCP's queue is a core part of its experience. Upload / download batches left unfinished by a disconnect or an exit are parked as "Paused" and restored after a restart, and "Resume" continues from the partial files (stored in SonnetDB `transfer_queue`, see [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §9); manual pausing and reordering are still missing (see also A4) |
| **Post-transfer verification, checksum** | ❌ None | |
| Completion notification | ✅ Implemented | |

### 3.4 Selection and Batch Operations

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Multiple selection, add via context menu, retain selection across navigation | ✅ Implemented | |
| **Wildcard selection (`*.log`)** | ❌ None | WinSCP's `Select Files` is a frequent operation |
| **Invert selection** | ❌ None | |
| **Recursive selection** | ❌ None | |
| Select all | ⚠️ Only through the default ListBox behavior | The ListBox's own `Ctrl+A` (when the list has focus); no menu entry |
| **Keyboard shortcuts (F5/F2/Del/Enter)** | ⚠️ Partial | Enter opens a folder, type-to-select and `Ctrl+L` exist (2026-10-12); F2 / Delete / F5 are still missing. **WinSCP veterans will find this very unfamiliar** |

### 3.5 Search and Filtering

| WinSCP capability | VelaShell | Description |
|---|---|---|
| **Current-directory filter box** | ❌ None | In a directory with 500 files, finding one requires scanning by eye |
| **Remote recursive search** | ✅ Implemented (2026-10-24) | One `find` / `grep` run over SSH exec (name wildcards, literal content match), and clicking a result locates it in the panel; capped at 500 items / 2 minutes, stoppable. Only on SSH session panels: FTP and plugin protocols have no exec, so the entry points do not appear. See C2 |

### 3.6 Editing and Viewing

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Built-in editor + automatic upload on save | ✅ Implemented, 5MB limit | **Syntax highlighting added on 2026-07-22**, see the next section |
| External editor | ✅ Implemented | A1 fixed |
| **File preview pane** | ❌ None | |

### 3.7 Views

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Sort remote pane by column / column visibility / resize columns | ✅ Implemented | |
| Local-pane sorting | ✅ Implemented (2026-10-25) | See A2 |
| **Remember column widths across restarts** | ❌ None | Exists only in the VM instance |
| **Remember pane ratio** | ❌ None | GridSplitter can be dragged but the value is not persisted |
| **Local pane has only 3 columns** | ❌ | Missing permissions/owner/group/type |
| Switch between icon view and detailed view | ❌ None | |

---

## IV. Category C: Architectural Gaps

These three capabilities **have no provision at the interface layer** (`ISftpService.cs` is 82 lines in full and has no related signatures), so they require dedicated design.
(C1 was completed on 2026-09-12; see section VII.)

### C1. Synchronization and Directory Comparison —— ✅ Implemented (2026-09-12)

WinSCP's three major synchronization capabilities are now available for SFTP, FTP, and FTPS dual-pane documents; the rules are in [section VII](#vii-appendix-directory-comparison-and-synchronization-2026-09-12):

- **Directory comparison**: "Compare Directories" on the dual-pane document toolbar selects the differing items in each pane (current level only, the same scope as WinSCP's command of that name)
- **Synchronization, one-way / two-way / mirror / timestamps only**: scan → preview (each step can be unchecked) → confirm deletions → execute in batch → automatic re-check
- **Keep remote directory up to date**: watch local changes and upload them automatically

~~Original assessment: zero lines of code; needs a comparison result model, difference calculation, a preview UI, and an execution engine.~~
By default differences are computed **by SHA-256 first** (digests computed on the server), falling back to size + modification time when that is unsupported or fails (see 7.4).

### C2. Search Capability — ✅ Implemented (2026-10-24)

~~`ISftpService` needs a new recursive search API. Pay attention to the cost of recursively traversing a remote directory. It needs streaming results and cancellation. The existing `ListDirectoryAsync` returns a `List` all at once and is not suitable for direct recursion.~~

In the end it **does not walk the tree through `ISftpService`**: it runs one `find` / `grep` over the SSH session's exec channel, so the
traversal happens on the server in a single round trip and results stream back one by one (`Core/Sftp/RemoteSearch` + `RemoteSearchService`).
The rules:

- **Entry points**: on an SSH session's file panel (terminal sidebar, single-pane SFTP document, the SSH side of a dual-remote tab) — the
  magnifier on the toolbar, "Search remote files…" in the blank-area context menu, and "Search in this folder…" on a folder row (starting
  there). One modeless window per panel; it closes when the panel is torn down (tab closed, session dropped).
- **POSIX first**: the remote shell is probed per host and cached; Windows cmd / PowerShell and the csh family get "not supported", and
  accounts with exec disabled or SFTP only get "could not tell" — in both cases **not a single command is sent**. bash, zsh, POSIX sh and fish
  can search.
- **No injection path**: the script body is a fixed constant, and user input is passed only as single-quoted positional parameters
  (`sh -c '<script>' vela-search '<root>' '<pattern>' …`), quoted by the rules of the login shell.
- **Only options BusyBox also has** (including `-maxdepth`, `tr` and `grep -q`); `/proc`, `/sys` and `/dev` are skipped only when searching from `/`; links are not followed. Names use
  `find` wildcards (no wildcard means "name contains"), content is a literal match only (`grep -F`), with no regex option.
- **Limits**: 500 items and 2 minutes, both fixed; hitting either says honestly that the results are incomplete, and the user can stop at any
  time, closing the window stops it too. Reaching the item limit cancels the exec at once, so the remote `find` / `grep` is stopped with its
  process group, and it reports "limit reached" rather than a time-out (since 2026-10-31).
- Known limits: on a very old BusyBox whose `find` lacks `-exec … +`, a content search finds nothing and names containing a newline still
  split into two results (since 2026-10-25 results are decoded with the session encoding, the search arguments are sent in it too, and a name
  with a newline counts as one result; before, Chinese names on a GBK server showed replacement characters and newline names split in two).
- The "current directory filter box" is still missing (3.5).

### C3. Terminal Integration

- The file pane's context menu has no "Open terminal here" action
- The terminal's current directory cannot be synchronized with the SFTP pane

The SFTP document is currently a pure dual-pane view (`SftpDocumentView.axaml` has no terminal control).
This project **already has a complete terminal stack**, so the architectural foundation for this feature is actually ready. The main work is UI orchestration.

---

## V. Recommended Priority

Ordered purely by return on investment, for reference:

**First tier, low cost and highly visible**
1. ~~Category A wiring debt (A1, A2, A3, A5, A6, A7, A8), the items users treat as bugs~~ — done except A3's per-item cancel; the remaining A3 / A4 / A9 / A12 wait for decisions, see the host's `feature-plan.md`
2. Keyboard shortcuts (F5 refresh / F2 rename / Del delete / Enter enter / Ctrl+A select all)
3. ~~Editable path input~~ — done
4. Current-directory filter box

**Second tier, moderate cost and clear demand**
5. Wildcard selection / invert selection
6. Visual transfer queue (A4 + individual cancel and retry)
7. Persist column widths and pane ratio
8. Complete the local pane, ~~sorting~~, hidden-file toggle, and create file
9. Drop onto a target-folder row

**Third tier, high cost and differentiation**
10. ~~**Directory comparison + synchronization** (C1), which determines "whether it can replace WinSCP"~~ — completed on 2026-09-12 (SHA-256-first comparison added on 09-13)
11. ~~Remote recursive search (C2)~~ — completed on 2026-10-24 (`find` / `grep` over SSH exec; not for FTP / plugin protocols)
12. Terminal integration (C3)
13. Timestamp modification (symbolic links were completed on 2026-09-12, chown on 2026-09-30)
14. Post-transfer verification

---

## VI. Appendix: Editor Enhancements Completed on 2026-07-22

The built-in editor (`RemoteFileEditorView`) originally had **no syntax highlighting at all**. The following was added:

- **Automatic file-type detection** (`Services/Syntax/FileTypeDetector.cs`): extension → special filename (`Dockerfile`/`sshd_config`/`fstab`/`.bashrc`…) → **shebang**.
  The third level is especially important for remote editing: many executable scripts on servers have no extension, and only `#!/bin/bash` identifies what they are.
- **Added missing AvaloniaEdit syntax definitions** (`Syntax/*.xshd`): among the 20 built-in definitions, **Shell, YAML, INI/conf, Dockerfile, and Log were conspicuously absent**. These are precisely the five types most often edited in daily operations, so we wrote the definitions ourselves.
- **Theme following** (`SyntaxHighlightingService`): AvaloniaEdit's built-in definitions are tuned for light backgrounds, with keywords in `Blue` and punctuation in `Black`. On this application's Dracula surface, `#282A36`, punctuation becomes invisible. Named colors are now recolored globally for Dracula (dark) / Alucard (light), with contrast fallbacks for roles not covered by the theme.

~~**Known limitation**: changing the theme after opening the editor does not update its colors in real time. Reopen the editor to apply the new colors.~~ Resolved on 2026-09-12; see below.

### 2026-09-12 Update: Colors Follow the Named Theme, Readable Links, More Languages, Larger Window

- **Colors are derived from the current UI theme**: previously there were only Dracula / Alucard palettes, so under Nord, Gruvbox, GitHub Light, and other themes the editor background followed the theme while code stayed Dracula-colored. Syntax colors now come from the current theme's seed colors (string = Yellow, keyword = Magenta, number = Accent, function = Success, type = Info, comment = TextTertiary). VelaDark keeps its Dracula colors exactly; VelaLight matches Alucard except for the variable color, which is now the midpoint of orange and red so it no longer collides with strings. Any role below 3:1 against the editor background is blended toward the body text color until it is readable.
- **URL / email link color**: AvaloniaEdit's default link color is pure blue `#0000FF`, about 1.7:1 on the Dracula background and barely readable. Links now use the `VelaInfo` token (Dracula cyan `#8BE9FD`, Alucard deep teal-blue `#036A96`, and each other theme's own Info color). Every theme reaches ≥ 3:1 against the editor background, enforced per theme by a regression test.
- **Every built-in color name is classified**: names that previously had no role and kept their light-theme colors, such as CSS selectors, HTML tags, Markdown links, and Patch added/removed lines, are now all mapped. A regression test checks every color name in every built-in and bundled definition.
- **New bundled syntaxes**: nginx, TOML, Makefile, Go, Rust, Lua, Ruby, Perl, generic SQL (MySQL / PostgreSQL / SQLite, replacing the SQL Server-only TSQL definition), and HCL / Terraform. Additional extension mappings were added (`.tsx`/`.jsx`, `.scss`/`.less`, `.axaml`/`.resx`, `.jsonl`, systemd `.timer`/`.socket`, and more).
- **nginx detected by directory**: detection now uses the full remote path, so files under `/etc/nginx/conf.d/*.conf` and `sites-available/` are highlighted as nginx, while `.conf` files elsewhere stay INI.
- **Live theme switching**: switching themes while the editor is open recolors it immediately (via `IThemeService.EffectiveThemeChanged`).
- **Larger default window**: 928×648 → 1160×820; on small screens or high scaling it shrinks to the screen working area and re-centers when opened.
- **Known limitation**: AvaloniaEdit 12's built-in TeX definition is itself broken (it throws "Could not find main RuleSet" on load), so `.tex` is not mapped for now and opens as plain text.

---

## VII. Appendix: Directory Comparison and Synchronization (2026-09-12)

Applies to SFTP, FTP, and FTPS dual-pane documents (plugin-protocol file documents work too, with the limits in 7.5). The entry points are on the toolbar at the top of the dual-pane document; the file browser in the terminal sidebar has no local pane and does not offer them.

### 7.1 Compare Directories (non-recursive)

- Compares the level of the **directories the two panes are currently showing**, using the criteria last chosen in the sync window (modification time + file size by default).
- The local pane selects items that exist only locally or are newer locally; the remote pane selects items that exist only remotely or are newer remotely. Items with the same time but a different size, and conflicts, are selected in both panes.
- The right side of the toolbar shows a one-line verdict ("Local: N different · Remote: N different · N identical"), cleared as soon as either pane changes directory.
- When the remote pane hides dotfiles, local dotfiles are left out of the comparison too, so every `.git` and `.env` is not flagged as "local only".

### 7.2 The Synchronize window

| Option | Values | Notes |
| --- | --- | --- |
| Direction | Local → Remote / Remote → Local / Both | In Both, the mode is fixed to "Synchronize files" and nothing is ever deleted |
| Mode | Synchronize files / Mirror files / Synchronize timestamps | **Synchronize**: transfer new, newer, and different-size files; never overwrite a newer target. **Mirror**: overwrite whenever different, even if the target is newer. **Timestamps**: transfer nothing; for files present on both sides with the same size but different times, set the target's time to the source's |
| Compare by | SHA-256 checksum (checked by default), modification time, file size | None checked = only fill in what is missing; SHA-256 rules are in 7.4 |
| Delete files | One-way only | Delete items that exist only on the target side; a directory is deleted as a whole and the items below it are not listed separately; confirmed before running |
| Existing files only | — | Only update files present on both sides; create nothing |
| File mask | WinSCP syntax `include \| exclude` | Masks separated by `;`; a trailing `/` makes a directory mask; a mask containing `/` matches the relative path; `*` and `?` are case-insensitive; `*.*` equals `*`. Excluded directories are **not scanned**, so they are never deleted |

Flow: **Compare** (both sides scanned in parallel; the status bar shows how many directories have been scanned; cancellable) → **Preview** (one row per step: action, path, size and time on each side; each row can be unchecked; changing any option discards the preview) → **Synchronize** → **automatic re-check** (the verdict states "no differences remain" or "N operations still pending").

Execution rules:

- Order: create directories → set times → transfer files → delete. **Deletions are not performed after a cancellation.**
- Transfers use the same pipeline as ordinary uploads and downloads (transfer panel, global concurrency limit, transfer log), but **bypass the "When a file already exists" conflict policy and skip resume detection** — the preview is the confirmation, and a target smaller than its source is the normal case of "different content", not a partial file to resume.
- After a transfer the modification time is **always** written back, regardless of the "Preserve timestamps" setting: uploads set the remote time with SFTP setstat / FTP `MFMT`, downloads set the local time. Without this, the next comparison would see a just-uploaded file as "newer on the remote". The window reports servers that cannot do it.
- Local paths built from remote names are validated segment by segment: names that are invalid on Windows (such as `a:b` or `CON`) are reported and skipped instead of being written somewhere else.
- Parent directories are created level by level by the executor, so unchecking a "create folder" row while keeping the files inside it still works.

### 7.3 Keep remote directory up to date

- Always Local → Remote, using the window's file mask, criteria, and "Delete files" (confirmed first if checked).
- Performs one full synchronization at start; then watches the local directory tree and, after a 1-second debounce, re-compares **only the directory level where the change happened**. Newly appearing directories (an extracted archive, a `git clone`) are compared as a whole tree, and a watcher buffer overflow (lost events) triggers a full re-check.
- The list area turns into an activity log (newest first) recording each upload and deletion. Closing the window or the document stops watching.

### 7.4 Comparison rules

- **SHA-256 first** (added 2026-09-13): files present on both sides **with the same size** are compared by content digest first (a different size already proves the content differs, so nothing is read).
  - Equal digests = the same file, **not transferred even if the times differ** (no more mass re-uploads after `git checkout`, extracting an archive, or `touch`). Different digests = the content really changed; the modification time decides which side is newer, and if the times are equal too the pair is "content differs" (transferred one-way, left alone in two-way).
  - Remote digests are **computed on the server**, without transferring content: SFTP runs `sha256sum` (or `shasum -a 256`) over the SSH exec channel, up to 200 paths per call; FTP / FTPS use `HASH` (SHA-256) or `XSHA256`. The remote side goes first, so no local byte is read when the server cannot hash.
  - **Fallback**: if the server cannot hash at all or fails (SFTP-only accounts with exec disabled, neither tool installed, Windows hosts, FTP servers without those commands, plugin protocols), the whole run compares by size and modification time and the window notes why; if a single file cannot be read, only that file falls back and the note says "M of N". With only SHA-256 checked (no time or size), the fallback uses both.
  - Also applies in timestamps mode: files whose digests differ are never re-stamped, so the difference is not hidden.
  - Digests are cached by "path + size + modification time" until the window / document closes, so the automatic re-check after a sync only re-hashes the files just transferred (which doubles as post-transfer verification).
  - "Compare Directories" also compares by SHA-256 first, and appends a hint to its verdict when it had to fall back.
- **Time precision**: both sides are truncated to the coarser precision before comparing. SFTP has seconds; FTP's Unix LIST only has minutes (and only a date for files older than six months), inferred from the raw value. Tolerance is 1 second (absorbs FAT/exFAT's 2-second granularity). When an FTP connection has a server time zone set, LIST times are shifted by that offset first, and day-level comparisons truncate both sides in **the server's calendar** (LIST gives only the server's local date for files older than six months, and truncating in the local calendar would put a file changed in the evening on the neighboring day).
- **Case**: on Windows / macOS paths are paired case-insensitively; two remote names that differ only in case are marked as a conflict and left alone.
- **Type conflicts**: a file on one side and a directory on the other is a conflict, and nothing below it is compared (otherwise "Delete files" would remove a tree nobody meant to touch).
- **Links**: neither side descends into links to directories (the number skipped is reported; same rule as folder downloads). Remote links to files are treated as files. Locally only real symbolic links and junctions are skipped; OneDrive "Files On-Demand" placeholders take part as usual.

### 7.5 Not done / known limitations

- **Cost of SHA-256**: same-size files are read in full on both sides (the local disk and the server's disk), so the first comparison of a large tree is noticeably slower; uncheck it when not needed. The cache does not outlive the window: a file whose content changed while its size and modification time did not (a deliberate `touch -r`) reuses the old digest on a second comparison in the same window.
- Sync options are remembered only within the same document tab, not saved to settings, and reset after a restart.
- ~~**FTP time zones**: LIST times are given in the server's local time and interpreted in the client's time zone. When the server and the client are in different zones, time comparisons are shifted as a whole — compare by file size only in that case. WinSCP offers a "time zone offset" in its session settings; VelaShell does not yet.~~ — resolved on 2026-10-24: set "Server time zone" on the FTP connection's Advanced page (`FtpSettings.ServerUtcOffsetMinutes`, the UTC offset of the server's clock; the default "Same as this computer" is the old behavior). Only the server wall-clock times LIST gives are shifted; MLSD / MLST times are UTC already and used as-is; the file panel's time column still shows the server's own clock. The sync window (including keep remote up to date) and "Compare folders" in single-pane and dual-remote tabs all use it. What remains: a fixed offset does not follow the server's daylight saving time, so it has to be changed by hand for that half of the year (WinSCP is the same).
- FTP servers without `MFMT`, and plugin protocols, cannot align the remote time after an upload. One-way Local → Remote is unaffected (a newer remote file is not treated as something to upload), but **two-way sync would download the just-uploaded files back**; the window warns about this.
- "Keep up to date" is Local → Remote only (as in WinSCP).

## VIII. Appendix: Remote + remote dual pane (2026-09-28, #524)

WinSCP's Commander interface can point both panes at remote servers, and FileZilla can drag between two remote tabs; VelaShell's dual pane used to be fixed at "local + remote", so moving files between two servers meant downloading and uploading again. Now Ctrl-select two connections in the explorer and right-click "Open in dual-pane SFTP" to get a file tab with each pane connected to a different machine. Interaction details are in [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §3 (multi-select) and §6.2.

### 8.1 Why "relayed through this computer"

**2026-10-30 single-connection and stream-piping update:** Each FTP/FTPS session uses one control connection, with transfers and metadata operations sharing its queue; legacy `MaxConnections` values no longer increase the connection count. SFTP concurrency is unchanged. The safe-commit FTP→SFTP path uses `OpenReadAsync` → `SourcePolledStream` (`System.IO.Pipelines`, 4 MiB backpressure threshold) → a remote temporary file, with separate source-reading and target-writing loops. Short source reads reuse the remaining space in the current pipe segment so small amounts of valid data cannot reserve many full blocks. In reverse, SFTP→FTP passes the seekable source through a length-checking wrapper directly to FTP `UploadStreamAsync`, reading while uploading without a complete local file cache; it need not use the same `Pipe` class. The commit gates and compatibility fallbacks below still apply: unsupported atomic commit, symbolic-link destinations, identical sessions and existing-target resume can still require full staging. This is not a claim that every case is disk-free.

**2026-10-29 streaming-commit update (supersedes the historical 2026-10-28 descriptions below):** Fresh FTP/FTPS-source transfers between distinct sessions use `OpenReadAsync` → bounded memory of at most 4 MiB per file → a random `.vela-relay-*` file in the destination directory when `CheckUploadCommitAsync` supports atomic replacement. Full consumption and length validation, the final FTP reply from source `DisposeAsync`, and target upload must all succeed before `CommitUploadAsync` publishes the final file, without a complete local file cache. `GetUploadCreateModeAsync` supplies creation permissions from the existing destination; commit retains existing ownership and permission handling. `SerializedSftpService` explicitly forwards all three methods. The destination directory must permit creation and rename; ordinary FTP rename is not treated as a universal atomic-replacement guarantee.

Pre-commit failure/cancellation only attempts to delete this transfer's remote temporary file, never the final file. Automatic retries reopen the source with a new temporary name instead of resuming the final destination. Commit uses an independent 30-second cancellation budget; a sent request cannot be undone. An unconfirmed commit raises a dedicated exception, disables automatic retries and preserves the final destination and any possible temporary file for manual inspection. Unsupported targets, identical sessions and requests to resume existing targets retain the full-cache path below, including cache reuse after successful preparation. SFTP→SFTP, plugin sources and same-session copy remain unchanged. This is not disk-free support for every protocol and adds no source snapshot, full-file remote hash or power-loss durability guarantee. Random temporary names reduce collisions, but the current upload interface does not provide exclusive creation; an untrusted shared writable directory is not a supported security boundary.

**Review follow-up:** Symbolic-link destinations fall back to full caching to preserve the previous write-through behavior. Definite commit refusals preserve the actionable error and clean up the temporary file; only ambiguous disconnection/timeout outcomes disable automatic retries and retain possible remnants. Commit bypasses the document metadata gate so long directory deletion/hash operations do not consume its confirmation budget, while remaining tracked for close-time draining. Unknown-size sources receive final progress from consumed bytes after successful commit. Before returning a stream, FTP `OpenReadAsync` releases an invalid lease on a 425/450 concurrency rejection, reduces the pool to one connection and retries at most twice, preserving full-download adaptation. Deleting an abandoned upload also discards its durable-resume hint.

- **Server-direct boundaries**: ordinary SFTP has no universal cross-server copy operation. Running `scp` / `sftp` / `rsync` on one endpoint through SSH is a separate approach requiring connectivity between the servers, the necessary tools, destination credentials and host-key verification (not disabling verification). FTP FXP depends on both servers' support, security policies and network conditions and cannot be assumed available. This change implements neither direct-transfer approach and does not forward credentials or an SSH agent to remote servers.
- **Fully cache FTP/FTPS sources first (2026-10-28)**: when either remote pane acts as an FTP/FTPS source, use high-level `DownloadStream` → complete cache → `UploadStreamAsync`, rather than feeding an FTP `OpenReadAsync` data stream directly into the target. All files, including empty, small, large and unknown-length files, use randomly named, exclusively opened temporary files with `DeleteOnClose` under `UserPathResolver.Home/.velashell/temp`. The ≤4 MiB memory cache and large/unknown system-temporary-directory split no longer apply. The disk holding this directory must have enough space for the complete file, whose contents may be sensitive. Local cache disk failures are permanent errors, not automatically retried.
- **Prepare before touching the target**: the complete source download, cache `Flush`, applicable length check and final FTP completion reply must all succeed before the cache can be handed to the target. Before that, do not touch the target file, including opening it for upload, truncation, resumed writes or failure cleanup. Show a download-preparation stage without counting cached download bytes as target ACKs. Upload progress still tracks target acknowledgements, with final full progress gated on successful upload and cleanup. Source errors/cancellation must not become EOF, and an early-returning target must not claim complete consumption.
- **Cache lifetime**: a cache belongs to one `Run`. Automatic target retries within that Run reuse the successfully prepared cache; only automatic retries after source-preparation failure download and prepare again. Manual Retry is a new Run and cannot reuse the old cache. Conflict rename, final failure, cancellation and success all release the cache, deleting temporary files on close; no cache survives across runs. Cache cleanup is separate from the policy for retaining or cleaning partial remote targets.
- **Unchanged paths**: SFTP → SFTP keeps its existing `OpenReadAsync` → `UploadStreamAsync` streaming path and seek-based resume verification. Non-seekable plugin sources retain `BufferedRelayReadStream`: 256 KiB blocks, with a per-file 4 MiB total budget covering producer-, queue- and consumer-held blocks. Known positive lengths ≤4 MiB are fully validated and closed at the source first; larger or unknown lengths start uploading after the first successful block through a bounded queue with backpressure. Cancellation or failure stops and joins the producer and clears and returns the blocks. About 12 MiB at concurrency three is this read-ahead budget, not a limit on FTP temporary disk usage. Same-session “Copy to” remains a separate path.

- **Completion gate (2026-10-28)**: when the source stat has a positive length, the relay checks the actual read length and fails on early EOF or a mismatch. The general `UploadStreamAsync` contract, in which length is only a progress denominator, is unchanged. The fully cached FTP/FTPS path passes its source-completion gate before touching the target; the streaming SFTP/plugin paths still require both target upload and source-stream cleanup to succeed before forwarding final full progress. FTP downloads must consume and validate the final completion reply before returning the connection lease; failed, cancelled, abandoned, or untrustworthy connections are not reused. Failed targets still follow the existing resume and cleanup settings; this does not provide atomic replacement or a full-file hash guarantee.

- **Retry follow-up (2026-10-28)**: FTP single-file stat and read opening switch to Binary before querying SIZE while holding the lease. LIST/MLST display sizes and sizes from another transfer mode are not treated as simultaneous exact truths. Unsupported SIZE uses the existing unknown-length semantics; final-reply validation remains required. A source-read failure is not replaced by a secondary target Flush/CLOSE error; target durable bytes are retained separately for resume bookkeeping. With transfer logging enabled, each failed attempt writes a safe structured `ATTEMPT_FAILED` record before backoff (type chains and numbers, not exception messages or server reply text), including failures followed by a successful retry. A 0-byte display may mean a small file is still buffered in SFTP and the first WRITE is not acknowledged, rather than that nothing was read. Normal batch regressions must check first-attempt success and RETR/OPEN counts, not only final hashes or mock-target results presented as real SFTP interoperability.

- **Cross-platform CI follow-up (2026-10-29)**: when a download with a known binary SIZE aborts with `426` and its actual byte count differs, retain `RemoteTransferLengthException` and the expected/actual byte counts. A nested command exception thrown by FluentFTP while closing the data stream must not hide that diagnosis. Local destination-write errors and cancellation retain precedence; busy retries and unknown-length semantics are unchanged. The loopback FTP test server sets `NoDelay` on its control connections to avoid roughly 40 ms per file from Linux Nagle/delayed ACK interaction, retaining the 2000-file coverage and default 60-second test timeout. Since 2026-11-01 a `426` abort is reported as the retryable `VelaFtpTransferInterruptedException` when the control connection still answers NOOP (otherwise as a connection error), with the `RemoteTransferLengthException` carrying the expected / actual byte counts as its inner exception, so the diagnosis is kept; a short read confirmed by 226 still surfaces directly as `RemoteTransferLengthException` and is not retried.

### 8.2 Known limitations

- **Resume depends on source and target capabilities**: the complete FTP/FTPS cache is now seekable and can support target-tail verification and resume; this is no longer limited to SFTP sources. It does not mean all targets support resume: the target protocol/implementation and resume settings still apply. SFTP → SFTP keeps its existing stream/seek verification. Non-seekable plugin sources still cannot seek for verification, and plugin streamed uploads do not gain resume support from this change. When verification is unavailable, the same-name conflict policy applies instead of silently retransmitting the whole file.
- **Unknown-SIZE integrity limit**: without binary `SIZE`, download and final-reply validation still apply, but no trusted expected length exists to prove that the read was not short or that the source should be nonempty. If a server falsely reports completion, full caching cannot guarantee detection of a 0-byte or short file. This is not a full-file hash guarantee.
- **Evidence boundaries**: confirmed code defects and controlled protocol reproductions do not establish the sole root cause at the user's real deployment. No on-site reproduction evidence is available, and no real OpenSSH testing is claimed for this round; protocol-test-server or mock success cannot stand in for real-server interoperability results. Moving all caches to disk does not establish that the roughly 2% failure rate reported by the user has been eliminated.
- **No synchronize window**: "Synchronize…" and "Keep remote up to date" are designed around local ↔ remote and are not offered in these tabs; "Compare directories" is, by size and modification time (no SHA-256).
- **Plugin file protocols** (added 2026-09-28): plugin protocols such as S3 and WebDAV can take part. As a source they use the SDK's existing `OpenReadAsync`; as a target the plugin must also implement `IProtocolStreamUpload` (SDK 2.0.6, see [../sdk/sdk-reference.md](../sdk/sdk-reference.md) §5), otherwise that pane is a source only. Workspaces (Redis…) and terminal protocols (Telnet…) cannot take part.
- With only "Restore sessions" on, startup does not reopen these tabs (reconnecting one by one remembers single profiles); with "Restore split layout" on they come back with the layout (as long as both profiles are still saved).
