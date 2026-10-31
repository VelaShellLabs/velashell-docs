# VelaShell Settings UI Functional Audit and Restructuring Checklist

> Audit scope: settings model, settings pages, persistence services, and primary runtime consumers  
> Audit objectives: identify functional duplication, unclear semantics, runtime conflicts, ineffective settings, and missing settings, and provide an actionable grouping plan  
> Status note: This document is a remediation ledger. The first remediation batch (Phases 1 to 3 and conditional visibility) was completed on 2026-07-11, and the "Status" columns in all tables have been synchronized. Source line numbers are snapshots from the audit and may have shifted after remediation.

## 1. Executive Summary

The current Settings Center contains 9 top-level entries and approximately 80 visible settings or management functions. The main issue is not the number of settings itself, but the following:

1. `BellMode` and `VisualBell` both control the terminal bell, creating a real runtime conflict. (Fixed)
2. The automatic reconnect count, default values, and "Show hidden files" settings have multiple sources of state. (Fixed)
3. The UI descriptions and actual behavior of export, language switching, and close confirmation are not fully consistent. (Fixed except for localization)
4. More than 12 unimplemented settings occupied the interface long-term as disabled controls. (All hidden or removed)
5. "General" handles startup, connections, notifications, security, logging, updates, and several other areas, and has become a miscellaneous collection. (Partially mitigated: the update group is hidden and reconnect settings have been grouped; overall IA restructuring remains pending)
6. Key Management and Snippets are functional management tools, not application preferences. (Pending)
7. Terminal colors are under "Appearance", while terminal font, encoding, cursor, and other settings are under "Terminal", requiring users to search across pages for settings for the same object. (Pending)

### 1.1 Priority Definitions

| Priority | Meaning | Handling requirement |
|---|---|---|
| P0 | A confirmed security risk or runtime conflict | Handle first to prevent further incorrect configurations |
| P1 | Clearly misleading behavior, inconsistent state, or a critical experience defect | Recommended for the next remediation release |
| P2 | Information architecture, discoverability, or advanced experience issue | Can be handled together during settings UI restructuring |
| P3 | Optional enhancement | Include in product planning evaluation |

## 2. Confirmed Conflicts and Inconsistencies

| ID | Priority | Affected function | Problem type | Current behavior | Recommended handling | Source evidence | Status |
|---|---|---|---|---|---|---|---|
| C-01 | P0 | Terminal Bell, visual flash | Actual runtime conflict | `VisualBell=true` overrides "system alert sound" and "silent", because the actual condition is `VisualBell \|\| BellMode == "visual"` | Remove the standalone `VisualBell`; retain only the three-state Bell setting: "system alert sound / visual flash / no alert"; migrate old configuration | `AppSettings.cs:222,226`;`VelaTerminalControl.cs:217-220,472`;`MainWindowViewModel.cs:1565-1566` | ✅ Done (2026-07-11): `BellMode` is now the sole authority; `VisualBell` is retained only as a migration slot. When the two settings services load, `AppSettings.Normalize()` migrates them to `BellMode=visual` and clears the old value; the terminal control property and independent settings page toggle have been removed |
| C-02 | P1 | Maximum retries | Multiple runtime authorities | `General.MaxRetries` in settings controls part of the automatic reconnect flow, but `TerminalTabViewModel` still hardcodes the maximum number of attempts as 3 | Rename to "Maximum automatic reconnect attempts" and make all automatic reconnect paths read the same value | `AppSettings.cs:65`;`MainWindowViewModel.cs:1325`;`TerminalTabViewModel.cs:297` | ✅ Done: `TerminalTabViewModel.MaxReconnectAttempts` is now a writable property supplied by the host according to the setting during disconnect handling; the UI was renamed "Maximum automatic reconnect attempts" and moved into the subordinate "Behavior > Automatic reconnect" position (range 1 to 20) |
| C-03 | P1 | Confirm before closing, minimize to tray | Unspecified priority | When minimize to tray is enabled, closing the window first minimizes it and does not enter the close confirmation flow | Rename to "Confirm before exiting the application" and explain in the description that it is not triggered when minimizing to tray | `AppSettings.cs:55,78`;`MainWindow.axaml.cs:297-310` | ✅ Done: renamed "Confirm before exiting the application"; the description clarifies tray precedence; the tray item description also says that the application actually exits only through the tray menu's "Exit" command |
| C-04 | P1 | Show hidden files | Settings and session state are inconsistent | The file browser toolbar modifies only the current ViewModel and does not write back to persistent settings | Recommend writing toolbar toggles back to `Transfer.ShowHiddenFiles`; otherwise remove it from the Settings Center and make it explicitly a session state | `AppSettings.cs:276`;`MainWindowViewModel.cs:508`;`FileBrowserViewModel.cs:80,248-260` | ✅ Done: the toolbar toggle writes back and saves `Transfer.ShowHiddenFiles` through the `ShowHiddenFilesToggled` callback; after settings are saved, all open file panels are synchronized through a broadcast; the settings page description documents the linkage between both locations; since 2026-10-25 this also covers the remote panes of single- and dual-pane file tabs (they previously neither read nor wrote it) |
| C-05 | P1 | Scrollback line count | Inconsistent default value | The settings model defaults to 50000, while `SettingsViewModel` defaults to 10000 | Keep only one source for the default value; preferably provide it through `AppSettings` or a default configuration factory | `AppSettings.cs:19`;`SettingsViewModel.cs:114` | ✅ Done: the VM initial value now comes from `new AppSettings()` and no longer declares its own business default |
| C-06 | P1 | Default language | Inconsistent default value | The settings model uses `zh-CN`, while the ViewModel uses `en` | Unify the source of the default value | `AppSettings.cs:8`;`SettingsViewModel.cs:90` | ✅ Done: as with C-05, the VM initial value now comes from the settings model |
| C-07 | P1 | Cursor style | Inconsistent default value | The settings model uses `bar`, while the terminal control internally defaults to `block` | Apply the default from the settings model in the terminal control instead of declaring a business default independently | `AppSettings.cs:218`;`VelaTerminalControl.cs:145` | ✅ Done: the control default now matches the model (`bar`), with a comment explaining that the runtime value is supplied by `ApplyLiveTerminalSettings` |
| C-08 | P1 | Settings import/export | Description and scope are inconsistent | The text promises to back up "all connections and settings", but `BuildExportJson()` serializes only `AppSettings` | In the short term, revise the description; in the long term, explicitly support selective export of connections, groups, known hosts, snippets, and other data | `SettingsViewModel.cs:474-482`;`GeneralSettingsPage.axaml:106-107` | ✅ Done (short term): renamed "Settings import/export"; description changed to "Back up or migrate application settings only; connections, groups, keys, and snippets are not included". ✅ Selective export done (2026-10-24, seventeenth batch): export / import by category with checkboxes, sensitive categories unticked by default, and the proxy password and webhook URL never written. Connections, groups and the like are still outside the settings export; they have connection-file export ([session-import.md](session-import.md) §7) |
| C-09 | P1 | Interface language | Incomplete localization | Much of the settings page text is hardcoded in Chinese, so the Settings Center may remain partly Chinese after switching to English | Move settings titles, labels, descriptions, and options to `.resx` | Approximately 148 Chinese text instances in `Views/Settings/*.axaml` | ⏳ Pending: the localization infrastructure is already available (`LocalizeExtension` + `Strings.resx/zh-CN.resx`), but migrating and translating approximately 150 strings is a large focused effort. It should be handled as a separate batch to avoid mixing it with this structural remediation |
| C-10 | P1 | Shortcut reference | Display may not match actual bindings | The page uses a separately maintained hardcoded table, and "New connection Ctrl+N" appears twice | Generate the page from the actual shortcut registry; until then, rename it to "Shortcut reference" | `SettingsViewModel.cs:318-355`, especially `:322,336` | ✅ Finished 2026-10-03 (joesdu/VelaShell#551): the global and tab bindings were consolidated into one factory table, `ShortcutBindings`; the main window generates its KeyBindings from it under the current keymap (nothing is hardcoded in `MainWindow.axaml` any more, and a test enforces that), and the shortcuts page and the command palette hints read from it too. The remaining fixed shortcuts are still listed one by one in `ShortcutCatalog`, also test-guarded. Before that: the page was renamed "Shortcut reference"; entries were rebuilt and checked one by one against the actual bindings (`MainWindow KeyBindings`, `KeyboardShortcutService`, `TerminalTabView`, `RemoteFileEditorView`), all fabricated and duplicate entries were removed, and the actual `Ctrl+N` binding, which had been labeled but missing from the menu/command palette, was added. |
| C-11 | P1 | Restore defaults | Destructive operation lacks confirmation | Clicking it immediately resets settings for every page | Add a confirmation dialog and consider adding "Restore defaults for current page" | `SettingsView.axaml:246`;`SettingsViewModel.cs:562-567` | ✅ Done: both Restore defaults and "Clear history" now show a dangerous-action confirmation dialog; "Restore defaults for current page" is an optional enhancement and has not been implemented |

## 3. Settings That Can Be Merged

| ID | Priority | Current settings | Merge conclusion | Recommended merged name | Data migration recommendation | Status |
|---|---|---|---|---|---|---|
| M-01 | P0 | `TerminalBehavior.BellMode` + `TerminalBehavior.VisualBell` | Must merge | Terminal bell behavior | When `VisualBell=true`, migrate to `BellMode=visual`, then delete the old field | ✅ Done: migration is implemented through `AppSettings.Normalize()`; the `VisualBell` field is retained as a read-only migration slot because deleting the field would silently discard old configuration, and it is no longer consumed at runtime |
| M-02 | P1 | `Transfer.AutoResume` + `Transfer.ResumeEnabled` | ~~Both currently unimplemented~~ Resuming interrupted transfers is implemented (probe → verify starting point → 2MB safety fallback, consuming `ResumeEnabled`) | Resume interrupted transfers | `ResumeEnabled` is the sole switch; retain `AutoResume` as a read-only migration slot and do not consume it at runtime | ✅ Done (2026-07-23): the UI now returns as a single "Resume interrupted transfers" switch (File Transfer > Conflict handling); failed items support retry within the panel. Since 2026-10-24, batches left unfinished by a disconnect or an exit go into the transfer queue, come back as "Paused" after a restart, and "Resume" continues them from the partial files (see [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §9 and the eighteenth batch) |

## 4. Similar Names That Must Not Be Merged

| ID | Current name | Actual scope | Merge? | Recommended name | Recommended group | Status |
|---|---|---|---|---|---|---|
| N-01 | Sound alert | SSH connection disconnected | No | Play a sound when the connection is disconnected | Notifications | ✅ Renamed (General > Behavior; description distinguishes it from the security alert sound) |
| N-02 | Security > Alert sound | Host fingerprints and other security events | No | Play a sound for security events | Security and Privacy > Security Alerts | ✅ Renamed (description distinguishes it from the connection-disconnected alert sound) |
| N-03 | Terminal Bell | Remote program sends the BEL control character | No | Terminal bell behavior | Terminal > Bell and Tab Alerts | ✅ Renamed and placed in the "Bell and Tab Alerts" group |
| N-04 | Alert when disconnected | SSH connection state | No | Show a notification when the connection is disconnected | Notifications | ✅ Renamed |
| N-05 | Show notification after transfer completes | File transfer completion | No | Show a notification when file transfer completes | File Transfer > Transfer Behavior | ✅ Renamed (description distinguishes it from the connection-disconnected notification) |
| N-06 | Maximum retries | SSH automatic reconnect | No | Maximum automatic reconnect attempts | Connection > Automatic Reconnect | ✅ Renamed and moved to the subordinate automatic reconnect position (see C-02) |
| N-07 | Maximum retry attempts | Retries after file transfer failure | No | Transfer failure retry attempts (shown as "Retry failed transfers") | File Transfer > Conflict handling (after "Resume interrupted transfers") | ✅ Returned with R-09 (2026-10-24, eighteenth batch) |
| N-08 | Log retention days | Terminal session logs | No | Terminal session log retention days | General > Data and Logs | ✅ Renamed (description distinguishes it from transfer logs) |
| N-09 | Log retention days | File transfer logs | No | File transfer log retention days | File Transfer > Transfer Logs | ✅ Renamed (description distinguishes it from session logs) |

## 5. Items Recommended for Removal or Temporary Hiding

| ID | Priority | Item | Current state | Recommendation | Condition for showing again | Status |
|---|---|---|---|---|---|---|
| R-01 | P1 | Check for updates at startup | Disabled, unimplemented | Hide from ordinary settings | Update service and update source integration are complete | ✅ **Restored (2026-08-30)** — the condition is met: the message center uses it to decide whether to check for a new version at startup, and posts a message that opens the About page when one is found (see [message-center-and-feed.md](message-center-and-feed.md)). The switch previously had no consumer anywhere in the repository |
| R-02 | P1 | Update channel | Disabled, unimplemented | Hide | Stable and preview update sources are supported | ✅ Hidden (the entire "Updates" group was removed; the About page's "Check for updates" was also changed to honestly say "The update service is not yet connected" instead of pretending that the application is up to date) |
| R-03 | P1 | Download updates automatically | ~~Disabled, unimplemented~~ Implemented: when the startup check finds a new version it is downloaded and verified in the background, then a restart finishes the update | Show again | Automatic download and installation flow is complete | ✅ **Wired and shown (2026-10-24, nineteenth batch)**: Settings → General, right next to "Check for updates on startup", disabled while the check is off, and shown only on installations that can update in place (a writable directory on Windows / macOS); off by default, and a stored `true` left by the old default is turned off once by migration |
| R-04 | P1 | Master password protection | Disabled, unimplemented | Hide | Key derivation, unlocking, and credential migration are fully implemented | ✅ Hidden (a comment explains that current credentials are encrypted with a local machine key). The first step (handing the local encryption key to the system keychain) was done on 2026-10-25, see the host's plan.md section of that day; the master password is in [master-password-design](master-password-design.md) |
| R-05 | P2 | Tab bar position | ~~Disabled, Dock fixed at top~~ Implemented: the docking layer can put horizontal tab strips at the bottom of the pane | Show again | The Dock template truly supports switching between top and bottom | ✅ **Shown again (2026-10-24, fifteenth batch)**: Appearance → Window, between "Sidebar position" and "Show Tabs in Multiple Rows"; it governs horizontal tab strips only, and every pane changes together. Before, the docking layer could only put tabs at the top and `TabBarPosition` had no consumer |
| R-06 | P1 | Load keys into Agent automatically | ~~Disabled, unimplemented~~ Implemented: after a successful private-key-file or certificate authentication, the key (together with the certificate for certificate authentication) is added to the local ssh-agent (like OpenSSH `AddKeysToAgent`) | Show again | `KeyOptions.AddKeysToAgent` now has a runtime consumer | ✅ Returned to the UI (2026-09-23, "SSH Agent" section of the Key Management page). The field was renamed from `AutoLoadToAgent` to `AddKeysToAgent` and is **off by default**: the old field defaulted to `true` but never had a consumer, so the `true` in existing configurations is not user consent. Since 2026-10-06 certificate authentication adds too ("certificate + private key"); when the OpenSSH agent service is not running it goes through the current user's Pageant (the SSH library's default endpoint, `ssh/spec/07-forwarding.md` §7.2) |
| R-07 | P1 | Resume automatically | Disabled, duplicates interrupted transfer resume | Remove current UI | Return as a single setting after interrupted transfer resume is implemented | ✅ Returned as planned as the single "Resume interrupted transfers" setting (2026-07-23, see M-02) |
| R-08 | P1 | Enable interrupted transfer resume | Disabled, duplicates automatic resume | Remove current UI | Return as a single setting after interrupted transfer resume is implemented | ✅ Returned as planned as the single "Resume interrupted transfers" setting (2026-07-23, see M-02) |
| R-09 | P1 | Maximum transfer retry attempts | ~~Disabled, unimplemented~~ Implemented: temporary failures such as a dropped connection or a timeout are retried on the same session with back-off | Show again | The transfer retry pipeline is integrated | ✅ **Wired and shown again (2026-10-24, eighteenth batch)**: `TransferMaxRetries` is consumed by transfer failure retry (`TransferRetryPolicy`); the setting is File Transfer → Conflict handling → "Retry failed transfers", 0–10, default 3 |
| R-10 | P1 | Clean up temporary files automatically | ~~Unimplemented~~ Implemented: delete partial destination files on failure/cancellation (delete the remote file for uploads and the local file for downloads); takes effect only when interrupted transfer resume is disabled | Clean up partial files | `AutoCleanTempFiles` already has runtime consumers | ✅ Returned to the UI (2026-07-23, shown as a subordinate setting of interrupted transfer resume) |
| R-11 | P1 | Record all production sessions by default | Disabled, unimplemented | Hide | Session recording, storage, and privacy documentation are complete | ✅ Implemented and shown again (2026-07-12): recordings are stored in SonnetDB time-series storage, and the Security Audit page provides a toggle and a "Replay Center" entry (design NceE6); renamed "Record all sessions automatically" |
| R-12 | P1 | Input redaction | Disabled, depends on session recording | Hide | Session recording and redaction mechanisms are complete | ✅ Disabled (no longer needed): recording captures only the terminal output stream, passwords are not echoed in the first place, and independent redaction is unnecessary; the field is retained |
| R-13 | P1 | Standalone visual flash toggle | Conflicts with BellMode | Delete | Do not restore after merging into terminal bell behavior | ✅ Deleted (with C-01/M-01) |
| R-14 | P2 | Duplicate "New connection" in shortcuts | Displayed twice | Delete one entry | Generate the page from actual bindings | ✅ Done (the reference table has been rebuilt without duplicates, see C-10) |

## 6. Settings to Retain and Review Results

The following settings were reviewed against their runtime consumers, are effective, and must not be mistakenly deleted.

| Setting | Current implementation evidence | Conclusion | Recommended optimization |
|---|---|---|---|
| Start automatically at system boot | `App.axaml.cs:77,102`;`StartupRegistration.cs` | Retain | Place under "General > Startup and Window" (✅ Implemented) |
| Minimize to tray | `App.axaml.cs:96,103`;`MainWindow.axaml.cs:297` | Retain | Clarify its relationship with exit confirmation (✅ Implemented, see C-03) |
| Upload/download bandwidth limits | `SftpService.cs:39,76,277-288` | Retain | Hide the speed input fields when the master switch is off (✅ Implemented) |
| Preserve file timestamps | `SftpService.cs:288` | Retain | Place under "File Transfer > Transfer Behavior" (already in that group) |
| Transfer completion notification | `FileBrowserViewModel.cs:1076` | Retain | Name it explicitly as the file transfer completion notification (✅ Implemented) |
| Transfer logs | `FileBrowserViewModel.cs:1293-1295` | Retain | Hide the path and retention days when logging is disabled (✅ Implemented) |
| Session log cleanup | `App.axaml.cs:80` | Retain | Rename to terminal session log retention days (✅ Implemented, and conditionally shown with its parent switch) |
| Transfer log cleanup | `App.axaml.cs:81-82` | Retain | Rename to file transfer log retention days (✅ Implemented) |
| Confirm fingerprint on first connection | `InfrastructureServiceCollectionExtensions.cs:199` | Retain | Put it in the same group as the fingerprint change policy (same group) |
| Block on fingerprint change | `InfrastructureServiceCollectionExtensions.cs:221` | Retain | Explain blocking and the manual override policy in the text |

## 7. Recommended New Settings

| ID | Priority | Recommended setting | Current hardcoded behavior or gap | Recommended options | Recommended group | Source evidence | Status |
|---|---|---|---|---|---|---|---|
| A-01 | P0 | Permission for remote OSC 52 clipboard writes | Remote programs can write to the local clipboard; read queries are blocked | Deny / Ask every time / Always allow | Security and Privacy > Clipboard Permissions | `TerminalEmulator.cs:16,412-428` | Pending evaluation |
| A-02 | P1 | Maximum automatic reconnect attempts | The UI setting does not uniformly control the hardcoded `TerminalTab` limit | 1 to 20 attempts | Connection > Automatic Reconnect | `TerminalTabViewModel.cs:297` | ✅ Done (with C-02: hardcoded limit removed, UI range 1 to 20) |
| A-03 | P1 | Enable remote resource monitoring | CPU, memory, and network metrics are polled at a fixed interval | On/Off | Advanced > Monitoring and Diagnostics | `MainWindowViewModel.cs:708` | Pending evaluation |
| A-04 | P1 | Resource monitoring refresh interval | Currently polled approximately every 1 second | 1 / 2 / 5 / 10 seconds | Advanced > Monitoring and Diagnostics | `MainWindowViewModel.cs:708` | Pending evaluation |
| A-05 | P1 | Enable network latency probing | Currently probed approximately every 3 seconds | On/Off | Advanced > Monitoring and Diagnostics | `MainWindowViewModel.cs:725` | Pending evaluation |
| A-06 | P2 | Semantic highlighting | URLs, IP addresses, and error words are always highlighted on the client | On/Off | Terminal > Display > Advanced | `VelaTerminalControl.cs:126` | Pending evaluation |
| A-07 | P2 | Cursor blink accessibility settings | There is only a toggle, with no rate or reduced-motion strategy | Follow system / Standard / Slow / No blinking | Terminal > Cursor | `VelaTerminalControl.cs` cursor timer | Pending evaluation |
| A-08 | P2 | Editable shortcuts | There is currently only a static reference table | Edit, conflict detection, restore individual default | Shortcuts | `SettingsViewModel.cs:318-355` | ✅ Done 2026-10-03 (joesdu/VelaShell#551, reversing the 07-12 "not planned"): the global and tab bindings can be changed, unbound, and reset one by one or all at once; a clash asks before replacing; bindings that take a key away from the terminal are flagged on the spot. See the tenth batch |

### 7.1 Internal Parameters Not Recommended as Ordinary User Settings

| Parameter | Reason |
|---|---|
| SSH stream read buffer | An internal throughput and latency tuning parameter; users cannot easily determine the correct value |
| ShellStream internal buffer | Should be tuned by the implementation according to the connection and platform |
| Bridge dispose wait time | An implementation detail of the lifecycle |
| Echo suppression count and time window | An implementation detail of protocol bridging |
| Network activity arrow byte threshold | A visual implementation detail with insufficient user value at present |

## 8. Boundary Between Functional Management Tools and Settings

| Current page | Assessment | Recommended destination | Content retained in Settings Center |
|---|---|---|---|
| Key Management | Generating, importing, deleting, and copying keys are credential management tools | A standalone "Credentials and Keys" entry; provide a link from the connection editor | Preferences such as default authentication key |
| Snippets | Snippets are user data and a productivity tool, not a preference | Main navigation, command palette, or terminal context menu | If snippet behavior preferences exist in the future, retain only those preferences |
| Shortcuts | This belongs in settings | Retain in the Settings Center | Renamed back to "Shortcuts": the global and tab bindings are customizable (A-08), and the page is generated from the factory table plus the current changes (C-10) |
| About | An information page, not a setting | It can remain the final entry or be placed at the bottom of settings | Version, license, diagnostic information, and update entry ("Check for updates" now honestly says it is not connected) |

> The "Verify login" and "Show problems only" switches in the title bar of the session health check window (since 2026-10-25, see [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §11C) are window switches remembered only for this launch — **not settings**, not written to the settings file; the concurrency of 6 and the 30-second limit per host are constants. Do not mistake them for settings left unwired during an audit.

## 9. Recommended Information Architecture

> Status: Not yet implemented as a whole (Phase 4). Local changes implemented in this batch: the General page was rearranged into "Startup and Window / Language / Connection Defaults / Data and Storage / Behavior", automatic reconnect settings were grouped, and the Terminal page gained a "Bell and Tab Alerts" group.

### 9.1 Top-Level Navigation

1. General
2. Appearance
3. Terminal
4. Connection
5. File Transfer
6. Security and Privacy
7. Shortcuts
8. Advanced
9. About

Key Management and Snippets are moved out of the Settings Center and become standalone functional entries.

### 9.2 Detailed Grouping Table

| Top-level group | Second-level group | Settings | Display rule |
|---|---|---|---|
| General | Startup and Window | Start automatically at system boot, restore last session, restore split layout (`General.RestoreLayoutOnStartup`, off by default), startup window state, minimize to tray when closing the window, confirm before exiting the application | The "Exit confirmation" description should explain tray behavior; restore split layout is subordinate to restore sessions and disabled while that is off (sixteenth batch) |
| General | Language | Interface language (`Language`, empty by default = follow the system: English when the system language is not one of the five UI languages; a language chosen in Settings takes precedence) | The settings page must be localized in sync; the first dropdown item is "Follow system" |
| General | Behavior | Follow the active terminal tab (`General.FollowActiveTerminalInExplorer`), collapse groups on startup (`General.CollapseGroupsByDefault`, off by default) | The collapse option only decides the initial state **the first time a group is seen in this run**; manual expand/collapse is remembered in-process and a full tree rebuild does not fold it back |
| General | Data Management | Import settings, export settings, clear recent connections, restore defaults | Restore defaults must require confirmation; export scope must be accurate. Export / import by category with checkboxes, Security & Audit and Proxy unticked by default (seventeenth batch) |
| General | Application Updates | Check for updates at startup, update channel, download automatically | Check for updates on startup (R-01) and download updates automatically (R-03, nineteenth batch) are shown; the latter appears only on installations that can update in place and is disabled while the check is off; the update channel is still not shown (R-02) |
| Appearance | Application Theme | Theme mode, theme color (`AccentColor`, factory empty = follow theme), interface font, interface font size | Keep real-time preview; "Follow theme" sits in front of the swatches; any other color comes from the color picker (interaction-and-ui-specs §14.3) |
| Appearance | Window and Layout | Window opacity, show menu bar, sidebar position, tab bar position (`Appearance.TabBarPosition`, top out of the box), show tabs in multiple rows (`Appearance.MultiRowTabs`, off by default), splash screen (`Appearance.SplashStyle`, Classic out of the box; Windows only) | Tab bar position governs horizontal tab strips only and changes every pane together; it previews immediately and cancelling rolls it back; panes moved left / right from the tab right-click menu are unaffected and land on the side chosen here when they go back to horizontal (fifteenth batch); multiple rows is the same value as the check item of the same name in the tab right-click menu; the splash screen dropdown has a live preview under it and takes effect on the next launch (thirteenth batch) |
| Terminal | Font and Display | Terminal font, font size, line height, padding, scrollback buffer, semantic highlighting | Semantic highlighting can be placed in an advanced collapsed section. Scrollback lines (`ScrollbackLines`, default 10000, adjustable 100–200000; out-of-range values in the settings file are clamped back on load): saving applies it to every open tab; **lowering it takes effect at once** — the history beyond the new limit is trimmed immediately and cannot be recovered (a scroll position inside the trimmed part is pulled back into range, and folds and the selection are cleared), so an idle tab with no further output gives the memory back right away; raising it only widens the limit and does not bring trimmed lines back. **It applies to the main screen only**: the alternate screen of full-screen programs (vim, less, htop) never has scrollback, and changing the value while a full-screen program runs still changes the main screen |
| Terminal | Colors | Color scheme, foreground color, background color, cursor color, selection color, ANSI palette | Collapse the ANSI palette by default; if it cannot be edited, label it as a preview |
| Terminal | Cursor | Cursor style, blink, blink accessibility settings | Support reduced motion |
| Terminal | Terminal Protocol | TERM type, output encoding | Explain that TERM affects new connections only |
| Terminal | Bell and Tab Alerts | Terminal bell behavior, background-tab Bell alerts | Delete the standalone visual flash toggle |
| Terminal | Scrolling | Auto-scroll while output is being received, scroll to bottom on keypress | Retain as independent settings |
| Terminal | Clipboard and Selection | Copy on select, paste on right click, trim trailing spaces, select word on double-click, copy with Ctrl+C, confirm multiline paste | Order by input/output risk |
| Terminal | Input and Session | IME, execute command after connection | A risk warning for startup commands would be more appropriate |
| Terminal | Output Rules (a page of its own, right after "Terminal") | The user's highlight and auto-answer rule table (`TerminalBehavior.OutputRules`) | Exported / audited with the "Terminal" category; auto-answer acts only on the prompt line the cursor is on, and rules that would match a password prompt are refused (twenty-third batch) |
| Connection | Connection Defaults | Default port, connection timeout, heartbeat interval | Move from General |
| Connection | Automatic Reconnect | Automatic reconnect, maximum attempts, reconnect interval | Hide subordinate settings when automatic reconnect is disabled |
| Connection | Authentication Preferences | Remember password, default authentication key | Provide a link to Key Management |
| File Transfer | Paths and Editor | Local download directory, default editor | Retain |
| File Transfer | Remote Editing | When double-clicking a file (`Transfer.DoubleClickAction`: system default program / built-in editor / configured editor, system default by default), upload after editing (`Transfer.AutoUploadOnEdit`, on by default) | Shown. Turning auto-upload off = local copies opened by external programs are not watched and nothing is uploaded on wrap-up either; the built-in editor's Save still uploads. The three entry points and the upload rules are in Interaction and UI Specs §6. Both settings descriptions were corrected on 2026-10-25: the double-click one no longer says all three upload automatically, and the auto-upload one says it only governs the external programs |
| File Transfer | Transfer Behavior | Maximum concurrency, preserve timestamps (an SFTP upload sets the modification time on the same handle before closing, one round trip), flush uploads to disk (`Transfer.FsyncAfterUpload`, off by default: `fsync@openssh.com` before an SFTP upload is closed, skipped on servers without it, and a failed flush fails the upload), conflict handling, show hidden files, completion notification, delete remote folders with rm -rf (`Transfer.UseRecursiveDeleteCommand`, on by default) | The hidden-file state must be unified; the `rm -rf` entry must state three things: it applies **only to SSH sessions that have an exec channel** (standalone SFTP / FTP / plugin protocols always walk the tree over SFTP), it **falls back to the SFTP walk automatically** when the command is unavailable or exits non-zero, and **there is no per-entry progress** on that path (the bar goes indeterminate, and cancelling can only close the channel). The delete confirmation prompt is unchanged |
| File Transfer | Bandwidth Limits | Master switch, upload limit, download limit | Hide both rate fields when disabled |
| File Transfer | Transfer Logs | Record logs, retention days, storage path | Hide subordinate settings when logging is disabled |
| File Transfer | Resume and Retry | Interrupted transfer resume, transfer retries, clean up temporary files | All three are implemented and shown, in the "Conflict handling" group: resume interrupted transfers (M-02), clean up partial files (only while resume is off, R-10), retry failed transfers (R-09, eighteenth batch) |
| Security and Privacy | Host Identity Verification | Confirm fingerprint on first connection, block on fingerprint change; trusted hosts (the list, OpenSSH known_hosts import / export, hash host names on export), trusted host CAs | Explain first trust and change policies together; actions on trusted hosts and trusted host CAs apply immediately and are not tied to "Save" (twentieth batch) |
| Security and Privacy | Clipboard Permissions | OSC 52 write permission | Recommend Deny or Ask every time by default |
| Security and Privacy | Credentials | Remember password, master password protection | If "Remember password" is placed on the connection page, provide only a link here and do not duplicate the control |
| Security and Privacy | Security Alerts | In-app notifications, security alert sound, Webhook, Webhook URL | Show the URL only after Webhook is enabled |
| Security and Privacy | Session Audit | Session recording (✅ Implemented: toggle + Replay Center entry), audit log (✅ Implemented: viewer window + retention days `Security.AuditLogRetentionDays`; 2026-10-24 added profile / settings / shared-credential events, a time-range filter and CSV export, see the seventeenth batch), input redaction (not planned, output-stream recording does not echo passwords) | Shown |
| Shortcuts | Shortcuts | Show actual bindings by command | The global and tab bindings can be changed, unbound and reset (A-08); everything else is a fixed, read-only shortcut |
| Advanced | Monitoring and Diagnostics | Resource monitoring, refresh interval, latency probing | Collapse by default or label the additional network overhead |
| About | Application Information | Version, runtime environment, license, configuration directory, update entry | Exclude from Restore defaults |

## 10. Conditional Display Rules

> Implementation prerequisite (landed on 2026-07-11): `GeneralOptions/TransferOptions/SecurityOptions/TerminalBehaviorOptions/KeyOptions` all inherit from `ObservableOptions` (INPC). Otherwise, `IsVisible` bindings for subordinate items would not refresh when the parent switch changes. The conditional display of the Webhook URL had failed for exactly this reason.

| Parent setting | Subordinate settings | Recommended behavior | Status |
|---|---|---|---|
| Automatic reconnect | Maximum automatic reconnect attempts, reconnect interval | Hide or disable when the parent is off, while retaining saved values | ✅ Implemented (hidden, value retained) |
| Enable bandwidth limits | Upload speed limit, download speed limit | Hide when the parent is off | ✅ Implemented |
| Record transfer logs | Retention days, log path | Hide when the parent is off | ✅ Implemented |
| Webhook | Webhook URL | Conditional display already exists; retain it | ✅ Fixed (actually works after INPC) |
| Terminal color scheme | Custom colors, ANSI palette | Collapse advanced editing when using a preset; expand after entering custom mode | ⏳ Pending (to be done with the Phase 4 Appearance/Terminal page restructuring) |
| Session logs | Terminal session log retention days | Hide when the parent is off | ✅ Implemented |

## 11. Recommended Implementation Order

### Phase 1: Fix Behavioral Correctness (✅ Completed 2026-07-11)

- [x] Merge BellMode and VisualBell
- [x] Unify automatic reconnect attempts
- [x] Unify the source of settings default values
- [x] Persist the Show hidden files state
- [x] Clarify the semantics of exit confirmation and minimize to tray
- [x] Add confirmation for Restore defaults (and for Clear history)

### Phase 2: Fix Misleading Text and Pages (Completed except for localization)

- [x] Correct the import/export scope description
- [x] Migrate localization text on settings pages (C-09, recommended as a separate batch)
- [x] Generate the shortcuts page from actual runtime bindings (manual verification and rebuilding completed, including the `Ctrl+N` binding; automatic generation remains pending until binding registration points are consolidated); since 2026-10-03 the global and tab bindings are consolidated into the factory table and generated from it, see C-10
- [x] Delete duplicate shortcut entries
- [x] Rename settings related to sounds, notifications, retries, and logs

### Phase 3: Clean Up Unimplemented Features (✅ Completed 2026-07-11)

- [x] Hide unimplemented update settings
- [x] Hide master password and ssh-agent settings
- [x] Delete duplicate resume-transfer UI
- [x] Hide unimplemented transfer retry and cleanup settings (later: cleanup on 2026-07-23 and retry on 2026-10-24 were implemented and shown again, see R-10 / R-09)
- [x] Hide unimplemented session recording and redaction settings (later, on 2026-07-12: recording was implemented and shown again, see R-11)
- [x] Hide unsupported tab bar position (later, on 2026-10-24: shown again once the docking layer supported the bottom, see the fifteenth batch)

### Phase 4: Adjust the Information Architecture

- [x] Move connection defaults from General to Connection
- [x] Move terminal colors from Appearance to Terminal
- [x] Move startup window state to General
- [ ] Move Key Management out of the Settings Center
- [ ] Move Snippets out of the Settings Center
- [ ] Add the Advanced group
- [x] Add conditional display for subordinate settings (§10, except terminal color scheme collapsing)

### Phase 5: Add High-Value Settings

- [x] Add OSC 52 clipboard permissions
- [x] Add resource monitoring and latency probing controls
- [x] Add a semantic highlighting toggle
- [x] Add cursor blink accessibility settings
- [x] Evaluate editable shortcuts (concluded "do not implement" on 2026-07-12; reversed and implemented on 2026-10-03 because of joesdu/VelaShell#551, see A-08)

## 12. Acceptance Criteria

- [x] Every persistent setting has one clear data source and one source for its default value
- [x] No two settings control the same final behavior with opaque priority
- [x] Every visible setting produces the actual effect described by its text (including changing "Check for updates" to an honest status message)
- [x] Unimplemented features no longer pretend to be configurable items
- [x] The settings UI can switch completely with the interface language (pending C-09)
- [x] The shortcuts page matches runtime bindings (currently manually checked; this item can be maintenance-free only after automatic generation); the global and tab bindings are generated from the factory table since 2026-10-03
- [x] All destructive operations have confirmation mechanisms (Restore defaults, Clear history)
- [x] Subordinate settings are shown only when the parent feature is enabled (except terminal color scheme collapsing, to be handled in Phase 4)
- [x] User data management such as keys and snippets is no longer mixed with preference settings (Phase 4)
- [x] Capabilities crossing the remote/local trust boundary, such as OSC 52, have explicit permission controls (A-01 pending evaluation)

## 13. Primary Source Code Index

| File | Purpose |
|---|---|
| `src/VelaShell.Core/Models/AppSettings.cs` | Settings model, default values, and settings groups; `Normalize()` handles migration of old configuration; option classes uniformly inherit from `ObservableOptions` |
| `src/VelaShell/ViewModels/SettingsViewModel.cs` | Settings loading, saving, index mapping, shortcut display, and import/export |
| `src/VelaShell/Views/SettingsView.axaml` | Settings window navigation, save, and Restore defaults operations (Restore defaults is confirmed in code-behind) |
| `src/VelaShell/Views/Settings/*.axaml` | Settings pages, labels, descriptions, and control states |
| `src/VelaShell/App.axaml.cs` | Consumers of settings for startup, tray, theme, log cleanup, and other behavior |
| `src/VelaShell/Views/MainWindow.axaml.cs` | Window appearance, close behavior, and session restoration |
| `src/VelaShell/ViewModels/MainWindowViewModel.cs` | Consumers of terminal, connection, automatic reconnect, and file browser settings; write-back entry for hidden files |
| `src/VelaShell/ViewModels/TerminalTabViewModel.cs` | Tab reconnect limit (supplied by the host according to settings) and disconnect alert |
| `src/VelaShell.Terminal/Rendering/VelaTerminalControl.cs` | Terminal display, Bell (BellMode only), cursor, and semantic highlighting |
| `src/VelaShell.Terminal/Emulation/TerminalEmulator.cs` | Processing terminal control sequences such as OSC 52 |
| `src/VelaShell.Core/Sftp/SftpService.cs` | Consumers of bandwidth limit and timestamp settings |
| `src/VelaShell/ViewModels/FileBrowserViewModel.cs` | Hidden files (toggle writes back to settings), completion notifications, and transfer logs |
| `src/VelaShell.Infrastructure/DependencyInjection/InfrastructureServiceCollectionExtensions.cs` | SSH timeout, heartbeat, and host fingerprint policies |
| `src/VelaShell.Core/Models/OutputRule.cs`, `src/VelaShell.Core/OutputRules/` | Output rule model, compilation (linear-time engine) and password guard (twenty-third batch) |
| `src/VelaShell.Terminal/Semantics/OutputTriggerScanner.cs` | Auto-answer decisions (cursor line only, no answer on a line carrying shell prompt marks, no answer while typing locally, cooldown and per-minute limit) |
| `src/VelaShell/ViewModels/OutputRulesViewModel.cs`, `src/VelaShell/Views/Settings/OutputRulesPage.axaml` | The "Output Rules" settings page |

## 14. Remediation Record

### 2026-07-11 First batch (Phases 1 to 3 + Conditional Display)

**Behavior fixes**

- `AppSettings.Normalize()`: when loading (through both settings services) and importing JSON, migrates `VisualBell=true` to `BellMode=visual` and clears the old value; `VelaTerminalControl` removes the `VisualBell` property, and Bell now follows only the three `BellMode` states.
- `TerminalTabViewModel.MaxReconnectAttempts` changed from a hardcoded 3 to a value supplied by the host in `OnTabDisconnected` according to `General.MaxRetries`.
- The file browser toolbar's "Show hidden files" toggle now goes through the `ShowHiddenFilesToggled` callback to `MainWindowViewModel.PersistShowHiddenFiles`, which writes back and saves it; after settings are saved, all cached panels are synchronized through a broadcast.
- The initial values of `Language/ScrollbackLines` in `SettingsViewModel` now come from `new AppSettings()`; the default value of `VelaTerminalControl.CursorStyle` is aligned with the model.
- Dangerous confirmation dialogs were added for Restore defaults and Clear history (in view code-behind).
- The option classes (`General/TerminalBehavior/Transfer/Security/Keys`) now uniformly inherit from `ObservableOptions` (INPC), supporting real-time conditional visibility and fixing the Webhook URL conditional display issue.

**Text and pages**

- General page rearranged: the update group and master password are hidden; reconnect settings are grouped under "Behavior" and shown conditionally; import/export, logging, notifications, sounds, and other items were renamed according to §4.
- Terminal page: added the "Bell and Tab Alerts" group, with the three-state "Terminal bell behavior", and removed "Visual flash".
- Transfer page: removed the three unimplemented UI items for resume, retry, and cleanup; bandwidth and transfer log subordinate items are shown conditionally; log retention days was renamed.
- Security Audit page: session recording and input redaction are hidden, and the alert sound was renamed.
- Appearance page: tab bar position hidden (since restored, see the fifteenth batch). Key page: Agent toggle hidden.
- Shortcuts page renamed "Shortcut reference" and rebuilt from actual bindings; `MainWindow.axaml` gained the actual `Ctrl+N` binding; the About page's "Check for updates" was changed to an honest status message.

**Verification**: `dotnet build` passed for the full solution; 601 tests passed. The only failure, `ConPty_SpawnsShell_HandshakesAndSignalsEof`, also fails on the unchanged working tree (environment-related and unrelated to this batch).

### 2026-07-12 Second batch (Three-Way Host Fingerprint Confirmation + Trusted Host Management)

Background: after the user enabled "Require manual fingerprint confirmation on first connection", no dialog appeared. The cause was not a missing link in the chain. The target host had already been automatically recorded by TOFU before the switch was enabled (persisted in the SonnetDB `known_hosts` collection, no longer in `~/.velashell/known_hosts.json`; that file is used only as a one-time import source on first run), and there was no UI for deleting trusted records.

- **Confirmation dialog changed to three choices** (consistent with mainstream SSH clients): permanently trust (write to known_hosts) / trust for this session only (in-process `HostTrustOnceCache`, not persisted, ask again after restart) / cancel (reject and abort the connection, fail-closed). `IHostKeyPrompt.ConfirmAsync(bool)` was refactored to `DecideAsync(HostKeyDecision)`; fingerprint changes (when the blocking switch is off) use the same three choices.
- **Added "Trusted Hosts" management** (Settings → Security Audit): list host:port, key type, and SHA256 fingerprint, with per-item deletion; deletion takes effect immediately, and the first-connection confirmation flow runs again on the next connection.
- **Fixed the gap where SFTP channels did not validate host fingerprints**: the standalone SFTP channel previously did not subscribe to `HostKeyReceived` (= trusted any fingerprint by default, creating MITM risk); it now strictly compares against known_hosts or temporary trust from the current run, rejecting mismatches without showing a dialog.
- The security alerts gained the `hostkey-trusted-once` event type.
- **Trusted host addresses are masked by default** (to prevent screenshot leaks): added a "Hide host addresses" switch, enabled by default (shows a fixed-length mask and does not reveal the address length); only manually disabling it shows the IP and port. This switch is intentionally not persistent and returns to hidden every time the settings are opened. Fingerprints remain visible to distinguish entries (public key hashes, not sensitive information).

**Verification**: full build passed; 268 `VelaShell.Tests` tests and 148 `Core.Tests` tests passed (including new three-choice dialog cases).

### 2026-07-12 Third batch (Session Recording and Replay + Webhook Repair + Cloud Sync)

- **Session recording and replay (design NceE6, R-11 implemented)**:
  - Storage: metadata is stored in the SonnetDB document collection `recordings`; output chunks are stored in the time-series measurement `session_recording_chunks` (tag: recording_id; fields: offset_ms/data-Base64; time = start time + offset);
  - Capture: uses the same hook point as session logs (`SshTerminalBridge.DataReceived`), buffers into 600ms/64KB chunks and writes them in the background; failures disable recording automatically without affecting the session; the `Security.RecordProductionSessions` switch applies to connections established afterward; retention days are cleaned at startup together with "Terminal session log retention days";
  - Replay Center (standalone window): recording list (name/time/duration/size), read-only terminal replay on a timeline, drag-to-seek (instant replay after reset), 1x/2x/4x speed, skip idle segments (retains a one-second pause feel), delete, export to asciicast v2 (`.cast`, asciinema-compatible), and an automatic recording toggle;
  - R-12 input redaction confirmed as not planned: recording captures only the output stream, and passwords are not echoed.
- **Webhook**: the complete feature path was confirmed (security event JSON POST + 5-second timeout + silent failure), so it was retained; the URL input area was changed to a subordinate block with a "Webhook URL" label and payload format description, shown after the switch is enabled.
- **Cloud Sync (new feature, user-requested)**: GitHub Gist multi-device synchronization (settings, connections including tunnels, and snippets), with native Gist revisions as version history, optional PBKDF2+AES-GCM end-to-end encryption, as described on Settings → Cloud Sync; sync configuration and tokens are stored only locally.
- Other changes: added a "Support and Donations" page to Settings; confirmed that the Shortcuts page is display-only (A-08 closed); trusted host addresses are masked by default.

### 2026-09-20 Fourth batch (#474: collapse groups on startup + delete remote folders with rm -rf)

Two new settings, both part of user request #474 (host side: `plan.md` §82).

- **`General.CollapseGroupsByDefault` (General, off by default)**: the explorer loads its groups collapsed.
  It only decides the initial state **the first time a group is seen in this run** — whatever the user expands or
  collapses by hand is remembered in-process, so the full tree rebuilds triggered by creating / editing / deleting a
  connection or by a cloud-sync write-back do not fold it back. The memory is not persisted; a restart returns to what
  the setting says.
  ⚠️ On its own this option is of limited use: it is only complete together with pinned connections (frequently used
  connections are hoisted to the very top of the tree and stay visible while everything is collapsed). See
  [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §4.
- **`Transfer.UseRecursiveDeleteCommand` (File Transfer > Transfer Behavior, on by default)**: deleting a remote folder
  first tries a single `rm -rf` instead of walking the tree entry by entry over SFTP.
  - **Why it defaults to on**: the SFTP path first lists the whole tree to compute a total (for the progress bar) and
    then sends one `SSH_FXP_REMOVE`/`RMDIR` per entry, so round trips scale with the file count; `rm -rf` is one round
    trip. The result is identical.
  - **Scope**: it applies only to **SSH sessions that have an exec channel**. Standalone SFTP profiles, FTP and plugin
    protocols such as S3 have no command channel and always walk the tree over SFTP. Three more gates: real directories
    only (symbolic links are still deleted as leaves, unchanged semantics), the path must be absolute, and it must not
    be the root directory.
  - **Failure means fallback**: if the command is missing, blocked by policy, or exits non-zero after deleting only part
    of the tree, it falls back to the SFTP walk — and the fallback surfaces the real error rather than a context-free
    `rm: exit 1`.
  - **Known cost (stated in the setting description)**: there is no per-entry progress on this path (the bar goes
    indeterminate), and cancelling can only close the channel — the remote `rm` may already have finished.
    **The delete confirmation prompt is unchanged.**

### 2026-09-20 Fifth batch (#476: fingerprint changes now ask instead of blocking)

The default of `Security.BlockOnFingerprintChange` **flipped from on to off** (host side: `plan.md` §83).

- **Why**: "the machine was replaced, the IP did not change" is the most common legitimate fingerprint
  change there is. The old default (block) reported it as a connection error — and specifically as
  Tmds.Ssh's English `UntrustedPeer`, which neither says the host key changed nor suggests what to do
  next; the only way out was to delete the record from `.velashell` by hand. Other SSH clients raise an
  "accept and overwrite / reject" dialog here, and VelaShell already had that dialog (the three-way
  prompt) — the default value was simply keeping it from ever showing.
- **What it does now**: a changed fingerprint goes through the same three-way prompt as first-connection
  confirmation (trust permanently = overwrite known_hosts / trust once / cancel). The dialog gained two
  things: **the recorded fingerprint from known_hosts** next to the current one (you need both to tell
  your own reinstall apart from an interception), and a line of guidance naming those two possibilities.
- **One-time migration**: the `true` sitting in existing configs cannot be told apart from the old
  default, so this follows the same pattern as session recording and the proxy default — turn it off
  once, stamp `Security.FingerprintChangeDefaultMigrated`, and respect the user from then on. Anyone
  wanting strict fail-closed behaviour turns "Block and alert on fingerprint change" back on and gets
  exactly the old behaviour.
- **Rejections say something useful**: whether the policy blocked it or the user cancelled, the error is
  now localized text naming the host, both fingerprints, and the Settings → Security & Audit → Trusted
  hosts route to remove the record.
- **Jump chains use the same policy**: the jump hop used to run a separate check that silently returned
  false for unknown *and* changed keys — so a machine only ever used as a jump host failed on its first
  use. It now goes through the same `AddHostAuthentication`.
- **The connect timeout during the prompt**: Tmds.Ssh 0.24's `ConnectTimeout` covers the whole handshake,
  **including** the HostAuthentication callback, so the timer keeps running while the dialog is open.
  Think for fifteen seconds and the connection is already timed out by the time you click. The decision
  has taken effect by then, so the wrapper **reconnects once** on this particular failure (once only).

### 2026-09-23 Sixth batch (new “X Server” page)

A new top-level section **X Server** (after “Proxy”) configures the local X server — on Windows, the VcXsrv the user
installed (host side: `plan.md` §100). All new settings live in `AppSettings.XServer` (`XServerOptions`); configs saved
before this section existed get the defaults from `Normalize`.

| Setting | Default | VcXsrv argument | Notes |
| --- | --- | --- | --- |
| VcXsrv executable | empty = auto-detect | — | Tries Program Files, Scoop, then PATH; **a filled-in path is the only one used** — if it is missing that is reported as such, never silently swapped for another install |
| Display number | Auto | `:N` | Auto = the first display nobody listens on, starting at :0; a fixed number that is taken fails before launching. Range :0–:15, clamped the same way by `Normalize` |
| Window mode | Multiple windows | `-multiwindow` / (none) / `-nodecoration` / `-fullscreen` / `-rootless` | Unrecognized values fall back to multiple windows. Shared by both engines since 2026-10-10: on the built-in engine one large window / no title bar / fullscreen start it in one-window mode (the whole X desktop in one window, a remote window manager takes over), while multiple windows and rootless both give one window per program; takes effect the next time the X Server starts |
| Enable clipboard | on | `-clipboard` / `-noclipboard` | |
| Copy on selection | on (off by default since 2026-10, see the eleventh batch) | `-primary` / `-noprimary` | Only meaningful with the clipboard on; disabled and omitted from the command line otherwise |
| Keyboard layout | Auto | `-xkblayout` | Shared by both engines. Auto = not passed to VcXsrv (it follows the Windows layout), and the built-in engine follows the system's current layout; a chosen layout is passed to VcXsrv as `-xkblayout` and the built-in engine uses the keymap tables shipped with the app |
| Keyboard model | pc105 | `-xkbmodel` | |
| Capture special Windows keys | off | `-keyhook` / `-nokeyhook` | |
| Native OpenGL (WGL) | on | `-wgl` / `-nowgl` | |
| Disable access control | off | `-ac` | The description states the risk; connections arriving through SSH X11 forwarding come from 127.0.0.1 and do not need it |
| Additional VcXsrv arguments | empty | appended verbatim, last | For the same switch the later one wins; the “Help” button next to it documents every VcXsrv argument in five languages |
| Start when VelaShell starts | off | — | |
| Start automatically for X11 forwarding | on | — | Stays out of the way silently when another X server already listens on :0 or VcXsrv is not installed |
| Show tray icon | off | `-trayicon` / `-notrayicon` | VcXsrv's own tray icon |

- **Every switch is written out in both states** rather than relying on VcXsrv's defaults (which have changed between
  versions); the page previews the exact command line for the next start.
- Changes **apply the next time the X server starts** (the subtitle says so); a running X server that is showing windows
  is never restarted behind the user's back.
- On non-Windows platforms the page is a single note: Linux uses the desktop's own X / XWayland, macOS uses XQuartz,
  both through `DISPLAY`.

### 2026-09-24 Seventh batch (X Server page: built-in engine)

The “X Server” page gains an **Engine** section at the top. The default is now the host's built-in X server
(`VelaShell.XServer`, ships with the app; host side in `plan.md` §105); VcXsrv becomes an option on Windows. The new setting
lives in `AppSettings.XServer.Engine`; unrecognized values fall back to built-in in `Normalize`.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Engine | Built-in | `builtin` / `vcxsrv` | Only shown on Windows; other platforms only have the built-in one, and a `vcxsrv` value (e.g. synced from Windows) is treated as built-in |

- **Shared by both engines**: display number, window mode (since 2026-10-10), enable clipboard, copy on selection, keyboard layout, start with VelaShell, start automatically
  for X11 forwarding. On the built-in engine the two clipboard switches map to the server's CLIPBOARD / PRIMARY sync (since 2026-10
  the built-in engine also exchanges only with the session that has the keyboard focus, see the eleventh batch).
- **VcXsrv-only**, shown only when VcXsrv is selected: program location, keyboard model, capture special
  keys, native OpenGL, disable access control, additional arguments and the command-line preview, tray icon. On Windows the
  built-in engine follows the system keyboard layout and needs no configuration.
- The “stay out of it” condition of “Start automatically for X11 forwarding” becomes: another X server is already in use —
  on Windows something listens on `localhost:0` (since 2026-10 only a process in the current user's session counts, see the eleventh
  batch), elsewhere `DISPLAY` is set; a selected-but-missing VcXsrv is silent too.
- The former “on non-Windows platforms the page is a single note” is gone: the built-in engine works everywhere, so the page is
  shown on every platform (except the Engine section).

### 2026-09-27 Eighth batch (#521: show tabs in multiple rows)

Appearance → Window gains a switch matching Visual Studio's "Show tabs in multiple rows". Host side in `plan.md` §127,
interaction in [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §4B.3.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Show Tabs in Multiple Rows | Off | On / off (`Appearance.MultiRowTabs`) | Off: when one row is too narrow, the widest tabs shrink first (120px floor), then the strip scrolls; on: tabs wrap onto more rows and the strip grows |

- **Two entry points, one value**: any tab's right-click menu has a check item of the same name, and toggling it writes this
  setting back; saving the settings page flows back into the tab strip the same way. Nothing is saved again when the value
  flowing back is unchanged, so the two sides never bounce writes off each other.
- One switch for the whole workspace, so every pane changes together; it only applies to horizontal tab strips (top or bottom;
  the bottom is in the fifteenth batch), and the menu item is disabled when the strip is docked left or right.
- The row label and the menu item use the same localization key (`Dock_MultiRowTabs`), so the two always say the same thing.

### 2026-09-29 Ninth batch (accent follows the theme + audit log viewing and retention)

Host side in `plan.md` §131 / §135, interaction in [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §14 and §15.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Theme color | Empty (follow theme) | Empty / `#RRGGBB` (`AccentColor`) | The factory value used to be `#E91E63`, and an accent override outranks the theme — so every one of the twelve themes had its own accent covered by the same pink. An empty string means no override: the current theme's own accent is used |
| Audit log retention days | 180 | 1–3650 (`Security.AuditLogRetentionDays`) | Shared by the audit log and the connection history (`conn_history`); older entries are pruned at startup |

- **Saved theme colors are not migrated**: a stored `#E91E63` cannot tell a user's choice from the old factory value, so it
  stays in effect. A "Follow theme" button now sits in front of the swatches, and the input's placeholder says "Follow
  theme" too — before, the only way back to the theme's accent was to clear the input.
- **Audit log retention also governs connection history**: connection history is the ledger behind "Recent connections";
  keeping it longer than the audit makes no sense, and keeping it shorter would leave sessions in the audit that the
  sidebar no longer knows. The clamp lives in `Normalize`, and the settings page's `NumericUpDown` uses the same range.
  Pruning happens **at startup only**, at the same time as session logs and recordings expire, as a time-based `DELETE`
  rather than the recordings' "stage → rebuild → restore".
- "View" in the same section opens the audit log window (read-only, not a setting).

### 2026-10-03 Tenth batch (joesdu/VelaShell#551: customizable shortcuts)

Host side in `plan.md` §155; the rules are under "Customizing shortcuts" in [Keyboard Shortcuts](keyboard-shortcuts.md).

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Custom shortcuts | Empty (every binding at its factory default) | Binding id → gesture (`Shortcuts.Overrides`); empty string = unbound | Only the changed entries are stored; an unrecognized gesture falls back to the factory default |

- **Reverses A-08's "not planned"**: #551 reported nano's `^K` (cut line) being swallowed by the command palette.
  Window-level bindings consume keys before the terminal control sees them, so any set of factory bindings collides with
  some remote program (bash's `^W`, tmux's `^B`, vim's `^F`…); letting users unbind them is the only answer that holds.
- **What can be changed**: the "Global" and "Tabs & Panels" groups (except pane focus `Alt+arrows`, which is only
  intercepted while split anyway), plus search, clear screen and the three zoom bindings in the "Terminal" group. Everything
  else (copy/paste, scrolling, the completion popup, each dialog's Esc / Enter, mouse gestures) stays fixed.
- **Rules for a new binding**: it needs Ctrl or Alt (or Win / Cmd); without a modifier only F1–F24 are allowed; fixed
  shortcuts cannot be taken; if another action already has the gesture you are asked first, and replacing it leaves that
  one unbound; recording the factory default again counts as no change and leaves no entry.
- **Staged until saved**: like every other setting, changes are written when you click "Save settings"; the main window
  then rebuilds its KeyBindings and the command palette hints follow. Settings export / import carries it, and cloud sync
  includes it when app settings are synced (it is not a device-local field).
- The page title went back from "Shortcut reference" to "Shortcuts", and the subtitle no longer says "read-only" (those
  words are what put off the reporter of #551).

### 2026-10-07 Eleventh batch (X Server: clipboard follows the focus, restricting programs from SSH sessions)

Changed together with the X Server's second library-wide review. Library behaviour: [X Server architecture](../xserver/design/architecture.md)
§7, "Several sessions sharing one display — what is tightened", and §10; interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md)
§4A.3 and §13.1; what users see and what to do: [X Server troubleshooting](../xserver/troubleshooting.md).

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Copy on selection | **off** (was on) | on / off (`XServer.CopyOnSelection`) | Shared by both engines; VcXsrv's default command line therefore has `-noprimary`. On the built-in engine it maps to the server's PRIMARY sync (`SyncPrimary`) — the host used to default it to on, the opposite of the library's own default |
| Restrict programs from SSH sessions | off | on / off (`XServer.RestrictForwardedClients`) | Built-in engine only, and only shown when it is selected (below the keyboard layout in the “Keyboard” section). Maps to the library's `RestrictForwardedClients`: programs forwarded over SSH cannot see XTEST, receive no raw key events and cannot change input devices |

- **Why “Copy on selection” is now off by default**: when on, whatever is copied locally (passwords included) can be pasted into any X program
  with a middle click, and selecting text in an X program quietly rewrites the system clipboard. **Only the factory default changed; saved
  configs are not migrated**: a saved “on” cannot be told apart from the old default, so it stays in effect until the user turns it off.
- **The clipboard following the session with the keyboard focus is not a setting**: it is always in effect on the built-in engine (the library's
  `ClipboardFollowsFocus` is on by default and the host leaves it so) — only the focused session can read what is copied locally, and only copies
  from that session reach the system clipboard; `xclip` / `xsel` in the same SSH session count as the same session. The sessions'
  selections (PRIMARY / SECONDARY / CLIPBOARD) are isolated from each other; copying across sessions goes through the local clipboard. The description of
  “Enable clipboard” says so.
- **The descriptions state the risks** (five languages): “Copy on selection” adds “Off by default: when on, whatever you copy on this computer can
  be pasted into X programs with a middle click”; “Restrict programs from SSH sessions” names the three restrictions and the cost — recommended
  when connecting to servers you do not fully trust, after which tools such as `xdotool` on those servers stop working.
- **Why the restriction is off by default and built-in only**: remote tools such as `xdotool` rely on XTEST, so the default keeps today's
  behaviour (a product decision); telling which connections came “from SSH” is only possible with the built-in engine (the host hands those
  connections in through `ServeAuthenticatedAsync`), VcXsrv cannot tell them apart. A per-connection trust level and “one display per session”
  are new features for later.
- **The “stay out of it” condition of “Start automatically for X11 forwarding” is narrower**: on Windows a listener behind `localhost:0` counts as
  “an X display already in use” only if it is a process in the current user's session — on a terminal server it may be another user's VcXsrv
  (often with `-ac`), and remote programs' windows, keyboard and clipboard used to end up on someone else's desktop; if it is not ours, the
  built-in engine starts automatically as usual (`:0` is taken, so the next free display number is picked). With a display address written into
  the config, or auto-start turned off, forwarding still follows `DISPLAY` / `localhost:0`.
- **Display number “Auto”**: when a number the probe found free turns out to be taken at start (someone got in between the probe and the bind, or
  a lock file / abstract name shows a use the probe cannot see), the next free number is tried, up to 4 times, with the state staying “Starting”
  meanwhile; a number set by hand still reports the error directly.
- **“Trusted (`-Y`)” on the connection** (not an X Server page setting, recorded here as well): the hint now explains that all forwarded sessions
  share one display — they can see each other's windows, read the clipboard and send input; the built-in X Server and most X servers on Windows
  support only trusted forwarding, and untrusted mode needs an external X server with the SECURITY extension plus a local `xauth`. When the
  display comes from the built-in X Server and the connection asks for untrusted mode, forwarding is not set up and a yellow line at the top of
  the terminal explains why, with two ways out (tick Trusted, or use an external X server). Since 2026-10-10 the built-in X Server supports
  both (an untrusted channel connects as an untrusted client) and that yellow line is gone; the hint is rewritten again to explain what each
  mode means — with an external X server untrusted mode still needs the SECURITY extension and a local `xauth`, and a few old programs do not
  work correctly when untrusted.

### 2026-10-10 Twelfth batch (X Server: the draft's new features)

Added with the built-in X server's new features (`xs_plan` F1–F30). Library behaviour: [X Server architecture](../xserver/design/architecture.md) §7
("Trust levels", "One display per SSH session", "One-window mode", "Added on 2026-10-10") and §10; interaction: the X Server part of
[interaction-and-ui-specs.md](interaction-and-ui-specs.md); symptoms and what to do: [X Server troubleshooting](../xserver/troubleshooting.md).
All apply to the built-in engine only (except "Window mode", which both engines share) and take effect the next time the X Server starts (except
"Show where X windows come from", which applies immediately). Where they sit on the settings page: the transparency and source items are in the
"Display" section (below window mode), the input method in "Keyboard", and "Restrict programs from SSH sessions", one display per session and the
TCP port in the new "Security" section (they all used to sit under "Keyboard").

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Window mode | Multiple windows | The existing five (`XServer.WindowMode`) | Used to affect VcXsrv only; now shared by both engines: on the built-in engine one large window / no title bar / fullscreen start it in one-window mode (the whole X desktop in one window, a remote window manager takes over), while multiple windows and rootless both give one window per program |
| One display per SSH session | off | on / off (`XServer.DisplayPerSession`) | X programs from different SSH sessions get separate displays: they cannot see each other's windows, read each other's keyboard or clipboard, or inject input; X programs on this computer keep using the shared display; each session costs a little memory and a thread. In one-window mode, closing one session's "X Desktop" window closes only that session's display |
| Use this computer's input method in X windows | **on** | on / off (`XServer.UseHostInputMethod`) | Text composed by the local input method is typed into X programs directly (no fcitx / ibus needed on the server); off sends keys to the remote side as they are, so its own input method works. This reverses the earlier "no XIM" decision: the built-in X Server is also an XIM input method server (`@im=velashell`); programs connected over XIM receive committed runs whole, the candidate window follows their insertion point, and on-the-spot programs draw the preedit themselves; `XMODIFIERS=@im=velashell` is set silently in the shell after connecting (an existing value is kept). The switch only applies to the built-in engine and takes effect after restarting the X Server |
| Show where X windows come from | **on** | on / off (`XServer.ShowWindowSource`) | Titles of windows forwarded over SSH (also the taskbar text) and tray icon tooltips read "user@host:22 — title", with the source first so nothing the remote side puts in the title can hide it; programs connected locally are not labelled. **Applies immediately**: open windows are retitled on the spot (the description used to say "applies to windows opened afterwards", but it actually needed an X Server restart). Windows that draw their own title bar show the source only in the taskbar |
| Transparent X windows | off | on / off (`XServer.CompositingManager`) | The server acts as compositing manager so GTK's rounded corners and shadows and Electron's transparent windows get alpha. Off by default: windows with alpha cost the system an extra compositing layer, not yet measured on real machines; while off, windows with alpha are shown opaque. No effect in the single-window modes |
| Also listen on a TCP port | off | on / off (`XServer.ListenTcpOnUnix`) | Only shown on Linux / macOS: by default only the Unix socket is open; when on, the port listens on 127.0.0.1 only, for programs that connect to this computer's loopback over TCP (containers using host networking); containers on a bridge network cannot reach it — on Linux mount `/tmp/.X11-unix` into them and use `DISPLAY=:N`. On Windows the port is always open (WSL and Cygwin programs only use TCP) and the setting is not shown |

- **Why these defaults**: the input method and source labels are on — without the former CJK users cannot type in remote programs, the latter guards
  against impersonation; the compositing manager is off (cost not measured); the TCP port being off on Linux / macOS is a behaviour change: loopback
  TCP used to be always open, and on those platforms that port was just one more way in for other local processes and users. The maintainer
  confirmed on 2026-10-10 that these defaults stay, with one display per SSH session off.
- **Descriptions follow the platform**: the built-in engine note and the display-number description say `DISPLAY=localhost:N` (TCP) on Windows
  and `DISPLAY=:N` (the Unix socket; TCP off by default) on Linux / macOS — they used to say `localhost:N` everywhere, which Linux users following
  them could not connect to. The display address in the start notification and the button tooltip likewise follows the listeners actually open.

### 2026-10-10 Thirteenth batch (splash screen)

A cold start takes seven or eight seconds before the main window appears, and nothing is on screen in between, so users think the app failed to
open. The splash screen shows as soon as the process is up (Windows, a native window from `Program.Main`) and fades out once the main window has
painted its first frame. Interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §14.5.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Splash screen | Classic | Classic / Terminal boot / Vela constellation / Prompt / Mascot / Off (`Appearance.SplashStyle`: `classic` / `terminal` / `constellation` / `prompt` / `mascot` / `none`) | Under Appearance → Window, with a live preview under the dropdown that uses the theme and accent being edited on the page; **takes effect on the next launch**. Windows only: the Dock bounce on macOS and the launch notification on Linux desktops already say "opening". Unknown values (hand-edited, written by a newer version) fall back to Classic |

- **Needed before the database opens**: the splash screen runs seconds before SonnetDB is up and cannot read the settings. Saving the settings
  mirrors the theme, accent, UI language and this item into `~/.velashell/startup.appearance` (a few `key=value` lines), and every launch aligns it
  once with the settings it read (no write when nothing changed) — the same approach as `render.mode`. The item takes part in cloud sync: it is
  a preference, not machine state.
- `VELASHELL_NO_SPLASH=1` turns the splash screen off (automated tests, screen recording, troubleshooting) and wins over the setting.

### 2026-10-19 Fourteenth batch (masking secrets in recordings and session logs)

Session logs and session recordings store the raw terminal output: the passwords and tokens printed by `cat .env`, `kubectl get secret -o yaml`
or `env | grep TOKEN` land on disk verbatim. A masking stage (`SessionOutputMasker`) now sits between the hook point both use
(`SshTerminalBridge.DataReceived`) and the recorders; the display path never goes through it.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Mask passwords and tokens in recordings and session logs | On | On / Off (`Security.MaskSecretsInCaptures`) | Under Security Audit → Session recording, below automatic recording; **applies to recordings and session logs started afterward**. The session log switch lives under General; masking covers both |

- **What is masked** (case-insensitive, erring on the side of masking): in `name=value` / `name: value` (JSON and YAML forms included; `grep --color` color sequences between the name and the separator do not get in the way) where the name contains password / passwd / passphrase / secret / token / api key / access key / private key / credential, only the value; the credential after `Bearer` / `Basic`; the body of PEM private key blocks (the BEGIN / END lines stay); tokens with a recognizable prefix (AWS `AKIA…` / `ASIA…`, GitHub `ghp_…` and friends, Slack `xox?-…`, `sk-…`). All become `***`. The current directory in `PWD=` is not a secret.
- **Bytes and lines**: every byte maps to one Latin-1 character before matching; rules and replacements are ASCII, so UTF-8 and GBK round-trip unchanged. Only complete lines are matched; the trailing incomplete line is held back only when it stops in the middle of a value (or inside a private key block), up to 4 KB, while `Password: ` prompts waiting for input and full-screen programs that never print a newline pass through immediately. Whatever is held is masked and released when recording stops.
- Not done: user-defined rules — the rule table now exists (the twenty-third batch, "Output Rules") but masking is not wired to it yet, see `feature-plan.md`; bare values that are neither named nor prefixed (such as a lone base64 line) are not recognized.

**2026-10-25 addendum** (the setting is unchanged; what is masked and how it is recognized changed):

- **Control sequences are stripped before matching**: each line is first projected to its visible text — CSI and short ESC sequences take no position, while the start and the ST of OSC / DCS / SOS / PM / APC each keep one ESC as a boundary (text inside a title or an OSC 8 hyperlink still goes through the rules but never joins the text around it into one value);
  broken sequences or ones longer than 256 bytes stay visible. All rules run on the visible text and positions are mapped back to the original. Keywords, tokens and values cut apart by grep highlighting, or values colored by jq / bat, are recognized all the same; control sequences inside a masked span are kept right after the `***`, so replay keeps colors and cursor state intact.
  Context truncation never lands inside a control sequence.
- **name=value / name: value**: a keyword in the name has to be a whole word — not when it is followed directly by a letter of the same case (`max_tokens`, `tokenizers`, `passwordless`, `SECRETARY` are not masked), while the next camel-case word, a digit, a separator or the end of the name count as boundaries;
  a plural `s` still counts except for token (`credentials.json`, `client_secrets.json` are still masked). A host name in `user@host` is not taken as a key; with no whitespace after the colon, a value starting with `~` or `/` does not count (`host:~#` prompts, `host:/path`).
  New keys without password-like words: names ending in `.key` (`tls.key`), names containing `key-data` (kubeconfig `client-key-data`), `.dockerconfigjson` / `.dockercfg`, and Docker `config.json`'s `"auth"` (only a quoted name with a string value).
  Unquoted values: `name=value` still stops at whitespace; `name: value` (whitespace after the colon: YAML, HTTP headers, `kubectl describe`) is masked to the end of the line, stopping before the next key on the same line (an identifier directly followed by `=` or ": ") or a `#` comment, with an adjacent comma or semicolon left to the next key.
  An escaped JSON string (`\"…\"`) cut off by a bare outer quote is masked up to that quote, and an empty value `\"\"` no longer masks a backslash.
- **Others**: `Basic` only takes base64 whose length is a multiple of 4, that decodes, and that contains a colon (`Basic configuration` is left alone); `Bearer` does not take a purely alphabetic English word shorter than 20 characters (`Bearer authentication` is left alone).
  Prefixed tokens add `github_pat_`. `.netrc` is matched in lower case only: a one-line `machine … login … password secret`, and, when split across lines, a `password secret` line on its own at the start of a line.
- **Private key blocks**: PEM and OpenPGP (`PGP PRIVATE KEY BLOCK`). Inside a multi-line block only base64 body lines become `***` (line-number and file-name prefixes added by `cat -n`, `nl`, `grep -n` / `grep -A`, color sequences and line endings are kept);
  blank lines and the `Proc-Type` / `DEK-Info` / `Version` / `Comment` / `Hash` / `Charset` / `MessageID` header lines go through the usual rules and the block continues; a line that cannot belong to a key block (an echoed BEGIN line, truncated output, a prompt) ends the block —
  before, everything after entering a block was masked line by line with its control sequences erased, so echoing a single BEGIN line blinded the rest of a recording.
- **What is held back**: besides "the trailing incomplete line is held only in the middle of a value (or inside a private key block)", the value of `name: value` and a netrc password line only end at the end of the line, so they are held until then;
  an unfinished line inside a key block is held only while it could still be a body / header / END line, and a prompt containing `@ ~ $ # > %` and the like is released at once.
- Still masked by mistake after the whole-word check: `token_type: Bearer`, `token_endpoint: https://…`, `secretName: tls-cert`, `token_count: 12`.

### 2026-10-24 Fifteenth batch (tab bar position shown again)

R-05 was withdrawn because the docking layer could only put horizontal tab strips at the top of a pane, so `TabBarPosition` was stored but
never read. Now that the docking layer supports the bottom, it is back under Appearance → Window, between "Sidebar position" and "Show Tabs in
Multiple Rows". Interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §4B.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Tab bar position | Top | Top / Bottom (`Appearance.TabBarPosition`: `top` / `bottom`) | Bottom: the horizontal tab strip and its 1px divider go below the content area; tabs look the same and the active accent line stays on the tab's top edge (which, at the bottom, is the side facing the content). Only `bottom` counts; any other value means top |

- **One value for the whole workspace**, like "Show tabs in multiple rows": every pane changes together (`DockWorkspace.TabsAtBottom` in the
  docking layer).
- **Horizontal strips only**: docking a pane's tabs left / right from the tab right-click menu ("Tab Position") is a per-pane choice this
  setting does not rewrite; when such a pane goes back to horizontal it lands on the side chosen here, and the menu's horizontal item reads
  "Top" or "Bottom" to match. The dropdown's two options share their text with those menu items.
- **Previews immediately**: changing the dropdown moves the strips in the main window right away (through the appearance preview), and
  cancelling the settings rolls it back with the preview; at startup it is applied before the main window's first frame. "Show tabs in multiple
  rows" still takes effect on save.
- "Show tabs in multiple rows" works on bottom horizontal strips too (see the eighth batch).

### 2026-10-24 Sixteenth batch (restore split layout)

"Restore sessions" (`General.RestoreSessionsOnStartup`, long-standing and on by default) reconnects the connections that were online at the
last exit one by one into a single pane, without the splits. The layout serialization behind named layout templates lets it restore the last
split layout too, as a subordinate option. Interaction: the layout templates section and §14 of
[interaction-and-ui-specs.md](interaction-and-ui-specs.md).

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Restore split layout | Off | On / Off (`General.RestoreLayoutOnStartup`) | Under General → Startup & Window, right after "Restore sessions", and disabled while that is off. When on, the panes' orientation and proportions, each pane's tab order, the selected tabs and the pinned state come back too |

- **Consumers**: `MainWindow.RestoreSessionsAsync` — when on and a layout was recorded, it reconnects and arranges by that layout (through the
  same restorer as layout templates, at most 6 handshakes at once, like opening a whole group); with no recorded layout it falls back to
  reconnecting `LastOpenProfileIds` one by one. `MainWindow.PersistWindowBounds` (on exit) — when on it records a layout as well; when off it
  clears the old one, so that turning the option back on some day, followed by an unclean exit, does not restore a layout from long ago.
  `LastOpenProfileIds` is written as before.
- **What is recorded**: only the tabs online at exit, the same rule as `LastOpenProfileIds`, plus local terminals and dual-remote file tabs.
  The exit path does not query the store; whatever no longer matches at the next start (usually temporary profiles that were never saved, such
  as quick connections) is skipped silently.
- **Not in the settings file**: the exit layout lives in the SonnetDB `layout_session` collection (one `last` document), named layout
  templates in the `layout_templates` collection, one document per template; neither travels with settings export, and neither holds passwords;
  named templates can travel with cloud sync since 2026-10-25 (a separate file, see the twenty-second batch), while the exit layout is still not synced.
- It defaults to off so that the existing default behavior of "Restore sessions" does not change; that option itself is untouched and still on
  by default.

### 2026-10-24 Seventeenth batch (settings import / export by category, audit log completed)

The "selective export" that C-08 left open. Export used to write the whole `AppSettings` (with only the proxy password blanked): the proxy's
address and account, the alert webhook URL (a DingTalk / Feishu / WeCom / Slack bot URL is itself a token) and the external-launch trusted
targets all went into the file; import overwrote everything, so `{}` or a wrongly picked `package.json` was taken in as an all-defaults set of
settings. Interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §14 (General page) and §15 (audit log). No new settings.

**Categories** (`Core/Models/SettingsCategories`, cut along the actual blocks of `AppSettings`, with loose root-level fields assigned to the
page they appear on; the order follows the left navigation, with the message center right after General; a reflection test pins down that
every root-level property belongs to exactly one category):

| Category | Block | Root-level fields assigned to it | Sensitive |
| --- | --- | --- | --- |
| General | `general` | `language`, `defaultPort` | |
| Message center | `notifications` | — | |
| Appearance | `appearance` | `theme`, `accentColor` | |
| Terminal | `terminalBehavior` | `terminalFont`, `terminalFontSize`, `scrollbackLines`, `terminalType`, `terminalEncoding` | |
| Key Management | `keys` | — | |
| Shortcuts | `shortcuts` | — | |
| File Transfer | `transfer` | — | |
| Security & Audit | `security` | — | ✅ |
| Proxy | `proxy` | — | ✅ |
| X Server | `xServer` | — | |

- **Sensitive categories start unticked**: export and import share one checklist; Security & Audit and Proxy carry a "Sensitive" tag and are
  unticked by default, and ticking one shows, in the warning color, what it carries (export: the proxy's address, port and user name; the
  host-key policy, the external-launch confirmation and trusted targets, the alert switches) / what it will change (import: the proxy every
  outbound connection uses from then on; host-key confirmation, the external-launch confirmation and its trusted targets). Import starts
  unticked too — a file from someone else should not quietly switch off external-launch confirmation or redirect outbound traffic.
- **Secrets are never written**: the proxy password and the alert webhook URL are left out whether or not their category is ticked. There is
  no encrypted export: those two are the only real secrets in the settings, and moving secrets to another machine is what cloud sync's
  end-to-end passphrase is for. Since 2026-10-25 cloud sync without an end-to-end passphrase leaves these two out as well, and a pull keeps the
  local values by the same rules as an import; an encrypted payload with a passphrase still carries them.
- **Runtime state is not exported**: window size and position and the last open sessions (`lastOpenProfileIds`) are not written, and import
  keeps the local ones; one-time migration markers do travel (without them an import would migrate values the user already chose a second time).
- **File shape**: still the `AppSettings` JSON, keeping only the selected blocks, plus an export header `velaShellExport` (format
  `velashell-settings`, version 2, export time, categories). It is not wrapped: an older version importing a wrapped file would reset
  everything to defaults, whereas with this shape it only resets the categories the file does not carry.
- **Import**: the categories a file carries are recognized by its JSON keys (an old whole-settings export = all of them); only those carried
  and ticked are overwritten, everything else stays; fields a block leaves out take the type's defaults (as before). When the file has no proxy
  password, the local password is kept only if the proxy's type, host, port and user name all match the local ones — a file pointing at another
  proxy does not get it, or the local password would be handed to whoever wrote the file; the webhook URL always stays local. Values the file
  does spell out (a webhook URL from an older export) are used. JSON in which no category is recognized is reported as "not a settings file"
  and nothing changes. Imported values only land in the settings page and are written when "Save" is clicked.
- Export now takes the settings page's current values (unsaved edits included); before, the grouped options carried unsaved edits while the
  root-level fields were whatever was last loaded.

**Audit log** (interaction in §15):

- The session repository, the settings service and the shared-credential repository each gain an auditing decorator
  (`Infrastructure/Audit/`), with three new event categories: `profile` (`profile-created` / `profile-updated` / `profile-deleted`),
  `settings` (`settings-changed`) and `credential` (`credential-created` / `-updated` / `-deleted`).
- **Only which part changed is recorded, never the values**: "settings changed" uses the same categories as above and records which ones
  changed; runtime state and migration markers do not count, so writing the window size on exit or cloud sync pulling identical data leaves no
  entry. Profiles record block names — name / address / login credentials / other settings; the last-connected time, directory bookmarks and
  pinning are not changes.
- Settings changes catch quite a few everyday actions: changing the font size with Ctrl+wheel in a terminal, switching columns or showing hidden
  files in the file panel, ticking "Don't ask again" for an external launch each leave an entry.
- Storage and retention are unchanged: still `audit_log`, pruned at startup by "Audit log retention days" (ninth batch). The viewer gains a
  time-range filter and CSV export (read-only, not settings).

### 2026-10-24 Eighteenth batch (retry failed transfers wired up)

`TransferOptions.TransferMaxRetries` had been persisted with no consumer since R-09, and the settings page had dropped it. It was thought to
depend on persisting the transfer queue — "a retry has to know what to retry"; on inspection that does not hold: an in-process retry needs
nothing beyond the item being transferred, with nothing on disk, so it was wired up on its own. Interaction:
[interaction-and-ui-specs.md](interaction-and-ui-specs.md) §9.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Retry failed transfers | 3 | 0–10 times (`Transfer.TransferMaxRetries`, 0 = never) | Under File Transfer → Conflict handling, after "Resume interrupted transfers" (and its subordinate "Clean up partial files"); out-of-range values in the settings file are clamped back by `Normalize` |

- **Which failures are retried** (`Core/Sftp/TransferRetryPolicy`, judged by Core's neutral exception types, never by message text): temporary
  failures — a dropped connection, a timeout, a closed channel, a lost FTP data connection — are retried on the same session; permission denied,
  a missing path, a full disk, a server refusing the write, authentication / host-key failures, an unreadable local file and user cancellation
  are not. Anything unrecognized is not retried — missing a temporary error at worst fails like before, while retrying a permanent one a few
  rounds makes the user wait tens of seconds longer to see the real cause.
- **Back-off**: 1 second, doubling, capped at 30 seconds; while waiting the row goes back to its waiting state with "Retry n/N" on the right,
  and it keeps its transfer slot. The wait honours the batch's cancellation, so "Cancel all" still stops it.
- **Where it continues**: with "Resume interrupted transfers" on and bytes actually moved in that attempt, the file continues from what it
  had itself transferred (the starting point is verified as usual); if not a single byte moved it retries from the original start; with resume
  off the whole file starts over. When the retries run out, the error reads "Still failing after N retries: …".
- **Not retried**: same-session copies (a single server-side operation); and nothing is retried on the old session once the panel is torn
  down (tab closed, session dropped) — in batches started from a file panel such failures turn into "Paused", and "Resume" continues them on a
  new connection (the transfer queue, stored in the SonnetDB `transfer_queue` collection, restored as paused after a restart without
  connecting; not a setting).
- After settings are saved, the terminal sidebar's file panel and open single- / dual-pane file tabs pick up the new value together (file tabs
  since 2026-10-25; before, the tab had to be reopened).
- Resumed downloads and dual-pane relays read a remote file stream that throws the neutral exceptions too (since 2026-10-25, `TranslatingSftpStream`), so a dropped
  connection is retried as a temporary failure; before, that stream was the SSH library's raw one and its exceptions went unrecognized and were not retried.

### 2026-10-24 Nineteenth batch (download updates automatically wired up)

`General.AutoDownloadUpdates` used to default to `true` with no consumer and no UI (R-03). About → Check for updates already went on to
download (download, SHA-256 verification, the ready snapshot and the re-check before swapping versions were all there); what was missing was
the startup-check path. Behavior: [message-center-and-feed.md](message-center-and-feed.md) §4.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Download updates automatically | **Off** (used to be on) | On / Off (`General.AutoDownloadUpdates`) | Under General, right next to "Check for updates on startup", and disabled while that is off; **shown only on installations that can update in place** (`CanSelfUpdate`: a writable install directory on Windows / macOS) — on Linux, the Store version or a read-only directory it would do nothing, and showing it there would be one more case of the UI lying |

- **Off by default, migrated once**: it adds a download of tens of MB on top of the plain check. A stored `true` can only be the old default
  (there was nowhere to choose it), so `Normalize` turns it off once, keyed on `General.AutoDownloadUpdatesOptInMigrated`, and follows the
  user from then on (the same approach as recording and fingerprint blocking). The marker is registered as a migration field, so the first save
  does not log a spurious "General" settings change (seventeenth batch).
- **Depends on "Check for updates on startup"**: there is only something to download after that check, so it is disabled rather than
  independent. With the check on, it runs if "Announce available updates" or this option is on — before, the check was skipped whenever
  announcements were off; now, with announcements off and automatic download on, it still checks and downloads, it just posts nothing.
- **No restart once downloaded**: a "ready to install" message is posted (clicking it goes to About, which has "Restart and update"); a
  failed download pops up nothing, falls back to the plain "update available" message and is retried at the next start. About → Check for
  updates downloads anyway and is not governed by this switch.
- Along the way: if the settings snapshot is not cached yet when the startup round checks for updates, the settings are now actually read
  first — the old `new()` fallback could treat someone who had turned "Check for updates on startup" off as the default (on) and send a request.

### 2026-10-24 Twentieth batch (Security & Audit page: known_hosts interop and trusted host CAs)

The SSH library has long implemented host certificates and `@cert-authority` ([03-key-exchange.md](../ssh/spec/03-key-exchange.md) §5.5),
but the host's trust store had no notion of a trusted CA, and trusted hosts could only be listed and deleted. The Security & Audit page gains
two OpenSSH interop items in the "Trusted hosts" section, followed by a new "Trusted host CAs" section. Interaction:
[interaction-and-ui-specs.md](interaction-and-ui-specs.md), the Security & Audit row of §14 and §15.

| Setting / action | Default | Values | Notes |
| --- | --- | --- | --- |
| OpenSSH known_hosts | — | "Import…" and "Export…" buttons | Import: multiple hosts, `[host]:port` and negation on one line are understood; `\|1\|` hashed lines are matched one by one against saved SSH / SFTP connections and trusted hosts, and the unmatched ones are counted; plain lines with wildcards are skipped and counted; a different key already stored for the same type is not overwritten (counted as a conflict); `@cert-authority` / `@revoked` lines go to Trusted host CAs below; plain lines for revoked keys are not imported; unknown markers skip the whole line. **It only adds, never changes.** Export: OpenSSH format; old records that have only a fingerprint and no stored public key are counted separately (filled in the next time the host is trusted on connect) |
| Hash host names on export | Off | On / Off | Like `HashKnownHosts yes`: host key lines are written as `\|1\|…`, while the patterns on `@cert-authority` / `@revoked` lines are written as-is. **Not persisted**; it is off again every time the settings open |
| Trusted host CAs | Empty | The list of CA / revocation records (tagged "CA" or "Revoked", deletable) + an add form (host name patterns + the CA's `.pub` line, or a whole `@cert-authority` / `@revoked` line copied from known_hosts, with an optional comment) | Stored in the SonnetDB `known_host_markers` collection and applied by host name pattern. A matching host presenting a valid certificate signed by one of these CAs is let through without being recorded among trusted hosts; a host covered by a CA that presents an unendorsed key or an invalid certificate is rejected outright, never falling back to first-connection confirmation — whatever "Require manual fingerprint confirmation on first connect" and "Block and alert on fingerprint change" say |

- **Applies immediately, not tied to "Save"**: the same approach as the trusted-host list — this is trust data, not a preference. Importing
  adds trusted-host records, and that table reloads right after the import.
- **Revocations come only from import or a pasted full line**: pasting an `@revoked …` line into the form creates a revocation record; there
  is no "trust this CA" button in the confirmation dialog — trusting a CA from the certificate a server presents is trusting blindly.
- Trusted CAs are not read from `~/.ssh/known_hosts` on every connection; they live only in the host's own trust store. To share them with
  the command-line ssh, import once.
- Trusting a host now also records its public key (only the fingerprint was stored before); older records get it the next time they are
  trusted on connect, and only then can the export write a host key line for them.

### 2026-10-25 Twenty-first batch (first-connection fingerprint confirmation on by default)

The **default of `Security.ConfirmFirstFingerprint` changes from off to on** (host side: the `plan.md` section of the same day). Interaction:
[interaction-and-ui-specs.md](interaction-and-ui-specs.md), the host fingerprint confirmation item in §13.2.

- **Why**: the user manual has always said that the first connection asks you to check the fingerprint. The trouble with TOFU is that the
  first time is exactly when checking matters most — with someone on the path, the key recorded is the impostor's, and every later
  connection "verifies" against it. On by default matches OpenSSH's `StrictHostKeyChecking=ask`.
- **One-time migration**: a stored `false` cannot tell the old default from a user's choice, so it follows the same pattern as the
  fingerprint-change block (fifth batch) — switch it on once, stamp `Security.FirstFingerprintDefaultMigrated`, and respect the user from
  then on. The flag is registered as a migration field (`SettingsCategories.MigrationFields`), so the audit does not count it as a settings
  change while export and sync still carry it (seventeenth batch). On pull every migration flag is OR-ed with the local one, so a
  migration done on either side is not run again (since 2026-10-31; before, a payload pushed by an older version that did not know the
  flag came in as `false` and the migration was redone on every sync round).
- **Behavior**: connections from anywhere in the UI, test connection and connection diagnostics, standalone SFTP, tunnels started by hand
  or by "Establish when the app starts", plugin `open_session`, "Resume" in the transfer queue and external launches (after their
  confirmation) all show the "Host Trust" dialog for an unseen host, and reject when there is no main window to show it on. Automatic
  reconnects (the terminal countdown, the batch after waking from sleep, a tunnel's automatic redial) reject an unseen host without a
  dialog and record a `hostkey-rejected` alert — with the old default off they quietly recorded it on first use. Hosts trusted "once"
  are still let through for the rest of the run; hosts covered by a trusted host CA are unaffected (twentieth batch).
- The switch keeps its place and wording; its description "When off, the fingerprint is recorded automatically on first connect (TOFU)"
  still holds.

### 2026-10-25 Twenty-second batch (layout templates sync through the cloud)

The named layout templates of the sixteenth batch lived only on the local machine, neither in Gist sync nor in settings export. This batch lets them travel with cloud sync, and adds export / import to the manage window (not settings). Interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §4C.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Sync scope → Layout templates | On | On / Off (`SyncSettings.SyncLayoutTemplates`) | Under "Cloud Sync → Sync scope", after snippets; stored locally in `app_config/sync`, not in the sync payload. Description: "A separate file merged template by template; templates only record connection IDs, so another device needs the same connections (synced with connection profiles) to open them" |

- **Consumer**: `GistSyncService.LayoutTemplatesInScope` (push, pull, reconcile and restoring a past version). A device with it off neither reads nor writes `velashell-layouts.json` in the Gist; not writing is not deleting, so other devices' templates stay.
- **On by default**, like app settings, connection profiles and snippets: it goes through the Gist the user already has on, not a new outbound call; templates hold no secrets and are no more sensitive than the connection profiles already syncing (tab titles are often `user@host`, and profiles carry host and user names anyway).
  Existing sync configurations lack the field and read as on, so the first sync after upgrading pushes the existing templates (`PRIVACY.md` says so).
- **Why a separate file**: the main payload is pushed whole, so a new section inside it would be dropped by an older client when deserializing and wiped from the cloud when it pushes back (that is how §158's `HiddenBuiltInIds` were lost); older clients only read and write the main file and never touch this one.
  Merging per template id with deletions kept as 180-day tombstones is described in §4C of the interaction specs; templates, nodes and tabs all carry `JsonExtensionData`, so fields added by newer versions round-trip unchanged.

### 2026-10-25 Twenty-third batch (output rules: custom highlights and auto-answer)

The built-in semantic highlighting is 7 compile-time regular expressions (`SemanticMatcher`) users cannot extend, and nothing in the code base did "output matches → send something". This batch adds one rule table shared by highlights and triggers / auto-answer, plus an "Output Rules" page in the settings window (after "Terminal", 15 pages in total). Interaction: [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §5.2, §5.2.3 and §14.6.

| Setting | Default | Values | Notes |
| --- | --- | --- | --- |
| Output rule table | Empty | `TerminalBehavior.OutputRules`, at most 64 | The "Output Rules" page of the settings window; applies to open tabs on save (`TerminalSettingsApplier`); exported / imported and audited with the "Terminal" category, synced with app settings |
| Rule: action | Highlight | `highlight` / `respond` | Unknown values do not take effect and are flagged on the page |
| Rule: scope | Global | `global` / `group` / `profile` + `scopeId` | Local terminals take only global rules |
| Rule: pattern / case sensitive / enabled | — / No / Yes | At most 512 characters, linear-time engine | Backreferences, lookarounds, atomic groups and conditionals are refused |
| Highlight: text color / background / bold | No change / No change / No | Palette index 0–15 or empty | At least one; out-of-range indices load as "no change" |
| Answer: text / then press Enter | Empty / Yes | At most 128 characters, one line of visible text | Empty text + Enter = just press Enter; rules that would match a password prompt are refused |
| Answer: cooldown / per-minute limit | 2 s / 6 | 1–3600 / 1–30 | Clamped on load |
| Answer: forward to sync channels | No | Yes / No | |

- **It lives in the "Terminal" block**: fields are strings / nullable numbers (the document store's JSON has no enum converters); export / import and the audit follow the "Terminal" category and cloud sync follows app settings — it travels with the user like quick commands, with no category of its own in `SettingsCategories`.
  `Normalize` clamps numbers, drops empty entries and truncates past 64; rules failing validation are not deleted but skipped by the compiler and flagged on the page. "Restore defaults" clears the table too.
- **Rules sync with the settings, on the condition that the table never holds a password**: an auto-answer rule whose pattern literally mentions password-like words (password / passphrase / PIN / OTP / token / secret and their CJK equivalents), or that would match one of 15 common password prompts,
  cannot be set to auto-answer (`OutputRuleSecretGuard`, erring on the side of refusing — patterns without keywords such as `:\s*$` or `.*` are caught too); for such a rule "Done" stays disabled in the editor, and a draft like that is dropped when the window's "Save" is clicked. Highlight rules are not restricted (marking "password" in red in a log is legitimate).
- **Trigger rules** (`Terminal/Semantics/OutputTriggerScanner`): remote output only; only the logical line the cursor is on, main screen, and not one carrying OSC 133 prompt marks (since 2026-10-31); no answer when there has been local input since the last Enter; wait for 300 ms of quiet; one answer per line; the cooldown postpones rather than drops, and the per-minute limit pauses for a minute;
  not forwarded to sync input channels by default. Details in §5.2.3 of the interaction specs.
- **Not done**: wiring recording and log masking (fourteenth batch) to this table so users can add their own masking rules — left in `feature-plan.md`; multi-step expect dialogs, variables and scoping by tab / marker are not done either.
