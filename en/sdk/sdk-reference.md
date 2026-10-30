# Plugin SDK Reference

> Applies to **SDK 2.1.3 / apiLevel 2**
> See also (other repositories): [Development Guide](../templates/dev-guide.md) (tutorial) · [CLI Manual](../cli/cli.md) · [Packaging and Publishing](../templates/publishing.md)

This page is the **lookup-oriented** view of the SDK: package layout, contract surface, what each
capability can do and what constrains it, version history, and the test doubles. For a
step-by-step walkthrough, read the [Development Guide](../templates/dev-guide.md) first.

---

## 1. Packages

| Package                         | Repository                                                                                | Referenced by                          | Contents                                                                                                                                       |
| ------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **`VelaShell.PluginSdk.Build`** | this repository (since 2026-09-11; previously velashell-plugin-cli)                       | **the plugin project (this one only)** | MSBuild props/targets, a version-pinned Avalonia, the bundled packer, manifest validation, the `PackVpx` target                                |
| `VelaShell.PluginSdk`           | this repository                                                                           | pulled in transitively                 | The contract assembly: entry interface, `IPluginContext` and every capability interface, DTOs, manifest model, `.vpx` container, host registry |
| `VelaShell.PluginSdk.Testing`   | this repository                                                                           | the plugin's **test** project          | `TestPluginContext` and in-memory doubles for every capability                                                                                 |
| `VelaShell.Plugin.Cli`          | [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli)             | developer machine (dotnet tool)        | `vela-plugin`: inner loop, validation, packing, signing, health check                                                                          |
| `VelaShell.Plugin.Templates`    | [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates) | developer machine                      | `dotnet new velaplugin` / `velaplugin-ui` / `velaplugin-protocol` / `velaplugin-workspace`                                                 |

The `PackageReference` line a plugin project needs — **including the current version** — lives in the
[Development Guide](../templates/dev-guide.md). That version belongs to
`VelaShell.PluginSdk.Build`, which now versions independently of this SDK, so repeating it here
would only guarantee it goes stale.

Do **not** reference `VelaShell.PluginSdk` or `Avalonia` separately — a version mismatch is a
build error (`VELA1001`) instead of a runtime control-cast failure on a user's machine.

The contract assembly depends only on the BCL. That is a discipline: it is the **only type
source shared between host and plugin**, so a third-party dependency inside it would be imposed
on every plugin.

---

## 2. Entry contract

```csharp
using VelaShell.PluginSdk;

[VelaPlugin]                                  // exactly one public, non-abstract, parameterless class
public sealed class DemoPlugin : IVelaPlugin
{
    public Task ActivateAsync(IPluginContext context, CancellationToken ct)
    {
        context.Log.Info("activated");
        return Task.CompletedTask;            // must return quickly (10-second limit)
    }

    public Task DeactivateAsync(CancellationToken ct) => Task.CompletedTask;  // ~2-second limit
}
```

| Constraint         | Detail                                                                                                                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Activation limit   | 10 seconds; **stretched to 10 minutes while a debugger is attached**                                                                                                                          |
| Deactivation limit | About 2 seconds (application exit path), abandoned on timeout                                                                                                                                 |
| Long-running work  | Start your own background task and observe `context.Shutdown`                                                                                                                                 |
| Resource cleanup   | Commands and event subscriptions registered through the SDK are cleaned up by the host; never put your own types into host static fields or long-lived events, or the ALC cannot be reclaimed |

---

## 3. `IPluginContext` capabilities

`IPluginContext` is the single entry point into the host. **Every interface is transport
agnostic** (async methods, DTOs, opaque ids only), so the same plugin source runs in both
`inProcess` and `isolated` mode.

### 3.1 Identity and infrastructure

| Member                       | Description                                                                                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `PluginId` / `PluginVersion` | From `plugin.json`                                                                                                                                                       |
| `DataDirectory`              | Private directory (already created). **All local writes belong here**; it is deleted on uninstall                                                                        |
| `Host` (`IHostInfo`)         | Host version, apiLevel, current locale. `Host.Theme` only holds the coarse `dark`/`light`/`system` value — for the actual theme identity and palette see `Theme` in §3.5 |
| `Log` (`IPluginLogger`)      | `Debug/Info/Warn/Error` into the host log pipeline (prefixed with the plugin id), and also into this plugin's own log file `~/.velashell/logs/plugins/<id>.log`, which users see through "View log" on the plugin management page |
| `Shutdown`                   | Shutdown token: capability calls may start failing afterwards, so wind down                                                                                              |

### 3.2 Data

| Capability                      | Key methods                                                                                                                                | Notes                                                                                                 |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `Storage` (`IPluginStorage`)    | `GetAsync<T>` / `SetAsync<T>` / `RemoveAsync` / `GetKeysAsync`                                                                             | Per-plugin namespaced KV backed by SonnetDB (JSON files in headless setups)                           |
| `Secrets` (`ISecretsApi`)       | `GetAsync` / `SetAsync` / `DeleteAsync`                                                                                                    | Encrypted private key-value store. **No plaintext fallback** — unavailable is reported as unavailable |
| `TimeSeries` (`ITimeSeriesApi`) | `OpenAsync` / `ListAsync` / `DropAsync`; on a series `WriteAsync` / `QueryAsync` / `CountAsync` / `DistinctTagValuesAsync` / `DeleteAsync` | A private embedded time-series store (append by time, retrieve by tag)                                |

### 3.3 Sessions and remote access

| Capability                          | Key methods                                                                           | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Sessions` (`ISessionsApi`)         | `ListAsync` / `GetAsync`; `ListSavedAsync` / `OpenAsync` / `CloseAsync` (since 2.0.2) | Enumerate SSH sessions, **redacted, never credentials**. The last three **ask the host to open a saved session** — only saved configurations can be opened (the user decides which machines exist), not one byte of credentials passes through the plugin, and the host may refuse (`PluginPermissionDeniedException`); `SessionOpenOptions.Reason` is shown to the user verbatim. A connection that fails gets `PluginSessionOpenException` instead: "you may not connect" and "it did not connect" call for different handling and should not share a type. `CloseAsync` **only closes what this plugin opened via `OpenAsync`** (a reused pre-existing session included — everyone else's is refused) |
| `RemoteFs` (`IRemoteFsApi`)         | directory / attributes / read / write / transfer / rename / delete                    | SFTP over an existing session                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `RemoteExec` (`IRemoteExecApi`)     | `RunAsync` (whole result) / `StreamAsync` (per line)                                  | A separate channel — **never the user's terminal**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `RemoteTunnel` (`IRemoteTunnelApi`) | `OpenUnixSocketAsync` / `OpenTcpAsync`                                                | A **raw byte duplex stream** to a remote endpoint (Docker Engine API, tar streams). `inProcess` only                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `Terminal` (`ITerminalApi`)         | `GetOutputAsync` / `SearchOutputAsync` / `WriteAsync`                                 | Read/search session output; **writing input requires user consent** (revocable on the manager page)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |

### 3.3a Dedicated sudo execution (SDK 2.1.0)

The dedicated contract is **officially published in SDK 2.1.0**, restored from NuGet.org. Host `src/tests` central packages and the explicit AI reference use 2.1.0; the AI manifest requires `minSdkVersion: 2.1.0` (`apiLevel: 2`) to prevent installation on older hosts. No local packages, temporary feeds or cross-repository project references are used. The existing global banner remains release-script-managed; this section states the version required by the added contract.

`IRemoteExecApi.StreamSudoAsync(string sessionId, string command, SudoExecOptions? options, IProgress<SudoExecEvent> progress, CancellationToken cancellationToken = default)` returns `Task<SudoExecResult>`.

- Linux sudo only: a separate PTY exec runs the **entire command as root**, including pipelines and redirections. Pass the actual command without a sudo wrapper. This is not `su`, arbitrary stdin or credential autofill for the user's terminal, and existing terminal credential-interaction decisions are unchanged.
- `SudoExecOptions` contains only nullable `Timeout` (execution deadline; null means unlimited), with no password, source identity, stdin or environment fields. The host determines the source plugin and actual target. Trusted host confirmation is required on every call, including NOPASSWD/cached authentication. Missing host UI or unsupported PTY/authentication fails explicitly, never by falling back to ordinary execution.
- `SudoExecEvent.Kind` is `State` or `Output`. States are `AwaitingApproval` / `AwaitingAuthentication` / `Running`; Output.Text contains **merged PTY text fragments**, preserving newlines and CR, without guaranteed line boundaries or a separate stderr. Authentication content, passwords and random markers never reach the plugin.
- `SudoExecResult` contains `Outcome`, nullable `ExitCode` and `OutputChunks` (Output events only). Outcomes are `Unknown = 0`, `Exited`, `AuthenticationCancelled` (including approval refusal), `AuthenticationFailed` and `NotSupported`. The latter three mean the user command did not start; nonzero process exit is still `Exited`, not authentication failure. `Exited` without an exit code is treated as Unknown, never defaulted to zero. `IsSuccess` requires `Exited` and a confirmed zero exit code. Never automatically replay Unknown operations.
- Caller cancellation throws `OperationCanceledException`; an execution deadline throws `TimeoutException`. A cancellation request does not confirm that the remote process stopped. State/output callbacks are synchronous and ordered, and drained before return. Receivers must forward directly and quickly, not use the asynchronous Post behavior of `Progress<T>`. Ordinary and sudo streams share the per-plugin limit of four streams.
- The default interface implementation and `FakeRemoteExec` without `SudoHandler` explicitly throw `NotSupportedException`, never invoke ordinary execution; the fake records sudo requests separately in `SudoStreamed`.

For isolated-process negotiation, real cancellation and event integrity, see [IPC §7.1](../plugins/05-ipc-protocol.md#71-execution-operations-sdk-210); for AI jobs, approval and recovery, see [Automation and AI §1.4](../plugins/11-automation-and-ai.md#14-managed-execution-and-recovery-in-the-ai-plugin-sdk-210).

### 3.3b Independent upload jobs (SDK 2.1.1)

**The independent upload contract is officially published in SDK 2.1.1 on NuGet.org.** Host central dependencies, the testing package and the explicit AI reference use 2.1.1; the AI manifest requires `minSdkVersion: 2.1.1` (`apiLevel: 2`). No local packages or temporary feeds are used. A published contract does not imply that every host implements an executor: callers must still check the actual capability and follow the lifecycle below.

`IRemoteFsApi.SupportsTransferOperations` defaults to `false`. Default implementations of the new methods throw `NotSupportedException`, never fall back to a long-running `UploadFileAsync`. Existing interface members and RPC DTO constructor/deconstruction signatures remain unchanged, as does apiLevel. The capability flag applies only when the new contract assembly is already loaded; new plugins must still declare the officially published minimum SDK version, not assume an old SDK assembly can load the added types.

| Method | Semantics |
| --- | --- |
| `StartUploadAsync(sessionId, localPath, remotePath, RemoteUploadOptions, ct)` | Returns `RemoteTransferSnapshot` without waiting for completion; idempotent per host-bound plugin identity and `OperationId`, rejecting changed arguments for the same id |
| `GetTransferAsync(transferId, ct)` | Reads an owned, currently authorized job without leaking metadata for unauthorized ids |
| `WaitTransferAsync(transferId, afterSequence, waitTimeout, ct)` | Waits for `Sequence > afterSequence` or a terminal state; `afterSequence >= 0`, window 0–30 seconds (`MaxTransferWaitSeconds`); expiry returns the current snapshot |
| `CancelTransferAsync(transferId, expectedGeneration, ct)` | Idempotently cancels the specified execution generation; reject a mismatch with current `Generation`, so stale cancellation cannot stop a resumed generation |
| `ResumeTransferAsync(transferId, RemoteTransferResumeOptions, ct)` | Explicit, reapproved resume; options contain nonblank `OperationId`, positive `ExpectedGeneration` and nonblank `TargetSessionId`, never paths, offsets or source identity |

`RemoteUploadOptions` contains only required nonblank `OperationId`, `Overwrite` (default false), and nullable positive `Timeout` (null means no execution deadline). There is no fixed file-size cap; 64-bit counters are not bounded by 32 MiB or `int.MaxValue`, but storage, authorization, protocol and network constraints still apply. Implementations must stream rather than load the whole file. The host writes an unpredictable temporary file and commits atomically only after validation and remote finalization succeed; an unsupported atomic commit/overwrite fails explicitly, never by deleting the destination first. The current host's temporary file is a short name in the destination's directory, `.vela-up-<16 hex digits>` (appending to the destination name would push long names past `NAME_MAX`). Before writing it runs a commit preflight: overwrite without server `posix-rename`, or no-overwrite with an existing destination, fails immediately with `ErrorCode` `not-supported` or `commit-rejected` respectively, the same classification as a refused commit. A preflight that cannot find out does not block, a destination that appears afterwards is still refused by the commit, and the commit remains the only authority. The temporary file of an overwriting upload is created with the replaced regular file's 0777 permission bits from the start (the server applies its umask as usual; 0600 when the destination exists but reports no mode; the server default when the destination cannot be read, with the commit still checking), so it is never wider than the original while being written; setuid / setgid / sticky bits and the owner are still applied by the commit. An overwrite commit carries the replaced regular file's permission bits (07777) over to the new file: the owner on a best-effort basis, setuid / setgid only once owner and group match. If the destination's mode cannot be read, or the temporary file's mode cannot be changed while it is wider than the destination's, the commit is refused and the destination left untouched (narrowing is allowed, widening is not).

`RemoteTransferSnapshot` contains job/start ids, session and source/destination paths, `State`, monotonically increasing `Sequence` starting at 1, `TransferredBytes`/`TotalBytes` (unknown -1, empty file 0), nullable speed/ETA, creation/update/completion timestamps, cancellation intent, remote-stop confirmation, resume checkpoint and error category/message. The default state is `Unknown = 0`; other states are `Queued`, `Starting`, `Running`, `CancelRequested`, `Succeeded`, `Failed`, `Cancelled`, and `TimedOut`. Full byte counts can still mean `Running`, not completion; `IsSuccess` requires `Succeeded`. `Unknown`/`TimedOut` are local terminal states, not proof that remote writes stopped; never automatically replay them. After a stop (cancel, deadline, disconnect, deactivation) the current host waits up to 30 seconds for the backend to stop before settling the terminal state; a disconnect is `Failed` / `session-disconnected` (write confirmed stopped, destination unwritten, safe to retry after reconnecting), distinct from `Cancelled`; too little space on the volume holding the Linux / macOS source snapshot is `Failed` / `insufficient-space`.

- Tokens for start/get/wait/cancel/resume control **that request only**. Wait expiry or request cancellation does not stop an accepted job; cancellation intent already recorded is not rolled back by request cancellation. Reuse the same `OperationId` after a lost start response instead of generating another id and reuploading.
- The host owns jobs under the authenticated plugin identity and rechecks authorization on every access. AI run/conversation isolation still needs a separate plugin-side layer; plugin ownership is not conversation isolation. Plugin deactivation, authorization revocation, SSH disconnect or host exit requires cleanup; without stop confirmation, use `Unknown`. Only the current application/plugin lifetime is covered: there is no promise of recovery after application restart, plugin reactivation or destruction of an MCP session.
- The current host resolves local source parent-directory aliases component by component (such as macOS `/var`) while rejecting a linked leaf file (on Windows only name-surrogate reparse points, that is symbolic links, directory junctions and mount points, count as links; filter-type tags such as OneDrive cloud-file placeholders do not). The opened object is verified by file identity (Windows volume serial + file id, Unix dev + inode), with the compared side reached without following links; mapped drives, subst, other hard-link names and case / normalization variants count as the same file; approval and snapshots show the resolved path. Idempotency receipts retain the original request path, so replaying the same OperationId never reopens a source after an alias changes or its path disappears. Path checks and snapshot validation are not an atomic filesystem-snapshot guarantee against malicious concurrent writes.
- Initial approval binds normalized source/destination paths, the actual source object and overwrite policy. The host checks local-read and remote-write authorization, maintaining source stability before queued work starts and throughout transfer: path replacement, symlinks or concurrent writes must not alter approved content; fail if stability cannot be guaranteed. `Overwrite=false` requires an atomic no-replace commit, not just an initial existence check.
- Start/resume `OperationId` values share a plugin-lifetime namespace recording operation kind and all arguments. Serialize mutations, check authorization and deduplication before current state, and return the current snapshot on replay without reexecuting. Retain receipts/tombstones for the entire lifetime; reject new registrations at quota rather than forgetting ids and replaying them. Argument conflicts throw `InvalidOperationException`; missing or foreign ids both throw `FileNotFoundException`.
- Snapshots add `Generation` (starts at 1, increments on resume) and `DestinationCommitConfirmedAbsent` (defaults false). `CanResume` also requires confirmed absence of final commit: unresolved commit acknowledgements must never be resumed. Revalidate the source object, temporary-file ownership and checkpoint. Reconnection uses explicit `TargetSessionId`; the host verifies server-key identity, account and filesystem namespace, not merely an equal address. Resume preserves `transferId` and the increasing `Sequence`, clears cancellation intent/completion time, and rejects invalid conditions rather than restarting from zero.
- `Timeout` starts when each generation is registered, including queueing and finalization. Explicit reapproved resume grants the original duration again rather than inheriting an exhausted deadline. Null still means no execution deadline.
- RPC adds `fs/transfer/startUpload|get|wait|cancel|resume|progress` with `FsTransferStartUploadRequest`, `FsTransferRef`, `FsTransferWaitRequest`, `FsTransferCancelRequest`, `FsTransferResumeRequest`, and `FsTransferProgressNotification`. Handshake `SupportsTransferOperations` defaults to false; support must not be inferred for old hosts. Progress carries a snapshot: discard stale frames by `TransferId + Sequence`, throttle to at least 100ms, send state changes promptly, and recover missed notifications via get/wait. These notifications do not acquire the exec-specific strict-send guarantee.
- `FakeRemoteFs` explicitly configures its capability flag and start/get/wait/cancel/resume handlers, records independent requests, and forwards arguments and request tokens. Missing handlers fail as unsupported, never fall back to the existing whole-file in-memory upload. Idempotency, authorization and actual execution belong to the host; test scripts must simulate them explicitly instead of receiving invented success from the fake.

The current host enforces trusted confirmation for the new upload API through a dedicated per-job lease. Before content is read, it displays the plugin, actual session, local source, **file size**, remote path and overwrite policy. Approval is limited to that job, never inherited from terminal/session-opening grants; AI `Bypass` cannot bypass host confirmation. Missing gate, prompt or desktop window means rejection. Revocation from plugin management synchronously invalidates pending/active leases and requests cancellation, with a pre-commit recheck and revoked metadata withheld from queries/subsequent progress. This does not promise rollback of an already committed file, extend to legacy synchronous file APIs or implement the universal permission Broker.

This contract does not change existing synchronous upload/download behavior, AI chat attachments or IM platform file limits. Host/AI integration uses independent jobs for visible progress, model status queries and uploads outside long requests; external MCP agents must still query explicitly, not assume server-driven model wakeups. The current host connection boundary cannot prove that server key, account and remote namespace remain identical after reconnection, so `ResumeTransferAsync` explicitly reports unsupported and `CanResume=false`. Having an interface does not mean resume is implemented; never automatically replay `Unknown`.

### 3.3c Independent download jobs (SDK 2.1.2)

**The independent download contract is officially published in SDK 2.1.2 on NuGet.org.** The host's central dependencies in `src/Directory.Packages.props` and `tests/Directory.Packages.props`, the testing package and the explicit AI reference use 2.1.2; the AI manifest requires `minSdkVersion: 2.1.2` (`apiLevel: 2`). No local packages or temporary feeds are used. A published contract does not imply that every host implements an executor: callers must still check `SupportsDownloadOperations`.

- `IRemoteFsApi.SupportsDownloadOperations` and `HandshakeResponse.SupportsDownloadOperations` are separate from upload support and default to `false`; unsupported downloads fail explicitly, without falling back to synchronous `DownloadFileAsync`.
- `StartDownloadAsync(sessionId, remotePath, localPath, RemoteDownloadOptions, ct)` returns a background snapshot. `RemoteDownloadOptions` contains required `OperationId`, `Overwrite=false` and `Timeout=null` (no default execution deadline). The request token cancels only receipt waiting, not registered work. Streaming has no fixed AI size cap; storage, authorization, user download bandwidth and shared resource budgets still apply.
- Reuse `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` ownership, sequence and generation semantics. Downloads have `CanResume=false`, with no checkpoint or cross-restart recovery. `LocalPath` is the final local destination and `RemotePath` the remote source; temporary paths are private. Idempotency includes direction and all arguments: a changed direction/argument under the same id is rejected. Reuse the original id after a lost receipt; never automatically redownload unknown outcomes.
- Snapshots add `Direction`: `RemoteTransferDirection.Upload=0`, `Download=1`; and `Phase`: `RemoteTransferPhase.Unknown=0`, `Approval`, `Transfer`, `Verification`, `Commit`. Missing fields default to Upload/Unknown; existing constructors/deconstruction remain unchanged. The current host reports `Phase` for **upload** snapshots too (since 2026-10-25; before, uploads were always `Unknown`): `Approval` for the receipt and while waiting for host confirmation, `Transfer` once approved, `Verification` during read-back / digest checks, `Commit` while committing — so a plugin can withdraw only jobs not yet approved; the SDK contract is unchanged, the host just fills the field now. 64-bit counters, speed, ETA and `RemoteStopConfirmed` describe progress. Full bytes do not mean success: flush, close, verification and confirmed commit precede `Succeeded`. An uncooperative backend may first produce local `Unknown`/`TimedOut` with `StopConfirmed=false`; slots and handles remain held until it actually exits.
- RPC adds `PluginRpc.FsTransferStartDownload="fs/transfer/startDownload"` and `FsTransferStartDownloadRequest`. Get/wait remain authoritative; there is no new download-push delivery promise. `FakeRemoteFs` separately configures download support/handlers and records starts; a missing handler is unsupported.
- Uploads and downloads share per-plugin limits of 1024 lifetime receipts, 32 active/queued jobs and four actual transfers; downloads do not double concurrency. Legacy synchronous APIs, model-context reads, chat attachments and IM platform attachment limits are unchanged.

The host uses a dedicated per-download lease checking remote read/local write, with per-job confirmation of plugin, actual session, source, final destination and overwrite policy. Upload, terminal or session-opening grants do not carry over; AI Bypass/a save picker cannot replace trusted host approval. Missing window/prompt means rejection. Revocation invalidates pending/active leases, requests cancellation, is rechecked before commit and redacts subsequent reads; cleanup cancellation remains available. Already committed destinations are not promised rollback.

**Safe commit currently supports Windows fixed local NTFS volumes and local filesystems on Linux / macOS (x64, arm64).** Both pin the destination directory chain before approval and create the temporary file, commit and clean up relative to the pinned parent. Windows holds every ancestor directory's handle with traverse plus read-attributes access, sharing read and write but not delete (a read-attributes-only handle is exempt from sharing checks; traverse access is what makes the share mode count), so no ancestor can be renamed or moved; only name-surrogate reparse points (symbolic links, directory junctions, mount points) are rejected, and filter-type tags such as cloud-file placeholders are not treated as links. The random temporary file is created exclusively with `CreateNew` in the pinned parent and its location verified by handle; the commit goes through `NtSetInformationFile` relative to the parent handle with only the leaf name. Linux / macOS first resolve the parent to its real path and record it as the snapshot's `LocalPath`, then walk from `/` with `openat(O_DIRECTORY | O_NOFOLLOW)` keeping only the parent directory fd, rejecting a symbolic link at any level. Only local, inode-addressed filesystems are accepted (Linux by a `statfs.f_type` allowlist, macOS by `MNT_LOCAL`); 9p / SMB / FUSE / NFS are refused before approval. The temporary file is created with `O_CREAT | O_EXCL | O_NOFOLLOW` and holds a flock; a no-replace commit uses `renameat2(RENAME_NOREPLACE)` / `renameatx_np(RENAME_EXCL)`, falling back to `linkat` + `unlinkat` when the flag is unsupported, and an overwrite uses `renameat`. A new file's permissions are narrowed by the umask; an overwrite keeps the old file's permission bits (without setuid / setgid / sticky). Default atomic no-replace also protects a destination appearing mid-transfer; only explicitly approved overwrite replaces it, never by deleting it first. Opening a destination sweeps orphaned `.vela-download-<48 hex digits>` files left in the same directory by a crash, removing only exact name matches no job is using; a same-shaped link is removed itself. UNC, non-NTFS/non-fixed volumes, other filesystems and unsupported backends fail explicitly with `ErrorCode` `not-supported`, never fall back to writing the final path. The old synchronous download never receives the user's final path; timestamp preferences are applied to the owned temporary handle.

Actual length, remote metadata before/after transfer and available digests are checked. Pure SFTP accounts are not required to support exec; a second full download is not the default. Detected source changes fail, but these checks **do not guarantee an atomic remote snapshot**. The commit tests have actually run on Windows and Linux (ext4 and tmpfs under WSL Debian), macOS only through CI; this is not real SSH/GUI end-to-end validation. See [Automation and AI §1.7](../plugins/11-automation-and-ai.md#17-dual-entry-background-ai-downloads-sdk-212) for lifecycle and terminal notification behavior.

### 3.3d File capabilities for plugin protocols (SDK 2.1.3)

**The contract is officially published in SDK 2.1.3 on NuGet.org, and the host now supports it.** The host central dependency is 2.1.4, and `PluginProtocolFileService` (link fields, modification times, digests) and `RemoteFsCapability.Map` use these capabilities. Protocol plugins using them declare `minSdkVersion: 2.1.3`. Everything is additive; `apiLevel` is unchanged.

- `RemoteFileEntry` gains two `init` properties (the eight-argument constructor and `Deconstruct` are unchanged): `IsSymbolicLink` is decided by lstat, while `IsDirectory`, `Size` and `LastModified` describe the link's **target** (a broken link keeps the link's own attributes); `LinkTarget` is readlink's raw text and may be relative. The existing `fs/list` / `fs/stat` JSON of isolated processes carries both automatically, and old data without them reads as "not a link".
- `IProtocolLastWriteTime` (optional, implemented alongside like `IProtocolStreamUpload`): `SetLastWriteTimeAsync(sessionId, path, DateTimeOffset lastWriteTime, ct)`.
- `IProtocolChecksum` (optional, implemented alongside): `ComputeChecksumsAsync(sessionId, RemoteChecksumAlgorithm algorithm, IReadOnlyList<string> paths, ct)` returns `IReadOnlyDictionary<string, string?>` — keyed by the requested paths, values in lowercase hex, null for a file that could not be computed. **Only digests the server produces itself may be returned**; reading a whole file back to make one up is not allowed. `RemoteChecksumAlgorithm` is `Unknown = 0`, `Sha256`, `Sha512`, `Sha1`, `Md5`.
- How "unsupported" is reported: when a protocol does not implement these interfaces, the host reports unsupported; when it does but this session cannot (no MFMT or HASH on the server), it throws `ProtocolUnsupportedException`. Both interfaces are for `inProcess` protocols only (protocols support only `inProcess` anyway), so no RPC is added.
- Out of scope: **creating** symbolic links on plugin protocols still has no contract, and the host's `CreateSymbolicLinkAsync` still reports unsupported.

### 3.3e Transfers refused before they start (SDK 2.1.3)

**The contract is officially published in SDK 2.1.3 on NuGet.org, and the host now supports it.** The host implements explicit pre-registration refusals: a missing source or parent directory returns `SourceNotFound`, a locked source returns `SourceInUse`, and a full active/receipt quota returns `QuotaExceeded`; the job is not registered, no transferId is created, and the OperationId remains reusable.

- `PluginTransferRejectedException` (derives from `InvalidOperationException`, deliberately not `IOException`; sealed) carries `Reason` (`RemoteTransferRejectReason`: `Unknown = 0`, `SourceNotFound`, `SourceInUse`, `QuotaExceeded`, `InsufficientSpace`). The RPC error code is `RpcErrorCodes.TransferRejected = "transfer-rejected"`, on the wire `transfer-rejected:source-in-use`; an unknown or missing suffix restores as `Unknown`; it comes before `InvalidOperationException` in `FromException`.
- Throwing it guarantees that the job was not registered, no transferId was created, nothing was read or written, and the OperationId was not recorded in the ledger. A plugin may try again once conditions change (the same OperationId will not hit "arguments differ"), but should not retry automatically right away.
- Keep it apart from the other two outcomes: "result unknown" can only be resolved by fetching the receipt again with the same OperationId; "user denied" (`PluginPermissionDeniedException`) is pointless to retry. Invalid arguments, unsupported platforms, missing sessions and OperationId argument conflicts keep their existing exception types. `IRemoteFsApi.StartUploadAsync` / `StartDownloadAsync` gained `<exception>` documentation.
- In the current host, insufficient space is detected while preparing the upload source snapshot after approval; it remains a registered job's `Failed` / `insufficient-space` snapshot (§3.3b), not a pre-registration refusal. `SourceInUse` detection is limited to Windows sharing/lock violations.

### 3.4 UI and extension points

| Capability                          | Key methods                                          | Notes                                                                                                                                                      |
| ----------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Commands` (`ICommandsApi`)         | `Register` / `TryExecute`                            | Command ids must be prefixed by the plugin id; manifest placeholders are replaced by real handlers on activation                                           |
| `Ui` (`IUiApi`)                     | `ShowPanelAsync(options, contentFactory)`            | Present your own Avalonia controls: dockable main-window tabs in `inProcess`, standalone card windows in `isolated`                                        |
| `TerminalView` (`ITerminalViewApi`) | `Create(...)`                                        | **Borrows the host's terminal emulator** (VT parsing, screen buffer, selection, IME, key encoding) as a control you embed in your own UI. `inProcess` only |
| `Protocols` (`IProtocolsApi`)       | register a protocol implementation                   | Your own remote **file** protocol, a first-class citizen of the connection page next to SSH/SFTP/FTP. `inProcess` only                                     |
| `Workspaces` (`IWorkspacesApi`)     | register a workspace provider                        | **Non-file** connection types (Redis, MySQL, …) whose session document you render entirely. `inProcess` only                                               |
| `Clipboard` (`IClipboardApi`)       | text read/write                                      | System clipboard                                                                                                                                           |
| `Events` (`IHostEvents`)            | session connect/disconnect, theme and locale changes | Subscriptions are cleaned up by the host on deactivation                                                                                                   |
| `Theme` (`IHostThemeApi`)           | `Current` / `Colors` / `GetColor` / `Changed`        | Theme identity, the fully resolved `Vela*` palette, and a signal that covers every kind of theme change. See §3.5                                          |

The host routes first-party `velashell.ai` `DisplayMode.Document` panels to a dedicated right region. The host owns width and pin/auto-hide preferences rather than using `PlacementRatio` or emitting `PlacementRatioChanged`. Hiding or collapsing keeps the handle open, and `ActivateAsync` restores the same control. Ordinary plugin document width contracts and the SDK package remain unchanged; see [UI extensions](../plugins/08-ui-extensions.md) for the full host policy.

> **The four `inProcess`-only capabilities** (`RemoteTunnel`, `TerminalView`, `Protocols`,
> `Workspaces`) throw "capability unavailable" in isolated mode. They hand out live native
> objects or raw streams, which have no equivalent across a process boundary — if you need them,
> set `hostMode` to `inProcess`.

### 3.5 Theming (SDK 2.0)

**You do not need this interface to colour your UI.** In Avalonia, always write
`{DynamicResource VelaXxx}` — the host derives those tokens per theme and swaps the whole set
at once, and they follow automatically in both `inProcess` and `isolated` mode.

What this interface is for is the three places `DynamicResource` cannot reach:

1. you need a `Color` rather than a `Brush` (syntax-highlighting definitions, custom drawing, image export);
2. you resolve colours once in code (converters, computations outside a control template) — those need a "re-read now" signal;
3. you want to branch on **theme identity** (a different illustration or icon set for one theme).

```csharp
HostThemeInfo theme = ctx.Theme.Current;
// theme.Id            "tokyo-night"  — already resolved, never "system"
// theme.Name          "Tokyo Night"
// theme.IsDark        true
// theme.Variant       "dark"
// theme.FollowsSystem whether the user picked "follow system"
// theme.Accent        "#7AA2F7"      — includes the user's accent override

string? bg = ctx.Theme.GetColor("VelaBgTerminal");   // "#FF1A1B26"

ctx.Theme.Changed += info => { /* Current and Colors are already updated */ };
```

> **Do not use `Host.Theme` to detect a theme change.** `IHostInfo.Theme` only ever holds
> `dark`, `light` or `system`: the host ships a dozen named themes and they are all collapsed
> to their light/dark name there. So for a VelaDark → Tokyo Night switch, the value of
> `Host.Theme` **and** the argument of `IHostEvents.ThemeChanged` are unchanged — code that
> compares that argument misses the change entirely. And when it reads `system`, it does not
> tell you whether the UI is currently light or dark. `IHostThemeApi.Changed` is the union of
> all three cases: named-theme switch, system light/dark flip, and accent override.

### 3.6 Tab icons (`PluginIcon`, SDK 2.0.4)

The host draws an icon on every session tab based on the connection type (one glyph for SSH,
another for SFTP / FTP, none for local terminals; see [Interaction and UI Specs §4B.2](../host/interaction-and-ui-specs.md#4b2-individual-tab-content)). **A plugin's tab gets a generic plug by default** — the host does not
know whether you are Redis or something else, and it should not: a host-side
"plugin id → icon" lookup table is a table third-party plugins can never get into.

So the plugin hands the icon over itself. **All three places take the same type**, `PluginIcon`:

| Where | What it draws | When you supply it |
| --- | --- | --- |
| `ProtocolDescriptor.Icon` | Session tabs for that protocol | At registration, one per protocol |
| `WorkspaceDescriptor.Icon` | Session tabs for that workspace | At registration, one per workspace |
| `PanelOptions.Icon` | That panel's tab **and its title bar in window mode** | Per panel opening |

The last one is per-opening rather than per-plugin on purpose: the chat page and the settings
page opened by the same plugin can each carry their own glyph.

`PluginIcon` has just three fields (`init`, a record type):

| Field | Type / default | Meaning |
| --- | --- | --- |
| `PathData` | `string`, **required** | SVG path data; several segments can simply be joined with spaces. If the host fails to parse it, it counts as not given |
| `ViewBoxSize` | `double`, default `24` | The native view-box size of `PathData` (always 24 for lucide). The host treats a non-positive or non-finite value as 24 |
| `IsFilled` | `bool`, default `false` | Whether it is a solid, filled shape; `false` renders it stroked |

The plugin does not pick the color: on session tabs (protocol / workspace) the host strokes and fills
with the connection's identity color (the same as the tab's color strip), and on panel tabs with the
accent color. When the user picks a tab icon in the connection profile, that icon overrides the one
the plugin reports.

Two factory methods cover every real case:

```csharp
// A lucide stroke glyph: copy the path, the view box is always 24
Icon = PluginIcon.Stroked("M6 12h12 M6 8h12a4 4 0 0 1 0 8H6a4 4 0 0 1 0-8Z")

// A brand logo: filled, and its view box is usually not 24 — say both
Icon = PluginIcon.Filled(RedisLogoPathData, viewBoxSize: 1030)
```

Three things that bite:

1. **An unreported view box means 24.** Brand logos are commonly exported at 1024 or so; leave
   it out and the host scales against 24, **blowing the glyph up more than fortyfold** — that is
   usually what is behind an "icon does not show up" report.
2. **Getting stroke vs fill wrong raises no error, it just stops being recognisable.** Stroking a
   filled shape yields its outline; filling a stroke glyph yields a pile of blobs.
3. **A malformed path will not take the UI down.** If the host fails to parse it, it treats the
   icon as absent and falls back to the generic plug.

> ⚠️ This is **not** the same thing as `PanelTitleAction.IconPathData`. That one draws an action
> button on a **window title bar**, is always a 24×24 stroke glyph, and has no notion of fill or
> view box (see [the development guide §5.9](../templates/dev-guide.md)).

### 3.7 Posting to the message center (SDK 2.1.3)

**The contract is officially published in SDK 2.1.3 on NuGet.org, and the host now supports it.** The in-process `PluginUiApi` and isolated `ui/notify` route enforce plugin identity, de-duplication, rate limiting and per-plugin muting; messages carry the plugin source and are removed on uninstall. A plugin that uses it declares `minSdkVersion: 2.1.3` and still checks `SupportsNotifications` before calling.

```csharp
if (ctx.Ui.SupportsNotifications)
{
    PluginNotificationOutcome outcome = await ctx.Ui.PostNotificationAsync(new PluginNotification
    {
        Id = "backup-failed:web-01",          // a de-duplication key, stable within this plugin
        Title = "Backup failed",
        Body = "The nightly backup of web-01 did not finish.",
        Level = PluginNotificationLevel.Warning,
        ActionCommandId = "acme.backup.open", // must be one of this plugin's commands
        ActionLabel = "View",
    });
}
```

- `PluginNotification` (record): `Id` is required, a de-duplication key stable within this plugin, 1–128 characters without control characters; `Title` is required plain text, truncated by the host past 200 characters; `Body` is plain text, truncated past 1000; `Level` (`Info` / `Warning` / `Critical`, mapping one to one onto the host's severities);
  `ActionCommandId` must be one of this plugin's commands (starting with `"<pluginId>."`), and `ActionLabel` is allowed only with a command; `Validate(pluginId)` is the one validation shared by the host, PluginHost and the test doubles. **There is deliberately no URL field and no "sender" field.**
- `IUiApi.SupportsNotifications` (default false) and `PostNotificationAsync(notification, ct)` (default implementation throws `NotSupportedException`) both have default implementations, so existing implementations still compile. It returns `PluginNotificationOutcome`: `Unknown = 0`, `Posted`, `AlreadyPosted`, `Muted`, `RateLimited`.
- Conventions: the message center takes only messages **worth keeping and reading again**; progress, runtime state, operation receipts and debugging output do not belong there ([message center and feed](../host/message-center-and-feed.md) §1). De-duplication comes before rate limiting: posting the same id again returns `AlreadyPosted` and does not count; each plugin may post at most 6 new messages a minute (the host may be stricter), beyond which it returns `RateLimited` without throwing; when the user has muted the plugin it returns `Muted`. Messages survive restarts within the message center's overall cap; uninstalling a plugin deletes its messages; a plugin cannot edit or withdraw a posted message. Action commands are best declared in the manifest's `contributes.commands`, so clicking works even before the plugin is activated.
- Permissions: no new permission — the ordinary `ui.contributions` already includes "notifications (rate limited)" ([06 §1](../plugins/06-permission-system.md)); `notifications.system` is about operating-system notifications and unrelated. The user's control lives on the host side: per-plugin muting on the plugin management page.
- Isolated processes: RPC `ui/notify` (`UiNotifyRequest` / `UiNotifyResponse`), with the handshake's `HandshakeResponse.SupportsNotifications` false when absent ([IPC §7.4](../plugins/05-ipc-protocol.md#74-notifications-pre-start-refusals-and-whole-frame-writes-sdk-213)).
- Test double `FakeUi`: `SupportsNotifications` (default true), `NotificationRequests` (every post), `PostedNotifications` (the ones actually queued), a `NotificationOutcome` script (to simulate rate limiting and muting) and `PluginId`; validation and de-duplication match the host, and `TestPluginContext` passes its PluginId on to it.

### 3.8 Version comparison `SemanticVersion` (SDK 2.1.3)

**Officially published in SDK 2.1.3 on NuGet.org.** The host now uses `SemanticVersion` for update checks, release notes, plugin load gates, plugin updates and feed version targeting. CLI and marketplace migration must be checked separately, not inferred from the host package upgrade.

- `VelaShell.PluginSdk.Versioning.SemanticVersion` (readonly struct): `Major` / `Minor` / `Patch` / `Revision` (the fourth part, 0 when absent), `PreRelease` / `BuildMetadata` (never null), `IsPreRelease`; `TryParse` / `Parse` (throws `FormatException` when invalid).
- `CompareTo` follows the full SemVer 2.0 §11 precedence: `beta.2 < beta.10`, numeric identifiers compare by length then ordinally without overflow at any length, numeric identifiers rank below alphanumeric ones, alphanumeric ones compare in ASCII order case-sensitively, and with everything else equal more identifiers rank higher; build metadata is kept but not compared (`Equals` / `GetHashCode` agree).
  `CompareCoreTo` compares only the numeric parts, for load gates such as `minSdkVersion` / the minimum host version; `TextComparer` compares version strings, treating unparseable ones as oldest and ordering equal precedence by the raw text.
- Parsing keeps the lenient numeric parts the host and `plugin.json` allow (an optional `v` prefix, two or four parts, leading zeros in numeric parts), while suffixes follow SemVer strictly: no empty identifiers, only `[0-9A-Za-z-]`, and no leading zeros in numeric prerelease identifiers.
  So `1.0.0-beta.01` and `1.0.0-a..b` pass the manifest regex but are invalid here (whether to tighten manifest validation is a separate decision).

---

## 4. Manifest

The full field table is in the [Development Guide §3](../templates/dev-guide.md); the three version gates are
in [Packaging and Publishing §1.2](../templates/publishing.md). Three things bite most often:

1. `id` is immutable after publication (command prefix + data namespace).
2. Using newer SDK surface requires `minSdkVersion`, otherwise old hosts fail at runtime with
   `MissingMethodException`.
3. Declarative contributions (`contributes.commands` / `protocols` / `workspaces`) take effect
   **during discovery**, without loading any assembly — that is what makes zero startup cost and
   lazy activation possible.

---

## 5. SDK version history

`apiLevel` only moves on **breaking** changes (2.0 raised it from `1` to `2`); additive surface is gated by
`minSdkVersion`.
`apiLevel` only moves on **breaking** changes; additive surface is gated by `minSdkVersion`.
The discipline is "SDK major == apiLevel" — the whole 1.x series is `apiLevel 1`,
and **2.0 onwards is `apiLevel 2`**.

| SDK       | Added                                                                                                                                                                                                                                                                                                                                                                                                                                | Does a plugin need `minSdkVersion`?                    |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ |
| 1.0       | First contract                                                                                                                                                                                                                                                                                                                                                                                                                       | —                                                      |
| 1.1       | `ExecResult` gains stderr and exit code; streaming remote exec                                                                                                                                                                                                                                                                                                                                                                       | Yes, if used                                           |
| 1.2       | `IRemoteTunnelApi` (raw byte duplex stream)                                                                                                                                                                                                                                                                                                                                                                                          | Yes, if used (`1.2.0`)                                 |
| 1.3       | `ITerminalViewApi` (borrow the host terminal control)                                                                                                                                                                                                                                                                                                                                                                                | Yes, if used (`1.3.0`)                                 |
| 1.3.1     | Workspace **variants**: `WorkspaceVariant`, `VariantKey`/`Variants`, `NoCredentials`/`NoEndpoint`                                                                                                                                                                                                                                                                                                                                    | Yes, if used (`1.3.1`)                                 |
| 1.4       | `HostRegistry` (host self-registration for `vela-plugin`)                                                                                                                                                                                                                                                                                                                                                                            | **No** — toolchain surface, plugin code never calls it |
| **1.5**   | Connection-form additions for protocols: `ProtocolFeatures.NoEndpoint` (hide the port column), `ProtocolSettingKind.DynamicChoice` + `IProtocolChoiceSource` (choices fetched when the form opens), `AllowsCustomValue` / `HostKind` / `HostChoices` / `HostAllowsCustomValue` (editable combo boxes; the host column can be a combo too). Driven by the serial plugin — ports are hot-plugged, baud rates have non-standard values. | Yes, if used (`1.5.0`)                                 |
| **2.0**   | **Generation bump (`apiLevel` 1 → 2)** — see "Migrating to 2.0" below. The surface it carries is `IHostThemeApi` (`ctx.Theme`, see §3.5): theme identity, the fully resolved palette, and a `Changed` signal covering every kind of theme change. Before it, a plugin could only see the three values of `IHostInfo.Theme` — no named theme, no accent, and no change at all when switching within the same light/dark variant.      | No — express it with `apiLevel: 2`                     |
| **2.0.2** | The "open a saved session" trio on the sessions capability: `ISessionsApi.ListSavedAsync` / `OpenAsync` / `CloseAsync`, with `SavedSessionInfo`, `SessionOpenOptions` and `PluginSessionOpenException`. Before it a plugin could only act on machines the user had **already connected by hand**, which caved in half of every unattended use.                                                                                       | Yes, if used (`2.0.2`)                                 |
| **2.0.3** | First cut at tab icons: an `IconPathData` / `IconViewBoxSize` / `IconIsFilled` trio on each of `ProtocolDescriptor` and `WorkspaceDescriptor`. ⚠️ **That trio is superseded by `Icon` in 2.0.4 — do not use it.** Panels were missing entirely at the time, and spelling one concept out as three parallel properties in two places was a design mistake to begin with. Replaced the next day; no plugin was affected. | ❌ Withdrawn — go straight to 2.0.4 |
| **2.0.4**   | Tab icons collapsed into a single entry point, `PluginIcon` (see §3.6), shared by all three places: `ProtocolDescriptor.Icon`, `WorkspaceDescriptor.Icon`, and the **new** `PanelOptions.Icon` — panel tabs do not travel the descriptor path, which 2.0.3 missed. Ships with the `PluginIcon.Stroked` / `PluginIcon.Filled` factories. | Yes, if used (`2.0.4`) |
| **2.0.6** | `IProtocolStreamUpload`: an interface an `IProtocolFileSystem` may **optionally** also implement — write a sequential stream to a remote file (overwrite). Lets the host relay a file between two remote sessions through memory instead of a temporary file (the remote + remote dual-pane file tab). The stream belongs to the caller; it may not be seekable, so the length is passed separately; no resume. Not implementing it is fine: the host then treats the protocol as unable to upload from a stream. Additive, `apiLevel` unchanged. | Yes, if used (`2.0.6`) |
| **2.1.0** | `StreamSudoAsync`, non-sensitive sudo events/results and a dedicated fake; execution-operation negotiation, cancellation and ordered complete notifications. Officially published; see §3.3a. | Required when used (`2.1.0`); declared by the AI manifest |
| **2.1.1** | Independent upload jobs: `SupportsTransferOperations`, `StartUploadAsync` / `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` / `ResumeTransferAsync`, `RemoteTransferSnapshot` and the `fs/transfer/*` RPCs. Officially published; see §3.3b. | Required when used (`2.1.1`) |
| **2.1.2** | Independent download jobs: `SupportsDownloadOperations`, `StartDownloadAsync` / `RemoteDownloadOptions`, snapshot `Direction` / `Phase`, and the `fs/transfer/startDownload` RPC. Officially published; see §3.3c. | Required when used (`2.1.2`); declared by the AI manifest |
| **2.1.3** | File capabilities for plugin protocols: `RemoteFileEntry.IsSymbolicLink` / `LinkTarget`, `IProtocolLastWriteTime`, `IProtocolChecksum` and `RemoteChecksumAlgorithm` (§3.3d); transfers refused before they start, `PluginTransferRejectedException` and the `transfer-rejected` error code (§3.3e); posting to the message center, `IUiApi.PostNotificationAsync` and `ui/notify` (§3.7); version comparison `SemanticVersion` (§3.8); `RpcConnection` writes a frame whole or not at all ([IPC §7.4](../plugins/05-ipc-protocol.md#74-notifications-pre-start-refusals-and-whole-frame-writes-sdk-213)). Officially published; wired in the host, whose current central dependency is 2.1.4. Additive, `apiLevel` unchanged. | Required when used (`2.1.3`); the whole-frame fix involves no plugin code and takes effect with the package bump |
| **TBD** | Workspace variant icons and custom pages in the connection dialog (version number replaced at release): `WorkspaceVariant.Icon` (icon on the type-picker card; `null` falls back to the descriptor `Icon`), `WorkspaceDescriptor.Pages`, `ProtocolSettingPage(Key, Label)`, `ProtocolSettingField.Page` (takes precedence over `IsAdvanced`; a key not in `Pages` is treated as `null`; ANDed with `VisibleWhen`). Semantics in [Development Guide §5.14.1](../templates/dev-guide.md). Additive, `apiLevel` unchanged. | Required when used (TBD, replaced at release) |

### Migrating to 2.0

2.0 is the **first** time this series moves `apiLevel`. Bumping the major version takes the
contract assembly's `AssemblyVersion` from `1.0.0.0` to `2.0.0.0` (see `$(VelaSdkMajor).0.0.0`
in the SDK repo's `src/Directory.Build.props`), so **every already-compiled plugin must be
rebuilt** before a 2.x host will load it.

Two things to do:

1. Move `VelaShell.PluginSdk.Build` to the matching version, rebuild, and repack the `.vpx`.
2. Set `apiLevel` to `2` in `plugin.json`.

Step 2 is optional for running on a 2.x host (a host accepts any generation up to its own),
but it is what you want **when the plugin lands on a 1.x host**: you get the discovery-time
message "Plugin targets apiLevel 2, host supports up to 1. Update VelaShell." instead of an
opaque assembly-binding failure. That is precisely what `apiLevel` exists for.

The new surface itself is still purely additive: nothing removed, no signature changed. The one
source-level incompatibility is the new `Theme` member on `IPluginContext` — plugins **consume**
that interface and are unaffected; anyone who wrote their own `IPluginContext` implementation
(usually a test double) must add the member, or switch to `TestPluginContext`.

---

## 6. Testing without starting the host

```csharp
using VelaShell.PluginSdk.Testing;

[TestMethod]
public async Task Activate_RegistersCommand()
{
    using var context = new TestPluginContext();
    var plugin = new DemoPlugin();

    await plugin.ActivateAsync(context, CancellationToken.None);

    Assert.Contains("acme.demo.run", context.RecordingCommands.Registered);
    await plugin.DeactivateAsync(CancellationToken.None);
}
```

Available doubles: `CollectingLogger`, `InMemoryStorage`, `InMemoryTimeSeries`, `FakeSessions`,
`FakeRemoteFs`, `FakeRemoteExec`, `FakeRemoteTunnel`, `FakeTerminal`, `FakeTerminalViewApi`,
`FakeUi` (with notification members from SDK 2.1.3 on, see §3.7), `FakeSecrets`, `FakeClipboard`, `RecordingCommands`, `RecordingProtocols`,
`RecordingWorkspaces`, `TestHostEvents`, `TestHostInfo`.

Besides `AddConnected`, `FakeSessions` offers `AddSaved` (build a saved configuration) and three hooks:
`DenyOpen` / `OpenFailure` simulate "the user said no" and "it would not connect", while `LastOpenReason`
lets you assert that the reason shown to the user actually made it through — an implementation that passes
"the plugin needs to connect" works perfectly and still turns the confirmation dialog into a blind button.

What unit tests cannot cover (real UI, real sessions, real protocol tabs) belongs to the inner
loop: `vela-plugin dev init` → F5, see the [CLI Manual](../cli/cli.md).

---

## 7. Loading model: three rules

1. **One collectible ALC per plugin**; the plugin's own dependencies resolve inside its
   directory through `deps.json`, so you may reference any NuGet package without colliding with
   the host.
2. **Two assembly families are always shared**: `VelaShell.PluginSdk` and `Avalonia*` fall back
   to the loading side. Your Avalonia version must match the host's (the SDK package pins it),
   and third-party packages whose name starts with `Avalonia` cannot be used.
3. **Development plugins load from a shadow copy** (`~/.velashell/dev-shadow/<id>/gen-N`), so
   the running host does not lock your `bin` and Reload on the manager page picks up a rebuild.
   The production path does not use shadow copies.

---

## 8. Host registry (`HostRegistry`, SDK 1.4)

`VelaShell.PluginSdk.Hosting.HostRegistry` reads and writes `~/.velashell/host.json`, where the
host registers its executable path, version, apiLevel, bundled SDK version, Avalonia version and
data root on every launch.

It targets the **toolchain** (`vela-plugin dev init` / `doctor` / `hosts`); plugin runtime code
does not use it. If you write your own build script or IDE integration:

```csharp
HostRegistryEntry? host = HostRegistry.Resolve();          // most recently started
HostRegistryEntry? preview = HostRegistry.Resolve("1.5");  // by version
IReadOnlyList<HostRegistryEntry> all = HostRegistry.List();
```

Every read path returns an empty list instead of throwing when the file is missing or corrupt —
it is a cache that speeds up tooling, and a broken cache should mean "you have to point at the
path yourself", never "something fails to start".
