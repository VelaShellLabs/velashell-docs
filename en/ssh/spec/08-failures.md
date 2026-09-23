# 08 · Failure Taxonomy and Observability

> Implementation: `Diagnostics/` (L9).
>
> What this file defines is a **contract**, not an implementation detail:
> users write their `catch` blocks and UI copy against it, so once released, none of the values here may be changed casually.
>
> 中文：[`../../../zh/ssh/spec/08-failures.md`](../../../zh/ssh/spec/08-failures.md)

---

## 1. Principle: exceptions carry data, not pre-assembled sentences

A failure must be able to answer four questions:

1. **What failed?** → `SshFailureReason` (strongly typed enum)
2. **At which step did it fail?** → `SshPhase`
3. **What did the peer say / what did we offer?** → structured context (algorithm lists, attempt records, status codes)
4. **What can be done next?** → `IsRetryable` + actionable information in the context

It is **forbidden** to put this information only in `Message` and leave callers to slice strings.
The reason is simple: that is exactly one of the things we set out to eliminate —
the underlying library hid the failure reason inside an internal exception and exposed only a fragment in the message prefix,
so the upper layer had to resort to `IndexOf(" - ")` to tell "authentication failed / timeout / negotiation failed" apart.
Code like that silently breaks when a library upgrade changes a single word of the wording.

`Message` is for humans; it is **not** an API.

---

## 2. Exception hierarchy

```
SshException                         Abstract base; carries Reason / Phase / IsRetryable
├── SshConnectException               Connection setup (TCP/proxy/version/negotiation/host key)
│   ├── SshNegotiationException       Algorithm negotiation failed — carries both sides' lists
│   └── SshHostKeyException           Host key rejected/changed — carries fingerprint and policy reason
├── SshAuthenticationException        Authentication failed — carries per-method attempt records
├── SshProtocolException              Peer violated the protocol
├── SshConnectionClosedException      Connection gone (closed by peer / keep-alive timeout / aborted locally)
├── SshChannelException               Channel-level failure
│   └── SshChannelClosedException     Channel closed
├── SshForwardException               Forwarding setup failed
└── SftpException                     SFTP operation failed — carries SftpErrorCode and the server's original text
    └── SftpTransferInterruptedException  Transfer interrupted — **carries DurableLength**
```

〔Decision〕**Channel-level failures do not derive from connection-level failures.** A channel failing to open
(the server's `MaxSessions` is full) and the whole connection dropping are two different things,
and the upper layer's reconnect policy should apply only to the latter. Put them in the same inheritance chain,
and a caller's `catch (SshConnectionException)` would swallow the former too.

---

## 3. `SshFailureReason`

| Value | Meaning | Retryable | Typical next step |
| --- | --- | :-: | --- |
| `DnsFailure` | Host name cannot be resolved | ✔ | Check host name/DNS |
| `TcpRefused` | Connection refused | ✔ | Check the port / whether the service is running |
| `TcpTimeout` | Connection timed out | ✔ | Check firewall/network |
| `TcpUnreachable` | Network unreachable | ✔ | |
| `ProxyRefused` | Proxy refused to forward | ✔ | **See §5.2** |
| `ProxyAuthRequired` | Proxy requires authentication | ✘ | Configure proxy credentials |
| `NotAnSshServer` | Peer does not speak SSH | ✘ | Wrong port |
| `VersionMismatch` | Protocol version is not 2.0 | ✘ | |
| `NegotiationFailed` | No algorithm in common | ✘ | **See §5.1** |
| `HostKeyRejected` | Host key rejected by policy | ✘ | See `SshHostKeyException.PolicyReason` |
| `HostKeyChanged` | Host key does not match the recorded one | ✘ | Show old and new fingerprints and let the user decide |
| `AuthenticationFailed` | An authentication attempt failed | ✔ | |
| `AuthenticationMethodExhausted` | All methods tried | ✘ | **See §5.3** |
| `TwoFactorRequired` | Server wants keyboard-interactive but we have none configured | ✘ | Prompt "this machine requires a one-time code" |
| `PasswordExpired` | Server requires a password change | ✘ | |
| `Timeout` | A phase timed out | ✔ | |
| `KeepAliveTimeout` | Declared dead by keep-alive | ✔ | **Automatic reconnect should apply only to this category** |
| `ClosedByPeer` | Peer closed actively | ✔ | |
| `Disconnected` | `SSH_MSG_DISCONNECT` received | Depends on `DisconnectCode` | |
| `ProtocolError` | Peer violated the protocol | ✘ | |
| `ChannelOpenFailed` | Channel could not be opened | Depends on reason code | |
| `Aborted` | Aborted locally (Dispose / cancellation) | ✘ | |
| `Unsupported` | The requested capability is not supported by the peer | ✘ | |

〔Decision〕**`IsRetryable` is the library's advice, not a promise.** It expresses
"whether this failure might go away on retry", not "you should retry" —
retry policy is the user's business (only they know whether a human is waiting or a batch is running).

---

## 4. `SshPhase`

```
Dialing → VersionExchange → KeyExchange → Authenticating → Open → Closing
                                 ↑                          │
                                 └──── Rekeying ←───────────┘
```

| Value | Description |
| --- | --- |
| `Dialing` | DNS, TCP, proxy handshake, jump chain |
| `VersionExchange` | Identification string exchange |
| `KeyExchange` | Initial KEX (including host key verification) |
| `Authenticating` | Authentication |
| `Open` | Normal operation |
| `Rekeying` | Re-negotiating |
| `Closing` | Teardown |

**A failure must carry its Phase.** The same `Timeout` in `Dialing` and in `Authenticating`
are completely different things: the former is a network problem, the latter is quite likely the user looking for a one-time code on their phone.

---

## 5. Three exceptions with structured context

### 5.1 `SshNegotiationException`

```
NegotiationCategory   Category       // KeyExchange / HostKey / EncryptionC2S /
                                     // EncryptionS2C / MacC2S / MacS2C / CompressionC2S / CompressionS2C
IReadOnlyList<string> OfferedByPeer  // the peer's KEXINIT list for this category, verbatim
IReadOnlyList<string> OfferedByUs    // the list we sent for this category, verbatim
string                PeerVersion    // "SSH-2.0-OpenSSH_9.6p1 ..."
```

**This directly replaces the practice of "after negotiation fails, open another plaintext TCP connection to probe the peer's KEXINIT".**
We already received that list during the handshake; we just never handed it over before.

〔Decision〕**`Message` gives a readable explanation of the difference**, but **the lists must also be provided in structured form** —
the UI needs to lay them out per category and highlight the category whose intersection is empty, which cannot be done by parsing `Message`.

〔Decision〕**Filter out the `*-cert-v01@openssh.com` variants when displaying** (done on the user's side; the library always provides the full list).
They are only relevant when the peer presents a certificate, and listing them buries six usable algorithms in twelve lines.

### 5.2 Proxy context of `SshConnectException`

```
IReadOnlyList<SshHopInfo> Hops   // each hop on the jump/proxy chain: kind, address, whether it succeeded, elapsed time
```

〔Decision〕Which hop on the chain the failure occurred at **must be discernible**.
"Cannot connect to 10.0.0.9" and "cannot connect to jump host jump.example.com" are completely different problems for the user,
yet today they look exactly the same.

`SshHopInfo` has at least `Kind` (`Tcp` / `Socks5` / `HttpConnect` / `SshJump` / `ProxyCommand`),
`Target`, `Succeeded`, `Elapsed`, `Detail`.
The filling rules (nearest to farthest, inner entries preserved as-is) are in `09-dialing.md` §2.2.

〔Decision〕When an HTTP CONNECT proxy refuses port 22, **give a suggestion directly in the message**
("the proxy may only allow 80/443; please use SOCKS5 or a direct connection instead").
This is a frequent failure that users have no way of guessing.

### 5.3 `SshAuthenticationException`

```
IReadOnlyList<AuthAttempt> Attempts      // see 04-authentication.md §3.4
IReadOnlyList<string>      ServerOffered // the methods the server last offered
bool                       PartialSuccessAchieved
```

`SkippedNotOffered` / `SkippedNoMaterial` in `AuthAttempt.Outcome` are the key:
without them, "the private key file could not be read" and "the server does not accept public key authentication" show up in the UI as the same
"incorrect username or password".

---

## 6. `SSH_MSG_DISCONNECT`

| # | Type | Field |
| :-: | --- | --- |
| 1 | `byte` | 1 |
| 2 | `uint32` | Reason code |
| 3 | `string` | Description (UTF-8) |
| 4 | `string` | Language tag |

| Code | Name |
| :-: | --- |
| 1 | `HOST_NOT_ALLOWED_TO_CONNECT` |
| 2 | `PROTOCOL_ERROR` |
| 3 | `KEY_EXCHANGE_FAILED` |
| 4 | `RESERVED` |
| 5 | `MAC_ERROR` |
| 6 | `COMPRESSION_ERROR` |
| 7 | `SERVICE_NOT_AVAILABLE` |
| 8 | `PROTOCOL_VERSION_NOT_SUPPORTED` |
| 9 | `HOST_KEY_NOT_VERIFIABLE` |
| 10 | `CONNECTION_LOST` |
| 11 | `BY_APPLICATION` |
| 12 | `TOO_MANY_CONNECTIONS` |
| 13 | `AUTH_CANCELLED_BY_USER` |
| 14 | `NO_MORE_AUTH_METHODS_AVAILABLE` |
| 15 | `ILLEGAL_USER_NAME` |

〔Decision〕**The description text given by the server must be preserved verbatim** (`SshDisconnectException.PeerDescription`).
It is often the only useful information (`"Too many authentication failures"`,
`"No supported authentication methods available"`).
It is also **untrusted text**, and the documentation must state that it is to be treated as untrusted content when displayed.

〔Decision〕**We also send `DISCONNECT` when we disconnect**, on a best-effort basis.
Not sending it leaves only a TCP reset in the server log, and the administrator cannot tell whether it was a network problem or the client disconnecting on purpose.

---

## 7. Metrics (`System.Diagnostics.Metrics`)

Meter name: `VelaShell.Ssh`

| Instrument | Type | Tags |
| --- | --- | --- |
| `velashell.ssh.connections.active` | UpDownCounter | `host` |
| `velashell.ssh.connect.duration` | Histogram (ms) | `host`, `outcome`, `phase` |
| `velashell.ssh.bytes` | Counter | `host`, `direction` |
| `velashell.ssh.packets` | Counter | `host`, `direction` |
| `velashell.ssh.rekeys` | Counter | `host` |
| `velashell.ssh.channels.active` | UpDownCounter | `host`, `type` |
| `velashell.ssh.channel.window` | Histogram (bytes) | `host`, `type` — actual values of the adaptive window |
| `velashell.ssh.sftp.inflight` | Histogram | `host` — actual values of the pipeline depth |
| `velashell.ssh.forward.*` | See [`07-forwarding.md`](07-forwarding.md) §5 |

〔Decision〕**The `host` tag uses the "logical target", not the IP.** The final target on the jump chain is what the user recognizes.

〔Decision〕**Do not tag with key fingerprints, usernames and the like.** Tags go into a time-series database; cardinality explosion is one concern,
and writing usernames into widely queryable metrics is another.

---

## 8. Tracing (`ActivitySource`)

Source name: `VelaShell.Ssh`

| Activity | When |
| --- | --- |
| `ssh.connect` | The whole connection setup; child spans are the Phases and the hops |
| `ssh.command` | One `RunAsync` |
| `ssh.sftp.<op>` | A single SFTP operation (created only when sampled; see below) |

〔Decision〕**Per-operation SFTP spans are not created by default.** One directory transfer produces tens of thousands of operations,
and the overhead of creating an Activity for each would itself change the performance characteristics. By default only
one aggregate span is created at the `SftpFileSystem` level; turn on fine granularity explicitly when needed.

---

## 9. Packet tap `IPacketTap`

```
interface IPacketTap
{
    void OnPacket(in PacketTapRecord record);
}

readonly struct PacketTapRecord
{
    PacketDirection Direction;     // Inbound / Outbound
    byte            MessageNumber;
    int             Length;        // payload length
    uint            SequenceNumber;
    uint?           ChannelNumber; // only for channel messages
    ReadOnlySpan<byte> Payload;    // **empty by default**, see below
}
```

**Three hard rules**:

1. **Disabled by default, and zero-overhead.** When the field is `null`, the entire call is eliminated by the JIT.
2. **The payload is not provided by default.** Providing it requires explicitly setting `TapOptions.IncludePayload = true`,
   and that option's documentation **must** state that it exposes passwords, keys and file contents.
3. **Payloads from the authentication phase are never provided**, even with `IncludePayload = true`.
   〔Decision〕There is no switch for this — no troubleshooting scenario is worth logging a password,
   and if a switch exists, someone will certainly turn it on in production.

Uses: connection diagnostics panel, protocol-level troubleshooting, record and replay.

---

## 10. Logging

Uses `ILogger`, but **logs are not an API**: anything users need to consume programmatically
must also appear in structured form in exceptions, events or metrics.

| Level | Content |
| --- | --- |
| `Trace` | Per-packet metadata (equivalent to `IPacketTap` without payload) |
| `Debug` | State transitions, negotiation results, window adjustments, pipeline depth changes |
| `Information` | Connection established/closed, authentication succeeded, forwarding started/stopped |
| `Warning` | Downgrades (SHA-1 signatures, no strict KEX), retries, single forwarded connection failures |
| `Error` | Connection-level failures |

〔Decision〕**"Peer does not support strict KEX" must be logged at Warning.** It does not block the connection (§03 6),
but users should be able to see it — it is a real missing security property.

〔Decision〕**Hot-path logging uses the `LoggerMessage` source generator.**
One Trace log per packet, if done with string interpolation, would become the main overhead during full-speed transfers.
