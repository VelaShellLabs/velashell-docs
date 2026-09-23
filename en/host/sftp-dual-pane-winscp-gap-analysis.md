# SFTP Dual-Pane and WinSCP Gap Analysis

> Date written: 2026-07-22　　Baseline code: `tmds-ssh` branch
>
> Purpose: **decision checklist**. List the gaps between VelaShell's dual-pane SFTP and WinSCP item by item, so features can be selected for future implementation.
> Each item is marked with its current status and implementation cost. Anything not verified is marked "unverified"; no assumptions are made.

---

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

| # | Problem | Location | Symptom |
|---|---|---|---|
| A1 | **The external editor always reports "not configured" in the dual-pane view** | `SftpDocumentViewModel.cs:33-40` does not assign `GetDefaultEditorPath` | The editor configured in settings cannot be used. The terminal-side host is wired (`MainWindowViewModel.cs:846`), but the dual-pane view is missing it |
| A2 | **The local pane header cannot sort** | `LocalFilePaneView.axaml:128-138` uses plain `TextBlock` headers | `SortCommand` (`LocalFilePaneViewModel.cs:49`) and `ToggleSort:499` are fully available, but not bound |
| A3 | **The transfer popup cannot cancel, retry, or clear an individual item** | `FileTransferView.axaml` binds only `CancelAllCommand` and `HidePanelCommand` | Three commands (`FileTransferViewModel.cs:106-112`) have no corresponding controls. **Also, `RetryTransfer:419` only changes the state and does not resend**, so connecting the UI would still be a no-op |
| A4 | **The entire `TransferManager` queue has zero call sites** | `Core/Sftp/TransferManager.cs` | Queuing, concurrency, and cancellation are implemented and DI is registered, but production code never calls `QueueTransferAsync`. The browser uses `SemaphoreSlim` directly |
| A5 | **The remote pane has no button for the parent directory** | `GoUpCommand` (`FileBrowserViewModel.cs:648`) has no axaml binding | The only option is double-clicking the `..` row. The local pane has a button |
| A6 | **The remote pane cannot be restored after being closed in the dual-pane view** | `ToggleVisibilityCommand` → `IsVisible=false` | This button was designed for the terminal host, which has a reopen path. In the dual-pane view, clicking it leaves only empty space |
| A7 | **Transfer settings in the dual-pane view are a construction-time snapshot** | `SftpDocumentViewModel.cs:36-37` | Changes to settings do not affect already-open SFTP documents. `OnSettingsSaved` only traverses the terminal-side cache |
| A8 | **The dual-pane view does not read or write column visibility and hidden-file settings** | `SftpDocumentViewModel.cs:33-40` | The terminal side wires `ShowHiddenFiles` and `ColumnVisibilityToggled`; the dual-pane view does not |
| A9 | **`DownloadItemCommand` is not bound** | `FileBrowserViewModel.cs:670` | This also makes `PickSavePathForDownload`, "save as when downloading a single file", unreachable from the UI |
| A10 | **Rate limiting is not applied to resumed downloads** | The resume branch in `SftpService.cs` does not wrap the stream in `ThrottledStream` | Bandwidth limiting silently stops working during resumable transfers |
| A11 | **The transfer log records Copy as DOWNLOAD** | `TransferLogService.cs:40` uses a ternary that distinguishes only Upload and everything else | The log type is inaccurate |
| A12 | **Dragging a file from the OS into the local pane refreshes but does not copy** | `LocalFilePaneView.axaml.cs:234-243` | Nothing happens after the drop |

> **Duplicate definitions**: `AppSettings.AutoResume` (:623) and `ResumeEnabled` (:668) overlap semantically; only the latter is consumed.
> `TransferMaxRetries` (:671) and `AutoCleanTempFiles` (:674) have code comments admitting that they are "planned, with no runtime consumer".

---

## III. Category B: Missing Capabilities (Compared by WinSCP Feature Domain)

### 3.1 Browsing and Navigation

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Editable path input | ❌ None | Both sides have only breadcrumbs, so a path cannot be pasted directly for navigation. **High-frequency basic need** |
| Forward/back history | ❌ None | Neither VM has a history stack |
| Bookmarks/favorites | ❌ None | WinSCP users rely on this heavily |
| Directory history dropdown | ❌ None | |
| Breadcrumbs | ✅ Implemented on both sides | |
| Hidden-file toggle | ⚠️ Remote only | The local pane always shows hidden files and has no toggle |
| Local drive switching | ✅ Implemented | |

### 3.2 File Operations

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Create folder/file | ✅ Complete on remote | The local pane has **create folder only, not create file** |
| Rename | ✅ Both sides | |
| Delete, recursive + progress + cancellation | ✅ Both sides | Good quality, with weighted progress on the remote side. **Deleting a folder on an SSH session takes an `rm -rf` fast path** (2026-09-20, `Transfer.UseRecursiveDeleteCommand`, on by default): the SFTP walk costs one round trip per entry while `rm -rf` costs one in total; it falls back to the SFTP walk when the command is unavailable or exits non-zero. The cost is that this path has no per-entry progress (the bar goes indeterminate); the confirmation prompt is unchanged |
| Move/copy, remote → remote | ⚠️ Requires typing an absolute path | No directory picker and no drag-and-drop move |
| chmod | ⚠️ Single file only | Has a 9-cell rwx matrix with octal synchronization, but **no batch operation, recursive application, or setuid/sticky bits** |
| **chown, change owner/group** | ❌ None | `ISftpService` has no interface; Owner/Group in the properties dialog are read-only |
| Create/identify symbolic links | ✅ Implemented (2026-09-12) | Link rows have dedicated icons and a "→ target" tooltip, the permission string starts with `l`, and links to directories can be opened directly; the context menu has "New Symbolic Link". Deleting removes only the link itself, copying produces a link (`cp -P`), and folder downloads do not descend into nested directory links. On FTP, links can be created only on servers that support `SITE SYMLINK`; plugin protocols cannot create them |
| **Modify remote timestamps** | ❌ None | Timestamps are preserved only during download; uploads do not write back mtime |
| Local file attributes/permissions | ❌ None | The local pane has only 5 context-menu items |

### 3.3 Transfers

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Drag and drop, between panes and OS → remote | ✅ Implemented | |
| **Drop onto a specific target-folder row** | ❌ None | Drop currently applies only to the "current directory" and does not resolve the drop target |
| **Double-click transfer** | ❌ Different semantics | Double-clicking remote files downloads them and opens them with an OS program; double-clicking a local file **does nothing** |
| Conflict handling, overwrite/skip/rename/ask | ✅ All four | Very good quality |
| Resumable transfers | ✅ Implemented | Includes offset verification and a safe fallback window |
| Rate limiting/concurrency | ✅ Implemented | See A10 |
| **Transfer queue, visualizable, pausable, reorderable** | ❌ See A4 | WinSCP's queue is a core part of its experience |
| **Post-transfer verification, checksum** | ❌ None | |
| Completion notification | ✅ Implemented | |

### 3.4 Selection and Batch Operations

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Multiple selection, add via context menu, retain selection across navigation | ✅ Implemented | |
| **Wildcard selection (`*.log`)** | ❌ None | WinSCP's `Select Files` is a frequent operation |
| **Invert selection** | ❌ None | |
| **Recursive selection** | ❌ None | |
| Select all | ⚠️ Only through the default ListBox behavior | No explicit `Ctrl+A` binding |
| **Keyboard shortcuts (F5/F2/Del/Enter)** | ❌ Completely absent | Neither View has `KeyBindings` or `KeyDown` handling. **WinSCP veterans will find this very unfamiliar** |

### 3.5 Search and Filtering

| WinSCP capability | VelaShell | Description |
|---|---|---|
| **Current-directory filter box** | ❌ None | In a directory with 500 files, finding one requires scanning by eye |
| **Remote recursive search** | ❌ None | `ISftpService` has no Find/Search API |

### 3.6 Editing and Viewing

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Built-in editor + automatic upload on save | ✅ Implemented, 5MB limit | **Syntax highlighting added on 2026-07-22**, see the next section |
| External editor | ⚠️ See A1 | Logic is complete, but not wired into the dual-pane view |
| **File preview pane** | ❌ None | |

### 3.7 Views

| WinSCP capability | VelaShell | Description |
|---|---|---|
| Sort remote pane by column / column visibility / resize columns | ✅ Implemented | |
| Local-pane sorting | ⚠️ See A2 | |
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

### C2. Search Capability

`ISftpService` needs a new recursive search API. Pay attention to the cost of recursively traversing a remote directory. It needs streaming results and cancellation. The existing `ListDirectoryAsync` returns a `List` all at once and is not suitable for direct recursion.

### C3. Terminal Integration

- The file pane's context menu has no "Open terminal here" action
- The terminal's current directory cannot be synchronized with the SFTP pane

The SFTP document is currently a pure dual-pane view (`SftpDocumentView.axaml` has no terminal control).
This project **already has a complete terminal stack**, so the architectural foundation for this feature is actually ready. The main work is UI orchestration.

---

## V. Recommended Priority

Ordered purely by return on investment, for reference:

**First tier, low cost and highly visible**
1. Category A wiring debt (A1, A2, A3, A5, A6, A7, A8), the items users treat as bugs
2. Keyboard shortcuts (F5 refresh / F2 rename / Del delete / Enter enter / Ctrl+A select all)
3. Editable path input
4. Current-directory filter box

**Second tier, moderate cost and clear demand**
5. Wildcard selection / invert selection
6. Visual transfer queue (A4 + individual cancel and retry)
7. Persist column widths and pane ratio
8. Complete the local pane, sorting, hidden-file toggle, and create file
9. Drop onto a target-folder row

**Third tier, high cost and differentiation**
10. ~~**Directory comparison + synchronization** (C1), which determines "whether it can replace WinSCP"~~ — completed on 2026-09-12 (SHA-256-first comparison added on 09-13)
11. Remote recursive search (C2)
12. Terminal integration (C3)
13. chown / timestamp modification (symbolic links were completed on 2026-09-12)
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
- **Time precision**: both sides are truncated to the coarser precision before comparing. SFTP has seconds; FTP's Unix LIST only has minutes (and only a date for files older than six months), inferred from the raw value. Tolerance is 1 second (absorbs FAT/exFAT's 2-second granularity).
- **Case**: on Windows / macOS paths are paired case-insensitively; two remote names that differ only in case are marked as a conflict and left alone.
- **Type conflicts**: a file on one side and a directory on the other is a conflict, and nothing below it is compared (otherwise "Delete files" would remove a tree nobody meant to touch).
- **Links**: neither side descends into links to directories (the number skipped is reported; same rule as folder downloads). Remote links to files are treated as files. Locally only real symbolic links and junctions are skipped; OneDrive "Files On-Demand" placeholders take part as usual.

### 7.5 Not done / known limitations

- **Cost of SHA-256**: same-size files are read in full on both sides (the local disk and the server's disk), so the first comparison of a large tree is noticeably slower; uncheck it when not needed. The cache does not outlive the window: a file whose content changed while its size and modification time did not (a deliberate `touch -r`) reuses the old digest on a second comparison in the same window.
- Sync options are remembered only within the same document tab, not saved to settings, and reset after a restart.
- **FTP time zones**: LIST times are given in the server's local time and interpreted in the client's time zone. When the server and the client are in different zones, time comparisons are shifted as a whole — compare by file size only in that case. WinSCP offers a "time zone offset" in its session settings; VelaShell does not yet.
- FTP servers without `MFMT`, and plugin protocols, cannot align the remote time after an upload. One-way Local → Remote is unaffected (a newer remote file is not treated as something to upload), but **two-way sync would download the just-uploaded files back**; the window warns about this.
- "Keep up to date" is Local → Remote only (as in WinSCP).
