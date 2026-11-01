# 05 · IPC Protocol

> **Implementation note (2026-08)**: The isolated-process mode has been implemented along the lines of this document, but one protocol-selection detail intentionally differs: StreamJsonRpc + MessagePack was not introduced (the dependency tree is too heavy and conflicts with this repository's zero-dependency discipline; both processes use the same library and version, and there is currently no need for cross-language negotiation). Instead, the implementation uses a **length-prefixed + lightweight bidirectional JSON RPC** (`plugin-sdk/VelaShell.PluginSdk/Rpc/`: three message types, req/res/evt, no request serialization, a unified error-code space, and typed exception restoration).
> The transport is a named pipe (.NET uses UDS underneath on macOS/Linux), with a random name + one-time token + `CurrentUserOnly`. The handshake, error model, and “credentials never leave the main process” behavior remain consistent with this document. Large data uses inline base64 (≤16 MB) and same-machine file paths as an intermediary. The streaming subchannel and shared-memory surfaces (§6-2/3) have not been implemented. If a non-.NET SDK or streaming channels are needed in the future, revisit the StreamJsonRpc/encoding negotiation evaluation. See `Rpc/PluginRpc.cs` for the method list.

## 1. Layering

```text
Application layer   IHostEndpoint / IPluginEndpoint (strongly typed interfaces, defined in PluginProtocol)
Protocol layer      JSON-RPC 2.0 (StreamJsonRpc), MessagePack binary encoding
Framing layer       StreamJsonRpc built-in length-prefixed framing
Transport layer     Windows: NamedPipeServerStream (one per plugin, bidirectional, asynchronous)
                    macOS/Linux: UnixDomainSocket (socket file in the user runtime directory, 0600)
Side channels       ① Streaming data channel (marshaled stream / chunked protocol on the same RPC connection)
                    ② Shared memory (MemoryMappedFile, only for high-bandwidth scenarios such as image surfaces, see 08)
```

Selection rationale, expanding on D3:

- StreamJsonRpc: bidirectional calls, notifications, `CancellationToken`, and `IProgress<T>` are automatically supported across processes, with exception serialization and a mature, stable implementation (used by the VS/VSCode family). MessagePack encoding avoids the expansion of JSON text.
- gRPC is not used because HTTP/2, TLS, and cross-machine communication are unnecessary, and we do not want protoc code generation in the plugin developer inner loop. The protocol itself remains language-neutral (JSON-RPC over a pipe); a future non-.NET SDK can implement the framing layer and encoding independently, with a JSON-encoding negotiation option provided at that time.

## 2. Connection Establishment and Authentication

```text
1. The host creates a pipe/socket (random name) → starts PluginHost (name and token passed through environment variables)
2. PluginHost connects; if it does not connect within 10s → the host considers startup failed
3. First frame (authentication, raw frame, not JSON-RPC): magic "VELA" + authToken
   Validation failure → disconnect immediately and record a security log
4. Attach JSON-RPC and enter the handshake
```

## 3. Handshake

```jsonc
// PluginHost → host  host/hello
{ "protocolVersion": 1,
  "apiLevels": [1],                  // Set of apiLevels supported by PluginHost
  "pluginId": "acme.image-viewer",
  "pluginVersion": "1.2.0",
  "sdkVersion": "1.3.0",
  "encodings": ["messagepack", "json"] }

// Host → PluginHost  response
{ "apiLevel": 1,                     // Negotiated result (highest in the intersection; empty set → reject and report both sides' versions)
  "encoding": "messagepack",
  "hostVersion": "0.3.0",
  "locale": "zh-Hans",
  "theme": "dark",
  "capabilities": ["remoteFs","terminal","ui","storage", ...] }  // Capability domains actually available from the host
```

Before the handshake completes, every call other than authentication and hello is rejected. Afterward, the host calls `plugin/activate(reason)` to enter the plugin lifecycle.

## 4. Endpoint Interfaces (Application Layer, Excerpt)

```csharp
// Exposed by the host to plugins (forwarded through PluginHost). Method name = "<domain>/<action>".
public interface IHostEndpoint
{
    // —— Capability-domain calls (full list in 07; the implicit plugin identity is the first parameter for all calls, bound to the connection and impossible to forge) ——
    Task<RemoteEntry[]> RemoteFsListAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> RemoteFsOpenReadAsync(string sessionId, string path, CancellationToken ct);
    Task TerminalWriteAsync(string sessionId, string input, CancellationToken ct);
    Task UiPatchAsync(string surfaceId, UiPatch[] patches, CancellationToken ct);
    Task<PermissionState> PermissionQueryAsync(string permissionId);
    // ... Notifications (host → plugin, no response): event delivery
}

// Exposed by the plugin to the host
public interface IPluginEndpoint
{
    Task ActivateAsync(ActivationReason reason, CancellationToken ct);
    Task DeactivateAsync(string reason, CancellationToken ct);
    Task ExecuteCommandAsync(string commandId, JsonElement? args, CancellationToken ct);
    Task UiEventAsync(string surfaceId, UiEvent e, CancellationToken ct);   // VelaUI event callback
    Task OnHostEventAsync(HostEvent e);                                     // Subscribed event (notification)
    Task<string> PingAsync();                                               // Heartbeat
    Task OnEnvironmentChangedAsync(EnvChange e);                            // Language/theme change
}
```

Discipline:

- Interfaces appear only in `PluginProtocol`. All DTOs are immutable records. MessagePack keys are explicitly annotated with integer keys (safe against renaming). **New fields may only be appended as optional fields**. This is the mechanical safeguard for the apiLevel compatibility promise, enforced by binary compatibility checks in CI.
- The plugin identity is bound to the connection. Capability implementations read the pluginId from connection metadata. **The protocol has no parameter for “call as a particular plugin”**, preventing impersonation at the root.

## 5. Error Model

Unified error-code space (`JSON-RPC error.code`):

| Range | Meaning | Example |
| --- | --- | --- |
| -32xxx | Reserved by JSON-RPC | Method not found, invalid parameters |
| 1xxx | General host error | 1001 HostShuttingDown, 1002 CapabilityUnavailable |
| 2xxx | Permission | 2001 PermissionDenied (including permissionId), 2002 PermissionPromptDismissed |
| 3xxx | Remote session | 3001 SessionNotFound, 3002 SessionDisconnected, 3003 SftpError (with embedded VelaSsh error classification) |
| 4xxx | File system | 4001 NotFound, 4002 AccessOutsideScope |
| 5xxx | UI | 5001 SurfaceClosed, 5002 InvalidPatch |

The SDK restores error codes as typed exceptions (`PermissionDeniedException`, etc.). Unknown codes fall back to the `VelaPluginException` base class. Internal host exception details (stacks and paths) are **not sent across processes**. Only the classification code and a safe message are provided, preventing information leakage.

## 6. Large Data and Streams

Three strategies are selected according to data size and shape:

1. **Small (≤256 KB)**: inline as a MessagePack `byte[]` in the response, such as when reading a small configuration file.
2. **Streaming (file transfer, terminal-output subscriptions)**: StreamJsonRpc stream marshaling, a multiplexed substream over the pipe, with natural backpressure. Cancellation immediately terminates the stream. Terminal-output subscriptions additionally use **host-side throttling and coalescing** (frame coalescing in windows of at least 16 ms), incorporating the lesson from the performance pass: high-frequency reporting must always be throttled so a plugin subscription cannot slow the terminal hot path.
3. **High-bandwidth surfaces (image frames)**: MemoryMappedFile shared memory, with RPC carrying only frame metadata (MMF name, dimensions, stride, and sequence number), see 08 §4. The shared-memory segment is created by the host and access is restricted to the current user.

## 7. Ordering, Concurrency, and Reentrancy

- Requests on the same connection execute concurrently (the StreamJsonRpc default). **There is no global serialization**. Domains requiring ordering semantics, such as `terminal/write`, queue requests by sessionId inside the host capability implementation, matching the existing rule that terminal input may only be enqueued and must not be written directly to the stream.
- All host → plugin calls include a timeout (5s by default, 2s for UI events). A slow plugin must not block any host path (fire-and-forget + timeout observation).
- Cancellation: `CancellationToken` propagates across processes. Disconnecting a connection is equivalent to cancelling all unfinished calls on that connection. Writing a single frame is atomic (since SDK 2.1.3): cancelling while queued for the write gate means the frame is never sent; cancelling once writing has started only stops the caller waiting — the frame is still written completely and the peer still executes it (§7.4).

### 7.1 Execution operations (SDK 2.1.0)

This describes the official SDK 2.1.0 contract and execution operations in the current length-prefixed JSON RPC implementation, not implementation of every StreamJsonRpc/MessagePack design above. Execution RPC regressions have run against the restored official package; they do not validate a complete real GUI/plugin/remote flow.

- `HandshakeResponse.SupportsExecutionOperations` is an added init property, absent/default false. The new host sets true for execution operations, cancellation and ordered notifications. With false, PluginHost throws `NotSupportedException` before sending sudo RPC: no prompt or ordinary fallback. Ordinary streams retain the legacy host's two-hour RPC fallback; only the new protocol removes that substitute for real cancellation on unlimited streams.
- `ExecStreamRequest.OperationId` on `exec/stream` and `ExecOutputNotification.Sequence` on `exec/output` are added init properties, preserving existing constructors and Deconstruct signatures. New `exec/sudoStream` requests contain only `SessionId` / `Command` / `OperationId` / `TimeoutSeconds`; `exec/sudoEvent` contains `OperationId` / `Sequence` / a non-sensitive Event; `exec/cancel` contains only OperationId and is an idempotent plugin-to-host notification. Identity is bound to that plugin connection, with no passwords or impersonation parameter.
- Each plugin connection registers operations/CTS independently, linked to connection and plugin shutdown. Cancellation arriving before registration must take effect; exit/disconnect cancels and asynchronously cleans up operations. At most 128 early cancellations are retained; overflow fails closed, stopping registration and active executions rather than discarding cancellation and starting stopped work. Request frames carrying an OperationId are not withdrawn when the caller cancels: cancelling while queued for the write gate only stops the caller waiting, the request is still sent, and the following `exec/cancel` either arrives before registration (cancelling it on registration) or cancels the registered operation. Otherwise the request frame would never be sent while its cancellation still was, and such ids, which never see a request, would needlessly fill the early-cancellation limit. Other RPC capabilities are not generalized here.
- Output/state enter a bounded 256-item notification queue and are sent in order by one asynchronous pump, with consecutive sequence numbers starting at 1; the new protocol rejects nonpositive, missing or out-of-order sequence numbers. SSH read pumps (ordinary streams, per line) and the sudo executor (per event) await each enqueue: when the queue is full they wait for the pump to free a slot, so backpressure travels back to the remote through the SSH window and output with thousands of short lines cannot overflow the queue. Only callers that can report synchronously still fail explicitly and cancel the operation on a full queue instead of silently dropping output. If the pump fails, waiting producers are woken immediately with the original failure. Drain before returning a terminal result. Ordinary Lines must match delivered callbacks; sudo OutputChunks must match Output events only. Missing tail events cannot succeed. Callbacks must not block or synchronously wait for RPC on the same connection; `Progress<T>` or unobserved notifications do not guarantee order.
- Execution-specific strict `NotifyExecutionAsync` propagates write failures. IO/disposal failures may leave a partial length header/body, so it terminates that RPC connection, fails all pending requests and cancels its executions, rather than sending an error terminal frame on the damaged connection. This is not ordinary `NotifyAsync`'s best-effort behavior. Since SDK 2.1.3 "a half-written frame invalidates the whole connection" applies to every write (requests and plain `NotifyAsync` included); see §7.4.
- `cancelled` / `timeout` / `not-supported` restore distinct cancellation, timeout and unsupported errors; a user stop is not an ordinary failure and missing results are not success. Delivery of cancellation does not confirm remote process termination.

SDK contracts, strict-send/partial-frame regressions and host `ExecutionRpcTests` against the official SDK 2.1.0 assembly have actually passed. The missing-sudo-type compilation blocker is resolved. An initial RPC timeout came from a fixture that had not started its read pump; it passed after fixing the fixture, rather than ignoring the timeout. **RPC regression coverage is not acceptance of a real GUI-to-remote flow.** See [Automation and AI §1.4](11-automation-and-ai.md#14-managed-execution-and-recovery-in-the-ai-plugin-sdk-210) for release and verification boundaries.

### 7.2 Independent upload jobs (SDK 2.1.1)

The host still streams file content from a local path rather than embedding it in JSON. New `fs/transfer/startUpload|get|wait|cancel|resume` requests exchange only job parameters and immutable snapshots; the transfer is not tied to one long RPC. Existing synchronous `fs/upload` / `fs/download` are unchanged, and their legacy two-hour RPC fallback must not be mistaken for unlimited transfer support.

- Handshake `SupportsTransferOperations` defaults to false and is advertised only by an implementing host. Unsupported calls fail explicitly, never silently fall back to synchronous upload.
- Start and resume `OperationId` values share the plugin-lifetime namespace. The host deduplicates by authenticated plugin identity and arguments; replay must not upload twice. Request cancellation or a lost receipt does not prove that work never started: reuse the same id to retrieve the receipt, never generate another id and reupload.
- A `wait` window is 0–30 seconds, waiting only for increasing `Sequence` or terminal state. Expiry returns the current snapshot, and cancelling a wait does not kill the background transfer. Queries are authoritative; progress can be throttled and reread rather than requiring every notification as a log stream would.
- Cancellation carries `ExpectedGeneration`, preventing stale cancellation from stopping a resumed generation. Resume includes explicit `TargetSessionId`; the host validates connection identity and checkpoint, without accepting caller-supplied resume offsets.
- `Succeeded` requires successful remote finalization, validation and atomic commit. Full byte counts, a sent cancellation request or a closed connection prove neither success nor termination. An unresolved commit outcome is `Unknown`, never automatically replayed or resumed.

See [SDK §3.3b](../sdk/sdk-reference.md#33b-independent-upload-jobs-sdk-211) for DTOs, ownership, recovery and timeout budgets. The contract provides no cross-application-restart recovery and cannot remove remote storage, authorization or network constraints.

### 7.3 Independent download jobs (SDK 2.1.2)

Downloads add `fs/transfer/startDownload` (`PluginRpc.FsTransferStartDownload`, `FsTransferStartDownloadRequest`), exchanging only session/source/final destination and `RemoteDownloadOptions`, never file content. `HandshakeResponse.SupportsDownloadOperations` defaults to false, separately from upload support; absence means explicitly unsupported, not a synchronous fallback. Missing snapshot `Direction` means Upload and missing `Phase` means Unknown. Downloads explicitly carry Download and Approval/Transfer/Verification/Commit phases.

Get/wait/cancel, authenticated plugin ownership, the lifetime `OperationId` ledger, monotonic Sequence and generations are shared. Direction is part of idempotency, so reusing an id across directions is rejected. Cancelling start only ends receipt waiting and does not prove nothing started; wait expiry does not cancel downloads. Queries remain authoritative, without a separate download-push promise. Default no-overwrite and successful verification, close and safe commit precede Succeeded; Unknown/TimedOut without stop confirmation must not be automatically redownloaded. Downloads cannot resume. Both directions share 1024 receipts/32 active queued jobs/four actual transfer slots.

The download contract is officially published in SDK 2.1.2 on NuGet.org; host central packages, the explicit AI reference and minSdk all use 2.1.2. Downloads have their own regression tests; the upload regression results above are not used in their place. Safe commit supports Windows fixed local NTFS and local filesystems on Linux/macOS (x64, arm64); UNC/9p/SMB/FUSE/NFS and other filesystems fail explicitly without a legacy API fallback. See [SDK §3.3c](../sdk/sdk-reference.md#33c-independent-download-jobs-sdk-212).

### 7.4 Notifications, pre-start refusals and whole-frame writes (SDK 2.1.3)

**These contracts are officially published in SDK 2.1.3 on NuGet.org.** The host has wired `ui/notify` and `transfer-rejected`; whole-frame writes need no code change and take effect with the package bump.

- **`ui/notify`** (`PluginRpc.UiNotify`, `UiNotifyRequest(PluginNotification)` / `UiNotifyResponse(PluginNotificationOutcome)`): an isolated plugin posting to the message center. The handshake's `HandshakeResponse.SupportsNotifications` is false when absent,
  in which case PluginHost throws `NotSupportedException` inside the plugin process; when supported it validates locally first, then sends the request. Semantics in [SDK §3.7](../sdk/sdk-reference.md#37-posting-to-the-message-center-sdk-213).
- **Error code `transfer-rejected:<reason>`** (`RpcErrorCodes.TransferRejected`): a transfer explicitly refused before registration (source missing, in use, quota full, not enough space), restored as `PluginTransferRejectedException`, with an unknown suffix giving `Unknown`. See [SDK §3.3e](../sdk/sdk-reference.md#33e-transfers-refused-before-they-start-sdk-213).
- **A frame is written whole or not at all** (`RpcConnection.WriteAsync`, public API unchanged): the caller's cancellation token only covers waiting in line for the write gate — cancelled before acquiring it, not a byte of the frame is written; once acquired, stream writes no longer look at the caller's token,
  so a cancellation then returns `OperationCanceledException` to the caller at once while the frame is still written completely in the background and the peer still executes the request (the same rule as "request frames are not cancelled with the caller" in §7.1). A failure halfway through a frame retires the whole connection: queued writes write nothing more,
  in-flight requests all fail, the disconnect fires and the connection is disposed — a plain `NotifyAsync` is still best-effort per notification, but never keeps using a connection with half a frame on it (before, only `NotifyExecutionAsync` did this, §7.1). Calls waiting for the write gate also react to the connection closing and leave with `ObjectDisposedException`
  (`RpcDisconnectedException` inside `RequestAsync`) instead of hanging. The host and PluginHost need no code change; raising the package version is enough.

### 7.5 Conditional withdrawal (SDK 2.1.4)

**The contract is officially published in SDK 2.1.4 on NuGet.org, and the host now supports it.**

- **`fs/transfer/withdraw`** (`PluginRpc.FsTransferWithdraw`, payload `FsTransferWithdrawRequest(TransferId, ExpectedGeneration)`, reply `RemoteTransferWithdrawResult`): withdraws a transfer only while this generation's approval has not taken effect; the host decides and withdraws in one step, first come first served against the approval taking effect — there is no round-trip gap between "the plugin's `get` sees `Approval`" and "it sends `cancel`". `RemoteTransferWithdrawOutcome` travels as a number; an unrecognized value reads as `Unknown` and does not count as withdrawn.
- Handshake `HandshakeResponse.SupportsTransferWithdraw` is false when absent. The host advertises the actual capability of its capability instance; when it is not advertised PluginHost never sends this request, throws `NotSupportedException` inside the plugin process, and does **not** fall back to `fs/transfer/cancel`.
- Same ownership, generation and revocation rules as `cancel`: a mismatched generation is rejected, repeated requests for the same generation give the same outcome, and it still works for cleanup after revocation with a redacted snapshot; the request frame is not cancelled with the caller and a registered withdrawal is not rolled back. Invalid or foreign ids throw `FileNotFoundException` in process; in an isolated process that error currently restores as `RpcRemoteException` (`RpcErrorCodes` does not map `FileNotFoundException`, the same as for the other `fs/transfer/*` requests).

Full semantics in [SDK §3.3f](../sdk/sdk-reference.md#33f-conditional-withdrawal-sdk-214).

## 8. Development Plan (This Work Item)

| Task | Description | Dependency | Estimate |
| --- | --- | --- | --- |
| P-1 | Transport-layer wrapper: pipe/UDS implementations + authentication frame + connection lifecycle (disconnect events, half-open detection) | A-2 | 3d |
| P-2 | Handshake and apiLevel negotiation + encoding negotiation; readable errors for rejection paths | P-1 | 2d |
| P-3 | Initial finalization of endpoint interfaces and DTOs (activate/deactivate/command/ping/events); MessagePack integer-key discipline + CI binary compatibility checks | P-2 | 3d |
| P-4 | Error-code space + SDK exception mapping | P-3 | 1d |
| P-5 | Streaming channel: stream marshaling wrapper, terminal-subscription throttling and coalescing, backpressure tests | P-3 | 3d |
| P-6 | Shared-memory surface channel (creation/authorization/reclamation protocol) | P-3 | 2d |
| P-7 | Protocol fuzz testing: out-of-order frames, oversized frames, invalid MessagePack, calls during the handshake (with the H-8 chaos group) | P-1..P-4 | 2d |

Acceptance: two-process loopback benchmark, empty-call P95 < 1 ms and 1 MB stream throughput ≥ 500 MB/s (local pipe); protocol fuzz testing produces zero host crashes and zero hangs.
