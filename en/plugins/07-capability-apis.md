# 07 · Capability APIs

Everything a plugin can do goes through a capability domain. Each domain maps to an SDK proxy interface on the plugin side, an RPC method family (05), and a host capability service, where the permission `Demand` bridges to an existing subsystem. This document defines the apiLevel 1 domain list and key signatures, in C# from the SDK perspective.

Design discipline:

- Every method documents its required permission. A method without permission is absent from the list and therefore absent from the protocol.
- Everything is asynchronous and cancellable. Long-running operations may include `IProgress<T>`.
- Paths, sessions, and other identifiers use opaque IDs issued by the host. Plugins never access internal objects.
- Bridge implementations **depend only on neutral interfaces in `Core.*`**, such as `Core.Ssh.ISftpClient`, continuing the rule that “library types do not leave Infrastructure”.

## 1. vela.sessions, Sessions

Permissions: `sessions.observe.basic` (redacted list) / `sessions.read` (full) / `sessions.create` (start).

```csharp
public interface ISessions
{
    Task<SessionInfo[]> ListAsync(CancellationToken ct);          // Fields are redacted according to the permission level
    IAsyncEnumerable<SessionEvent> WatchAsync(CancellationToken ct); // Connect/disconnect/reconnect events
    Task<string> ConnectAsync(string savedProfileId, CancellationToken ct); // sessions.create
}
```

## 2. vela.remoteFs, Remote Files (Core Scenarios S1/S2/S7)

Permissions: `remote.files.read` / `remote.files.write`, with scope supporting sessions and path prefixes. The implementation bridges the existing SFTP wrapper. Note the existing semantics: `GetAttributes` returns null for a nonexistent path and does not use an exception to signal existence. The SDK's `StatAsync` likewise returns a nullable value.

```csharp
public interface IRemoteFs
{
    Task<RemoteEntry[]> ListAsync(string sessionId, string path, CancellationToken ct);
    Task<RemoteEntry?> StatAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenReadAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string sessionId, string path, WriteMode mode, CancellationToken ct);
    Task DownloadAsync(string sessionId, string remotePath, string localPath,   // localPath also requires fs.local.write
                       IProgress<TransferProgress>? progress, CancellationToken ct);
    Task UploadAsync(...);
    Task DeleteAsync/RenameAsync/MkdirAsync(...);
    IAsyncEnumerable<RemoteChange> PollWatchAsync(string sessionId, string path,
                       TimeSpan interval, CancellationToken ct);  // Polling watch, minimum interval 2s (protects the remote)
}
```

Host-side protections: a per-plugin limit on concurrent SFTP operations (4 by default), staggering of bandwidth use against the user's foreground transfers (plugin traffic enters a low-priority queue), and throttled progress callbacks (≥100 ms; the full 100% frame is sent once, only after the transfer succeeds, and never on failure). This incorporates the historical lesson of large-transfer stalls: plugin traffic must not flood the UI scheduler.

Link fields on entries (SDK 2.1.3, now wired in the host): the `RemoteFileEntry` values returned by `ListAsync` / `StatAsync` carry `IsSymbolicLink` (decided by lstat) and `LinkTarget` (the readlink text verbatim, possibly relative); `IsDirectory` / `Size` / `LastModified` describe the link **target**, and a dangling link keeps the link's own attributes. Both in-process and isolated modes carry them (in the JSON of `fs/list` and `fs/stat`), and an older host or older data without the two fields reads as "not a link". See [SDK §3.3d](../sdk/sdk-reference.md#33d-file-capabilities-for-plugin-protocols-sdk-213). Transfers explicitly refused before registration are covered by `PluginTransferRejectedException` in §2.1.

### 2.1 Independent upload jobs (SDK 2.1.1)

`IRemoteFsApi` adds `SupportsTransferOperations` and `StartUploadAsync` / `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` / `ResumeTransferAsync`; see [SDK §3.3b](../sdk/sdk-reference.md#33b-independent-upload-jobs-sdk-211) for the complete contract. Unlike synchronous transfers above, start returns a snapshot while the host continues streaming content. A query/wait deadline does not cancel the transfer, and the legacy two-hour RPC limit is not an execution deadline. Refusals made before a job is registered (source missing, Windows sharing/lock conflicts, quota full) come back as `PluginTransferRejectedException` (SDK 2.1.3, implemented by the host; see [SDK §3.3e](../sdk/sdk-reference.md#33e-transfers-refused-before-they-start-sdk-213)). Insufficient snapshot space remains a registered job's `Failed / insufficient-space` snapshot. Plugin protocols get the optional `IProtocolLastWriteTime` (setting modification times) and `IProtocolChecksum` (server-provided digests only, `RemoteChecksumAlgorithm`) and `RemoteFileEntry.IsSymbolicLink` / `LinkTarget` in SDK 2.1.3, now wired in the host ([SDK §3.3d](../sdk/sdk-reference.md#33d-file-capabilities-for-plugin-protocols-sdk-213)).

The current host permits four concurrent transfers shared by uploads and downloads and 32 active/queued jobs per plugin, retaining up to 1024 lifetime idempotency receipts. Quota exhaustion rejects new jobs; reacquiring an existing id is not blocked by the active-job quota. These are resource/concurrency budgets, not file-size limits.

Local source parent-directory aliases are resolved component by component. Approval and snapshots show the resolved path. Linked leaf files, unsupported handle verification and unresolvable paths are explicitly rejected. On Windows only name-surrogate reparse points (symbolic links, directory junctions, mount points) count as links; filter-type tags such as OneDrive cloud-file placeholders do not. After opening, the handle is verified by **file identity** to be the file at that path (Windows volume serial + file id, `FILE_ID_INFO`, with both sides falling back to `BY_HANDLE_FILE_INFORMATION` when it is unavailable; Linux statx dev + inode; macOS fstat st_dev + st_ino), and the side being compared is reached without following any link: Windows pins each ancestor directory and allows no name surrogates; Linux walks `openat(O_NOFOLLOW)` from `/`; macOS uses `fstatat(AT_SYMLINK_NOFOLLOW_ANY)`. That is the real defence. Mapped drives, subst drives, other hard-link names and case / normalization variants are accepted as the same file, and an identity that cannot be determined is always refused (since 2026-10-25; before, the handle's final path was compared character by character). Repeated starts are deduplicated by original request arguments, never reopening a file because an alias changed.

The new upload API uses dedicated trusted host confirmation and a per-job authorization lease. It displays the plugin, actual session, local source, **file size**, remote destination and overwrite policy, with only Allow This Job or Deny; terminal/session-opening grants never carry over. Missing gate, prompt or desktop window means rejection, and waiting for confirmation does not block the start RPC. Plugin permission revocation cancels pending/active uploads, prevents subsequent commit and withholds new job metadata. An uncertain result after commit has begun still means `Unknown`, not a rollback promise. AI `Bypass` skips only plugin approval, never this host confirmation.

Progress uses 64-bit byte counters, monotonic snapshot sequence, execution generations and explicit terminal states. Full bytes can still mean remote finalization; only `Succeeded` means success. The remote temporary file is `.vela-up-<16 hex digits>` in the destination's directory. Before writing, a commit preflight fails a commit that is bound to be refused (overwrite without server `posix-rename`, or no-overwrite with an existing destination; `ErrorCode` `not-supported` / `commit-rejected` respectively) instead of transferring the whole file first; the commit itself remains the only authority. The temporary file of an overwriting upload is created with the replaced regular file's 0777 permission bits from the start (the server applies its umask as usual; 0600 when the destination exists but reports no mode; the server default when the destination cannot be read, with the commit still checking), so it is never wider than the original while being written; setuid / setgid / sticky bits and the owner are still applied by the commit. An overwrite commit carries the replaced regular file's permission bits over to the new file, the owner on a best-effort basis, and setuid / setgid only once owner and group match; if that cannot be done without widening the permissions, the commit is refused and the destination left untouched. Cancellation targets a generation; resume requires fresh approval and validation of connection identity, source object, checkpoint and final commit outcome. Never automatically replay an uncertain side effect. New methods explicitly default to unsupported: capability false must not fall back to synchronous large-file upload. `ErrorCode` `insufficient-space`: the volume holding the Linux / macOS source snapshot has too little space (see [Automation and AI §1.6](11-automation-and-ai.md)). The stop grace period is 30 seconds; a disconnect ends as `Failed` / `session-disconnected`, meaning the write is confirmed stopped and the destination unwritten. This contract does not implement the universal permission Broker; interpret actual authorization and lifecycle boundaries according to the host implementation.

To withdraw only transfers whose approval has not taken effect, use conditional withdrawal (SDK 2.1.4): `SupportsTransferWithdraw` (default false, independent of the upload and download capabilities) and `WithdrawPendingTransferAsync(transferId, expectedGeneration, ct)`. The host decides and withdraws in one step, first come first served against the approval taking effect; the outcomes are `PastApproval` (approval already in effect, nothing changed) / `Withdrawn` (withdrawn, ends `Cancelled` with nothing written) / `AlreadyEnded` (ended for another reason before approval). Unsupported hosts throw `NotSupportedException` rather than degrading into `CancelTransferAsync`. The current host supports it for uploads and downloads; see [SDK §3.3f](../sdk/sdk-reference.md#33f-conditional-withdrawal-sdk-214).

### 2.2 Independent download jobs (SDK 2.1.2)

`SupportsDownloadOperations` (default false) and `StartDownloadAsync(sessionId, remotePath, localPath, RemoteDownloadOptions, ct)` provide a separate download entry; unsupported hosts reject rather than fall back to synchronous downloading. Reuse §2.1 get/wait/cancel and shared budgets: both directions together have four actual transfers, 32 active/queued jobs and 1024 lifetime receipts per plugin. Request/wait cancellation does not cancel work; `Timeout=null` has no default execution deadline, and downloads have `CanResume=false`. See [SDK §3.3c](../sdk/sdk-reference.md#33c-independent-download-jobs-sdk-212) for direction/phases, 64-bit progress and stop confirmation.

Downloads use a **per-job host lease separate from uploads**, covering remote read and local write. Every job confirms plugin, actual session, source, final destination and default `Overwrite=false`; replacement requires explicit approval. A save picker and AI Bypass cannot skip this gate; missing gate/prompt/window means rejection. Revocation cancels pending/active downloads, rechecks before commit and withholds subsequent metadata; cleanup cancellation remains possible. There is no rollback promise for committed files. This enforcement does not extend to legacy synchronous APIs or implement a universal file Broker.

The host supports safe commit on Windows fixed local NTFS and on local filesystems under Linux / macOS (x64, arm64): the destination directory chain is pinned before approval, a random temporary file is created exclusively, and the commit is atomic relative to the pinned parent. Windows holds every ancestor's handle with traverse plus read-attributes access, sharing read and write but not delete, so no ancestor can be renamed or moved; only name-surrogate reparse points are rejected, so cloud-file placeholders are not treated as links; the temporary file is verified by handle to live in the pinned parent, and the commit renames it through `NtSetInformationFile` relative to the parent handle. Linux / macOS walk from `/` with `openat(O_NOFOLLOW)` and keep only the parent directory fd, rejecting any symbolic link in the chain; 9p, SMB, FUSE, NFS and other non-local filesystems are refused before approval; no-replace uses `renameat2(RENAME_NOREPLACE)` / `renameatx_np(RENAME_EXCL)`, falling back to `linkat` when the flag is unsupported. Atomic no-replace protects even destinations appearing mid-transfer; approved overwrite never deletes first, and on POSIX the new file keeps the old file's permission bits. UNC and other filesystems/backends fail explicitly (`not-supported`) without writing the final path directly. User download throttling and timestamp preferences remain. Length/before-and-after metadata/available digests detect some changes, not an atomic remote snapshot. Full bytes still require verification, close and commit; an uncooperative backend with unconfirmed stop keeps slots/handles and reports Unknown/TimedOut, never automatic redownload.

The download contract is officially published in SDK 2.1.2; host central packages, the explicit AI reference and minSdk have all moved to 2.1.2. The commit tests have actually run on Windows and Linux, macOS only through CI, and a real SSH / GUI end-to-end flow has not been accepted yet. See [Automation and AI §1.7](11-automation-and-ai.md#17-dual-entry-background-ai-downloads-sdk-212) for model/link entry points.

## 3. vela.remoteExec, Remote Execution (S5/S6)

Uses a **separate execution channel**, never the user's terminal or its shell history/environment. The actual interface is `IRemoteExecApi`; see the [SDK reference §3.3a](../sdk/sdk-reference.md).

| Method | Actual contract and limits |
| --- | --- |
| `RunAsync` | Returns `ExecResult.Output` / `Error` / `ExitCode`; nonzero exit is a result, not an exception. Default 30 seconds, maximum 10 minutes |
| `StreamAsync` | stdout/stderr callbacks through `IProgress<ExecOutput>`, returning `ExecStreamResult`; no execution deadline by default, caller cancellation or explicit Timeout ends execution; no stdin |
| `StreamSudoAsync` (SDK **2.1.0**) | Returns `SudoExecResult`; `IProgress<SudoExecEvent>` carries non-secret states and merged raw PTY text fragments. Linux sudo only; the entire command runs as root, including pipes/redirections. Not a general interaction API |

Ordinary and sudo streams share the limit of four per plugin. Receivers must forward synchronously and quickly, not reorder output with `Progress<T>`. A slow receiver does not abort execution; it slows the remote stream instead: when an isolated plugin's notification queue is full the host waits for space, and backpressure travels back to the remote through the SSH window, while in-process plugins are already called synchronously on the line-reading thread. Ordinary output without newlines is split into fragments of at most 4096 characters; CR may delimit fragments, and long lines carry `[VelaShell: continued]`. Do not assume one callback is one complete application record. Like ordinary streams, sudo output replaces non-UTF-8 bytes with U+FFFD, so GBK logs, binary content or non-UTF-8 file names do not turn a running root command into Unknown.

**Separate current permission behavior from design goals.** This blueprint's `remote.exec` Broker, universal Demand and non-disableable audit are design goals, not guarantees delivered here. The current `PluginPermissionGate` covers `terminal.write` / `session.open` and separate dedicated per-job leases for the new upload/download APIs (§2.1/§2.2), not generic grants for other remote file operations. This change adds **trusted host confirmation on every sudo call**, not a complete generic permission system. Source identity is bound from the host manifest and the real target comes from the connection; even NOPASSWD cannot skip confirmation. In the confirmation dialog "Approve" stays disabled for about 0.6 seconds after it opens (counted from the window opening, since 2026-10-25), Cancel is always available, and Enter and the initial focus stay on Cancel — so a stray Enter while the user is typing cannot approve a root command.

AI-plugin approval and host confirmation are separate layers. `Ask` / `ReadOnlyAuto` use the distinct `run_command_elevated` kind, display the target and whole-command root scope, and have no RepeatKey. Missing approval Handler means rejection; even root `cat` is not read-only auto-approved. Explicit user-selected `Bypass` skips only plugin approval, never host confirmation. The external MCP route has no approval card, so it hands the human confirmation of elevation entirely to the host (`AgentToolbox.ElevationConfirmedByHost`): no mode rejects `run_command_elevated` at the plugin layer, and every elevated call goes straight to the desktop host's root confirmation and password prompt. Non-elevated writes are still rejected in the first two modes.

Only the host's trusted masked dialog collects the password: no login-password reuse, remembering, SDK/RPC/plugin/model/history/log exposure. The controlled PTY must confirm echo is off and match this call's random prompt before sending it. Once the command starts, password responses are permanently disabled and stdin is closed. The complete raw startup marker must not appear directly in argv that sudo may echo; only an authorized root wrapper may generate it. A sudoers refusal diagnostic must not be mistaken for Running/Exited or leak authentication-tail content as command output. Complex PAM/OTP, missing UI, rejected PTY or sudoers refusal of the root-shell wrapper fails explicitly; no policy bypass or ordinary-execution fallback. A fish or tcsh login shell works too: the login shell receives only a `/bin/sh` invocation with the controlled script encoded in a `printf %b` payload, which it cannot rewrite (see [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210)). Ordinary commands containing sudo are not regex-rewritten or automatically rerun elevated, avoiding replay of an already-executed script prefix. This is not terminal autofill and does not change existing terminal credential decisions.

See [SSH specification §9.1](../ssh/spec/08-failures.md#91-connection-wide-payload-suppression-leases-for-sensitive-interactions) for diagnostic leases and conservative asynchronous teardown, and [Security §1.1](12-security-threat-model.md#11-actual-boundaries-of-this-sudo-and-managed-ai-execution-change-sdk-210) for scope-cache/revocation limits and unverified areas.

## 4. vela.terminal, Terminal (S3/S4)

Permissions: `terminal.read` / `terminal.write` (high sensitivity).

```csharp
public interface ITerminal
{
    IAsyncEnumerable<TerminalOutput> SubscribeAsync(string sessionId, CancellationToken ct);
        // Host-side frame-coalescing throttle ≥16 ms; call Demand once when the subscription is established, and cut the stream on revocation
    Task<TerminalSnapshot> GetSnapshotAsync(string sessionId, CancellationToken ct);  // Screen + tail of scrollback (line limit)
    Task<string?> GetSelectionAsync(string sessionId, CancellationToken ct);
    Task WriteAsync(string sessionId, string input, CancellationToken ct);
        // Through the existing input serialization queue (do not write directly to the stream); ≤4 KB per call by default
    Task<string> CreateOutputChannelAsync(string title, CancellationToken ct);
    Task AppendToChannelAsync(string channelId, string text, CancellationToken ct);
        // Plugin-specific read-only output page (attached as a VelaDock document); requires no terminal permission, only ui.contributions
}
```

Additional `terminal.write` safeguards: injected input is briefly marked in the terminal with a superscript identifying the source plugin, which the user can disable in settings. Per-plugin write frequency is rate-limited (20 calls/min by default to prevent runaway screen spam); exceeding the limit trips a circuit breaker and notifies the user.

## 5. vela.localFs, Local Files

Permissions: `fs.local.read` / `fs.local.write` (scope = directory prefix). **Permission-free paths**: paths returned by `PickFileAsync` / `PickFolderAsync`, where the host displays the system picker, automatically receive temporary authorization. The user made the selection directly, following the Android SAF model. Plugins are encouraged to prefer the picker over requesting a broad directory scope. This blueprint's picker grant does not replace the new download API's per-job trusted host confirmation (§2.2); selecting a final path does not authorize overwrite or pre-create/truncate it.

```csharp
public interface ILocalFs
{
    Task<string?> PickFileAsync(FilePickerOptions opts, CancellationToken ct);
    Task<string?> PickFolderAsync(..., CancellationToken ct);
    Task<Stream> OpenReadAsync(string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string path, WriteMode mode, CancellationToken ct);
    Task<LocalEntry[]> ListAsync(string path, CancellationToken ct);
    // Regular Stat/Delete/Move/Copy…
}
```

## 6. vela.storage / vela.secrets / vela.settings

```csharp
public interface IStorage      // Permission: storage.private (normal)
{
    Task<T?> GetAsync<T>(string key);   Task SetAsync<T>(string key, T value);
    Task RemoveAsync(string key);       string DataDirectory { get; }   // Direct file access is also restricted to this directory
}
public interface ISecrets      // Permission: secrets (dangerous); namespace forced to <pluginId>/
{
    Task<string?> GetAsync(string name); Task SetAsync(string name, string value); Task DeleteAsync(string name);
}   // Uses the OS credential store underneath (DPAPI/Keychain/libsecret), a separate namespace from host credentials and invisible to each other
public interface IPluginSettings   // Permission: settings.own (normal)
{
    T Get<T>(string key);  Task SetAsync<T>(string key, T value);
    IAsyncEnumerable<SettingChange> WatchAsync(CancellationToken ct);   // Pushed when the user changes a setting in the settings page
}
```

## 6b. vela.timeSeries, Private Time-Series Store (Implemented)

```csharp
public interface ITimeSeriesApi          // Permission: comes with storage.private, no separate escalation
{
    Task<ITimeSeries> OpenAsync(TimeSeriesDefinition definition, CancellationToken ct);  // Create the measurement (idempotent)
    Task<IReadOnlyList<string>> ListAsync(CancellationToken ct);
    Task<bool> DropAsync(string name, CancellationToken ct);
}
public interface ITimeSeries
{
    Task WriteAsync(TimeSeriesPoint point, CancellationToken ct);
    Task WriteManyAsync(IEnumerable<TimeSeriesPoint> points, CancellationToken ct);
    Task<IReadOnlyList<TimeSeriesPoint>> QueryAsync(TimeSeriesQuery query, CancellationToken ct);
    Task<long> CountAsync(string field, TimeSeriesQuery query, CancellationToken ct);
    Task<IReadOnlyList<string>> DistinctTagValuesAsync(string tag, CancellationToken ct);
    Task<int> DeleteAsync(IReadOnlyDictionary<string, string>? tags, CancellationToken ct);
}
```

It lives in the host's single SonnetDB; the physical measurement name is `pts_<plugin namespace>_<short name>`, the
namespace is derived from the plugin id (the plugin cannot choose it), and uninstalling drops everything under the prefix.
Semantics and quotas are in [dev-guide §5.2b](../templates/dev-guide.md#52b-timeseries-private-time-series-store); isolated processes are routed to the same
implementation through the `ts/*` methods (see 05). **Plugins never connect to the database directly and get no SQL entry
point** — external input such as tag values is always parameterized, and column names are limited to `[a-z0-9_]`.

## 7. vela.ui, UI (See 08)

Permission: `ui.contributions` (normal). Includes commands/menu items/status-bar dynamic updates, VelaUI surfaces (panels and document pages), dialogs (Message/Confirm/Input/QuickPick), in-app notifications (rate limit: ≤6 per plugin per minute), and progress indicators (in the status bar or notifications).

> The actual contract for in-app notifications (SDK 2.1.3, now wired in the host): `IUiApi.SupportsNotifications` / `PostNotificationAsync(PluginNotification)`,
> returning `Posted` / `AlreadyPosted` / `Muted` / `RateLimited`; de-duplicated by id before rate limiting (6 per plugin per minute), only for messages worth keeping and reading again,
> with actions limited to the plugin's own commands and no URL field. See [SDK §3.7](../sdk/sdk-reference.md#37-posting-to-the-message-center-sdk-213).

## 8. vela.events, Host Event Subscriptions (S5)

Permission: most event payloads follow the permission for the corresponding domain, such as session events requiring `sessions.read`. `appStartup/appShutdown/themeChanged/localeChanged` require no permission.

```csharp
public interface IEvents
{
    IAsyncEnumerable<HostEvent> SubscribeAsync(EventFilter filter, CancellationToken ct);
    // HostEvent: SessionConnected/Disconnected, TransferCompleted, ThemeChanged,
    //            LocaleChanged, AppShutdown (gives the plugin a chance to flush), ScheduleFired (see 11)
}
```

## 9. vela.audio, Audio Output (S2)

Permission: `audio.playback`. Design choice: **the plugin decodes, the host outputs**. The host does not include codecs, avoiding codec dependency bloat and licensing issues. It only provides PCM mixing output. MP3 decoding is supplied by the plugin using a managed decoder, such as NLayer, as demonstrated by the official sample.

```csharp
public interface IAudio
{
    Task<IAudioTrack> CreateTrackAsync(AudioFormat fmt, CancellationToken ct); // fmt: sample rate/bit depth/channels
    // IAudioTrack: WriteAsync (PCM chunks, backpressure), pause/resume/stop, volume, position feedback
}
```

Host output backend: pending a selection spike, with candidates including a miniaudio binding, OpenAL Soft, and three platform-native implementations using WASAPI + CoreAudio + ALSA. The placeholder interface comes first, so selection does not block other domains. Global mixing applies at a given time. Plugin volume is constrained by the host master volume and mute switch. When a plugin is disabled or crashes, its audio track is muted and reclaimed immediately.

## 10. vela.net, Network (Declarative)

Permission: `network` + a domain-list declaration in the manifest. v1 provides no physical interception (see the premise in 06). The SDK provides `ctx.Http`, a convenience wrapper based on HttpClient that automatically includes the plugin UA. Requests made through the SDK wrapper are recorded through the host audit channel with the target domain. Directly constructing an HttpClient cannot be prohibited in v1. Compliance depends on review and signatures, and the documentation states this limitation plainly.

## 11. vela.clipboard / Notifications / i18n

- `clipboard.read/write`: read/write the system clipboard. Read is a dangerous permission because the clipboard often contains passwords.
- `notifications.system`: OS-level notifications. In-app notifications belong to vela.ui and do not require this permission.
- `i18n.read`: the current locale and translated common host strings, such as the “OK” and “Cancel” buttons, so plugin copy matches the host.

## 12. vela.ai, AI Gateway (Longer Term, See 11-automation-and-ai)

Permission: `ai.invoke`. Reserved in apiLevel 1, with the interface held for future use and the host able to declare the capability unavailable. Full implementation is covered in [11-automation-and-ai.md](11-automation-and-ai.md).

## 13. Development Plan (This Work Item)

Batch divisions correspond to the milestones in 14:

| Task | Content | Dependency | Estimate |
| --- | --- | --- | --- |
| C-1 | Capability-service foundation: registry, Demand base class, audit hooks, per-plugin rate limiter | B-2, P-3 | 3d |
| C-2 | First batch: storage / settings / secrets / events (foundation domains, minimum dependencies for sample plugins) | C-1 | 4d |
| C-3 | Second batch: sessions / remoteFs (bridge Core.Ssh, low-priority transfer queue, progress throttling) | C-1 | 5d |
| C-4 | Third batch: localFs (picker temporary-authorization model) + clipboard + notifications | C-1 | 3d |
| C-5 | Fourth batch: terminal (subscription frame coalescing, snapshots, write-rate circuit breaker) + remoteExec (auditing) | C-3 | 5d |
| C-6 | Audio-output spike (evaluation report for three candidates) → IAudio implementation | C-1 | 5d |
| C-7 | vela.net SDK wrapper + auditing | C-1 | 1d |
| C-8 | Contract test suite: for every domain, “permission matrix × normal/boundary/cancellation/disconnection” cases (connects to 13) | Each batch | Ongoing |

Acceptance: the domains required by S1 (C-2/C-3) support the image-viewer sample end to end. Once C-5 is complete, all domains for S3/S4/S5 are available.

---

## protocols — plugin-provided remote file protocols (apiLevel 1, landed 2026-08)

Once a plugin registers a protocol, it is a first-class citizen on the connection page alongside the built-in ones; the host
routes that protocol's sessions to the plugin's `IProtocolFileSystem`, and the dual-pane browser, transfer queue, rate limiting,
drag and drop and conflict policy are all reused unchanged.

| Member | Notes |
| --- | --- |
| `IProtocolsApi.Register(descriptor, fileSystem)` | Registers a protocol; disposing the return value unregisters it. The protocol id must equal the plugin id or start with it |
| `IProtocolsApi.GetTransferOptionsAsync()` | Reads the host's current bandwidth cap and timestamp policy (a global user preference, not something each protocol should configure separately) |
| `IProtocolFileSystem` | Connect/disconnect + list/stat/read/write/delete/rename/copy/permissions + protocol actions, all taking the host-assigned opaque session key as the first argument |
| `ProtocolDescriptor` | Tab name, default port, relabelling of the three standard fields, a **declarative settings form**, **declarative context-menu actions**, feature bits, the field the certificate fingerprint is written back to |
| `ProtocolFeatures` | `Permissions` / `ServerSideCopy` / `ResumeUpload` / `ResumeDownload` / `AnonymousAccess` / `CertificateTrust` |
| Exception family | `ProtocolAuthenticationException` / `ProtocolCertificateTrustException` / `ProtocolConnectionException` / `ProtocolUnsupportedException` |
| Manifest contribution | `contributes.protocols` + lazy activation via `onProtocol:<protocol id>` |
| Optional interfaces implemented alongside (SDK 2.1.3, now wired in the host) | `IProtocolLastWriteTime` (setting modification times), `IProtocolChecksum` (server-provided digests only, `RemoteChecksumAlgorithm`); `RemoteFileEntry` gains `IsSymbolicLink` / `LinkTarget`. See [SDK §3.3d](../sdk/sdk-reference.md#33d-file-capabilities-for-plugin-protocols-sdk-213) |
| `PluginManifestReader.IsValidProtocolId` | The authoritative protocol-id check (all-lowercase `[a-z0-9.-]`, ≤128, must start with the plugin id); shared by manifest validation and run-time registration so the two cannot drift apart |

Design choices:

- **The settings form is a declarative schema, not a control tree.** The connection dialog is a core host surface whose layout,
  theming, validation and localization the host owns; the plugin only describes "which parameters it needs". This is not the
  general declarative UI (VelaUI) that blueprint 08 explicitly rules out — that is arbitrary UI, this is a closed-shape parameter table.
- **Context-menu actions are declarative too**: the menu has to be drawn on the frame the right button goes down, it cannot wait
  for an asynchronous round trip.
- **`inProcess` only**: a protocol is a high-frequency channel on which the host calls back into the plugin (streaming reads
  included), while isolated-process RPC only carries the plugin → host direction. Opening it up needs bidirectional requests,
  which is follow-up work for 05. In an isolated process `IProtocolsApi` is a degraded implementation that throws on register,
  not null.
- **Re-registering the same id = replacement**: replacing it with a **different** implementation raises one unregistration
  (the host uses it to close the sessions under the old implementation), replacing it with the same instance (a plugin
  re-registering to refresh localized strings) does not — otherwise switching the UI language would drop every tab the user
  has open. Disposing the old handle after it has been replaced is a no-op.
- **The registry never calls back into the plugin under its lock**: event subscription and unsubscription always happen outside
  it. The UI thread holds that lock too, and a plugin doing a blocking `Dispatcher.Invoke` in an event accessor would deadlock.

---

## ui addendum: `IPluginPanel.ActivateAsync` (apiLevel 1, added 2026-08)

Brings the panel to the front: in window form it restores from minimized and calls `Activate()`, in document form it switches
to that tab. Idempotent, callable from any thread, and a no-op once the panel is closed.

Why it was added: a plugin's convention for "the same target triggered again" is to focus the one already open rather than open
another, but the SDK only had `CloseAsync`, which cannot express "focus" — so plugins could only `return`, and a same-sized
`CenterOwner` window happens to cover the previous one exactly, so what the user saw was "I clicked and nothing happened".
