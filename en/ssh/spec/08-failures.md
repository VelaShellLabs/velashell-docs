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

〔Decision〕**Text from the peer is sanitized before it goes into `Message`**: control characters (`ESC`, `CR`, `BEL`, …), `DEL`, C1 control codes and bidirectional text controls
are replaced with `?`, and the text is truncated to a fixed length. The identification string, the `DISCONNECT` description, the reason for refusing a channel open, SFTP status messages
and proxy replies all come from the peer, often from a peer that has not been authenticated yet — and `Message` ends up printed to a terminal or a UI,
so embedding it verbatim is an escape-sequence injection surface (rewriting the clipboard, clearing the screen to fake a prompt, overwriting the start of a line).
**The verbatim text is still kept in dedicated properties** (`PeerDescription`, `ServerMessage`), which are treated as untrusted text.

Multi-line peer output (a remote command's stderr) **contributes only its tail**: the message from `SshCommandResult.EnsureSuccess` carries a cleaned summary of the last 1024 characters of stderr,
with line breaks collapsed into a single " ⏎ " so it stays on one line (a single newline is enough to forge another record in a log); the line saying what went wrong is usually the last one.
A signal name from the peer is cleaned the same way and cut to 32 characters. The full original stays in `SshCommandFailedException.Result`.

〔Decision〕**These rules are public** (`VelaShell.Ssh.Diagnostics.PeerText.Sanitize` / `SanitizeTail`): consumers who put the original text (`ServerMessage`,
`PeerDescription`, SFTP paths) into their own UI text use the same rules instead of writing their own — the host once appended the original text after a message that had already been cleaned,
showing the same sentence twice, with the second copy bypassing the cleaning.

---

## 2. Exception hierarchy

```
SshException                          Abstract base; carries Reason / Phase / IsRetryable
├── SshConnectException               Connection setup (dialing, version, negotiation, host key) — carries Hops
│   ├── SshNegotiationException       Algorithm negotiation failed — carries both sides' lists
│   └── SshKeyExchangeException       Key exchange computation failed (invalid public value from the peer)
├── SshAuthenticationException        Authentication failed — carries per-method attempt records
├── SshProtocolException              Peer violated the protocol
├── SshConnectionClosedException      Connection gone (closed by peer / keep-alive timeout / DISCONNECT received / aborted locally / host key changed on rekey)
├── SshChannelException               Channel could not be opened, or a request on the channel was refused — carries the reason code and the peer's text
├── SshCommandFailedException         Remote command did not succeed (EnsureSuccess) — carries the full result
├── SshForwardException               Forwarder could not start
├── SshPublicKeyException             Public key blob / text malformed, type unsupported
├── SshPrivateKeyException            Private key file cannot be read or decrypted — carries NeedsPassphrase
├── SshCertificateException           Certificate cannot be read, or does not pair with the private key
├── SshAgentException                 Cannot reach the agent, or the agent refused
├── SftpException                     SFTP operation failed — carries StatusCode and the server's original text ServerMessage
├── SftpTransferInterruptedException  Transfer interrupted — **carries DurableLength**
└── SftpUnavailableException          SFTP subsystem could not start
```

The types below each correspond to a single kind of failure, and their `Reason` and `Phase` are fixed:

| Type | `Reason` | `Phase` |
| --- | --- | --- |
| `SshNegotiationException` | `NegotiationFailed` | `KeyExchange` |
| `SshKeyExchangeException` | `ProtocolError` | `KeyExchange` |
| `SftpUnavailableException` | `Unsupported` | `Open` |
| `SshCommandFailedException` | `CommandFailed` | `Open` |

The types below have a fixed `Phase`, and their `Reason` is **reported as it actually is**:

| Type | `Reason` | `Phase` |
| --- | --- | --- |
| `SshPublicKeyException` | `KeyFormatInvalid`, `Unsupported` (type not supported) | `None` |
| `SshPrivateKeyException` | `KeyFileUnreadable`, `KeyFormatInvalid`, `KeyPassphraseRequired`, `KeyPassphraseIncorrect`, `Unsupported` | `None` |
| `SshCertificateException` | `KeyFileUnreadable`, `KeyFormatInvalid`, `KeyMismatch`, `Unsupported` | `None` |
| `SshAgentException` | `AgentNotRunning`, `AgentUnavailable`, `AgentRefused`, `LimitExceeded` (the key to add exceeds the message size limit), `ProtocolError` | `Authenticating` |
| `SshChannelException` | `ChannelOpenFailed` (with `OpenFailureReason`), `ChannelRequestRejected`, `LimitExceeded` (this side's channel count / session window budget reached its limit), `ProtocolError` (the peer announced a max packet of 0, so what is written to stdin cannot be sent) | `Open` |
| `SshForwardException` | `ForwardRejected`, `ForwardBindFailed`, `ForwardSetupFailed`, `LimitExceeded`, `ProtocolError`; when agent forwarding cannot be set up because the local agent cannot be reached, the agent side's `AgentNotRunning` / `AgentUnavailable` is carried over | `Open` |
| `SftpTransferInterruptedException` | Follows the cause of the interruption (the inner exception): a dropped connection is `ClosedByPeer`; a write the server rejected (disk full, quota, permission) takes the reason code of the inner `SftpException`; caller cancellation and local disposal are `Aborted`; a timeout waiting for acknowledgements on close is `Timeout`. 〔Decision〕It used to be fixed at `ClosedByPeer`, so callers deciding by reason code would treat "disk full" as a dropped connection and try to resume anyway | `Open` |

Reading a private key, reading a certificate and parsing a public key happen outside any connection (there may be no connection at all), so `Phase` is `None`.
When parsing the peer's host key fails during key exchange, the key exchange wraps it into an `SshConnectException` with `Reason` `HostKeyRejected`.

〔Decision〕**`Reason` must tell the truth, not be hard-coded.** Private key, certificate, agent and forwarding exceptions used to report `Unsupported` across the board,
a refused exec / pty-req / shell was reported as `ChannelOpenFailed` (the channel had in fact opened), and a jump-host cycle was reported as a retryable `ProxyRefused`.
Callers could then only parse the message sentence —— and the message is diagnostic text for developers, not UI copy, with no promise of stable wording.
If no suitable value exists, add one instead of borrowing a similar one; only when the exception type itself corresponds to a single kind of failure is the reason fixed in the type (the first table above).

A rejected host key has no dedicated type: it is an `SshConnectException` with `Reason` `HostKeyRejected`, and the policy's reason is its `Message` (§3).

〔Decision〕**Public APIs throw only the exceptions of this hierarchy.** Something from the peer that cannot be parsed (a truncated name list in KEXINIT, a frame with an empty payload) is handled the same way during the handshake and authentication
as during the session: `SshProtocolException` (`ProtocolError`, with `Phase` set to the step where it happened); a peer that disconnects in the middle of a packet is an `SshConnectionClosedException`
(`ClosedByPeer`). The parsing layer's exceptions are internal and kept as the inner exception. The handshake used to lack this mapping, and they leaked out of `ConnectAsync` as they were — callers' `catch (SshException)` could not catch them.
A frame with an empty payload is rejected at the frame layer (spec/01 §5) instead of being passed up.

〔Decision〕**Channel-level failures do not derive from connection-level failures.** A channel failing to open
(the server's `MaxSessions` is full) and the whole connection dropping are two different things,
and the upper layer's reconnect policy should apply only to the latter. Put them in the same inheritance chain,
and a caller's `catch (SshConnectionClosedException)` would swallow the former too.

〔Decision〕**`SshPublicKeyException` and `SshKeyExchangeException` are both under `SshException`.**
Both used to derive directly from `Exception`: callers using `catch (SshException)` as the catch-all for library failures missed them, and the host's exception translation did not recognize them;
`SshKeyExchangeException` also leaked to callers as-is during connection setup.
〔Decision〕**`SshKeyExchangeException` derives from `SshConnectException`**, on the same level as `SshNegotiationException`: it is a handshake-phase failure,
and getting-started says handshake-phase failures are `SshConnectException`. It used to derive directly from `SshException`, so callers that branch on that type (the host's exception translation)
sent it down the fallback branch and a connection failure was treated as a generic error. A failure during re-keying is also this exception (the session goes down with it), just like a negotiation failure during re-keying.
`SshPublicKeyException` has `Phase` `None` (see above); it used to have `KeyExchange`, a phase that does not hold when reading a local `.pub`.
`SshKeyExchangeException` has `Reason` `ProtocolError`: what it reports is almost always an invalid public value from the peer (wrong length, not on the curve, a weak value);
the "algorithm not implemented" kind is already stopped before connecting (`SshConnection.ConnectAsync` checks the algorithm lists first).
Should the lists and the implementation table ever disagree, that is the library's own programming error: it is reported as `InvalidOperationException`, not under the name of `SshException`.

### 2.1 A connection that dies mid-way: the cause is normalized to public types first

Whatever ends an **established** connection — any error in the receive loop, the send pump, keep-alive or rekeying —
is handed as-is to every caller on that connection and to every channel reader (the read throws it instead of completing as if the peer had sent EOF;
see `05-connection.md` §4.4). 〔Decision〕**Before it is handed out, it is normalized to the library's public types:**

| What ended the connection | What callers get | `Reason` |
| --- | --- | --- |
| Already an `SshException` | As-is | As-is |
| `OperationCanceledException`, `ObjectDisposedException` | As-is — that is the local side shutting down (cancellation, disposal), not a fault | — |
| The peer closed the connection in the middle of a packet | `SshConnectionClosedException` | `ClosedByPeer` |
| `IOException`, `SocketException` (socket dropped or reset) | `SshConnectionClosedException` | `ClosedByPeer` |
| Frame format or integrity check failure, message parse failure | `SshProtocolException` | `ProtocolError` |
| Anything else unexpected | `SshConnectionClosedException`, with the original exception in `InnerException` | `Unknown` |

The newly wrapped ones always carry `Phase` `Open`, even when the fault happened during a rekey (a known limitation); exceptions passed through keep their own `Phase` (a rekey timeout, for example, is `Rekeying`).
〔Decision〕**When the key exchange itself fails during a rekey** (signature verification fails, negotiation fails, the exchange reply is malformed, the peer sends `DISCONNECT`): the reason code is kept and `Phase` becomes `Rekeying`;
a protocol error is still an `SshProtocolException`, everything else is an `SshConnectionClosedException`, with the original exception attached as the inner one (for a negotiation failure, both sides' lists are still on it).
The exchanger does not distinguish the first exchange from a rekey and reports in first-exchange terms, `SshConnectException` / `KeyExchange` — that used to be passed through as-is, and callers that branch on type took a long-established connection for "could not connect".

Reason: this cause lands in the users' `catch` blocks and reconnect policies, and both of those branch on `SshException` and its `Reason`
(§3: automatic reconnect should apply only to "the connection dropped"). It used to be thrown as-is: internal parse exception types could not be caught by type;
raw socket exceptions bypassed `Reason`, so a reconnect policy could not tell it was a dropped connection; and the host's exception translation did not recognize them.

〔Decision〕**The connection hands out how it ended**: `SshConnection.Completion` (`Task<SshException>`) completes **with a result** when the connection ends —
on a drop it is the cause normalized by the table above (keep-alive timeout, peer closed, `DISCONNECT` received with its reason code and the peer's words, protocol error…),
and on local disposal an `SshConnectionClosedException` with `Aborted`; `CloseReason` is its synchronous form (`null` while the connection is alive).
It never completes faulted, so it never becomes an unobserved task exception when nobody awaits it. The `Disconnected` token only says "it's over" and cannot carry a cause —
consumers hooked on it used to report "closed by peer" for everything (the host's tunnel panel did so even when the user disconnected the session), or infer the cause from how the read pipe ended.

**During connection setup** (from the moment dialing succeeds until authentication finishes) the same rules apply, except that `Phase` records the step where the failure happened:

| What happened during setup | What callers get | `Reason` |
| --- | --- | --- |
| `IOException` / `SocketException` on the stream during version exchange, key exchange or authentication | `SshConnectionClosedException`, with the original exception in `InnerException` | `ClosedByPeer` |
| The peer closed the connection in the middle of a packet (during key exchange or authentication) | `SshConnectionClosedException` | `ClosedByPeer` |
| The peer closed the connection during the version exchange | `SshConnectException` | `ClosedByPeer` |
| Frame format or integrity check failure (during key exchange or authentication) | `SshProtocolException` | `ProtocolError` |
| Key exchange computation failed (invalid public value from the peer) | `SshKeyExchangeException` | `ProtocolError` |
| An exception thrown by a caller's own callback (host key policy, `IHostKeyTypePreference`, banner handler) | Returned as-is — it is not a library failure, so it is not classified as a dropped connection | — |
| A cancellation thrown by a callback itself (neither the caller's token nor the library's timer fired: the user clicked "Cancel" on a prompt) | `SshConnectException`; during authentication a `DISCONNECT(AUTH_CANCELLED_BY_USER)` is sent first | `Aborted` |

Depending on the step and on who notices it, the same "the connection dropped" can be an `SshConnectException` or an `SshConnectionClosedException`,
but `Reason` is always `ClosedByPeer` — to tell whether the connection dropped, look at `Reason`, not the type.
〔Decision〕A mid-packet close during key exchange used to be reported as `ProtocolError`, which made callers think reconnecting was pointless, and socket exceptions after dialing leaked out as-is. Both now match the rules for an established connection.
〔Decision〕IO errors inside callbacks (the host's trust store failing, the host unable to write its own files) used to be rewritten by the first row above into a retryable `ClosedByPeer` — retrying only fails again;
an `UnauthorizedAccessException` from the library's own read of `known_hosts` leaked out as-is. Now callback exceptions are returned as-is, and the library's own `known_hosts` read and write failures are reported as `HostKeyStoreFailed` (§3).
〔Decision〕**Only the library's own timers decide a timeout** (the connect timer, the authentication timer, the host key decision timer). A cancellation the caller did not request used to be reported as a timeout every time:
clicking "Cancel" on a one-time-code prompt produced "authentication timed out (limit 120 seconds)", and a host key decision with no time limit reported "decision timed out (-00:00:00.001)",
so the host had to record the cancel in its callback and claim it back after the failure. A cancellation requested by the caller still propagates as a cancellation.

The dialing phase is not covered here: each dialer maps its own failures to an `SshConnectException` with a reason (`DnsFailure`, `TcpRefused`, `ProxyRefused`… in §3; see `09-dialing.md`).
Timeouts, negotiation failures and rejected host keys during setup are likewise reported by each step itself (§3).

---

## 3. `SshFailureReason`

| Value | Meaning | Retryable | Typical next step |
| --- | --- | :-: | --- |
| `DnsFailure` | Host name cannot be resolved | ✔ | Check host name/DNS |
| `TcpRefused` | Connection refused | ✔ | Check the port / whether the service is running |
| `TcpTimeout` | Connection timed out | ✔ | Check firewall/network |
| `TcpUnreachable` | Network unreachable | ✔ | |
| `ProxyUnreachable` | The proxy itself cannot be reached (the specific cause is in `InnerException` and `Hops`, `09-dialing.md` §2.2) | ✔ | Check the proxy address and port and whether it is running |
| `ProxyRefused` | Proxy refused to forward | ✔ | **See §5.2** |
| `ProxyAuthRequired` | Proxy requires authentication, and no credentials are configured | ✘ | Configure proxy credentials |
| `ProxyAuthFailed` | Proxy rejected the configured credentials | ✘ | Correct the proxy username or password |
| `TlsFailed` | TLS handshake failed (`DialerChain.Tls`): untrusted certificate, mismatched name, or the peer does not speak TLS (`09-dialing.md` §4.4) | ✘ | See `Message`; a self-signed certificate needs its fingerprint pinned |
| `NotAnSshServer` | Peer does not speak SSH | ✘ | Wrong port |
| `VersionMismatch` | Protocol version is not 2.0 | ✘ | |
| `NegotiationFailed` | No algorithm in common | ✘ | **See §5.1** |
| `HostKeyRejected` | Host key rejected (`SshConnectException`, `Phase` `KeyExchange`): by policy (`SshHostKeyVerdict.Reject`) — unseen and not allowed to ask, declined by the user, `@revoked`, fingerprint not on the allow-list; `K_S` unparsable, signature does not verify, RSA too short, or not matching the negotiated algorithm (including a certificate algorithm negotiated while `K_S` is not a certificate, or vice versa); a CA-vouched host certificate that is invalid (`03-key-exchange.md` §5.5) | ✘ | Read `Message`: the policy's reason (`SshHostKeyVerdict.Message`) is placed there as-is, with fingerprints and `known_hosts` line numbers |
| `HostKeyChanged` | The host key **has changed**, in two situations: ① at the initial exchange the policy rejects with `SshHostKeyVerdict.RejectChanged` —— `KnownHostsPolicy` reports this when the recorded key has changed, or only other types are recorded (`SshConnectException`, `Phase` `KeyExchange`; the message carries the old and new fingerprints and line numbers); ② on rekey the host key the peer presents differs from the one pinned at the initial exchange (`03-key-exchange.md` §8.4), and the connection drops with `SshConnectionClosedException` (`Phase` `Rekeying`) | ✘ | Possibly a man-in-the-middle: do not reconnect automatically, and do not offer a "trust and remember" shortcut —— if the server really was reinstalled, have a person delete the old line from `known_hosts` |
| `HostKeyStoreFailed` | The host key records cannot be read or written: no permission on `known_hosts`, held by another process, disk full (`SshConnectException`, `Phase` is `KeyExchange`). When they cannot be read the connection is not allowed; when "trust and remember" cannot write them the connection does **not** fail, and the reason is recorded in `SshConnection.HostKeyPersistFailure` (`03-key-exchange.md` §5.4) | ✘ | See `Message`: it contains the file path and the IO error |

| `AuthenticationMethodExhausted` | All methods tried | ✘ | **See §5.3** |
| `TwoFactorRequired` | Server wants keyboard-interactive but we have none configured | ✘ | Prompt "this machine requires a one-time code" |
| `PasswordExpired` | Server requires a password change first, and it was not changed (no new-password callback, not changing it this time, the server did not change it, or out of attempts; 04 §5.1) | ✘ | |
| `KeyFileUnreadable` | A private key / certificate / public key file cannot be read (does not exist, no permission, I/O error) | ✘ | The message contains the path |
| `KeyFormatInvalid` | The content of a private key / certificate / public key is malformed (corrupt, truncated, invalid parameters) | ✘ | |
| `KeyPassphraseRequired` | An encrypted private key needs a passphrase, and none was given | ✘ | Show a passphrase prompt (`SshPrivateKeyException.NeedsPassphrase`) |
| `KeyPassphraseIncorrect` | A passphrase was given, but it does not decrypt this private key | ✘ | Ask for the passphrase again |
| `KeyMismatch` | The credential material does not match: the public key in the certificate does not pair with the private key, or a host certificate was used to log in | ✘ | A certificate must be used together with the private key it was issued for |
| `AgentNotRunning` | No ssh-agent is running locally: `SSH_AUTH_SOCK` is not set, the socket does not exist or nobody is listening on it, or the Windows named pipe did not appear within 3 seconds (`07-forwarding.md` §7.1). When agent forwarding cannot be set up, `SshForwardException` carries this reason code over | ✘ | Start the agent (Windows: `Start-Service ssh-agent`) |
| `AgentUnavailable` | All other cases where ssh-agent cannot be reached: the endpoint is not trusted (wrong owner on the named pipe), or communication failed midway | ✘ | See `Message` |
| `AgentRefused` | ssh-agent refused the request (the `ssh-add -c` confirmation was declined, the agent is locked, the key is no longer there) | ✘ | |
| `Timeout` | A phase timed out | ✔ | |
| `KeepAliveTimeout` | Declared dead by keep-alive | ✔ | **Automatic reconnect should apply only to this category** |
| `ClosedByPeer` | Peer closed actively | ✔ | |
| `Disconnected` | `SSH_MSG_DISCONNECT` received | ✘ 〔Not implemented yet: meant to depend on `DisconnectReason`; today never retryable〕 | The reason code is in `SshConnectionClosedException.DisconnectReason`, the peer's own words in `PeerDescription` |
| `ProtocolError` | Peer violated the protocol | ✘ | |
| `ChannelOpenFailed` | Channel could not be opened (`CHANNEL_OPEN_FAILURE`; the reason code is in `OpenFailureReason`) | ✘ 〔Not implemented yet: meant to depend on the reason code; today never retryable〕 | |
| `ChannelRequestRejected` | The channel opened, but an `exec` / `pty-req` / `shell` / `subsystem` on it was refused | ✘ | Check the server's `ForceCommand`, `PermitTTY` and `Subsystem` configuration |
| `ForwardRejected` | The server does not accept the forwarding request (`AllowTcpForwarding no`, `AllowAgentForwarding no` and the like), or this side refused an inbound channel that matches no forward | ✘ | |
| `ForwardBindFailed` | The local listening port cannot be opened (in use, no permission) | ✘ | Use another port |
| `ForwardSetupFailed` | Preparing the forward on the local side failed: no X display available, `xauth` could not run or failed | ✘ | |
| `LimitExceeded` | One of this side's limits was reached (concurrent channels, the session's total receive window budget, forwarded connections, agent / X11 channels) | ✘ | |
| `CommandFailed` | A remote command did not end with exit code 0 (the `SshCommandFailedException` thrown by `SshCommandResult.EnsureSuccess`) | ✘ | Look at `Result`: stderr, exit code or signal |
| `InvalidConfiguration` | The configuration itself does not hold: a `ProxyJump` cycle, too many hops, an invalid `ProxyCommand` template | ✘ | Fix the configuration —— otherwise the next attempt will fail the same way |
| `Aborted` | Aborted locally (Dispose / cancellation); also a cancellation thrown by a callback itself during connection setup (the user clicked "Cancel" on a prompt, §2.1) | ✘ | |
| `Unsupported` | The requested capability (algorithm, key type, format version) is not supported by the peer or by this library | ✘ | |
| `Unknown` | Unclassified: the connection ended because of an unexpected error (§2.1) | ✘ | Look at `InnerException` |

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
Whether to send it, and with which code:

| When | Reason code |
| --- | --- |
| The connection is disposed normally (queued frames are flushed first, within a time limit; `DISCONNECT` goes last, so once it is out, everything before it is out too) | `BY_APPLICATION` (11) |
| What the peer sent cannot be decoded or violates the protocol (during the handshake, authentication or the session; a failed integrity check at the frame layer is reported this way too, not as a separate `MAC_ERROR` — echoing check details back to the peer is a side channel; a peer public value failing its checks during key exchange is separate, see two rows down) | `PROTOCOL_ERROR` (2) |
| Algorithm negotiation fails | `KEY_EXCHANGE_FAILED` (3) |
| A peer public value fails its checks during key exchange (length, encoding, not on the curve, all-zero X25519, DH out of range, a non-prime GEX `p` or an out-of-range `g`, a hybrid method's total length; [03 §3.7.4](03-key-exchange.md)). The local reason is still `ProtocolError` | `KEY_EXCHANGE_FAILED` (3) |
| The host key is rejected, fails verification, or changes during a rekey | `HOST_KEY_NOT_VERIFIABLE` (9) |
| Authentication methods are exhausted (including a one-time code being required but not configured, and a required password change) | `NO_MORE_AUTH_METHODS_AVAILABLE` (14) |
| The user cancelled authentication at a prompt (§2.1) | `AUTH_CANCELLED_BY_USER` (13) |
| The connection is already gone (socket error, the peer left mid-packet), a timeout, a local configuration problem | Not sent — nobody would receive the first two, and the rest are not the peer's business |

It is not sent before the version exchange has completed (the peer may not even speak SSH yet). Each attempt waits at most 2 seconds, and a failure to send is not reported.

---

## 7. Metrics (`System.Diagnostics.Metrics`)

Meter name: `VelaShell.Ssh` (`SshMetrics.MeterName`; the forwarding set has its own Meter `VelaShell.Ssh.Forwarding`, see [`07-forwarding.md`](07-forwarding.md) §5).
Only the name is public; the instruments themselves are not — handing out writable counters would let anyone record into them. With no subscriber, each recording costs one check.

| Instrument | Type | Tags | When it records |
| --- | --- | --- | --- |
| `velashell.ssh.connections.active` | UpDownCounter | `host` | +1 when authentication completes, -1 when the connection ends (peer disconnect, failure, local dispose) |
| `velashell.ssh.connect.duration` | Histogram (ms) | `host`, `outcome`, `phase` | Once per `ConnectAsync`: from dialing to authentication success or failure. `outcome` is `Success`, an `SshFailureReason` name, `Canceled` (the caller cancelled) or `CallbackFailed` (a user callback threw); `phase` is where it stopped (`Open` on success) |
| `velashell.ssh.bytes` | Counter | `host`, `direction` | Every byte actually sent and received on the stream (identification string, packet headers, padding, MAC, after compression); matches `SshConnection.BytesSent` / `BytesReceived` |
| `velashell.ssh.packets` | Counter | `host`, `direction` | Once per packet; matches `SshConnection.PacketsSent` / `PacketsReceived` |
| `velashell.ssh.rekeys` | Counter | `host` | Once per completed rekey (matches `SshConnection.RekeyCount`) |
| `velashell.ssh.channels.active` | UpDownCounter | `host`, `type` | +1 when the peer confirms (or we accept a channel the peer opened), -1 when the channel winds down |
| `velashell.ssh.channel.window` | Histogram (bytes) | `host`, `type` | Actual values of the adaptive window: once when the channel opens, then again on every grow and shrink |
| `velashell.ssh.sftp.inflight` | Histogram | `host` | Actual values of the pipeline depth: on every request sent, the number in flight at that moment (itself included) |
| `velashell.ssh.forward.*` | See [`07-forwarding.md`](07-forwarding.md) §5 | | |

`direction` is `sent` / `received` (as in the forwarding set). `type` only takes known channel types (`session`, `direct-tcpip`, `forwarded-tcpip`, `x11`,
`auth-agent@openssh.com`, `direct-streamlocal@openssh.com`, `forwarded-streamlocal@openssh.com`); anything else is recorded as `other` —
the type of a channel the peer opens is a string the peer chose, and taking it as-is would let the peer decide the tag cardinality.

Live numbers for a single connection or channel do not need a metrics subscription; read the properties on the objects: `SshConnection.BytesSent` / `BytesReceived` / `PacketsSent` / `PacketsReceived`
(on the wire), `SshChannel.BytesSent` / `BytesReceived` (application data: the data part of `CHANNEL_DATA` and `CHANNEL_EXTENDED_DATA`, no protocol overhead; received bytes count whether or not they were read).

〔Decision〕**The `host` tag uses the "logical target", not the IP.** The final target on the jump chain is what the user recognizes: through jump hosts, every hop's connection
(including the jump hop's `connect.duration` and `bytes`) is recorded under the final target. Connections built directly in tests, without `ConnectAsync`, record no metrics.

〔Decision〕**Do not tag with key fingerprints, usernames and the like.** Tags go into a time-series database; cardinality explosion is one concern,
and writing usernames into widely queryable metrics is another.

---

## 8. Tracing (`ActivitySource`)

〔Status〕**Not implemented yet**; the table below is the plan.

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

readonly ref struct PacketTapRecord
{
    PacketDirection Direction;     // Inbound / Outbound
    byte            MessageNumber;
    int             Length;        // payload length (including the message number)
    uint            SequenceNumber;
    uint?           ChannelNumber; // only for channel messages (91–100)
    ReadOnlySpan<byte> Payload;    // **empty by default**, see below
}
```

Hooked up through `SshConnectionOptions.PacketTap`. Each hop on a jump chain is its own connection; to watch a hop, set it in that hop's connection options.

**Three hard rules**:

1. **Disabled by default.** When `PacketTap` is `null` (the default), the send and receive paths pay one null check.
2. **The payload is not provided by default.** Providing it requires explicitly setting `SshConnectionOptions.AllowPacketTapPayload = true`,
   and that option's documentation **must** state that it exposes passwords, keys and file contents.
3. **Payloads of authentication messages are never provided**, even with `AllowPacketTapPayload = true`: for messages numbered 50–79
   (the range [RFC 4252](https://www.rfc-editor.org/rfc/rfc4252) reserves for the user authentication protocol, keyboard-interactive prompts and answers included)
   the tap sees metadata only.
   〔Decision〕There is no switch for this — no troubleshooting scenario is worth logging a password,
   and if a switch exists, someone will certainly turn it on in production.

〔Implementation〕The tap sits at the two points where the framing layer sends and receives: inbound packets after decryption and decompression, outbound ones before compression and encryption — the tap sees the plaintext payload layer,
and the sequence number is that packet's number in its direction (reset by every `NEWKEYS` under strict KEX, see [`03-key-exchange.md`](03-key-exchange.md) §6).
`Payload` borrows the transport's buffer and is only valid for the duration of that `OnPacket` call (hence a `ref struct`); copy it to keep it.
The callback runs on the send/receive loop, so it must be fast and must not block; exceptions it throws are swallowed — the tap is user code, and its failures should not take the connection down.

Uses: connection diagnostics panel, protocol-level troubleshooting, record and replay.

---

## 10. Logging

Uses `ILogger`, but **logs are not an API**: anything users need to consume programmatically
must also appear in structured form in exceptions, events or metrics.

〔Current state〕**The library has no logger yet**: `SshConnectionOptions` has no `ILoggerFactory`, and the table below is a plan.
Wherever the specs say "log one debug entry", only the action itself (discard, ignore) is done today, with nothing recorded;
"the peer does not support strict KEX" can be read from `SshConnection.Algorithms.StrictKeyExchange`, but no Warning is logged.

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
