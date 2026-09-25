# VelaShell.Ssh — Architecture and Principles

> Status: **Decided; M0–M5 complete + user-facing connection layer** (2026-09-21). This document sets the direction, architecture and principles.
> The basis for implementation is not this document but the behaviour specifications under [`spec/`](../spec/) — see discipline #2 in §2.2.
> 中文：[`../../../zh/ssh/design/architecture.md`](../../../zh/ssh/design/architecture.md)
>
> Basis: a file-by-file read-through of `Tmds.Ssh` (MIT, 2026-09 snapshot, 28,769 lines) and VelaShell's current SSH stack
> (`Infrastructure/Ssh/` + `Core/Ssh/` + `Core/Sftp/`, about 8,600 lines).
> Every "current state" claim in this document can be pointed to a file and line number; nothing here is unverified.
>
> **2026-09-23 update**: this library has been merged into the host repository (`src/VelaShell.Ssh`, tests in `tests/VelaShell.Ssh.Tests`,
> engineering scripts in `scripts/ssh/`) and is no longer published as a separate NuGet package. The passages below about a standalone repository, a NuGet package and packaging smoke tests
> are the decision record of the time and are kept as they were; the paths in the text have been changed to their new locations in the host repository.
> The CI similarity gate formerly described in §2.4 was removed at the same time, and the related passages have been deleted.

---

## 0 One-line conclusion

**Worth doing — but the reason is not "we want to write one ourselves". It is that VelaShell has already written 1,200+ lines of patches to work around the underlying library,
and two capabilities (2FA, PTY pixel size) are stuck hard upstream.**

The approach: **a brand-new API + an independent implementation + an optional compatibility shim package**.
No fork, no file-by-file rewrite, no reuse of its type system — so the "plagiarism suspicion" is not something we argue away;
it **simply does not exist**: the internal models on the two sides are not the same thing.

---

## 1 Why it is worth building ourselves: the price VelaShell has already paid

Not a feeling — a list. Every row below is **host code that exists to work around a limitation of the underlying library**:

| # | Host code | Lines | The only reason it exists |
| :-: | --- | :-: | --- |
| 1 | `Infrastructure/Net/LoopbackProxyRelay.cs` | ~200 | Tmds only accepts `host:port` and has no transport extension point. Going through an HTTP/SOCKS5 proxy means **spinning up a loopback relay on the local machine** to fool it |
| 2 | `Infrastructure/Ssh/SshAlgorithmProbe.cs` | 199 | When negotiation fails the library does not hand over the peer's KEXINIT lists, so we **open another plaintext TCP connection to probe them** |
| 3 | `Infrastructure/Ssh/SshAlgorithmDiagnostics.cs` | 105 | Same as above: diff the probed lists against ours and turn the result into human language |
| 4 | `Infrastructure/Ssh/MeteredPortForwardHandle.cs` | 376 | The library does the forwarding data plane internally and exposes no counters. For the tunnel panel to show "3 connections · 1.4 MB" we had to **reimplement forwarding ourselves** |
| 5 | `Infrastructure/Ssh/OpenSshPrivateKey.cs` | 303 | The library's private key parser only accepts the OpenSSH envelope. Users' PKCS#1 / PKCS#8 / encrypted PKCS#8 keys are **silently skipped**, so the host converts them first |
| 6 | `ExtractConnectFailedReason` in `Infrastructure/Ssh/TmdsSshInterop.cs` | ~40 | `ConnectFailedException` is `internal`. To distinguish "authentication failed / timed out / negotiation failed" we can only **slice strings by message prefix** |
| 7 | `TmdsSftpClientWrapper.ResumeSafetyMargin` | — | The library does not guarantee contiguous writes. Before resuming a transfer we must **blindly back off 2 MB** (64 buffers × 32 KB), because the file length only means "highest acknowledged offset" and there may be holes before it |
| 8 | `TmdsSftpClientWrapper.PosixRenameFileAsync` | — | The library does not expose `posix-rename@openssh.com`, so we can only fall back to a plain rename — which is exactly the path some servers reject |
| 9 | The whole comment block in `InfrastructureServiceCollectionExtensions.AddCredential` | — | The default credential list must be **replaced wholesale** rather than appended to; otherwise on Windows every connection first hits an `ArgumentException` from `SSH_AUTH_SOCK` |

Plus **four blocked capabilities**:

| Capability | Status |
| --- | --- |
| **keyboard-interactive (2FA / OTP)** | Not implemented upstream at all. `KeyboardInteractiveSupportTests` is a "tripwire" test that asserts it is **not supported yet** — Google Authenticator / Duo on bastion hosts simply cannot connect, and the UI text can only say "this version cannot connect" |
| **PTY pixel size** | Blocked on [tmds/Tmds.Ssh#519](https://github.com/tmds/Tmds.Ssh/pull/519), neither merged nor released. The pixel fields of `pty-req` / `window-change` are always 0 |
| **Compression zlib@openssh.com** | We submitted [#513](https://github.com/tmds/Tmds.Ssh/pull/513) upstream ourselves; it is stuck awaiting merge |
| **SSH agent forwarding** | The one cell in the comparison matrix where all six competitors have it and we have nothing |

> **That is the entire rationale.** It is not that "the library is badly written" — Tmds.Ssh is quite well written: async-first, AOT-friendly,
> no legacy baggage, and migrating to it from SSH.NET was the right call. The problem is **the boundary**:
> it is an "SSH client library", while VelaShell needs **the SSH engine of a terminal product** —
> observable, pluggable, able to report failures in human language, able to feed the product's UI. Those two call for different API trade-offs.

---

## 2 How to actually settle the "plagiarism suspicion"

### 2.1 Get the legal facts straight first

Tmds.Ssh is **MIT**. MIT permits forking, modifying, closing the source, redistributing and selling commercially;
**the only obligation is to keep the copyright notice and the full license text**. Therefore:

- **Forking it is not illegal** — even selling it unchanged is not illegal (as long as the LICENSE comes along).
- The real risks **are not legal; they are in three other places**:
  1. **Reputation** — someone diffs it and finds "this is Tmds.Ssh with a new name"; for a product that sells licenses that is a serious wound.
  2. **Dual-licensing friction** — VelaShell is dual-licensed. MIT code mixed into closed-source distributions must carry attribution forever,
     and once it is mixed at file level, every future compliance review has to re-establish "whose lines are these".
  3. **Upgrade isolation** — after a fork you must manually rebase every upstream change; within a few months it drifts beyond merging,
     and you end up with "neither the freedom of writing it yourself nor the benefits of upstream". **This is the most expensive hidden cost of forking.**

So the goal is not "legal"; it is **provable independence**: anyone who runs a diff / similarity scan over the two codebases
should conclude "these are two different implementations".

### 2.2 Clean-room procedure (six rules, executable and auditable)

| # | Discipline | How |
| :-: | --- | --- |
| 1 | **Specifications first; source code is not a basis** | The only permitted bases for implementation are RFC 4250–4254 / 4256 / 4419 / 5656 / 8308 / 8332 / 8709, the `PROTOCOL*` files in the OpenSSH repository, and draft-ietf-sshm-*. **Every protocol implementation file states in its header which section of which document it implements**; if you cannot, it means you copied someone else's code |
| 2 | **Two-phase isolation** | The analysis phase produces **behaviour specifications** (pure natural language + message sequence tables, zero code snippets); the implementation phase looks only at the specs and RFCs. With AI assistance this rule is especially practical: analysis sessions and implementation sessions **do not share context** |
| 3 | **An entirely separate identifier scheme** | Do not reuse the **combination** of its class names / method names / field names / enum member names. §6.1 gives a mapping table. Note the word "combination" — colliding on a generic name like `SshClient` is fine; colliding on a whole set is evidence |
| 4 | **A genuinely different architecture, not a rename** | §4–§5 give **a different internal model** (Pipelines instead of Tmds's own Sequence type, a state machine instead of semaphore handshakes, a unified ledger instead of three pending mechanisms, IDuplexPipe instead of dual-buffer reads). This rule is the foundation of the others: **as long as the internal model really is different, the two codebases naturally won't look alike** |
| 5 | **Test vectors only from public sources** | RFC test vectors, NIST CAVP, the **ideas** behind OpenSSH regress cases. **Do not copy its test files** — not a single one |
| 6 | **An honest NOTICE** | See §2.4 |

### 2.3 What does **not** count as plagiarism (don't overreact)

These things **are necessarily identical**, and being identical is not infringement — they are facts dictated by the protocol,
protected by merger doctrine / scènes à faire (expression is not copyrightable when there is only one way to express it):

- Message number constants: `SSH_MSG_KEXINIT = 20`, `SSH_MSG_CHANNEL_DATA = 94` …
- Algorithm name strings: `"curve25519-sha256"`, `"chacha20-poly1305@openssh.com"` …
- Message field order and wire layout (fixed by the RFCs)
- The order in which inputs are concatenated for the exchange hash `H` (RFC 4253 §8)
- SFTP packet type numbers, `SSH_FX_*` error codes
- Limits hard-coded by the RFCs, such as the "35000-byte maximum packet length"

**Do not change these to "look different"** — changing them breaks protocol compatibility, and that would be the real disaster.

### 2.4 How to write the NOTICE

Honesty is far safer than concealment. Put a `NOTICE.md` at the repository root:

```
VelaShell.Ssh is an independently implemented SSH client library.

It is not a fork of any existing library and contains no code from other SSH implementations.
It is implemented from the IETF RFCs and OpenSSH's protocol documents (see the citations in each file header).

In its design we studied, and benefited from, the ideas demonstrated by the following projects —
their source code was not copied or adapted:
  - OpenSSH (BSD) — the de facto standard for protocol extensions and interoperability behaviour
  - Tmds.Ssh (MIT) — the pioneer of async-first SSH clients on .NET
  - libssh2 / golang.org/x/crypto/ssh — public reference implementations of message handling
```

**Include the Tmds.Ssh line.** An honest acknowledgement is ten times better than a silence someone later digs up;
and it states clearly two different things at once: "benefited from the ideas" and "did not copy the code".

---

## 3 Design principles (six of them, used to settle every disagreement that follows)

1. **Async is the only form.** No synchronous overloads, no `.Result`. Hot paths use `ValueTask` +
   pooled `IValueTaskSource` throughout.
2. **Backpressure is structural, not bolted on.** Buffering capacity is expressed jointly by `PipeReader`/`PipeWriter` and the SSH window;
   no layer may open an unbounded queue to **absorb** the peer's production rate.
3. **Failures must be structured.** Every point that can fail must be able to answer "what failed, what did the peer say,
   what did we offer, what should happen next". **Exceptions carry data, not pre-assembled English sentences.**
4. **Everything observable is exposed.** Byte counts, round trips, windows, in-flight requests, negotiation results, per-hop timing —
   whatever the product needs to display, the library must already know; if the library knows and doesn't say, the host has to rewrite it
   (which is exactly how item 4 in §1 came about).
5. **Pluggable beats configurable.** Rather than adding a 41st boolean switch, open an interface.
   Dialing, cipher suites, authentication methods, host key policy, SFTP extensions — all interfaces (§8).
6. **We do not write cryptographic primitives ourselves. One exception, on record.** We only do "protocol assembly".
   Whatever the BCL has comes from the BCL (AES / SHA-2 / ECDH / ECDSA / RSA / ML-KEM — which also gets hardware acceleration);
   whatever the BCL lacks comes from BouncyCastle (raw ChaCha20, standalone Poly1305, Ed25519 curve arithmetic, sntrup761,
   Argon2id).
   **The only exception is `Keys/BcryptPbkdf.cs`** (`bcrypt_pbkdf`, needed for encrypted OpenSSH private keys) —
   no existing primitive can be assembled into it, and not doing it means "the key most people actually have cannot be read".
   The scope, rationale and verification of the exception are in §11.2.18.
   〔Tightened 2026-09-21〕There used to be a "write it ourselves" exception for raw ChaCha20 as well; it was withdrawn during implementation:
   hand-written cryptographic primitives are a real liability for a library that is meant to be audited, and their bugs don't raise errors —
   they silently produce wrong results on particular inputs.

---

## 4 Architecture overview

```mermaid
flowchart TB
    subgraph API["VelaShell.Ssh — user-facing"]
        CONN["SshConnection<br/>state · lifecycle · events"]
        CMD["SshCommand · SshShell"]
        SFTP["SftpFileSystem · SftpFileStream"]
        FWD["PortForwarder · SshTunnelStream"]
    end

    subgraph CORE["Protocol core (same assembly, layered by folder)"]
        L7A["L7 auth method chain IAuthMethod"]
        L7B["L7 SFTP pipeline SftpRequestPipeline"]
        L6["L6 request ledger RequestLedger"]
        L5["L5 channel layer IDuplexPipe · adaptive window"]
        L4["L4 session state machine SessionMachine · SendGate"]
        L3["L3 cipher suite ISshCipherSuite"]
        L2["L2 framing layer PipeReader · PipeWriter"]
        L1["L1 dialing ISshTransportDialer"]
    end

    subgraph EXT["Extension surface"]
        DIAL["Socks5 · HttpConnect · SshJump · InMemory"]
        ALGO["Algorithm registry KEX·HostKey·Cipher·MAC·Compression"]
        SIGN["ISshSigner file·Agent·PKCS#11·KeyVault"]
        POL["IHostKeyPolicy known_hosts·CA·TOFU"]
        OBS["Metrics · ActivitySource · IPacketTap"]
    end

    API --> CORE
    L7A --> L6
    L7B --> L6
    L6 --> L5
    L5 --> L4
    L4 --> L3
    L3 --> L2
    L2 --> L1
    L1 -.-> DIAL
    L3 -.-> ALGO
    L7A -.-> SIGN
    L4 -.-> POL
    CORE -.-> OBS
```

**One package, one job.** (Decided 2026-09-21; the original plan was two: Core + main package.)

| Package | Contents | Runtime dependencies |
| --- | --- | --- |
| `VelaShell.Ssh` | All of L1–L9, layered by **folder** (`Protocol/` `Transport/` `Crypto/` `Session/` `Channels/` `Threading/` `Auth/` `HostKeys/` `Sftp/` `Forwarding/` `Diagnostics/` `Client/` `Config/`) | `BouncyCastle.Cryptography` (only for cryptographic primitives the BCL lacks) |

It is not split into `Core` + main package because, once split, the first thing users agonize over is "which one do I reference" —
and the answer is always "the main package". To date there is not a single scenario that needs only the protocol core.
**Extension points are opened through interfaces (§8), not by splitting assemblies** — splitting assemblies buys you a version-alignment burden,
not extensibility.

> The compatibility shim `VelaShell.Ssh.Compat.Tmds` (§6.3), if it is ever built, will be **a separate package** —
> its lifetime is finite (obsolete as soon as VelaShell finishes migrating) and it should not be tied to the main package's release train.
> This is decided at M6; M0–M5 reserve no interfaces for it.

---

## 5 Layer-by-layer design and principles

> Each section first states **how it fundamentally differs from the corresponding Tmds layer**, then why that difference is right.
> This is both a design explanation and the record of honouring discipline #4 in §2.2.

### 5.1 L1 dialing layer — `ISshTransportDialer`

```
ValueTask<Stream> DialAsync(SshDialTarget target, CancellationToken ct)
```

**Difference: Tmds has no such layer.** Its `ConnectCallback` / `ConnectContext` are `internal`;
the `Proxy` class is public, but its only member `ConnectToProxyAndForward` is `internal abstract`
— effectively **an abstract class that cannot be implemented from outside**. VelaShell's `LoopbackProxyRelay` was forced into existence by exactly this.

Built-in implementations: `TcpTransportDialer`, `Socks5Dialer`, `HttpConnectDialer`, `SshJumpDialer` (jump host),
`ProxyCommandDialer` (`ProxyCommand`), `InMemoryTransport.CreateDialer` (for tests, see §10.2).
The entry point is `DialerChain`; **nesting is done with `Via`**: every proxy-type dialer has an inner dialer for "how to reach the proxy itself",
defaulting to direct TCP. A jump host's nesting is written in the jump host's own connection parameters (it is, after all, a complete connection with its own way of dialing),
and `DialerChain.Jumps(a, b, c)` corresponds to `ProxyJump a,b,c`.
**Jump chains and network proxies are two applications of the same mechanism** — rather than, as today, jump hosts going through the library's internal `SshProxy`
while proxies go through the host's loopback relay, two completely different paths. The behaviour spec is `spec/09-dialing.md`;
`ProxyJump` / `ProxyCommand` from `ssh_config` land here directly via `SshConfigFile.CreateConnectionOptionsAsync`.

> **VelaShell gain**: `LoopbackProxyRelay.cs` is deleted entirely; the three pieces `PrepareProxyRelay` / `DisposeRelay` / `DescribeProxyError`
> in `TmdsSshClientWrapper` go with it.

### 5.2 L2 framing layer — handed to `System.IO.Pipelines`

**Difference: Tmds builds its own `Sequence` + `Sequence.Segment` + `SequencePool`
(250 + 100 + 60 lines), essentially re-implementing half of Pipelines**,
plus a `Packet` struct whose ownership is managed by hand with `Move()` / `Clone()`.

We use `PipeReader` / `PipeWriter` directly:

- Read: `ReadResult.Buffer` is a `ReadOnlySequence<byte>`; frames are sliced out **zero-copy**; `AdvanceTo` expresses consumption progress.
- Write: `GetSpan` / `Advance` / `FlushAsync`; backpressure is expressed by `PipeOptions.PauseWriterThreshold`.
- Ownership: **there is no ownership problem** — buffers belong to the Pipe; there is no "who should Dispose this Packet".

Two quantifiable gains (both facts read from the source, not inferences):

| | Tmds today | VelaShell.Ssh |
| --- | --- | --- |
| **Receive** | `StreamSshConnection.ReceiveAsync` only does `AllocGetMemory(4096)` each time; one 32 KiB SFTP data packet takes **8+ `stream.ReadAsync` calls** | PipeReader's per-read limit is adaptive (default 64 KiB); read in one go |
| **Send** | `SendLoopAsync` does one `stream.WriteAsync` per packet. 64 in-flight 32 KB SFTP writes = **64 writes per round** | The send gate takes packets in batches → one `FlushAsync`, **1–2 writes per round** |

### 5.3 L3 cipher suite — a single interface `ISshCipherSuite`

**Difference: Tmds splits it into two families, `IPacketEncryptor` / `IPacketDecryptor`,
plus a composed assembly of `TransformAndHMacPacketEncryptor` / `EncryptionCryptoTransform` / `IHMac`**;
AEAD (GCM, ChaCha) and "Transform + HMac" are two parallel class trees.

We make **one interface describe a whole suite**:

```
interface ISshCipherSuite : IDisposable
{
    CipherSuiteShape Shape { get; }   // whether the length field is encrypted · AAD length · tag length · block size · whether EtM
    int  Seal(ReadOnlySpan<byte> payload, ulong seq, IBufferWriter<byte> output);
    OpenResult TryOpen(ref SequenceReader<byte> input, ulong seq, IBufferWriter<byte> output);
}
```

`CipherSuiteShape` turns three **shape differences** — "chacha20-poly1305 encrypts the length field separately with another key",
"AES-GCM's length field is plaintext AAD", "EtM's MAC covers the ciphertext" —
into **data**; the framing layer only needs to look at `Shape` to know how to cut packets — instead of every decryptor
implementing "read 4 bytes first or decrypt first" on its own.

> The encryption algorithms themselves are not hand-written: AES-GCM uses the BCL's `AesGcm` (AES-NI hardware acceleration).
> `chacha20-poly1305@openssh.com` is OpenSSH's custom construction (**two keys, length field encrypted separately**);
> the BCL's `ChaCha20Poly1305` is the RFC 8439 one and is of no use — what is needed here is the raw ChaCha20 block function
> and a standalone Poly1305, both from **BouncyCastle**.
> AES-CTR uses the BCL's AES-ECB to generate the counter blocks one by one (the definition of CTR mode, NIST SP 800-38A §6.5),
> to get AES-NI hardware acceleration; the counter and XOR are bookkeeping, not cryptography.

### 5.4 L4 session state machine — explicit states + a send gate

**The difference here is the most critical point of this design.**

This is how Tmds does rekeying: the receive loop sees `SSH_MSG_KEXINIT` → creates a `SemaphoreSlim` and assigns it to
`_keyReExchangeSemaphore` → pushes its own KEXINIT into the send queue → `await`s that semaphore;
the send loop picks up the KEXINIT → **reassigns `_keyReExchangeSemaphore` to yet another new semaphore** →
releases the old one → `await`s the new one; the receive loop wakes up → takes the new semaphore → completes kex → releases it.
**One field, two assignments, held crosswise by two loops** — it works, but nobody dares touch it.

We replace it with **a one-way state machine + a send gate**:

```
Dialing → VersionExchange → KeyExchange → Authenticating → Open
                                 ↑                           │
                                 └────── Rekeying ←──────────┘
                                                   Closing → Closed
```

- **Send gate (SendGate)**: outbound frames fall into two classes — *transport-layer messages* (msg id 1–49) and *everything else*.
  The gate is a single boolean: during `Rekeying` only transport-layer messages pass; everything else **stays in the queue**
  (not dropped, not blocking the caller). Kex completes → gate opens → the backlog flows out in order.
- This is exactly what RFC 4253 §7.1 says ("…MUST NOT send any messages other than…");
  **the code is written the way the spec reads**, with no need for two loops handing semaphores to each other.
- State transitions are centralized in a single `SessionMachine.Advance(Event)`; illegal transitions throw immediately
  — rather than being scattered across `if`s in two loops.

The send path is **single-writer + batch coalescing**:

```
multiple producers → Channel<OutboundItem>(SingleReader) → SendPump
SendPump: take as many as possible per TryRead (capped at 64 KiB or 32 items)
          → process item by item (messages pass the gate then Seal into PipeWriter / close gate / switch keys / open gate and drain)
          → one FlushAsync → notify the senders waiting in this round (pooled IValueTaskSource)
```

> This is where the "1–2 writes per round" in the §5.2 table comes from.

Decisions made during implementation (2026-09-22, §11.2.19):

- **Closing the gate, NEWKEYS + key switch, and opening the gate and draining are each an item in the queue**, in the same queue as messages —
  "what was enqueued before the gate closed is sent as usual, what comes after is held back" and "no frame can slip in between NEWKEYS and the key switch"
  are both guaranteed by enqueue order, with no locks needed.
- **Backpressure applies at enqueue** (bytes queued ahead of the pump and not yet sealed onto the wire, including held-back ones, capped at 16 MiB),
  and only affects data-plane senders. **The receive loop and key exchange never wait on backpressure** —
  backpressure can only be relieved by the receive loop reading a `WINDOW_ADJUST` or the peer's NEWKEYS; making it wait means it is waiting on itself.
- **Messages the receive loop must send in reply are always "posted", never awaited** (CLOSE replies, request replies, OPEN_CONFIRMATION,
  UNIMPLEMENTED): a receive loop that stalls as soon as the TCP send buffer fills is exactly the deadlock where each end waits for the other to read.

### 5.5 L5 channel layer — `IDuplexPipe` semantics + adaptive window

**Difference one: the shape of the read API.** Tmds's
`SshChannel.ReadAsync(Memory<byte>? stdout, Memory<byte>? stderr, …)`
requires the caller to pass two nullable buffers at once, and uses two fields, `_skippingStdout` / `_skippingStderr`,
to remember "did we skip last time"; pass the wrong combination and it throws `InvalidOperationException`.
And the data is **copied** from the internal `Sequence` into the user's `Memory`.

We give each channel:

```
PipeReader StandardOutput;   // zero-copy: ReadOnlySequence<byte>
PipeReader StandardError;    // a separate one; the two cannot starve each other
PipeWriter StandardInput;
ValueTask<SshChannelEvent> ReadEventAsync(…);  // Eof / Closed / ExitStatus / ExitSignal / custom requests
```

The data plane and the control plane are separated — `ExitStatus` no longer shares the return value of the same `ReadAsync` with the byte stream.

**Difference two (the performance headline): adaptive window.**
Tmds's `DefaultWindowSize` is fixed at 2 MB, and the window is topped up when half has been consumed (that part is correct in itself,
and memory is bounded because of it). The problem is that **a fixed window also sets the throughput ceiling**:

```
throughput ceiling ≈ window / RTT
2 MB / 200 ms ≈ 10 MB/s   ← on a transoceanic link SFTP can never beat this number
```

We let the window adapt within `[256 KiB, 64 MiB]`, but **without estimating RTT or BDP**. The criterion looks only at two things visible locally:
whether the window **ran low** in this round (between two top-ups; ≤ 1/8 left), and whether the reader was **starved — read everything and waited** — recently (this round or the previous one).
Only when both hold does it double — with too small a window, a fast reader drains the data and waits a round trip; with a slow reader, running low is the reader's doing and a bigger window buys no throughput.
After 3 rounds that never ran low it shrinks ×0.75. No timing criterion like "how long between two top-ups": that is always true on a LAN and would inflate the window to the maximum for nothing.
The policy is exposed via `SshChannelOptions.WindowPolicy` (`SshWindowPolicy.Fixed(n)` / `Adaptive(min, max)`);
the exact rules are in `spec/05-connection.md` §3.3.
**Saves memory on a LAN, saturates bandwidth on transoceanic links.**

> The window top-up trigger hangs off `PipeReader.AdvanceTo` — if the consumer doesn't read, the window isn't topped up;
> backpressure is structural (principle 2), with no extra rate limiter needed.

### 5.6 L6 request ledger — `RequestLedger<TKey,TResult>`

**Difference: Tmds has three separately written pending mechanisms** —
global requests use `Queue<(TaskCompletionSource, bool)>` (FIFO, aligned by order, no id),
channel requests use the two success/failure messages,
SFTP uses `ConcurrentDictionary<int, PendingOperation>` + an `IValueTaskSource` pool it builds itself.
Cancellation semantics, timeout semantics, and cleanup on disconnect are each written their own way in the three places.

We build one: a pooled `IValueTaskSource<TResult>` + an id table + a single, unified path for
"connection drops → everything completes with the same exception", shared by all three.
Three fewer sets of concurrent code; the correctness of cancellation and disconnect **only has to be proven once**.

〔Correction during implementation, 2026-09-22〕The protocol actually has **two kinds** of ledger; forcing them into one would require introducing fake ids:

| Ledger | Where it is used | Alignment |
| --- | --- | --- |
| `FifoRequestLedger<TResult>` | Global requests, channel requests | No id, first sent first answered; **registration and enqueueing happen under the same lock** (`spec/05` §6.1) |
| The id table in `SftpRequestPipeline` | SFTP | Has a request-id; late replies arriving after cancellation need **cleanup** (closing handles the server has already opened) |

The first two share one implementation, and the completion semantics (disconnect → everything in flight completes as "failed") are written only once;
SFTP's "late replies need cleanup" is unique to it and stays with it.

### 5.7 L7 authentication — pluggable method chain

```
interface IAuthMethod
{
    string Name { get; }        // "publickey" / "password" / "keyboard-interactive" / ...
    ValueTask<AuthOutcome> RunAsync(IAuthChannel ch, AuthContext ctx, CancellationToken ct);
}
```

Built in: `none`, `password`, `publickey` (including certificates), **`keyboard-interactive`**,
`gssapi-with-mic`, `hostbased`, ssh-agent signing.

The callback shape of `keyboard-interactive` (**this directly unlocks VelaShell's 2FA**):

```csharp
new KeyboardInteractiveCredential(async (challenge, ct) =>
{
    // challenge.Name / .Instruction / .Prompts[i].Text / .Prompts[i].Echo
    return responses;   // the host pops a dialog, or takes it from a TOTP generator
});
```

Signing is decoupled from keys as `ISshSigner`:

```
interface ISshSigner
{
    SshPublicKey PublicKey { get; }
    ValueTask<byte[]> SignAsync(ReadOnlyMemory<byte> data, string algorithm, CancellationToken ct);
}
```

Built in: file private keys, ssh-agent, PKCS#11, Azure Key Vault — **the private key never has to enter the process**.
This is also where VelaShell's "credential manager integration" track lands.

Private key formats supported: OpenSSH v1 (including bcrypt_pbkdf encryption), PKCS#1, PKCS#8 (including encrypted), PuTTY `.ppk` v2/v3.

> **VelaShell gain**: `OpenSshPrivateKey.cs` (303 lines of PEM→OpenSSH conversion) is deleted entirely.

### 5.8 L7 SFTP — a three-stage pipeline

**Difference: Tmds's `SftpChannel.cs` is a 2,291-line partial class**;
encoding/decoding, pipelining, file semantics, directory enumeration, upload and download are all in it.

Split into three layers, each independently unit-testable:

| Layer | Responsibility | Testability |
| --- | --- | --- |
| `SftpWire` | Pure encoding/decoding: `bytes ↔ SftpMessage`. Stateless, no I/O | **Pure functions**; assert byte by byte against message samples |
| `SftpRequestPipeline` | Pipeline depth, in-flight window, reordering, cancellation, extension negotiation | Tested with `InMemoryDialer` + message scripts, no server needed |
| `SftpFileSystem` | User-facing: paths, attributes, directory enumeration, upload/download, progress | Integration tests |

**Three capability improvements:**

1. **Write watermark — eliminates `ResumeSafetyMargin` outright.**
   The pipeline records the "highest **contiguously** acknowledged offset" (contiguous acked offset, rather than "highest acknowledged offset"),
   and `SftpFileStream.DurableLength` exposes it. Resuming continues from that number, **no blind 2 MB back-off**;
   when the channel drops, it is written into the exception so the host can use it directly.
2. **OpenSSH extensions as first-class citizens**: `posix-rename@openssh.com`, `hardlink@openssh.com`,
   `fsync@openssh.com`, `statvfs@openssh.com`, `limits@openssh.com`, `copy-data`,
   `home-directory`. The negotiation result is exposed as `SftpCapabilities`, **so the host can ask "does this server support it"**
   — instead of, as today, `PosixRenameFileAsync` silently falling back to a plain rename.
3. **Block size from `limits@openssh.com`, pipeline depth adapted by "did requests wait"**, instead of hard-coding 64 × 32 KB:
   the server's `max-read-length` / `max-write-length` cap the block size of a single request;
   the in-flight request count starts at 64, and every 32 requests it checks how many of them **had to wait for an in-flight slot** —
   more than half waiting means depth is the bottleneck, so it doubles (up to 256); none waiting gives a quarter back (not below the starting value).
   No BDP estimate from RTT: that needs a bandwidth estimate first, which is unstable on a link that carries other traffic; "did we hit the limit" is more direct and harder to get wrong.

### 5.9 L8 forwarding and tunnels — metering inside the library

**Difference: Tmds does the data shuttling of `LocalForward` / `SocksForward` / `RemoteForward` internally
and exposes not a single counter.** VelaShell had to write the 376-line `MeteredPortForwardHandle` to redo forwarding itself
(local/dynamic listen on their own; remote forwarding even has the library forward to a temporary local listener and then relays once more — an extra loopback copy).

We make `PortForwarder` carry its own:

```
int  ActiveConnections { get; }
long TotalConnections  { get; }
long BytesUp { get; }  long BytesDown { get; }
event EventHandler<ForwardConnectionEventArgs> ConnectionOpened, ConnectionClosed;
event EventHandler<ForwardErrorEventArgs>      Error;
```

and these numbers also go through `System.Diagnostics.Metrics` (`velashell.ssh.forward.bytes` etc.),
so the host can either subscribe to the events or hook up OpenTelemetry — both paths work.

> **VelaShell gain**: `MeteredPortForwardHandle.cs` shrinks from 376 lines to a thin adapter (about 60 lines),
> and the "loopback relay" for remote forwarding disappears entirely.

### 5.10 L9 observability — failures must carry data

**① The negotiation report arrives with the exception.**

```
class SshNegotiationException : SshException
{
    NegotiationCategory   Category      { get; }  // KeyExchange / HostKey / Cipher / Mac / Compression
    IReadOnlyList<string> OfferedByPeer { get; }  // the peer's KEXINIT verbatim
    IReadOnlyList<string> OfferedByUs   { get; }
    string                PeerVersion   { get; }  // "SSH-2.0-OpenSSH_9.5"
}
```

> **VelaShell gain**: `SshAlgorithmProbe.cs` (199 lines, **opening another TCP connection just to get this list**)
> and `SshAlgorithmDiagnostics.cs` (105 lines) together shrink to a single pure formatting function.
> It also removes a side effect: reconnecting after a failure left an extra connection in the peer's logs that was established and immediately dropped.

**② Failure reasons are strongly typed, not string prefixes.**

```
enum SshFailureReason {
    DnsFailure, TcpRefused, TcpTimeout, ProxyRefused, ProxyAuthRequired,
    VersionMismatch, NegotiationFailed, HostKeyRejected, HostKeyChanged,
    AuthenticationFailed, AuthenticationMethodExhausted, TwoFactorRequired,
    Timeout, ClosedByPeer, KeepAliveTimeout, ProtocolError, Aborted
}
```

Every failure carries `Reason` + `Phase` (at which step) + `Attempts` (which authentication methods were tried and how each fared).

> **VelaShell gain**: the part of `TmdsSshInterop.ExtractConnectFailedReason` that
> "slices strings by the `"The connection could not be established - "` prefix" is deleted.
> Also: `TwoFactorRequired` turns "this machine wants an OTP" into a decidable state,
> rather than a vague "incorrect username or password".

**③ Packet-level tap (`IPacketTap`).**
An optional interface that receives the **metadata** of every sent and received message (direction, msg id, length, channel number, sequence number);
disabled by default with zero overhead (when the interface is null the whole block is eliminated by the JIT).
Used for: the connection diagnostics panel, protocol-level troubleshooting, record and replay. **Payloads are not provided by default**; to get them you must explicitly enable
`TapOptions.IncludePayload`, and the documentation must state that it will expose credentials.

---
## 6 Public API shape

### 6.1 Name mapping (which is also how rule 3 of §2.2 is honored)

| Tmds.Ssh | VelaShell.Ssh | Why the change |
| --- | --- | --- |
| `SshClient` | **`SshConnection`** | A different metaphor: the ADO.NET `SqlConnection` / SignalR `HubConnection` family — with `State`, `StateChanged`, `OpenAsync`/`CloseAsync`. **A "client" is an object; a "connection" is a thing with a lifecycle**, and the latter is what VelaShell actually has to manage |
| `SshClientSettings` | `SshConnectionOptions` | The .NET `*Options` convention |
| `Credential` | `SshCredential` | — |
| `RemoteProcess` | **`SshCommand` / `SshShell`** | **Split in two.** A one-shot command and an interactive shell differ in lifecycle, read/write shape and exit semantics; today they are crammed into one 1,198-line class, and properties like `HasTerminal` are what that cramming squeezes out |
| `SftpClient` | `SftpFileSystem` | It is not a "client"; it is a view of a file system |
| `SftpFile` | `SftpFileStream` | It is a subclass of `Stream`, and the name should say so |
| `SftpDirectory` / `ISftpDirectory` | `SftpDirectoryHandle` | — |
| `SshDataStream` | `SshTunnelStream` | — |
| `LocalForward`/`RemoteForward`/`SocksForward` | `PortForwarder` + `ForwardKind` | The three classes have nearly identical public surfaces; they differ only in "who listens" |
| `HostAuthentication` (delegate) | `IHostKeyPolicy` (interface) | A policy needs to carry state (a known_hosts handle, a CA trust chain, temporary trust for this run); a delegate can't |
| `SftpProgressHandler` (abstract class) | `IProgress<SftpTransferProgress>` | An existing BCL convention; don't invent another |
| `SshChannel` (internal) | `SshChannelCore` + `SshChannelPipe` | Different internal model, see §5.5 |
| `SshSession` (internal) | `SessionMachine` + `SendPump` + `ReceivePump` | Same, see §5.4 |
| `Sequence` / `SequencePool` / `Packet` | **(does not exist)** | Uses Pipelines, see §5.2 |

### 6.2 Mainline usage (mapped against VelaShell's existing call sites)

```csharp
var options = new SshConnectionOptions("root@10.0.0.1:22")
{
    ConnectTimeout = TimeSpan.FromSeconds(15),
    KeepAlive      = new KeepAlivePolicy(TimeSpan.FromSeconds(30), maxMissed: 3),
    Credentials    = [ new PrivateKeyCredential(path, passphrase),
                       new KeyboardInteractiveCredential(PromptAsync) ],
    HostKeyPolicy  = new TofuHostKeyPolicy(store, onDecision: AskUserAsync),
    Dialer         = DialerChain.Socks5("127.0.0.1", 10808),   // the proxy itself is dialed directly by default; nest with .Via(...)
    Algorithms     = SshAlgorithmSet.Default.WithCompression(Compression.ZlibOpenSsh),
    AutoConnect    = false,   // explicit connect (this is how VelaShell is configured today, and it's right)
    AutoReconnect  = false,
};

await using var conn = new SshConnection(options, loggerFactory);
await conn.OpenAsync(ct);

// Interactive shell — pixel dimensions are first-class
await using SshShell shell = await conn.OpenShellAsync(new ShellOptions {
    TerminalType = "xterm-256color",
    Size  = new TerminalSize(cols, rows, widthPx, heightPx),
    Modes = modes,            // encoded terminal modes for pty-req
}, ct);
shell.Resize(new TerminalSize(cols, rows, widthPx, heightPx));

// One-shot command — get all three things in one go
SshCommandResult r = await conn.RunAsync("uname -a", ct);  // StdOut · StdErr · ExitCode · ExitSignal

// Long-running command — line by line, TERM first on cancellation
await foreach (var line in conn.StreamAsync("tail -F /var/log/x", StreamOptions.IncludeStderr, ct)) { }

// SFTP
await using SftpFileSystem fs = await conn.OpenSftpAsync(ct);
if (fs.Capabilities.HasPosixRename) await fs.PosixRenameAsync(a, b, ct);

// Forwarding — metering lives in the library
await using PortForwarder fwd = await conn.ForwardAsync(
    ForwardKind.Local, bind: "127.0.0.1:8080", target: "10.0.0.9:80", ct);
Console.WriteLine($"{fwd.ActiveConnections} conn · {fwd.BytesUp + fwd.BytesDown} B");
```

### 6.3 Compatibility layer `VelaShell.Ssh.Compat.Tmds` (optional)

A **thin shim**: the type names and method signatures of `Tmds.Ssh`, forwarding to VelaShell.Ssh.

- The goal is not long-term maintenance; it is **to let VelaShell switch over in two steps**:
  first swap the engine and get all 3,194 tests passing, then migrate call sites to the new API one by one.
- It covers only the surface VelaShell actually uses (`SshClient`, `SshClientSettings`, `RemoteProcess`,
  `SftpClient`, the `Credential` family, the exception family), **with no ambition of full compatibility**.
- **It contains no Tmds code itself** — just classes with the same names, forwarding to our implementation.
  (Type names and method signatures are not copyrightable; the implementation is.)
- Marked `[Obsolete]`, and removed in a major version once VelaShell has finished migrating.

> **Decision point**: whether a compat layer "saves effort" or "drags you down" is debatable. Recommendation: **build it, but only the surface VelaShell uses,
> and mark it Obsolete from day one** — its value is splitting "swap the engine" and "swap the API" into two changes that can each be rolled back independently.

---

## 7 Performance design: seven techniques and expected gains

| # | Technique | How it works | Expected gain |
| :-: | --- | --- | --- |
| 1 | **Send coalescing** | A single writer drains the queue in one go, Seals each frame into the PipeWriter, and Flushes once | High-concurrency SFTP writes: syscalls per round **64 → 1–2** |
| 2 | **Large-block receive** | PipeReader reads in 64 KiB, replacing a fixed 4 KiB | 32 KiB packets: reads **8 → 1** |
| 3 | **Adaptive channel window** | Ran low and the reader was starved → ×2, 3 rounds without running low → ×0.75, `[256 KiB, 64 MiB]`; no RTT / BDP estimate (§5.5) | SFTP over a 200 ms RTT link: **~10 MB/s → near the bandwidth ceiling** |
| 4 | **Adaptive SFTP pipeline** | Block size from `limits@openssh.com`; depth scaled by how often requests wait for an in-flight slot (starts at 64, capped at 256) instead of a hard-coded 64×32 KB (§5.8) | Markedly higher throughput on high-latency links; small servers no longer get flooded |
| 5 | **Zero-copy read path** | `PipeReader` hands over a `ReadOnlySequence<byte>` that consumers can parse directly | Removes the Sequence→Memory copy (one per packet) |
| 6 | **Default cipher order by hardware** | With AES-NI / ARM Crypto extensions AES-GCM comes first, without them chacha20-poly1305 comes first (`SshAlgorithmSet.Default`); ChaCha20 and Poly1305 come from BouncyCastle's `ChaChaEngine` / `Poly1305`, not hand-written, with the engines kept resident and only the nonce changed per packet (§11.2.22) | Devices without AES hardware acceleration (some ARM) don't fall into software AES, which is an order of magnitude slower |
| 7 | **Pooling on every path** | `ArrayPool` + `IValueTaskSource` (unified ledger §5.6) + `PoolingAsyncValueTaskMethodBuilder` | Near-zero GC allocation in steady-state transfers (target: < 50 Gen0 collections for a 1 GB SFTP transfer) |

**Acceptance**: a BenchmarkDotNet suite that runs the same set of scenarios against `Tmds.Ssh` / `SSH.NET` on the same target machine
(1 GB upload / download / 10k small files / interactive echo latency / connection setup time / steady-state allocation),
with `tc netem` injecting three tiers: 5 ms / 50 ms / 200 ms RTT and 0.1% packet loss.
**The numbers go in the README, not hidden.**

---

## 8 Extension points (this is what decides whether things are easy to add five years from now)

| # | Extension point | What it's for |
| :-: | --- | --- |
| 1 | `ISshTransportDialer` | Proxies, jump hosts, TUN, in-memory transport (tests), heterogeneous carriers |
| 2 | `ISshCipherSuite` + `CipherRegistry` | New ciphers. **Including Chinese national standards SM4-GCM / SM3** (a real requirement for Chinese government and enterprise customers) |
| 3 | `IKeyExchange` + `KexRegistry` | New KEX. Post-quantum (ML-KEM, sntrup761) built in; future hybrid schemes get added the same way |
| 4 | `IHostKeyAlgorithm` | New host key types, including **CA-signed host certificates** (`*-cert-v01@openssh.com`) |
| 5 | `ISshSigner` | Where the private key comes from: file / Agent / PKCS#11 / HSM / KeyVault / OS keychain |
| 6 | `IAuthMethod` | New authentication methods, including bastion hosts' private extensions |
| 7 | `IHostKeyPolicy` | Trust model: known_hosts / CA / TOFU / enterprise allowlist |
| 8 | `IIncomingChannelHandler` | Server-initiated channels: **agent forwarding**, X11, `forwarded-tcpip` |
| 9 | `IGlobalRequestHandler` | Server global requests, e.g. `hostkeys-00@openssh.com` (host key rotation) |
| 10 | `ISftpExtension` | Vendor SFTP extensions |
| 11 | `IPacketTap` / Metrics / ActivitySource | Diagnostics, recording, APM |
| 12 | `ISshConfigSource` | Configuration sources: `~/.ssh/config`, enterprise-pushed, UI |

> This table is the whole answer to "extensibility later on". The test is simple:
> **each of the nine host-side patches in §1 maps to some row of this table.**
> In other words — had these twelve extension points existed from the start, not one of those nine patches would have had to be written.

---

## 9 What VelaShell can delete / unlock

**What can be deleted (about 1,200 lines):**

| File | Lines | Outcome |
| --- | :-: | --- |
| `Infrastructure/Net/LoopbackProxyRelay.cs` | ~200 | **Delete** → `Socks5Dialer` / `HttpConnectDialer` |
| `Infrastructure/Ssh/SshAlgorithmProbe.cs` | 199 | **Delete** → the negotiation exception carries the lists itself |
| `Infrastructure/Ssh/SshAlgorithmDiagnostics.cs` | 105 | Shrinks to one formatting function (~30 lines) |
| `Infrastructure/Ssh/MeteredPortForwardHandle.cs` | 376 | Shrinks to a thin adapter (~60 lines) |
| `Infrastructure/Ssh/OpenSshPrivateKey.cs` | 303 | **Delete** → the library reads PKCS#1 / PKCS#8 / ppk directly |
| `TmdsSshInterop.ExtractConnectFailedReason` + two call sites | ~60 | **Delete** → strongly typed `SshFailureReason` |
| `TmdsSftpClientWrapper.ResumeSafetyMargin` (blind 2 MB rewind) | — | **Delete** → `DurableLength` watermark |
| The "reconnect once after the prompt times out" bit in `TmdsSshClientWrapper` | ~40 | **Delete** → the host key decision doesn't count toward the connect timeout (see below) |

> The last one deserves its own note: today, the time the host fingerprint prompt sits on screen **counts toward the connect timeout**,
> so by the time the user clicks "Trust", this attempt has already been declared dead and the only option is to reconnect once on the spot.
> In VelaShell.Ssh, the `IHostKeyPolicy` decision is **timed separately** (`HostKeyDecisionTimeout`, infinite by default),
> and this patch disappears along with its explanatory comment.

**What gets unlocked:**

| Capability | Today | After |
| --- | --- | --- |
| **2FA / OTP (keyboard-interactive)** | Can't connect; the UI copy says "this version cannot connect" | Native support; the tripwire in `KeyboardInteractiveSupportTests` can be removed |
| **PTY pixel dimensions** | Always 0, blocked on upstream PR #519 | First-class, the four fields of `TerminalSize` |
| **Compression zlib@openssh.com** | Blocked on upstream PR #513 | Built in |
| **SSH agent forwarding** | None (the most glaring cell in the comparison matrix) | `IIncomingChannelHandler` + `auth-agent-req@openssh.com` |
| **Configurable algorithm negotiation** | Halfway (can diagnose, can't configure) | `SshAlgorithmSet` fully configurable, and listable directly from the UI |
| **posix-rename** | Degrades to plain rename | Explicit support + capability query |
| **known_hosts interop with OpenSSH** | View and delete only | Reads and writes the OpenSSH format (including hashed, `@cert-authority`, `@revoked`) |
| **Live metering per tunnel** | Forwarding reimplemented by hand | Emitted directly by the library + Metrics |

---

## 10 Testing and acceptance

### 10.1 Layered tests

| Layer | Approach |
| --- | --- |
| Codec | Byte-by-byte assertions on message samples; **SharpFuzz fuzzing** (the decoder is the only place facing untrusted input directly — it must be fuzzed) |
| Cipher suites | RFC 8439 / NIST CAVP vectors one by one; cross-checked against the BCL |
| State machine | Property tests: random event sequences must never reach an illegal state or deadlock |
| Channels / windows | In-memory transport + message scripts, asserting when windows are replenished and how backpressure behaves |
| SFTP | Pure-function tests for `SftpWire` + scripted pipeline tests |

### 10.2 In-memory transport (`InMemoryDialer`) — this one matters

Almost all of Tmds' tests need an sshd running in Docker. We let `ISshTransportDialer` return
**a pair of in-memory duplex streams**, with the other end wired to a **minimal test server stub** we write ourselves (exists only for tests, never shipped).
The result: **the vast majority of protocol-layer tests need no network, no containers, and finish in milliseconds**,
so they can gate every commit — instead of the current approach of excluding `DockerIntegration` and running the rest.

### 10.3 Interop matrix (this is the real acceptance test for an SSH library)

| Peer | Why it must be tested |
| --- | --- |
| OpenSSH 8.x / 9.x / 10.x | The de facto standard; 8.x is still very common |
| OpenSSH for Windows | Behaves differently from the POSIX version |
| Dropbear | Embedded devices, routers |
| Cisco IOS / Huawei VRP / H3C | **Old network gear**: only `aes128-ctr` + `hmac-sha1`, sometimes even `diffie-hellman-group14-sha1`. Some VelaShell users will certainly need to connect to these |
| OpenSSH 7.4 on CentOS 7 | Same as above |
| AWS Transfer / Azure SFTP / other managed SFTP | Huge variance in supported SFTP extensions |
| GitHub / GitLab (exec only) | The most common public endpoints |

**Landed and verified** (see §11.2.9, §11.2.13): the interop matrix runs two OpenSSH versions,
`latest` and `9.3`, with 13 `[TestCategory("Interop")]` cases.
**Local run on 2026-09-21: 13/13 passing on both OpenSSH 10.3 and 9.3** —
and the very first run was **0/13**, which caught a real key-derivation bug (§11.2.13).
Locally, `scripts/ssh/interop/Start-TestServer.ps1 -X11` brings up the same environment.
**The remaining peers in the table are not wired up yet** — they need either purchased hardware (network gear) or accounts
(managed SFTP), not something a few lines of YAML can solve.

### 10.4 Security acceptance

- Gates: `CodeQL` + `dotnet list package --vulnerable`
- **Must implement**: `strict KEX` (`kex-strict-c-v00@openssh.com`, the Terrapin mitigation),
  RSA minimum key length check, algorithm downgrade protection, authentication failure counting and backoff
- Third-party audit: commission one external audit before 1.0 (this is a product that can be sold under license, not a toy)

---

## 11 Engineering and milestones

### 11.1 Repository shape

A new repository `VelaShellLabs/velashell-ssh`, alongside the VelaShell main repository.
NuGet package name **VelaShell.Ssh (single package)**; the solution has two projects in total (library + tests),
with internal layering done by folders, see §4.
TFM: **`net11.0` single target** (no multi-targeting — this saves polyfills, conditional package references and `#if` branches,
and lets us use the .NET 11 BCL directly: `AesGcm`, `ChaCha20Poly1305`, ML-KEM, the new Pipelines APIs).
`LangVersion` is `preview` (tracks the latest C# syntax; the library itself is the earliest proving ground).
`IsAotCompatible=true`, zero reflection, with `PublicApiAnalyzers` pinning the public surface.

### 11.2 Milestones

| M | Contents | Exit criteria | Estimate |
| :-: | --- | --- | :-: |
| **M0** | Behavioral spec (the "spec" of rule 2 in §2.2) + skeleton + in-memory transport + test server stub | Spec review passed; dry-run CI all green | 2 weeks |

| **M1** | L1–L4: dialing, framing, cipher suites, version exchange, KEX, authentication (password / publickey / kbdint) | Can connect to OpenSSH and authenticate | 4 weeks |
| **M2** | L5: channels, exec, shell, pty, signal, exit-status | Can run an interactive shell and one-shot commands | 3 weeks |
| **M3** | L7 SFTP: three-tier pipeline + extension negotiation + watermark | SFTP upload / download / listing / attributes / links all working | 4 weeks |
| **M4** | L8 forwarding: `-L` / `-R` / `-D` + metering + agent forwarding | All three forwarding kinds work, metering is accurate | 2 weeks |
| **M5** | Periphery: known_hosts, ssh_config, all private key formats, ppk, agent client, compression | Shares a single known_hosts file with OpenSSH | 3 weeks |
| **M6** | Compat layer + **full VelaShell switchover** | All 3,194 VelaShell tests green | 2 weeks |
| **M7** | Performance and interop: benchmarks, window tuning, interop matrix, fuzzing, audit | §7 targets met; §10.3 matrix all green | 4 weeks |

### 11.2.1 M0 progress (2026-09-21)

| Item | Status |
| --- | --- |
| Repository skeleton (MIT / NOTICE / net11.0 single target / LangVersion=preview / central package management) | ✅ |
| **9 behavioral specs** (`spec/00`–`08`) | ✅ awaiting review |
| CI (build & test · public-surface gate · packaging smoke test · interop) | ✅ written, not yet run on a real runner. interop was a placeholder at the time; it is now wired to real containers (§11.2.9) |
| Public-surface gate + `scripts/ssh/Update-PublicApi.ps1` | ✅ |
| Skeleton code: wire primitives, send gate, failure classification, dialing abstraction | ✅ **72 cases all green** |
| **In-memory transport** (`InMemoryTransport` + `ISshTransportDialer`) | ✅ including half-close, backpressure, cancellation |
| Test server stub (the end that speaks SSH) | ✅ landed with M1 (`TestSshServer` + `TestAuthServer`) |
| Link characteristics simulation (one-way latency / bandwidth / packet loss) | ⏳ the adaptive window and SFTP pipeline depth can only really be tested with it |
| Dependency footprint | ✅ **zero runtime dependencies** (`Microsoft.Extensions.Logging.Abstractions` ships with the framework on net11) |

**One deviation from the original plan, recorded:**

1. **Consolidated into two projects** (library + tests), layered internally by folders — the original plan was two packages, Core + main.
   Rationale in §4.

### 11.2.2 M1 progress (2026-09-21)

L1–L4 all landed, **197 cases all green**. The full handshake and authentication run end to end in memory:
version exchange → algorithm negotiation → key exchange → signature verification → host key policy → rekey → encrypted send/receive → user authentication.

| Item | Status |
| --- | --- |
| L1 transport: `SshPacketTransport` (Pipelines line-mode + frame-mode IO, send coalescing) | ✅ |
| L2 cipher suites: `ISshCipherSuite` + `CipherSuiteShape` | ✅ plaintext / AES-GCM / ChaCha20-Poly1305 / AES-CTR+HMAC (both EtM and MtE paths) |
| L3 key exchange: curve25519 · ECDH (3 curves) · DH group14/16 · hybrid PQ (ML-KEM-768 / sntrup761) | ✅ 7 kinds |
| Exchange hash and key derivation (including extension rounds) | ✅ H computed independently on both sides and compared byte for byte |
| Strict KEX (Terrapin mitigation, CVE-2023-48795) | ✅ including the attack case "injecting IGNORE during KEX must disconnect" |
| Host keys: ed25519 / ecdsa×3 / rsa parsing, fingerprints, signature verification (mismatched algorithm name is rejected outright) | ✅ 5 kinds |
| L4 authentication: `SshAuthenticator` + four credential kinds + `ISshSigner` | ✅ none / password / publickey / keyboard-interactive |
| **Partial success (2FA)** | ✅ end-to-end case for two-step public key + one-time code authentication |
| `server-sig-algs` (RFC 8308) decides the RSA signature algorithm | ✅ SHA-1 downgrade rejected by default; requires an explicit switch |
| Authentication failure diagnostics (per-attempt `SshAuthAttempt`) | ✅ "can't read the private key", "server doesn't accept this key" and "one-time code required" are distinguishable |
| Coverage | 7 KEX × 5 host key types × 6 ciphers × 4 MACs, plus failure paths |

**Three real bugs caught during M1** (none of them were mistakes in the tests; recorded because all three show up only under real timing):

1. **Invalid ECDH points weren't rejected on Windows** — CNG wraps the `CryptographicException` in another layer,
   so the original `catch (CryptographicException)` let it through. Changed to treat "anything that isn't our own exception" as failure.
2. **`DisposeAsync` threw `IOException`** — `PipeWriter.CompleteAsync` throws when the peer is already gone,
   masking the real failure reason on every error path. The dispose path no longer throws.
3. **Version exchange wrapped reads but not writes** — if the peer vanished while we were sending our identification string, what got thrown was a bare `IOException`
   instead of `SshConnectException(ClosedByPeer)`.

**One design gap only surfaced while writing tests**: the "explanation" of an authentication step (e.g. "the server requires a password change first")
was constructed in `ReadAuthOutcomeAsync` and then thrown away; only the outcome was passed along.
Now `AuthStepResult` carries the outcome and the explanation upward together — otherwise that failure would degrade in the UI into
a contentless "authentication failed", and that explanation is exactly the sentence the user most needs to see.

**Not done in M1 (deferred to later milestones, recorded)**:

- Private key file parsing (OpenSSH v1 / PKCS#1 / PKCS#8 / ppk) and the ssh-agent client → M5.
  For now there is only `InMemorySshSigner` (holds the private key in-process).
- `known_hosts` and `ssh_config` → M5. For now there is only the `IHostKeyPolicy` abstraction.
- Password change flow (`PASSWD_CHANGEREQ`) — **explicitly not doing it**, but a readable failure reason is given.
- Interop matrix against real OpenSSH — needs a CI runner, can't run locally.
  **→ Done since, see §11.2.9.**

### 11.2.3 M2 progress (2026-09-21)

The L5 connection protocol landed, **228 cases all green**. Sessions, channels, flow control and requests all work.

| Item | Status |
| --- | --- |
| `SshConnection`: a multiplexer built from a receive loop + a send lock | ✅ |
| `SshChannel`: three pipes + one ordered event stream | ✅ stdout / stderr are **two independent PipeReaders** |
| Flow-control window, **replenishment hooked to `AdvanceTo` rather than to packet arrival** | ✅ `WindowedPipeReader` |
| Adaptive window policy (`Fixed` / `Adaptive`) | ✅ feedback loop wired up, see §11.2.8 |
| Total session window budget + channel count cap | ✅ when full, new channels are refused **without dropping the session** |
| Delayed channel number reuse (prevents crosstalk) | ✅ 30 seconds by default |
| EOF one-way half-close / CLOSE two-way | ✅ end-to-end case for "output still arrives after sending EOF" |
| Channel requests FIFO (**no id; aligned purely by order**) | ✅ reports `ProtocolError` when out of step |
| Global requests FIFO + keepalive | ✅ unknown requests **always** get a `REQUEST_FAILURE` reply (silence would permanently misalign the peer's FIFO) |
| `exec` / `shell` / `subsystem` / `pty-req` / `window-change` / `env` / `signal` | ✅ |
| `exit-status` / `exit-signal`, `ExitCode` as `int?` | ✅ each of the three cases (has a code / killed by signal / nothing at all) has a case |
| Pixel-level terminal size | ✅ carried all the way through to `window-change` |
| Server-initiated channels | ⏳ explicitly answered with `ADMINISTRATIVELY_PROHIBITED`; implemented in M4 |

**Two real bugs caught in M2**, both of which appear only under specific timing:

1. **`exit-status` was read as some other number.** The channel received the whole message payload but skipped only one byte of
   message number — while the `CHANNEL_REQUEST` header also contains the channel number and a **variable-length** type string.
   Now `SshConnection` slices out "the type-specific part" using `reader.Consumed` before handing it to the channel:
   leave downstream code to skip a variable-length header itself, and sooner or later someone will skip a fixed length.
2. **The window leaked away bit by bit.** When the window replenishment pump hadn't accumulated enough to reach the threshold, it **discarded** the bytes consumed in that round.
   The window as seen by the peer would shrink gradually to zero; the symptom is a channel that stalls permanently after transferring for a while,
   even though the local books show the window as full. Most easily hit when the consumer reads line by line (a few dozen bytes each time).
   Changed to accumulate across rounds, and added a case "read only 64 bytes at a time, 6 windows total" —
   inject the bug back in and both this case and another window case hang until timeout.

**Not done in M2 (recorded)**:

- **The RTT feedback loop for the adaptive window.** `SshWindowPolicy.Adaptive` currently only provides the lower and upper bounds;
  auto-scaling by RTT has to wait until `InMemoryTransport` can simulate one-way latency and bandwidth —
  without link characteristics simulation, logic like "double the window" has nothing to be verified against; writing it would just be spinning wheels.
  This is the same item as the to-do recorded in M0.
- Server-initiated channels (`forwarded-tcpip`, `auth-agent@openssh.com`) → M4.
- The `hostkeys-00@openssh.com` global request → M5.

### 11.2.4 M3 progress (2026-09-21)

The three SFTP tiers landed, **287 cases all green** (23 of them are byte-for-byte assertions on `SftpWire` that need no peer).

| Item | Status |
| --- | --- |
| `SftpWire`: pure-function codec (bytes ↔ messages, no state, no I/O) | ✅ decodes multi-segment `ReadOnlySequence` too |
| `SftpRequestPipeline`: **aligned by request-id, not by order** | ✅ still matches up when the server deliberately replies in reverse |
| `SftpFileSystem`: the full user-facing API | ✅ |
| **Write watermark `DurableLength`** | ✅ exact resume point, no blind rewind by the in-flight window |
| `AckedRangeSet`: sorted array + tail fast path | ✅ set size stays at 1 for sequential writes |
| `SftpWriteMode.Sequential` | ✅ trades throughput for "the file length is a trustworthy length" |
| Block size set from `limits@openssh.com` | ✅ falls back to 32 KiB without the extension |
| **Queryable capabilities** (`SftpCapabilities`) | ✅ **no silent degradation** when `posix-rename` isn't supported |
| Symlink semantics: `LSTAT` + follow-up `STAT`/`READLINK`, **issued concurrently** | ✅ including a broken-link case |
| Argument order of `SSH_FXP_SYMLINK` (OpenSSH, not the draft) | ✅ pinned by two cases |
| The three ATTRS pitfalls (shared flag bits ×2, signed times, type hidden in the high bits of permissions) | ✅ one case each |
| `EOF` is not an error; `READ` may return short | ✅ reads the whole thing even when the server returns just 100 bytes each time |
| Cleaning up handles from late replies after cancellation | ✅ `onLateResponse` closes the leaked handle |
| Handle leak check | ✅ server handle count returns to zero after read, write and directory listing |

**Two defects of our own caught in M3** (both exposed while writing tests):

1. **The `Order` out-of-order stub would hang.** The test server "accumulates two replies, then sends them in reverse",
   while the handshake (`INIT` → `VERSION`) is strictly one request, one reply — the only reply got held back forever.
   That's a bug in the stub, not the library, but it exposes the same class of problem:
   **any "wait until you have N, then process" logic must have a way out for "it will never come"**.
   It now decides whether to hold back based on "does the client have any other request in flight".
2. I got the `HighestAckedOffset` assertion wrong myself (wrote the start of the range instead of the end); the implementation was correct.

**Not done in M3 (recorded)**:

- Wrappers for `statvfs@openssh.com`, `copy-data`, `home-directory` and `expand-path@openssh.com`
  — their capability bits can already be queried, but no convenience methods yet. None of them are in VelaShell's existing usage.
- The public `ISftpExtension` extension point (architecture §8 item 10) —
  for now, adding a vendor extension means using `SftpRequestPipeline.SendAsync` + `SftpWire.WriteExtended` directly,
  which works but isn't elegant.
- BDP adaptation at the SFTP layer (in-flight request count tuned by RTT) — the same to-do as M2's adaptive window;
  both have to wait for link characteristics simulation. **→ Done since, see §11.2.9.**

### 11.2.5 M4 progress (2026-09-21)

All four forwarding forms landed, **311 cases all green**.

| Item | Status |
| --- | --- |
| `DuplexRelay`: a copy loop **with nothing to do with SSH** | ✅ 5 cases that don't touch SSH at all |
| Per-direction half-close (TCP `shutdown(SEND)` ↔ `CHANNEL_EOF`) | ✅ case for "after one end finishes sending, the other can keep sending" |
| Metering: **both** events + `System.Diagnostics.Metrics` | ✅ bytes are counted in the copy loop, excluding protocol overhead |
| Local forwarding `-L` | ✅ end to end: real socket → real SSH session → back |
| Dynamic forwarding `-D` (SOCKS5 subset) | ✅ **domain names are not resolved locally**; failure codes mapped faithfully |
| Remote forwarding `-R` | ✅ including "for port 0, take the actual port from the reply payload" |
| Direct tunnels (no listener) | ✅ both TCP and Unix sockets |
| Incoming channels (`IIncomingChannelHandler`) | ✅ unregistered types are **explicitly rejected**, not met with silence |
| A single failed connection doesn't affect the forwarder | ✅ still accepts the next one after a rejection |
| No half-alive listener left behind when the port is taken | ✅ |
| Binds to loopback by default | ✅ exposing it externally must be spelled out explicitly |

**One real gap filled in M4**: `SendGlobalRequestAsync` originally only returned success or failure,
but when a `tcpip-forward` request asks for port `0`, **the actual port allocated by the server is in the
`REQUEST_SUCCESS` payload**. Returning only a boolean means that port number can never be obtained,
and routing incoming connections by `(bind_addr, 0)` afterwards never matches a single one —
the symptom is "the forward looks established, but every incoming connection is rejected".
`SendGlobalRequestWithReplyAsync` has been added, returning `SshGlobalRequestReply` (with the payload).

**Two tooling lessons from M4** (recorded because both wasted time):

1. When using Perl's `s/\Q…\E/…/` for code replacement, **XML doc comments can't be written directly in the pattern** —
   the `/` in `///` terminates the regex early. Always extract the literal with a heredoc and replace from that instead.
2. `q{…}` as a Perl literal delimiter requires balanced braces,
   while C# snippets are often unbalanced (e.g. cut from the middle of a block). Same fix: use a heredoc.

**Not done in M4 (recorded)**:

- **Agent forwarding** (`auth-agent-req@openssh.com`). Mechanically it only lacks one
  `IIncomingChannelHandler` implementation + a connection to the local agent, but the three security constraints in §7.2
  (off by default, forward only specified keys, per-signature confirmation) only become complete together with M5's agent client —
  you first need the ability to "connect to the local agent" before "forward only some of its keys" even makes sense.
  **→ Done since, see §11.2.9.**
- `streamlocal-forward@openssh.com` (remote Unix socket forwarding) —
  the direct direction, `direct-streamlocal`, already exists; the reverse direction will be added along with agent forwarding.
- The grace period after cancelling a remote forward is currently a hard-coded 2 seconds, not configurable.

### 11.2.6 M5 progress (2026-09-21)

The configuration and key layer landed, **356 cases all green**, source at 78 files / about 17,700 lines.

| Item | Status |
| --- | --- |
| `known_hosts` parsing (plain / `[host]:port` / **hashed form**) | ✅ malformed lines are skipped rather than invalidating the whole file |
| `KnownHostsPolicy`: TOFU / strict / pinned fingerprint | ✅ "never seen" and "changed" take two different paths |
| Message when a key has changed | ✅ says which host, the new fingerprint, and **which line number to delete** |
| `ssh_config` parsing | ✅ **the first value seen wins** (the opposite of most config formats) |
| Private keys: OpenSSH v1 (unencrypted) | ✅ ed25519 / rsa / ecdsa×3, only counted as passing after **sign-then-verify** |
| Private keys: PKCS#8 (including passphrase-protected), PKCS#1, SEC1 | ✅ via the BCL |
| `ssh-agent` client (Windows named pipe / Unix socket) | ✅ the private key never enters this process |
| `SshPublicKey.SignatureAlgorithms` | ✅ agent signing uses it to pick the algorithm |

**One real bug caught in M5 — and a security-relevant one**:

`KnownHostsFile.Lookup` used to return `Known` as soon as it hit "host + key both match".
But `@revoked` lines are usually **appended** to the end of the file (the most natural way to revoke a key
is to add a line at the end). So "revoked" meant "not revoked at all" —
the old trust line sits earlier, and the revocation never gets its turn to be seen.
It now scans the whole table before concluding, and a matching revocation wins immediately.

**One gap M5 deliberately left open** — **filled on 2026-09-22, see §11.2.18**:

**Encrypted OpenSSH private keys couldn't be read** (`-----BEGIN OPENSSH PRIVATE KEY-----` with
`cipher != none`). Its passphrase derivation uses `bcrypt_pbkdf`, which requires access to Blowfish's
**key schedule internals** (`EksBlowfishSetup` / `expandstate`) —
the BCL doesn't have that, and BouncyCastle's `BlowfishEngine` only exposes `Init` + `ProcessBlock`.

> That assessment still holds today. What changed is the trade-off: this format is the default output of `ssh-keygen` when a passphrase is set,
> so "can't read it" means "can't read the key most people actually have".
> In the end a **documented, tightly scoped** exception was made to "don't write our own cryptographic primitives";
> see §11.2.18.

**Not done in M5 (recorded)**:

- PuTTY `.ppk` — the exception message suggests converting with PuTTYgen. **→ Done since, see §11.2.9.**
- Compression (`zlib@openssh.com`) — it already has a slot in algorithm negotiation, but the codec isn't wired up yet.
  It brings almost no benefit for interactive sessions (terminal output is small to begin with); it only matters for SFTP.
  **→ Done since, see §11.2.9.**
- `Include` and `Match` in `ssh_config` — judged "deliberately not doing" at the time.
  **→ Done since, see §11.2.12** (`Match exec` is still not executed by default).
- Agent forwarding (the same item as in M4; now that the agent client exists, it can be wired up).
  **→ Done since, see §11.2.9.**

### 11.2.7 Connection layer (the user-facing layer, 2026-09-21)

M0–M5 built every layer, but callers still had to assemble
dialing → version exchange → key exchange → decision → authentication themselves — that's not what "usable" means.
This step fills in the shape written in §6.2, **369 cases all green**.

| Item | Status |
| --- | --- |
| `SshConnectionOptions` (`user@host:port` parsing, including IPv6 brackets) | ✅ |
| `SshConnectionFactory.ConnectAsync` / `options.ConnectAsync()` | ✅ one step to usable |
| `TcpTransportDialer` | ✅ socket errors translated into `DnsFailure` / `TcpRefused` / `TcpTimeout` / `TcpUnreachable` |
| **Three independent timers** | ✅ connect / decision (infinite by default) / authentication (2 minutes by default) |
| Keepalive: **baseline is "last time any message was received"** | ✅ never sends one while busy |
| Declaring the link dead throws `KeepAliveTimeout` rather than a generic `Timeout` | ✅ upper-layer reconnect policy should apply to this class only |
| `conn.RunAsync()` → `SshCommandOutput` (all three things in one go) | ✅ `EnsureSuccess()` carries stderr into the exception |
| The default host key policy is **not** "accept any key" | ✅ pinned by a case |
| [`getting-started.md`](../getting-started.md) | ✅ |

**A design flaw fixed in this step**: the factory's own timeout used to always report
`SshFailureReason.Timeout`, regardless of whether it was stuck in dialing, version exchange, key exchange or authentication.
But the next step for each of these four is completely different —
"DNS is slow", "the port is dropped by a firewall", "the peer isn't an SSH service" and "the user hasn't entered the one-time code"
should say four different things in the UI. Reason codes and messages are now given per stage:

```
Timed out at the "key exchange" step while connecting to 10.0.0.1:22 (limit 30 seconds).
```

It also explains an intermittent test failure: when connecting to `127.0.0.1:1`,
if the socket is silently dropped, the factory's timer fires before the dialer's,
so the reason code degrades from `TcpTimeout` to `Timeout`. After the fix, 6 consecutive runs were all green.

### 11.2.8 Link simulation and adaptive window (2026-09-21)

This is the to-do recorded back in M0 and put off until now: **without link characteristics simulation,
the "adaptive" code has nothing to be verified against** — on a zero-latency, infinite-bandwidth
in-memory link, a 32 KiB window runs exactly as fast as a 64 MiB one.

**376 cases all green.**

| Item | Status |
| --- | --- |
| `LinkCharacteristics` (one-way latency + bandwidth) | ✅ including `LocalNetwork` / `Intercontinental` presets |
| `DelayedStream`: wraps any stream with latency and rate limiting | ✅ rate limiting uses "the next moment sending is allowed", so scheduling jitter doesn't accumulate |
| Adaptive feedback loop for the channel receive window | ✅ |
| Comparison case: adaptive vs fixed | ✅ compares **round-trip counts**, not wall-clock time |

**Two corrections of judgment** (the first version got them wrong; both were exposed by tests):

1. **Growing the window must grant the extra credit to the peer on the spot.**
   The first version only changed the local `Size`, with the result that the replenishment threshold (`Size/2`) went up
   while the window in the peer's hands didn't — it couldn't send more data, and we couldn't accumulate enough for the next replenishment.
   **Both sides stall, and neither moves.** Two window cases hung straight to the 60-second timeout.

2. **The criterion for growing the window is "was the window eaten down to the bottom", not "how long between two replenishments".**
   The first version used an interval < 250 ms as the signal — but on a LAN that's always true,
   which would grow the window all the way to 64 MiB and waste memory on a link where the window isn't the bottleneck at all.
   `OnData` now records the fact "remaining dropped below 1/8", and doubles only when things are genuinely tight.

**There was also a correction in test methodology**: the comparison case originally compared wall-clock time,
and failed one run in four — that kind of assertion is inherently unreliable on a busy machine,
and **a test that occasionally fails is worse than no test**. It now compares the number of
`WINDOW_ADJUST` messages received by the server: each replenishment means the peer used up its window first and waited one RTT,
so fewer round trips means higher throughput.

**But the claim "this number is deterministic" was premature at the time.**
It later failed once more in a full run (always passes when run alone), for two reasons, both on the test side:

1. **Consumption granularity followed the scheduler.** The replenishment pump is triggered by "how much was consumed",
   so consumption granularity directly determines the number of replenishments; and each round used to `AdvanceTo(buffer.End)`,
   eating "however much happened to have arrived right now" in one go — when the machine is busy, several packets merge into one read,
   and a few fewer replenishments get sent. What was being measured was scheduling noise, not window policy.
   **Changed to consume a fixed 8 KiB per round** (`AdvanceTo(consumed, consumed)`;
   examined must equal consumed — writing `buffer.End` would lock the remaining data inside the pipe).
2. **The counter was read while replenishments were still in flight.** On a link with 20 ms one way,
   at the moment all the data has been received, the last few `WINDOW_ADJUST` messages haven't reached the server yet,
   so what gets read is "how many have arrived so far" — a random number.
   **Changed to poll until it stops changing.** That isn't a performance assertion of the "wait long enough and it passes" kind;
   it waits for a termination event that is certain to happen: once all the data has been received there will be no further replenishments.

Along the way, the observation counter was switched to `Interlocked.Increment` + `Volatile.Read` —
the thread incrementing it is the server loop, the thread reading it is the test thread, and neither side was synchronized before.

**But neither of those two was the real cause of that failure.** Once the failure information was captured in full,
the assertion was `expected 524288, actual 294912` — not a count mismatch, but **EOF after receiving only half the data**.
Reproduced once more, it wore a different face: the consumer side threw
`Reading is not allowed after reader was completed`.

Two faces of the same bug, and it was in the library, not the stub:

#### The receive pipe's watermarks were computed from the **initial window**, and the window grows past it

```csharp
// Old code
int window = options.WindowPolicy.InitialBytes;
PipeOptions pipeOptions = new(
    pauseWriterThreshold: window * 2L,   // ← twice the initial value
    resumeWriterThreshold: window, …);
```

Behind this line was a comment saying "the peer **cannot possibly** send more unconsumed data than the window,
so the write side should never block". That statement is right — **but only for a fixed window**.
Under the adaptive policy the window grows all the way up to `MaximumBytes` (64 MiB by default),
**and once it grows past twice the initial value, the premise is gone.**

And the receive loop relies on exactly that premise to dare not to wait on flush:

```csharp
ValueTask<FlushResult> flush = writer.FlushAsync();
if (flush.IsCompletedSuccessfully) { … return; }   // normal path
_ = ObserveFlushAsync(flush);                      // old code: toss it into the background, keep writing
```

Once the watermark is hit, flush no longer completes synchronously, so when the next message arrives,
**the same `PipeWriter` gets written again before the previous flush has completed** — a misuse of `Pipe`.
It doesn't fail on the spot; it just, at some later moment, hands the consumer a
"Reading is not allowed after reader was completed", or simply loses half the data.

Two changes:

1. **Watermarks are computed from `MaximumBytes`**, not `InitialBytes`.
   Under the fixed policy the two are equal, so it's unaffected; under the adaptive policy the premise holds again.
   A watermark is just a waterline — it preallocates no memory, and the actual backlog is bounded by the window anyway.
2. **The "toss it into the background and keep writing" path now throws.** With the new watermarks it's unreachable —
   actually reaching it means our own window bookkeeping is broken, and at that point
   **failing with a pointer to the real cause is far better than going on to corrupt the pipe**.

The regression case `窗口长满之后数据依然一字节不差` ("data is still byte-exact after the window grows to full") sets the upper bound to 16 times the initial value,
forcing the window past the old watermark, and asserts only that "the data is byte-exact".
Inject the bug back in and it fails **every time** (no longer one run in eight).

> Three lessons, in order of weight:
> 1. **Don't rush to believe your first guess about a failure.** I was initially sure it was statistical noise in the replenishment count,
>    and changed two things on that basis — those two changes were right (they really were noise sources), but neither was the cause.
>    It was **capturing the failure information in full** that found the real culprit.
> 2. **When you write down "impossible", write down what it depends on.** That comment wasn't wrong;
>    what went wrong was that its premise was later changed by the adaptive window, and neither the comment nor the code moved with it.
> 3. **Don't paper over intermittent failures by rerunning.** At the time it had only a one-in-eight chance —
>    and its counterpart on a real link is "a large file occasionally loses half its data in transit".

#### Along the way: teaching the stub to "speak up when it dies"

The most expensive stretch of the investigation was spent on **a 30-second timeout carrying no information at all**.
The reason: **three** background paths in the test server were all silently swallowing exceptions:

- `RunAsync` (the receive loop) had no catch-all — once it threw, the loop stopped,
  and the server never replied again;
- `PlayScriptAsync` (script playback) only caught `OperationCanceledException` —
  any other exception quietly faulted the Task, so EOF and CLOSE were never sent;
- `PumpHandlerOutputAsync` (subsystem output copying) read
  `catch (Exception) { /* Channel's gone. The test stub doesn't make a fuss about it. */ }` —
  the comment treated "not making a fuss" as a virtue. In SFTP-type cases, once the copy died,
  the client sat waiting for a response that would never come. **This was the third instance of the same ailment**,
  and the last one to be dug out.

In all three cases what the client saw was not an error but **an `await` that never returns**,
finally ended by the global timeout in `test.runsettings`.
`Harness.DisposeAsync` also swallowed the exception from `await _serverChannels` —
so the real cause never got a single chance to show itself.

All three paths now: **record the cause**
(`Observation.ServerFault` / `ScriptFault` / `SubsystemFault`),
**actively wake up the other side** (close the connection, or send the missing EOF + CLOSE —
so the client immediately reads "the peer has quit" instead of hanging),
and in `Harness.DisposeAsync` raise it into a failure that names names.

> **The stub's diagnostic quality determines the cost of an investigation.** Between a stub that can say "here's why I died"
> and one that can only time out, the difference isn't a few lines of code, it's a few hours.
>
> And "the same ailment showing up three times in a row" is itself a signal:
> **a silent `catch` isn't stability; it's charging the bill to the future.**
> Every catch on a background path should answer the question "who will find out about this" —
> the ones that can't answer are the next 30-second timeout.

During the investigation I also fixed a real race in the test stub (not the cause this time, but it would bite sooner or later):
the server used a `Dictionary<uint, long>` to track "how much can still be sent to the client";
the receive loop adds to it when handling `WINDOW_ADJUST` and the send pump subtracts when spending credit, both doing
"read old value → compute new value → write back" with no synchronization — a textbook lost update.
All reads and writes now happen under the same lock, and **the check and the deduction are in the same critical section** (splitting them is the same as no lock).

**Still not done at the time**: adaptive SFTP pipeline depth (in-flight request count tuned by RTT).
Link simulation exists now, so the precondition for verifying it is in place — but the SFTP layer's in-flight window
(in-flight requests × block size) defaults to 64 × 32 KiB = 2 MiB,
which likewise caps at 10 MB/s over a 200 ms RTT; this one still stands. **→ Done since, see §11.2.9.**

**Total: about 24 weeks** (one person, full time). Size estimate **15,000–22,000 lines** (excluding tests; tests roughly another 1.5×).

> This is not a weekend project. **Accept this volume before deciding to do it.**
> If the goal is only to solve 2FA and pixel dimensions, the cheaper route is to send PRs upstream — we've already sent two
> (the #513 / #519 line), and that route works.
> **The case for writing it ourselves has to be the whole table in §1, not one or two rows of it.**
### 11.2.9 Six finishing items (2026-09-21)

Every section from §11.2.1 to §11.2.8 ends with a "not done" list. This section settles all of
those "→ done, see §11.2.9" entries in one go: **compression, agent forwarding, adaptive SFTP
pipeline depth, `.ppk`, performance benchmarks, and an interop matrix against real OpenSSH.**

**427 test cases: 414 green, 13 interop cases report themselves Inconclusive when no server is available.**

| Item | Status |
| --- | --- |
| Compression `zlib@openssh.com` / `zlib` | ✅ Per-direction persistent stream + `Z_PARTIAL_FLUSH`; decompression is capped |
| Agent forwarding `auth-agent-req@openssh.com` | ✅ Parsed forwarding (not byte passthrough); all three security constraints in place |
| Adaptive SFTP pipeline depth | ✅ Scales on "did we ever wait on the semaphore", no RTT guessing |
| PuTTY `.ppk` v2 / v3 | ✅ Including encrypted private keys (v2 via SHA-1 KDF, v3 via Argon2id) |
| Performance benchmarks (BenchmarkDotNet) | ✅ File-based app, `scripts/ssh/benchmarks/benchmarks.cs` |
| Interop matrix (real sshd container) | ✅ 13 cases × 2 OpenSSH versions, run locally on demand (`scripts/ssh/interop/`) |

#### Compression: the hard part isn't zlib, it's flush semantics

SSH compression is not "compress each packet independently" — that would barely compress
anything. It is **one persistent zlib stream across packets**: after each packet is compressed,
a flush squeezes the bytes out, **but the dictionary is not reset**: packet 100 can still
reference a string that appeared in packet 1. In zlib terms that is `Z_PARTIAL_FLUSH`.

The first version drew a **wrong conclusion** from this: "The BCL's `ZLibStream` doesn't expose
the flush mode, so we have to bring our own zlib" — and so it used BouncyCastle's `ZStream`,
and recorded it as "BouncyCastle's only non-cryptographic use".

**That conclusion is wrong**; the correction is in §11.2.10.

Two judgments:

1. **`zlib@openssh.com` and `zlib` are not the same thing**, and they differ in exactly one way:
   the former is **only enabled after authentication succeeds**. The reason is that the
   compression ratio is itself a side channel — the compressed length leaks the compressibility
   of the plaintext, which during authentication means a distinguishing attack on the password
   (the CRIME family). That is what `SshCompressorFactory.IsDelayed` encodes, and the enabling
   point is in `SshConnectionFactory` after authentication succeeds.
2. **Decompression must be capped, and the check must happen before writing out.**
   A 32 KiB packet can decompress to hundreds of MiB — if the peer wants to, it can blow up our
   memory with one packet. Compute the length first and then write; don't discover the overrun
   halfway through and roll back.

The benchmark conclusion is blunt. Measured (22,800 bytes of synthetic log lines):

```
Compressible text  22,800 → 159 bytes      (about 1/143 — this is the best case for synthetic
                                            input; real logs are usually 1/5 ~ 1/10)
Random data        22,800 → 22,813 bytes   **it got bigger**
```

So compression is still off by default —
`Compression` must be set explicitly. Turning it on for an interactive session is basically a net
loss; it only pays off for SFTP transfers of text.

#### Agent forwarding: why it must be parsed, not passed through

Treating the agent channel as a byte pipe hooked up to the local agent is the easiest approach —
and it is the one we **explicitly did not choose**. Once you pass bytes through, none of the three
security constraints in §7.2 can be implemented: you don't know which key the peer is asking to
sign with, so "forward only the specified keys" or "confirm each use" are out of the question.

So `AgentForwarder` **parses the agent protocol, then forwards**:

- When `AllowedKeys` is non-empty, keys not on the list are **filtered out** of the
  `REQUEST_IDENTITIES` reply (`KeysHidden` counts them), and a `SIGN_REQUEST` for a key not on
  the list gets `FAILURE` straight away.
- `ConfirmEachSignature` gives the host an async callback, which can pop a dialog to ask a human.
  If refused, reply `FAILURE`.
- Messages such as `ADD_IDENTITY` / `LOCK` / `UNLOCK` that would **change local agent state**
  **always get `FAILURE`** and are not forwarded. A remote server has no reason whatsoever to
  modify our local keyring.
- `MaxConcurrentChannels` caps the channel count, default 8.

The whole thing is **off** by default: no forwarding unless `AgentForwarder.RequestAsync` is called.

#### SFTP pipeline depth: "did we wait", not RTT

M2's adaptive window already fell into a trap once (§11.2.8 item 2): it used "how long between two
refills" as the signal, which is always true on a LAN, so the window grew for nothing. This time
we didn't repeat it — **the signal is "were we actually blocked".**

The in-flight request count in `SftpRequestPipeline` is a `SemaphoreSlim(initial, ceiling)`.
When sending a request we **first try `Wait(0)`**: if we get it, the pipeline isn't full yet; if we
don't, we count one `_saturationHits` and then wait normally. Every 32 requests we evaluate:

- More than half of the requests had to wait → depth **doubles** (and the extra quota is
  `Release`d), capped at `ceiling`;
- None waited at all → shrink by 25%, but not below the initial value.

The benefit of this signal is that it **carries link information by itself**: on a high-RTT long
fat pipe, in-flight requests quickly fill the semaphore, so it grows; on a LAN there's room for the
next request before the previous one has even finished sending, so it doesn't. No RTT estimate
needed, so no RTT estimate to get wrong.

#### `.ppk`: doable because the KDF PuTTY uses is available

〔At the time〕 M5's "encrypted OpenSSH private keys can't be read" (§11.2.6) still stood —
`bcrypt_pbkdf` needs Blowfish's key-schedule internals, BC only exposes `Init` + `ProcessBlock`,
and architecture principle 6 forbids writing our own primitives. **This was filled in on
2026-09-22; see §11.2.18.**

`.ppk` was doable even then: v2's KDF is concatenated SHA-1, v3's is Argon2id,
**and BC provides both directly**. So `.ppk` is supported including encrypted keys —
that's not a double standard, it's a direct consequence of the "is there a ready-made primitive" rule.

The only place in the implementation that's easy to get wrong: **the MAC is computed over the
decrypted private-key plaintext, not the ciphertext.** Get it backwards and the symptom is "MAC
mismatch even with the right password", and that message points people in completely the wrong
direction. `VerifyMac` additionally tolerates two length candidates (padded and `NaturalLength`),
because historical generators were not fully consistent here.

#### Performance benchmarks: if you claim it, you must be able to measure it

The performance claims in the README (coalesced writes, zero-copy framing) **can't just be talk**.
`scripts/ssh/benchmarks/benchmarks.cs` is a **file-based app** (src/VelaShell.Ssh/AGENTS.md §3.2:
scripts go through PowerShell or C# single-file apps; no new projects) that measures four groups:
cipher suite throughput, framing and send coalescing, compression, and SFTP wire encode/decode.

```
dotnet run -c Release -p:SignAssembly=false scripts/ssh/benchmarks/benchmarks.cs
dotnet run -c Release -p:SignAssembly=false scripts/ssh/benchmarks/benchmarks.cs -- --filter '*Wire*'
```

Two engineering pitfalls, both written into the comment at the top of the script:

1. **You must use `InProcessNoEmitToolchain`.** By default BDN regenerates a project for each
   benchmark group and compiles it, and a file-based app has no `.csproj` for it to find. The
   `Emit` variant throws outright while building the wrapper code (reporting only
   `Build Error: Exception!`).
2. **Don't pass `--job` on the command line** — it overrides that toolchain, which lands you back
   at item 1.

It caught a real problem on the spot: `SftpWire.WriteWrite`, like the other `Write*` methods,
used to **assemble the whole message into an intermediate buffer first, then write it out as a
whole** — for 32 KiB data blocks, that's an extra copy per block. After changing it to "write the
fixed-length header with one `GetSpan`, then `Write` the data body directly to output", the A/B on
the same machine:

```
| Method              | Mean     | Ratio | Allocated |
|---------------------|----------|-------|-----------|
| Direct write        |  2.53 us | 1.00  |  32.1 KB  |
| Intermediate buffer | 11.06 us | 4.37  | 160.1 KB  |
```

**4.4× faster, 5× less allocation.** Every block on the upload path goes through this once, so
this spot is worth it.

> The "intermediate buffer" in the table is a comparison implementation rewritten in the old style,
> not a verbatim restoration of the code at the time (its allocation is somewhat higher than the
> 98 KB recorded then, because the intermediate buffer grows from its default capacity).
> The direction and magnitude of the conclusion are solid, **but don't cite 4.37 as a historical fact**.

> These numbers **are not a CI gate**. BDN results are too sensitive to machine load; as a gate it
> would only produce false alarms every day. Its use is **comparing before and after a change on the
> same machine**.

#### Interop matrix: this is the real acceptance test

The 414 cases above all run on `InMemoryTransport` plus a hand-written test server.
They can verify only one thing: **that our understanding of the spec is consistent with itself.**
The real pitfalls live in the **gap** between "what OpenSSH actually does" and "what the spec says" —
the argument order of SFTP's `SYMLINK` is the most famous one (the warning comment above
`SftpWire.WriteSymLink` exists specifically to stop "well-meaning fixes").

13 cases (`[TestCategory("Interop")]`), covering: handshake and running a command,
**one handshake per KEX**, **one send/receive per cipher**, public-key authentication,
integrity of 4 MiB of output, stdin, exit code vs signal, PTY, SFTP 1 MiB round trip,
symlinks in a real directory, compression negotiation, local forwarding, keepalive replies.

Two deliberate design choices:

1. **With no server, they report `Assert.Inconclusive`, not red.**
   A test that is permanently red on your local machine will soon be ignored by everyone —
   and then on the day it really breaks, nobody will look at it either.
2. **There are two ed25519 test keys: one without a passphrase, one with.**
   〔2026-09-22〕 Encrypted OpenSSH private keys can now be read (§11.2.18),
   and that path can only be verified against a file written by the real `ssh-keygen`,
   so the script also makes a passphrase-protected copy of the same key.

Running locally:

```powershell
pwsh scripts/ssh/interop/Start-TestServer.ps1 -X11   # start sshd in docker, install xauth, wait for banner
. artifacts/interop/env.ps1
dotnet test tests/VelaShell.Ssh.Tests/VelaShell.Ssh.Tests.csproj --filter "TestCategory=Interop"
pwsh scripts/ssh/interop/Stop-TestServer.ps1
```

The interop matrix runs **two** OpenSSH versions: `latest` (the only place post-quantum KEX is
available) and `9.3` (to verify we don't silently depend on new algorithms — the real world has
plenty of devices stuck several years back). It is **not in CI**: it needs a container, is slow and
depends on the network. The former CI `interop` job only ran on main and on manual trigger, and showed
up as a permanently Skipped check on every PR; it was removed on 2026-09-23 and the matrix is now run
locally on demand with the commands above.

**At this point, the only remaining to-dos listed across §11.2 are these** (all left on purpose,
not forgotten):

- Encrypted OpenSSH private keys — `bcrypt_pbkdf` isn't available. **→ Done, see §11.2.18.**
- `streamlocal-forward@openssh.com` **→ Done, see §11.2.12.**
- `Include` and `Match` in `ssh_config` **→ Done, see §11.2.12.**
- A public `ISftpExtension` extension point; convenience wrappers for extensions such as `statvfs@openssh.com`.
- The grace period after cancelling a remote forward is hard-coded to 2 seconds, not configurable.

### 11.2.10 Compression switched to native zlib (2026-09-21)

§11.2.9 contained this conclusion:

> This rules out the BCL's `ZLibStream` — it doesn't expose the flush mode.

**That sentence was wrong, and expensively so**: it made us pull in a third-party managed zlib
implementation for something the BCL could already do, and cut a notch into the "small dependency
surface" selling point.

#### Where it went wrong

The premise was right: SSH needs "flush without resetting the dictionary", OpenSSH uses
`Z_PARTIAL_FLUSH`, and the BCL indeed doesn't let you choose the flush mode.

What was missed is the next step: **`Stream.Flush()` on a deflate stream does a `Z_SYNC_FLUSH`,
which also doesn't reset the dictionary.** `Z_SYNC_FLUSH` and `Z_PARTIAL_FLUSH` differ in only one
way — the former emits an extra empty stored block (`00 00 FF FF`), **4 extra bytes per message**,
and decompressors on both sides accept it. In other words: the BCL always had the semantics we
needed, just not under that name.

> Lesson: **"The API doesn't expose that option" is not the same as "it can't be done".**
> The question to ask is "does the semantics I need go by another name", not "is there that parameter".

#### After the switch

`ZlibCompressor` now uses `ZLibStream` to drive the runtime's own native zlib,
one persistent stream per side, one `Flush()` per message.
Two very thin `Stream` adapters connect the BCL's streaming API to our
`IBufferWriter<byte>` / `ReadOnlySequence<byte>` —
**whatever the compressor emits is written straight into the caller's writer, decompression reads
segment by segment, and neither side has an intermediate buffer.**
(The old implementation had to `ToArray()` the payload first when decompressing.)

A/B on the same machine (one stream compressing 8 messages in a row):

```
| Payload | BCL native | BouncyCastle | Faster | Allocated                 |
|---------|-----------|--------------|--------|---------------------------|
| 4 KiB   |  17.0 us  |     97.3 us  |  5.7×  |  1.43 KB vs 311 KB (218×) |
| 32 KiB  |  57.9 us  |    850.9 us  | 14.7×  |  4.48 KB vs 536 KB (120×) |
```

The size of the gap is no surprise: BouncyCastle's zlib is a **managed port of JZlib**,
while the BCL goes through the native zlib in the runtime.

**Incidentally, the dependency story is now clean too**: BouncyCastle now **only provides
cryptographic primitives**, with no second use — `Directory.Packages.props` and `NOTICE.md` have
both been updated.

#### A pitfall that must be handled: `UseStrictValidation`

`System.IO.Compression.UseStrictValidation` is an AppContext switch (off by default).
When on, "read reached no more data" is judged as a truncated stream and throws
`InvalidDataException`. And **an SSH zlib stream is flushed continually and never ends** — so
**every single message hits it after decompression**. Left unhandled, any app with that switch on
fails across the board as soon as it connects.

How it's handled: only treat it as end-of-payload when "this particular read produced no data at
all" (the data has already come back from the previous reads). **A payload that decompresses to
nothing is not exempt** — that still errors, otherwise invalid zlib data would slip through silently.

This can't go into a regular unit test: the switch can only be set once at process start,
and the static field that reads it is read once per process, so cases running in parallel in the
same process can't each set their own. So it's a **separate file-based script**, run as its own
step in CI:

```bash
dotnet run scripts/ssh/compression/verify-strict-validation.cs
```

It verifies three things: messages round-trip with the switch on, the dictionary really is
retained across messages (on-the-wire bytes shrink message by message), and invalid data is
still rejected.

#### Where this code came from

The implementation approach comes from VelaShell Labs' own upstream contribution
[tmds/Tmds.Ssh#513](https://github.com/tmds/Tmds.Ssh/pull/513) (author `joesdu`,
the author of this project), **not third-party code**. What was taken from there are three
**facts**: `Z_SYNC_FLUSH` and `Z_PARTIAL_FLUSH` are equivalent in the SSH scenario (at a cost of
4 bytes/message), `CompressionLevel.Optimal` maps to zlib level 6 (the level OpenSSH uses),
and this `UseStrictValidation` pitfall.

The code itself was rewritten against our own interfaces (`ISshCompressor` +
`IBufferWriter`/`ReadOnlySequence`) — the upstream version is wrapped around their
`IPacketEncryptor`/`IPacketDecryptor`, a completely different structure.

> Worth writing down: **clean-room discipline guards against "copying someone else's structure",
> not against "knowing facts".**
> That `Z_SYNC_FLUSH` doesn't reset the dictionary is a fact from RFC 1951; it's the same no matter
> where you learned it. The difference is whether, once you know it, you write it yourself or copy
> someone else's file over.

### 11.2.11 Rekeying (2026-09-21)

This item isn't "filling in a protocol detail"; it's **fixing a hole that drops long-lived connections**.

OpenSSH's `RekeyLimit` defaults to 1 GiB or 1 hour, and when it's reached it sends
`SSH_MSG_KEXINIT` on its own. A client that doesn't respond doesn't just "lack a feature"; instead:

- A shell that's been open all afternoon suddenly drops;
- A large file transfer drops halfway through.

VelaShell is a terminal product; long-lived connections and large files are its daily bread — so
this is a hard requirement. Two more reasons: AES-GCM's nonce advances deterministically, and nonce
reuse under the same key leaks the authentication key and leads directly to forgeability (which
is why the spec says to **force** a rekey as it approaches 2⁶⁴); and every rekey is a fresh DH /
ML-KEM, so a leaked session key only exposes that one window.

**437 test cases: 424 green + 13 interop cases self-reporting Inconclusive.**

| Item | Status |
| --- | --- |
| Handle server-initiated rekey | ✅ Runs to completion in place on the receive loop |
| Send gate actually wired into the send path | ✅ **Previously it was just an ornament with tests** |
| Compression context reset along with keys | ✅ Swapped in the same critical section as the keys |
| `session_id` unchanged across rekeys | ✅ Pinned by a test |
| Channel data during rekey | ✅ Receive direction dispatches as usual; send direction is stashed, then drains in order |
| Several consecutive rekeys | ✅ |

#### The send gate was never wired in

`SendGate` is the component §5.4 describes in the most detail, and it has a whole
`SendGateTests.cs`, **but it was never wired into `SshConnection.SendAsync`.** In other words,
before this change, even if we could answer a rekey, we would have kept sending channel data
between `KEXINIT` and `NEWKEYS` — a clear violation of RFC 4253 §7.1.

> Lesson: **"has an implementation + has unit tests" is not the same as "wired in".**
> A component's unit tests all being green only says the component itself is correct, not that
> anyone is using it. This kind of hole grows especially easily out of architecture docs — the doc
> describes it as core, so readers of the doc (including me three hours ago) assume it's already working.

Once wired in, `SendAsync` looks like this: take a ticket → take the send lock → ask the gate
(`Send` / `Stashed` / `StashFull`) → if full, release the lock, wait, and retry.
**When full, wait — never drop**: if the peer is slow to finish the rekey, better to apply
backpressure to the sender than to accumulate without bound.

#### Two "must be atomic" spots

1. **Sending `NEWKEYS` and switching the send-side state.** The next message we send after
   `NEWKEYS` must use the new keys; if another sender slips in between, that frame goes out under
   the old keys while the peer is already decrypting with the new ones. So both are done inside
   the same `_sendLock` critical section.
2. **Swapping the compression context together with the keys.** Compression comes before
   encryption, and both are bounded by `NEWKEYS`. For this, both switch methods of
   `ISshKexTransport` take an `ISshCompressor?` —
   **the initial exchange passes `null`**, because `zlib@openssh.com` is only enabled after
   authentication succeeds (passing non-null would break that CRIME protection). This "nullable"
   isn't laziness; it encodes exactly the semantic difference between the initial exchange and a rekey.

#### Why it runs in place on the receive loop

The reason the `ISshKexTransport` abstraction exists: during the initial exchange the transport has
only one reader and one writer, so it can read and write directly; during a rekey, reads belong to
the receive loop and writes to the send lock, so the key exchange has to go through them.

And the rekey's key exchange **runs to completion on the receive loop** rather than in a separate
task — because **not a single message may slip in** between "read the peer's `NEWKEYS`" and
"switch to the new receive keys". Put it on another task and that ordering would have to be
guaranteed by extra synchronization, which is asking for trouble.

The cost is that the receive loop is occupied during the rekey, so
`RekeyKexTransport.ReadPacketAsync` **dispatches non-transport-layer messages in place** and only
hands over the key exchange's own messages — otherwise a running SFTP transfer would stall
completely for the one or two RTTs of the rekey.

#### Caught a more serious bug along the way: the compression scratch buffer was shared by both directions

While writing the rekey + compression test, it failed **before the rekey** — an 8 KB payload
received 0 bytes. Meanwhile the same connection, same compression, with a 1.9 KB payload passed every time.

The cause was in `SshPacketTransport`:

```csharp
// Old code — one buffer, two directions
private readonly ArrayBufferWriter<byte> _compressionBuffer = new(4096);

WritePacket:      _compressionBuffer.ResetWrittenCount(); … Compress(payload, _compressionBuffer);
DecompressPayload: _compressionBuffer.ResetWrittenCount(); … return _compressionBuffer.WrittenMemory;
```

**Receiving and sending are concurrent**: the receive loop is reading while N channel pumps are
writing. Decompression had just written the payload into the buffer, and before the caller was
done with it, a concurrent `WritePacket` called `ResetWrittenCount()` and wiped it —
the caller got a payload of length 0 or a truncated one.

Why the symptom was so confusing: with small payloads there's almost no chance of interleaving
between a send and a receive, so it doesn't hit; once the data volume rises you start seeing
"received 0 bytes" or "message payload empty, no message number",
**and neither of those points at compression**.

The fix is simply one buffer per direction. But what's worth recording is **why the unit tests
didn't catch it**: `CompressionTests` even includes "large payloads round-trip", but they are all
driven sequentially — **compress fully, then decompress** — and this bug only appears when
compression and decompression interleave.

> Lesson: **test concurrent things concurrently.**
> Sequentially driven cases can verify protocol understanding, not shared state.
> There's now an added case `整条连接开着压缩也能跑通` ("a whole connection works with compression on"):
> it goes through the full connection setup and genuinely sends and receives concurrently — this one
> catches it; inject the bug back and it fails immediately.

#### Initiating it ourselves (completed the same day)

The thresholds live in `SshRekeyPolicy` (default **1 GiB / 1 hour / 2³¹ messages**, **on** by default).
`SshConnection.StartRekeyAsync()` is also public, so the host can pick its own moment.

**The message-count one is the hard line.** SSH sequence numbers are 32-bit, and the AES-GCM nonce
advances once per message — both break at 2³², and nonce reuse is **catastrophic** for GCM
(the authentication key can be recovered, and from there forgeries made). Byte count and duration
are just recommendations from RFC 4253 §9.
As an aside: **there's no need to expose GCM's invocation counter** —
it advances once per message, so "how many messages have been sent under this key" is its exact
value, and every cipher suite has that number; the same goes for ChaCha20-Poly1305's 32-bit
sequence number.

The three paths converge into one in `OnPeerKexInitAsync`:

| Who initiates | `_ourPendingKexInit` | Behavior |
| --- | --- | --- |
| Peer | `null` | The runner sends our `KEXINIT` |
| Us | The one already sent | Hand it to the runner, **don't send a second one** (sending twice is a protocol violation) |
| Both at once | The one already sent | Exactly the same as "we initiated" — RFC 4253 §7.1 says this is legal and happens only once |

So the "both sides initiate simultaneously" case, which looks like it needs special handling,
**needs not a single extra line of code**.

`StartRekeyAsync` **returns as soon as `KEXINIT` is sent**, without waiting for negotiation to
finish: the key exchange has to read the peer's messages, and the only reader of this transport
is the receive loop. Calling it again while negotiation is in progress is a no-op.

Diagnostics: `LastRekeyReason` says which threshold triggered it ("单向报文数达到 1024（阈值 1024）" —
"messages in one direction reached 1024 (threshold 1024)"), and `RekeyCount` is the number of
completed rekeys. When troubleshooting you need to be able to answer "why did this connection just change keys".

##### An API trap of our own making

In the first version of `SshRekeyPolicy`, `default` for `MaxInterval` was translated into "1 hour"
(because `TimeSpan` can't be a default parameter value in a record struct). The result:
**the duration threshold in `Disabled` couldn't be turned off at all** — `Disabled.IsEnabled`
was actually `true`.

It was the `Assert.IsFalse(Disabled.IsEnabled)` in `阈值低于下限会被当场拒绝` ("thresholds below
the floor are rejected on the spot") that flushed it out.

The fix was to flatten the semantics: **for all three thresholds, "0 means ignore this one"**, so
`default(SshRekeyPolicy)` and `Disabled` are the same thing —
an all-zero struct should mean "do nothing". The opinionated set of values is written in `Default`,
spelled out in full, not implied through default parameter values. The test gained an
`Assert.AreEqual(Disabled, zeroed)` to pin it down.

> Lesson: **default parameter values shouldn't carry semantics.**
> They look convenient, but in effect you're writing half the truth in each of two places —
> and once the two halves disagree (say, `Disabled` wants to turn off a threshold that a default
> value turned on), the compiler won't say a word.

##### How to accumulate a thousand messages in a test (two failed attempts)

Verifying "the monitor loop is really watching" requires pushing the message count past 1024.
Two approaches were tried, and neither worked:

1. **Run commands repeatedly** — each command is a full round of open channel + request + data +
   exit status + close. Fine on its own, but under a full parallel run it's slow enough to hit the
   25-second timeout.
2. **Pour 4000 one-byte chunks into stdin** — the stdin pump coalesces them; measured, 4000 writes
   produced only **one or two** messages (the whole connection had just 10 in total).
   This was measured directly by the `PacketsSent` diagnostic output.

The knob that's actually controllable is **the peer's fragmentation granularity**: the server
splits according to the max packet we advertise. Set `ReceiveMaxPacketBytes` to 256 bytes, have the
server emit 300 KB, and that's over a thousand messages — cheap and deterministic; the test dropped
from 6–25 seconds to **1.2 seconds**, and incidentally verifies "a rekey happening mid-transfer
doesn't lose data".

> Lesson: **to push a quantity up, first find the knob that actually determines it.**
> The first two attempts pushed on "do it more times", but what determines the message count isn't
> the number of times, it's the fragmentation granularity.

#### Not done yet

- Behavioral differences on the peer side when the client initiates (for example, some
  implementations may refuse or delay their response) —
  not verified in real interop; the interop cases have none that crosses 1 GiB yet
  (one run would move 1 GiB, which doesn't belong in regular CI).

### 11.2.12 Closing the "deliberately left gaps" (2026-09-21)

Three gaps: two closed, one couldn't be — and **the root cause of the one that couldn't was a
single wrong line of documentation**.

**449 test cases: 436 green + 13 interop cases self-reporting Inconclusive.**

#### `streamlocal-forward@openssh.com`

Reverse-direction Unix socket forwarding (`ssh -R /remote/path:/local/path`).
The typical use is handing the local `docker.sock` or a database socket to the remote —
going through a socket instead of a port means other users on the remote machine **can't see it
or connect to it** (file permissions decide).

It differs from `tcpip-forward` in only three places: the name of the global request, replacing
`addr ‖ port` with a single `string socket_path`, and the callback going through
`forwarded-streamlocal@openssh.com`.
**Metering, concurrency slots, data pumping, events and teardown are all identical** — so no new
class was created; instead `RemoteForwarder` gained a nullable `RemoteSocketPath`,
and non-null means Unix socket mode. Three branch points, everything else shared.

Along the way, one piece of public surface was added: `RemoteEndpointName` — `bind:port` for TCP,
the path for sockets. The tunnel panel needs to show "where this forward is open", and the two forms
need a unified way of saying so; otherwise callers would have to write that ternary themselves
(architecture principle 4).

4 test cases, one of which is **truly end-to-end**: stand up a real Unix domain socket,
have the server open a `forwarded-streamlocal` back, and pump data both ways.
(`Socket.OSSupportsUnixDomainSockets` is true on Windows 10+, so no skipping needed.)

#### `Include` and `Match` in `ssh_config`

The original verdict was "deliberately not done", on the grounds that `Match exec` means parsing a
config file can run arbitrary programs.
**That reasoning only holds for `exec`**, yet it killed all of `Match` and the innocent `Include` along with it.

Both are now done, each with a constraint:

- **`Include` is only expanded in `LoadAsync`.** `Parse` is pure text parsing, has no base
  directory, and shouldn't touch the file system. Expansion has a **depth limit (16, same as
  OpenSSH) and cycle detection** — `a` including `b` and `b` including `a` is easy to write, and
  without cycle detection the symptom is **the whole process freezing while reading config**.
  One test case pins exactly this.
  Glob results are **sorted**: directory enumeration order differs across file systems,
  and `ssh_config` is "first value wins" — nondeterministic order means nondeterministic results.
- **`Match exec` doesn't execute by default.** Without an evaluator, blocks with an `exec`
  condition **never match**. To use it, pass in your own `SshConfigMatchContext.ExecEvaluator` —
  so the decision "should an external command be run on this machine" **lands explicitly on the
  caller**, rather than hiding in the library's default behavior.

`Match` supports `all` / `host` / `originalhost` / `user` / `localuser` / `exec`,
conditions are ANDed, and `!` negation is supported. `canonical` / `final` **never match** —
we don't do hostname canonicalization, and silently treating them as true would make a config
written for canonicalization produce completely different results.

Two judgments: **insufficient information means no match** (when the user is unknown,
`Match user` doesn't guess one), and **unrecognized conditions don't match** (misreading one
would make a block take effect that shouldn't).

A parsing pitfall: in `Match exec "test -f /etc/special"` the command is **one** argument.
Without quote handling it gets split into three pieces, the last two become two unrecognized
conditions, and the whole block silently stops working —
it was that exec test case that caught it. `exec`'s argument also **must not be split on commas**
(commands can contain commas).

#### Encrypted OpenSSH private keys: couldn't be closed at the time, root cause found

> **〔2026-09-22 postscript〕 This has now been filled in; see §11.2.18.**
> All of the technical judgments below still hold — what changed isn't the facts but the trade-off
> in the last row of that table: the cost of "keep the status quo" was re-estimated, and the
> conclusion is that it's more expensive than "open one documented exception".

`src/VelaShell.Ssh/AGENTS.md` §3.3 said "what the BCL lacks goes through BouncyCastle (…… `bcrypt_pbkdf` ……)".

**That line was wrong, and the error directly caused this gap.**

Actually checking BouncyCastle 2.7.0: `Org.BouncyCastle.Crypto.Generators.BCrypt`
does have `DeriveRawKey` / `EncryptMagicString` / `ProcessTableWithSalt`,
but they are **all `internal`**, and the public `Generate` does
**standard bcrypt with 2^cost rounds**, not bcrypt_pbkdf's variant of
"64 rounds of `expand0state` + a different magic string + byte-order swap".
`BlowfishEngine` likewise only offers `Init` + `ProcessBlock`, with no access to the key-schedule internals.

So the conclusion didn't change — **but the reason went from "I remember it can't" to "checked,
and it really can't"**, and that line in `src/VelaShell.Ssh/AGENTS.md` has been corrected.

The three remaining paths are not mine to choose on behalf of users:

| Path | Cost |
| --- | --- |
| Write `bcrypt_pbkdf` ourselves | Violates §3.3 "no hand-written cryptographic primitives. **No exceptions.**" ← **This is the one ultimately chosen, with that sentence changed to "one exception, documented on record" (§11.2.18)** |
| Pull in another dependency | Dependency surface is one of this library's selling points; and nothing suitable was found (license + maintenance + for this one thing only) |
| Keep the status quo | Two actionable paths are already in the error message: `ssh-keygen -p -m PKCS8`, or go through ssh-agent |

> Worth writing down here: **"deliberately not done" and "thought it couldn't be done at the time" are two different things.**
> Of the three gaps, two belonged to the latter — one inflated the risk of `exec` to all of `Match`,
> the other was a wrongly written dependency list entry.
> A gap list should be **re-verified** periodically, not just copied forward forever.
### 11.2.13 Hooking up real OpenSSH: a bug we could never have caught ourselves (2026-09-21)

The interop cases had been written but never run — there was no Docker. This time they ran,
and **the very first connection to a real OpenSSH failed across the board**.

Meanwhile the 449 local cases were all green.

#### Symptoms

```
✘ kex=curve25519-sha256          -> packet integrity check failed
✘ kex=ecdh-sha2-nistp256         -> packet integrity check failed
✘ kex=mlkem768x25519-sha256      -> packet integrity check failed
✘ cipher=aes256-gcm@openssh.com  -> packet integrity check failed
✘ cipher=aes128-gcm@openssh.com  -> packet integrity check failed
✘ cipher=chacha20-poly1305       -> invalid frame header: packet_length=231406730
✘ cipher=aes256-ctr              -> packet integrity check failed
```

**Every KEX failed, every cipher suite failed** — so the problem was not in any particular algorithm
but in the part they all share. The chacha20 line is especially telling: the length field it decrypted was pure noise,
which means **the keys we were holding were simply wrong**.

#### Root cause: two extra length prefixes in key derivation

RFC 4253 §7.2:

```
K1 = HASH(K || H || "A" || session_id)
```

That `||` is **plain concatenation**. `K` looks like it "has a length prefix", but that is because it is itself
encoded as an mpint — the prefix is part of the mpint. **`H` and `session_id` are raw bytes.**

We had written:

```csharp
WriteValue(ref prefixWriter, sharedSecret, secretEncoding);
prefixWriter.WriteString(exchangeHash);   // ← an extra 4-byte length prefix
...
roundWriter.WriteByte((byte)letter);
roundWriter.WriteString(sessionId);       // ← and another 4 bytes
```

After changing these to `WriteRaw`, all seven combinations connected, **including the post-quantum `mlkem768x25519-sha256`**.

#### Why not one of 449 cases caught it

This is the single most worth-recording thing in the whole project.

1. **The signature still verified.** `H` is computed on a different code path
   (`SshExchangeHash.Compute`, where the `string` prefix is correct).
   So both sides agreed **completely** on `K` and `H`, and verification of the server's signature went through —
   the handshake looked perfectly fine right up until the first encrypted packet failed to decrypt.
2. **Our own test server used the same `DeriveKey`.** Both sides were wrong in the same way, so they interoperated perfectly.
   `TestSshServer` never said "you got it wrong", because it computed it the same way.
3. **The unit test copied the mistake.** In the case `派生的第一轮就是公式本身` ("the first derivation round is the formula itself"),
   the expected value was assembled with `w.String(h)` — it asserted the bug itself.
   After the implementation was fixed, those two cases were the first to go red.

> The opening sentence of §10.3 already said it:
> "No matter how complete our own test stubs are, they can only verify that **our understanding of the spec is consistent with itself**."
> This was the first time that sentence actually paid out — and the price was that until then I had believed this library worked.

#### Two bugs in **the test cases themselves**, fixed along the way

The real server also exposed places where the tests were written too loosely:

| Case | Problem |
| --- | --- |
| `压缩能与OpenSSH协商上` ("compression negotiates with OpenSSH") | Asserted `StandardOutput.Length > 50000` — that is a **character count**, and each line of 13 CJK characters = 14 chars / 40 bytes. Changed to assert on line count and byte count |
| `本地转发能穿过真实服务端` ("local forwarding goes through a real server") | The tunnel connected to `127.0.0.1:22`, but sshd in the container listens on **2222** — the server answered "Connection refused", which looked like forwarding was broken |

There was also an environment issue: 13 cases connecting to the same sshd **in parallel** ran into OpenSSH's
`MaxStartups` (default 10 concurrent unauthenticated connections), and the excess ones were dropped **before the version identification string was sent**
— the symptom was "the peer closed the connection before sending its version identification string", which looked like our bug.
Added `[DoNotParallelize]` to that class.

#### Current state

```
OpenSSH 10.3 (latest)    13/13 ✔
OpenSSH 9.3              13/13 ✔
449 local cases           436 green + 13 interop (report Inconclusive when there is no server)
```

Compression was verified separately: over real `zlib@openssh.com`, 100 / 2000 / 20000 lines
(up to 800 KB), **byte counts identical to the byte**, with compression both on and off.

> Lessons, in order of weight:
> 1. **"All tests green" and "it works" are two different things.** The gap between them is called "the peer is not you".
> 2. **A single bug can fool the implementation, the test stub and the unit tests all at once, as long as they come from the same head.**
>    Interop tests are not "more tests"; they are **the only kind of test that brings in outside facts**.
> 3. **It should have run long ago.** The cases were written two weeks earlier; all that was missing was a container.
>    The cost of "we'll do it once we have the environment" is that all confidence in the library up to that point was hollow.

### 11.2.14 X11 forwarding (2026-09-21)

〔Scope〕**Only the forwarding end, no X server** (that is what §12 "Explicitly not doing" already said).
`x11` channels opened back by the server are connected to an X display that already exists on the local machine.

**479 cases: 464 green + 15 interop (all run when a server is present; report Inconclusive when there is none).**

#### The security core: a fake cookie

X11 has no client isolation — **any client connected to the same display can read other clients' keystrokes,
capture other clients' windows, and inject events into other clients' windows**. So handing the local display to the remote side
means handing the input and output of every graphical session on this machine to the remote side.

Therefore:

1. By default it is **not requested**; turning it on requires an explicit call to `X11Forwarder.RequestAsync`.
2. The default is **untrusted** mode (corresponding to `ssh -X`).
3. **What is sent to the server is always a random fake cookie.** The real cookie never leaves the local machine.
   A remote X client connects with the fake cookie → we check it → swap in the real cookie → only then pass it on to the X server.
   If the check fails, it is rejected, and the local X server is **not even touched**.

Comparison goes through `CryptographicOperations.FixedTimeEquals`. A short-circuiting byte-by-byte comparison leaks
"how many of the first few bytes were right", and an attacker can open channel after channel and slowly try.

#### Three places that are easy to get wrong

| Place | Symptom if wrong |
| --- | --- |
| The cookie field of `x11-req` is **hexadecimal text**, not raw bytes | What the remote `xauth` stores does not match what we check, and the only error is "connection refused" |
| The length fields of the X11 setup message **must be read in both byte orders** (the first byte, `'B'`/`'l'`, decides) | "Some clients can connect, some can't" — looks exactly like a random failure |
| The setup message may **arrive in several pieces** | Random failures on small MTUs or slow links |

#### One deliberate difference from OpenSSH

OpenSSH's `ForwardX11Timeout` only applies to untrusted mode. **Ours applies to both modes.**

Reason: "an expiry that only takes effect in one of the modes" is an API that trips people up —
and trusted mode is precisely the far more dangerous one, yet it would have no expiry at all, which makes no sense.
For long sessions that need it indefinitely, set `Timeout = TimeSpan.Zero` explicitly.

This came up while writing the cases: `过期之后不再接受新的x11通道` ("no new x11 channels accepted after expiry") failed in trusted mode,
and only on looking into it did I realize that "copying OpenSSH" had copied a counterintuitive behavior here.

#### How far it has been verified

| Layer | Status |
| --- | --- |
| Pure functions (DISPLAY parsing / `.Xauthority` / setup message) | ✅ 17 cases |
| Forwarder (fake cookie, cookie swap, rejection path, expiry, concurrency limit) | ✅ 7 cases, including **end-to-end**: start a fake X server and verify that what lands on it is the **real** cookie |
| **Rejection path** on real OpenSSH | ✅ Explicitly rejected when `X11Forwarding no`, with a message that points to `sshd_config` |
| **Success path** on real OpenSSH | ✅ **15/15 on each of OpenSSH 10.3 and 9.3** (see below) |

#### Checked against real OpenSSH

On the day it was written only the "rejection path" had been verified; the success path had not yet been checked against a real server —
and §11.2.13 had just shown how deadly that gap can be. So it was filled in.

`Start-TestServer.ps1 -X11` installs `xauth` in the container and turns on `X11Forwarding`,
so two interop cases can run:

| Case | What it verifies |
| --- | --- |
| `真实服务端接受x11_req并把假cookie存进xauth` ("real server accepts x11_req and stores the fake cookie in xauth") | What the real `xauth list` stores is **exactly the fake cookie we sent** — this one directly nails down the "cookie field is hexadecimal text" pitfall |
| `真实服务端开回的x11通道会被接受并换成真cookie` ("x11 channel opened back by the real server is accepted and swapped to the real cookie") | A process in the container connects to the `DISPLAY` that sshd set up, sshd opens an `x11` channel back, we check the fake cookie, swap in the real cookie, and it lands on a fake X server on the local machine |

In the second case, the X11 setup message is **fed in through stdin**, not spliced into the command line —
the fake cookie in the message is only known after `x11-req` has been sent, whereas the command line has to be fixed before that.

##### Two pitfalls hit (both in the scripts, not in the library)

1. **The location of sshd's config file cannot be hard-coded.**
   The `latest` image from `linuxserver` runs `sshd -f /config/sshd/sshd_config`,
   while the `9.3` one uses `/etc/ssh/sshd_config`. Editing the wrong file is extremely confusing:
   the file clearly says `X11Forwarding yes`, yet the server still refuses — because that file is never read.
   Now the `-f` is read out of `/proc/<pid>/cmdline`.
2. **`apk` needs retries, and its output must be printed on failure.**
   For the first second or two after the container starts, the network is not up yet. The first version sent the output to `/dev/null`,
   so the only error left was "failed to install xauth" — which points nowhere.

#### Why no Dockerfile

`scripts/ssh/interop/Dockerfile` was written, but on this machine **Docker Hub cannot be pulled from**
(`auth.docker.io` times out), while `lscr.io` works. So we fell back: install packages and edit config
inside an already-running container. Docker Hub is reachable on CI, so it can switch back to building the image there.

#### Independence

The implementation approach comes from the upstream PR submitted by this project's own author,
[tmds/Tmds.Ssh#514](https://github.com/tmds/Tmds.Ssh/pull/514) (author `joesdu`).
What was taken from it are **behavioral facts**: the fields and timing of `x11-req`, the fake-cookie swap flow,
the two byte orders, how to use `xauth generate`, and whether a failure should throw or degrade.

The code was rewritten in our own structure (`IIncomingChannelHandler` + the existing
`DuplexRelay` / `ChannelRelayEndpoint` relay layer),
and the spec was written first (`spec/07` §7.5) and then implemented.

#### Addendum (2026-09-24): local connector

The host now has a built-in X server (`VelaShell.XServer`, in the same repository, neither library referencing the other). When
the local display lives in the same process, connecting to a local port is only a detour. `X11ForwardOptions.LocalConnector`
lets the caller supply a connector: it is called once per `x11` channel for a duplex stream, and the setup message is written
into it with the cookie replaced by `LocalCookie` (empty if none was given) (`spec/07` §7.5.9). **The fake-cookie check is
unchanged** — that layer guards against the remote side. Trusted mode only: untrusted mode needs `xauth` to reach the local
display and sign a restricted cookie, so setting both makes the request fail instead of silently falling back to trusted.
An unavailable connector side is treated as "local display unreachable". Three tests (relaying with cookie replacement,
unavailable connector, combined with untrusted mode).

### 11.2.15 That intermittent hang: the channel was registered too late (2026-09-22)

Twenty full runs, three red, and a different case red each time —
`深度不够时会自己长大` ("grows itself when depth is insufficient"), `窗口长满之后数据依然一字节不差` ("data still byte-exact after the window fills"), `带宽确实被限住了` ("bandwidth really is limited"),
and before that `多条通道互不干扰` ("multiple channels don't interfere"), `动态转发的目标由SOCKS握手给出` ("dynamic forwarding target comes from the SOCKS handshake").
The symptom was either the 30-second global timeout, or half the data missing, or the server reporting an AEAD tag mismatch.

It finally turned out to be **two independent things**, plus a problem in one test itself.

#### 1. `_channels` registration came after sending — the reply got dropped on the floor

```csharp
// old code
lock (_stateLock) { …; _pendingOpens[localId] = completion; }

await SendAsync(CHANNEL_OPEN);          // ← the bytes are already on the wire

lock (_stateLock) { _channels[localId] = channel; }   // ← registered after that

return await completion.Task…;
```

The receive loop is **another thread**. On an in-memory transport the server can send `OPEN_CONFIRMATION` back
before that `await` resumes, and then:

```csharp
lock (_stateLock)
{
    _pendingOpens.Remove(recipient, out completion);   // taken out
    _channels.TryGetValue(recipient, out channel);     // not there yet → null
}
if (completion is null || channel is null) return;     // reply dropped
```

`completion` has been removed from the table, yet nobody completes it — `OpenChannelAsync`
then waits until the end of time. Which case hangs depends purely on which of the two threads runs first, so **it shows a different face every time**.

The fix is to move the registration into the lock that creates the channel, **before sending**:

```csharp
_pendingOpens[localId] = completion;
_channels[localId] = channel;          // ← same lock as pending, same moment
```

This also fixed an old debt it exposed: the limit check used to be
`_channels.Count + _pendingOpens.Count >= MaxChannels`.
A channel being opened is now **in both tables**, so adding them counts it twice —
effectively halving the limit. Changed to count only `_channels`.

The regression case `发出通道打开请求之前通道就已经登记好` ("channel is registered before the open request is sent") doesn't gamble on timing;
instead it uses an observer wrapped around the client's stream to pin down the invariant:
**at the moment the CHANNEL_OPEN bytes land on the stream, the channel must already be in `_channels`**.
Move the registration back after the `await` and it goes red in 126 milliseconds, not a 30-second timeout.

#### 2. `DelayedStream` disposed its write lock during teardown

```csharp
await _inner.DisposeAsync();
_writeLock.Dispose();     // ← whoever is waiting in WaitAsync will now wait forever
```

`SemaphoreSlim.Dispose` **does not wake waiters**; their Tasks never complete.
And this stream is exactly the place where "writes wait for latency", so someone queued behind the write lock during teardown is the norm.
So "closing the stream" turned into "that writer hangs until the timeout".
Fix: don't Dispose — we never take `AvailableWaitHandle`,
and without that handle a SemaphoreSlim holds no unmanaged resources.

#### 3. The assertion in `带宽确实被限住了` ("bandwidth really is limited") sat right on the boundary

The rate limiter **waits for budget first, then records the cost**, so N writes only wait N-1 times:
the expected value of 4 × 50 KB at 1 MB/s is exactly 150 ms, and the assertion said `>= 150 ms`.
Measure 149 ms and it goes red — that isn't bad luck, it's **a miscalculation**.
Changed to 5 writes per the actual model (expected 200 ms), with a 25% margin on the threshold.

#### How it was found

The first two rounds of guessing were wrong, and **convincingly wrong**:
first I suspected the watermark bug from §11.2.8 hadn't been fully fixed (its symptom is identical),
then I suspected the sync-over-async in `SftpFileStream` was starving the thread pool.
Both were investigated; neither was it.

What actually worked were three things:

1. **Turn the intermittent failure into a scene you can capture.** Write a watchdog: run the whole suite, and if it hasn't finished after 25 seconds,
   `dotnet-dump collect` a full dump, then use `dumpasync` to see who is waiting on whom.
   At a glance: `OpenChannelAsync` stuck waiting on a TCS, while both receive loops were idle.
2. **Diagnostics must not change the timing.** The first version wrote every message number to a file,
   and forty runs didn't reproduce it once — the I/O covered up the race window.
   Switched to recording into a `byte[64]`, zero allocation and zero I/O, and it was caught on the twelfth run.
   Reading from the dump, the message numbers the client received were `20 31 21 6 51 52 91`,
   and the last one, `91`, is exactly `OPEN_CONFIRMATION`: **the reply had arrived, and someone was still waiting**.
3. **Don't read `Dictionary`'s fields on assumption.** `_count` is **the number of slots ever used**;
   `Remove` doesn't decrement it, it increments `_freeCount`.
   I stared at `_count == 1` and reasoned for ages that "the entry is still there", when in fact
   `_freeCount == 1` — it had long since been taken out.
   That one misreading sent me several extra laps in the wrong direction.

> **An intermittent failure is either root-caused or not yet investigated.** In tests it is a random red light;
> on a real link it is "occasionally can't open a channel" and "occasionally loses half the transfer".
> Until thirty runs in a row are green, you cannot say it's fixed.

### 11.2.16 Self-checking pixel dimensions against upstream #519 (2026-09-22)

[tmds/Tmds.Ssh#519](https://github.com/tmds/Tmds.Ssh/pull/519) (author `buchmiet`)
says: `pty-req` and `window-change` each carry four `uint32`s (columns, rows, width in pixels, height in pixels),
while upstream only exposed the character dimensions, and the two pixel ones were **always 0**.

It isn't our PR, so only its problem description was read, not its implementation — the conclusions come from
self-checking our own code against RFC 4254 §6.2 / §6.7.

**The main problem does not exist here.** Pixel dimensions have been first-class in this library from the start (a decision in §5.3,
and indeed one of the two reasons we originally decided to write it ourselves): the four fields of `TerminalSize` run through
`OpenShellAsync` → `pty-req` and `ResizeAsync` → `window-change`,
both paths have cases, and `伪终端能开起来` ("a pseudo-terminal can be opened") even runs against real OpenSSH.

**But the second thing it mentions in passing, we really do have — and more broadly.**
That PR rejects negative **pixel** dimensions; all four of our fields were unchecked casts like
`writer.WriteUInt32((uint)size.PixelWidth)`,
with not a single validation — columns and rows were missing it too.

So `-1` went out as `4294967295`. The remote side accepts it as-is and lays out for four billion columns:
**that is not "wrong size", it is garbage**, and the error shows up in the remote program, with no way to trace it back to the call site.

The fix puts validation into the type itself rather than at the two serialization points:

- All four properties get hand-written `init` accessors — auto-properties can't hold validation,
  and with it in `init`, `with` can't bypass it either;
- Hence it can no longer be a positional record (that would force all four to be auto-properties);
  `Deconstruct` is added back by hand, and a case pins down that it still exists;
- `0` is still legal. It means "unknown"; rejecting it would only force callers to make up a number.

Three cases: construction rejects negatives (once for each of the four fields), `with` to a negative is rejected just the same,
`0` is legal and `Deconstruct` is still there.

> The value of this one isn't in those few lines of validation, but in **"the main problem isn't here" not meaning "this PR has nothing to do with us"**.
> Following someone else's bug report into your own code is far more reliable than judging from the title whether you've been hit.

### 11.2.17 CI narrowed to a single platform, and the repo turned out not to build clean at all (2026-09-22)

#### CI: three-platform matrix cut to one

This is a class library with a single TFM (net11.0) — the same source compiles to the same IL on all three platforms,
so "per-platform builds" buy nothing; they just multiply CI time by three.

What the matrix was really meant to buy was platform coverage for **running** the tests. And there are only two places with genuinely platform-specific code:
`SshAgentClient` (Windows named pipes vs `SSH_AUTH_SOCK`) and
`X11Display` (Windows has only TCP; Unix-likes add `/tmp/.X11-unix/X{n}`,
and Linux adds one more in the abstract namespace).

The Windows branches of those two **are exercised every time tests run in day-to-day development** (the dev machine is Windows),
so running Linux on CI is exactly the complement of the dev environment. The only difference between macOS and Linux is
the single "skip the abstract namespace" branch, guarded by `OperatingSystem.IsLinux()`.

> The premise of this trade-off is written into the workflow's comments: **it depends on "the dev machine is Windows"**.
> If the main environment ever changes, or platform-specific code grows, put the matrix back.

#### The big thing found along the way: `dotnet build` had been lying all along

After changing CI I checked whether `dotnet pack` would pass — it **blew up** with a pile of IDE analyzer errors.
At the same moment, `dotnet build` was green.

The reason is that **incremental builds don't re-run analyzers on unchanged files**.
This repo had never been green in a clean build; nobody had ever run `--no-incremental`:

```
110 error CA1707     (55 of them, underscores in test method names)
 10 error IDE0004    (redundant casts)
  6 error IDE0230    (should use UTF-8 string literals)
  2 error IDE0062    (local function can be static)
```

**CI would have been all red on its first run** — and CI had never run on a real runner (the table in §11.2.9 says as much:
"written, not yet run on a real runner"), so this debt had just been hanging there.

How it was handled:

- `CA1707` is `NoWarn`-ed in the test projects, with the reason spelled out:
  the underscores in Chinese case names **are not a naming style, they are the protocol identifiers themselves**
  (`原子覆盖式重命名走posix_rename` "atomic overwriting rename uses posix_rename", `终端尺寸变化发出window_change` "terminal size change sends window_change").
  Converting to camel case would only make the case names diverge from the names in the spec, and the one job of a case name
  is to show at a glance which rule was broken when it goes red. CA1707 protects public APIs; test methods aren't one.
- The other 9 are real problems, fixed one by one (`private static readonly` should be PascalCase per `.editorconfig`;
  local `const` should be camelCase; redundant casts removed;
  `new byte[] { x }` hoisted into a local array).

> **Don't use incremental builds as a gate.** Green doesn't mean clean —
> it only means "the few files you changed are clean".
> Run `--no-incremental` at least once before committing; that is what CI sees.

#### License changed to MIT

Changed from Apache-2.0 to MIT (SPDX headers in 136 `.cs` files, `LICENSE`,
`PackageLicenseExpression`, and the wording in README / AGENTS / NOTICE / CI).

**What is given up is the patent grant clause** — Apache-2.0 §3 has one, MIT does not.
The README used to advertise it as a selling point: "includes an explicit patent grant clause, safe for commercial use";
under MIT that sentence is false, so it has been changed, and this difference is now stated explicitly.

`NOTICE.md` is kept: MIT doesn't require it, but it records third-party attributions
(BouncyCastle, etc.), whose own licenses do require it. The wording of the corresponding CI check was changed accordingly —
it used to cite "Apache-2.0 Section 4(d)".

### 11.2.18 Closing the last two gaps: encrypted OpenSSH private keys and certificate authentication (2026-09-22)

Before VelaShell switches over, first fill in the two things it currently gets from `Tmds.Ssh` that this library didn't yet have.
This was **the user's call**: each gap would be hit by real users,
and switching with them in place would take people from "it used to work" to "what used to work doesn't anymore".

#### 1. Encrypted OpenSSH private keys — a recorded exception to "don't write our own cryptographic primitives"

§11.2.12 said this one couldn't be closed, because `bcrypt_pbkdf` needs **the internals of Blowfish's key schedule**,
which the BCL doesn't have and BouncyCastle only exposes as `Init` + `ProcessBlock`. **That assessment still holds today**
(in 2.7.0, `BCrypt.DeriveRawKey` is still `internal`, and the public `Generate` does standard bcrypt with
2^cost rounds). What changed isn't the facts, it's the trade-off:

> This format is the default output of `ssh-keygen` when a passphrase is set. In other words, "the key most people actually hold"
> is one this library couldn't read. Making the product unusable on the most common path for the sake of a rule's literal completeness isn't worth it.

So "**no exceptions**" in §3.3 of `src/VelaShell.Ssh/AGENTS.md` became "**one exception, stated here**",
with the scope of the exception nailed down:

- **Only the one file `Keys/BcryptPbkdf.cs`**, doing only this one KDF and exporting no block-cipher capability —
  Blowfish is purely an internal detail in there and cannot be reached from outside.
- **The initial tables are not copied by hand.** The P array and the 4 S-boxes, **1042 32-bit words** in total, are computed from the hex digits of π
  (Machin's formula, 33344 digits + 128 guard digits). Copy one word wrong and it compiles, the cases pass
  (as long as the test stub uses the same wrong table), and it only shows up against a real private key —
  at which point the symptom is "the passphrase is definitely right but it won't decrypt". Something computed can be checked against its definition;
  `BlowfishTableTests` additionally pins a few well-known values at the start and end (`P[0]=0x243F6A88`,
  `S3[255]=0x3AC372E6`).
- **Verification goes through real artifacts.** All samples were generated by a real `ssh-keygen` (OpenSSH 10.5p1) and committed under
  `tests/.../Keys/Fixtures/`; the assertion is "the public key we decrypt is byte-for-byte identical to the `.pub` it wrote".
  Write the encryption side yourself and decrypt your own output, and the two sides will **be wrong together** — and that kind of wrong doesn't report errors.

Supported algorithms: `aes{128,192,256}-{ctr,cbc}`, `aes{128,256}-gcm@openssh.com`,
`chacha20-poly1305@openssh.com`. No `3des-cbc` — carrying 3DES in a security library just to read a format nobody uses
isn't worth it; when encountered, it reports the error honestly and suggests `ssh-keygen -p -Z aes256-ctr`.

**A pitfall hit while working on this, worth a line of its own:**

> **The AEAD authentication tag is outside the private-key section's `string`**, as raw bytes at the end of the container,
> without a length prefix. It was first written as "ciphertext ‖ tag together in one string", and the symptom was extremely confusing:
> a stream cipher still decrypts the **prefix** correctly, so the two checkints match and the private key looks decrypted,
> only the tag never verifies. The ChaCha20-Poly1305 path got stuck on this for a long time,
> while the AES-GCM path failed on the spot — the same bug, two completely different presentations.
>
> What located it wasn't more reasoning, it was **measuring**: print each sample's `section` length and how many bytes
> remain after it, and one line — `section=176 / 16 remaining after it` — explained everything.

#### 2. OpenSSH user certificate authentication

Certificate authentication **is not a third authentication method** — it still goes through `publickey`, only the
"public key blob" field in the request carries the whole certificate. The real pitfall is an asymmetry:

| Field | Content |
| --- | --- |
| "Public key algorithm name" in the request | `ssh-ed25519-cert-v01@openssh.com` (**with** the suffix) |
| "Public key blob" in the request | The whole certificate |
| Algorithm name inside the signature blob | `ssh-ed25519` (**without** the suffix) |

Getting either one backwards produces the same `Permission denied (publickey)`,
indistinguishable from "CA not trusted", "principal mismatch" or "certificate expired".

In the implementation, this asymmetry is **entirely contained in `SshCertificateSigner`**, and not one line of the authenticator changed —
it only does "present `Signer.PublicKey.Blob`, sign with the chosen algorithm name".
`SshPublicKey` gains a constructor that "puts on the certificate identity": `KeyType` and `Blob` are the certificate's,
verification is still done by the plain key inside, and the suffix is stripped before comparing algorithm names.

`SshCertificateSigner.Create` **checks on the spot that the certificate and the private key are a pair**.
Without that check, a misconfiguration also shows up as that same `Permission denied`,
and this is a local byte comparison that costs nothing.

**This library does not verify the CA signature**, deliberately: verification is the server's job, the client verifying it doesn't change the outcome,
and the client side doesn't even have the list of "which CAs are trusted". What is handed out are the **facts** —
`SignatureKey` / `ValidBefore` / `CriticalOptions` —
so that users can display them and say something in plain language when it has expired.

#### 3. `SshConnection.Disconnected`

VelaShell's `ISshClientWrapper.Disconnected` is a `CancellationToken`;
the terminal's read loop and auto-reconnect both hang off it. Previously this library only had the `IsAlive` property,
so the upper layer could only poll — with up to one polling period of delay between the drop and the tab changing color.

It is **separate** from the internal `_lifetime`: `_lifetime` is used to stop the send/receive pumps,
while this one is the **external** signal, released at `Fault()` time, when the pumps may still be winding down.
The token is taken once at construction and kept, not read from the CTS each time —
that property throws after the source is disposed, and "the connection has already been disposed" is exactly when it gets read most.

#### Verification

All 517 unit cases green (22 new). Interop ran against real OpenSSH 10.3,
and the three new ones all passed with none skipped: an encrypted private key can log in, a wrong passphrase is reported locally,
certificate authentication can log in (the server is configured with `TrustedUserCAKeys`,
installed by `scripts/ssh/interop/trust-ca.sh`).

### 11.2.19 Fixes after a read-through review (2026-09-22)

A read-through review of the whole library was done, focusing on "is it really async", "are compression and X11 complete and reliable", and "do proxy settings take effect".
Conclusions and fixes are recorded together here.

#### Async: no fake async of `Task.Run` wrapping sync code, but a few real problems

`Task.Run` is only used in the library to start async loops (it just moves the loop's synchronous beginning onto the thread pool; the loop itself truly `await`s IO),
not to "wrap synchronous blocking in a Task". The real problems were these:

| Problem | Fix |
| --- | --- |
| Each keep-alive probe had **no deadline** — on a half-open connection the first probe waits forever, making the dead-detection logic a dead letter | Each probe has `KeepAliveInterval` as its deadline; timed-out probes stay in the ledger, so late replies still line up |
| On protocol violation, the session was marked dead first and then `DISCONNECT` sent, but the send path's first step rejects a faulted session — `DISCONNECT` never went on the wire | Send first (waiting at most 2 seconds), then mark dead; marking dead actually stops the send/receive pumps and closes all channels |
| The send path was "one lock + one Flush per frame", not matching §5.4; the receive loop could get stuck on TCP backpressure | Implemented the send pump from §5.4 (see above); the receive loop only posts and never waits |
| When the staging area is full during renegotiation, the receive loop replying with a CLOSE would wait on the gate, and the gate only opens after the receive loop completes KEX — a self-deadlock | Backpressure moved to enqueueing and distinguishes senders; the receive loop and KEX are exempt from backpressure |
| The sync overloads of `SftpFileStream` were sync-over-async, violating principle 1 | Sync read/write throws `NotSupportedException`; sync `Flush` is a no-op; sync `Dispose` does not block (`spec/06` §6.4) |
| Host key verdict time was counted against the connect timeout (feedback from the host side) | The connect timer pauses during the verdict, and persistence honors only the caller's token; the jump hop also pauses the outer timer |

A few other session-layer correctness problems: the receive buffer was reused on the next packet read, while the "peer channel request" event and incoming channel handlers
were both holding onto it — now copied; a race between `OnData` checking state and writing to the pipe could let one channel's close kill the whole session — now under the same lock;
when a consumer `Complete`s early, the remaining bytes were never credited back to the window — now they are; the post-authentication packet limit was "said to be relaxed but wasn't" — now relaxed and validated when opening channels;
the reason code and original text of `DISCONNECT` were dropped — now surfaced; the sequence number in `UNIMPLEMENTED` was hard-coded to 0 — now the rejected packet's sequence number.

#### Along the way: a hang that only showed up in the host's integration tests

After the host (VelaShell) switched to this library, half of its Docker shell integration tests randomly hung on the 15-second timeout:
after an exec probe opened and closed once on the same connection, the interactive shell never echoed again. Diagnostic instrumentation (thread IDs + every message)
showed the outgoing data had long since been flushed to the wire, and the server's echo had already reached the socket — it was **the receive loop itself stopping for 15 seconds**:
while handling the probe channel's `CHANNEL_CLOSE` it called `CancellationTokenSource.Cancel()`,
and `Cancel()` runs callbacks synchronously on the current thread, so a chain of synchronously completing continuations (channel pump ends → the probe's DisposeAsync resumes
→ RunAsync returns → the test method) started running on the receive loop's thread; the test then polled with `Thread.Sleep`.

This bug existed before this round (the same `Cancel()`); it's just that Tmds didn't have the same path.
The fix is a rule: **never run user code on the receive loop or the send pump** (`spec/05` §8),
and cancellation always uses `CancelAsync()`. The regression case `调用方在await之后同步阻塞时接收循环不受影响` ("receive loop unaffected when caller blocks synchronously after await") failed 3/3 before the fix;
after the fix the host's integration suite is 32/32, with timing on par with the Tmds baseline (19 s / 18 s).

#### Compression

The core implementation (persistent per-direction streams, sync flush, bomb limit, reset on renegotiation, delayed activation for `zlib@openssh.com`) is correct.
One semantic hole: plain `zlib` was also deferred until after authentication, whereas it should take effect from the first NEWKEYS — now installed on the spot at NEWKEYS.
`Compression yes` in `ssh_config` now actually takes effect via the mapping.

#### X11

The security core (fake cookie, constant-time comparison, hex cookie field, both byte orders, fragmentation, expiry) is correct. Fixes:

- `xauth generate` in untrusted mode had no `-f`, and would **overwrite the fully authorized cookie in the user's `.Xauthority`** — now writes to a temporary file;
- Interactive shells couldn't open X11 at all (the most common use of `ssh -X`) — `SshShellOptions.X11` / `AgentForwarding` / `BeforeStart`;
- The handler was a per-connection singleton, so multiple sessions pushed each other out — added `X11ChannelRouter`, which dispatches by cookie;
- Concurrency slots of incoming handlers leaked when the local side rejected a channel (same for agent / remote forwarding) — `IIncomingChannelHandler.OnOpenAborted`;
  multiple handlers of the same type (two `-R`s pushing each other out) — `Add/RemoveIncomingChannelHandler`;
- `xauth` timeouts didn't kill the process, display names lost the host and socket path, single-connection mode wasn't enforced locally — all fixed.

#### Proxies: previously there were **none at all**

`Socks5` / `HttpConnect` / `SshJump` in `SshDialKind`, the examples in §5.1 and §6.2,
`ProxyRefused` / `ProxyAuthRequired`, `SshConnectException.Hops` — all empty shells;
`ProxyJump` in `ssh_config` was only parsed, never consumed; `ProxyCommand` wasn't even parsed;
and the whole `SshConfigFile` had no mapping to connection parameters.

Following the clean-room procedure, **the spec was written first** (`spec/09-dialing.md`, based on RFC 1928 / 1929 / 9110 / 7617 / 4254 §7.2
and the behavior described in `ssh_config(5)`), then implemented per the spec: `Socks5Dialer`, `HttpConnectDialer`, `SshJumpDialer`,
`ProxyCommandDialer`, `DialerChain`, channel-as-stream `SshChannelStream` (`SshChannel.AsStream()`),
per-hop failure information, `SshConfigFile.CreateConnectionOptionsAsync` and `SshHostConfig.ApplyToShell`.

〔Stated plainly〕On Windows `ProxyCommandDialer` uses anonymous pipes, and anonymous pipes don't support overlapped IO —
its async reads and writes are completed by the runtime blocking on thread-pool threads. This is a platform limitation; when a purely async path is needed, use the other three dialers.

#### Verification

568 unit cases (549 passed, 19 interop skipped when there is no sshd); interop against real OpenSSH 10.3 is 19/19 (including the new "jump through a real server"). Coverage added in this round:
keep-alive marks dead when the peer is silent, late keep-alive replies don't misalign, DISCONNECT really goes on the wire on violation, DISCONNECT reason code and original text,
UNIMPLEMENTED sequence number and no echo, payloads not overwritten by subsequent packets, early completion still credits the window back, receiving 128 KiB packets after authentication,
multiple handlers of the same type, `OnOpenAborted`, plain zlib, multi-session X11 dispatch, local enforcement of single-connection, shell X11 timing,
xauth always has `-f`, timer paused during verdict, SFTP sync API, SOCKS5 (including auth, rejection, IDN), HTTP CONNECT
(including 407, the hint for proxies that only allow 80/443, **the reply and the SSH identification string arriving in the same write**), per-hop info for nested proxies, jump hosts,
the proxy command's stderr, `ssh_config` mapping (jump chains, cycle detection, host key policy, IdentityFile, session options).

### 11.2.20 A whole-library review after merging into the host, and the fixes (2026-09-24)

The library was read through subsystem by subsystem (framing and cipher suites, KEX and rekeying, session core, channels, authentication and private keys, SFTP, forwarding, dialing and configuration),
and the review kept to the same clean-room discipline (the only references were RFCs, OpenSSH `PROTOCOL*`, and this repository's specs). Issues that directly hurt users were fixed first,
then the DoS limits and the window budget; the remaining findings are left for later. Every fix has a test that was confirmed to fail with the fix removed and to pass with it.

#### Issues that directly hurt users

| Problem | Root cause | Fix |
| --- | --- | --- |
| A single certificate, FIDO or DSA key in the agent breaks agent authentication, agent forwarding and auto-add-to-agent | Listing identities caught `SshWireFormatException`, while `SshPublicKey.Parse` throws `SshPublicKeyException` | Catch the right exception; skip identities we don't recognize |
| An agent refusing to sign aborts the whole credential chain; a dropped connection is instead recorded as a "skip", and the next credential reads the previous one's reply | The chain decided "credential problem" by exception **type** | Only exceptions thrown by the credential's callbacks and the signer count (`spec/04` §3.4) |
| Callers writing stdin hang forever when the channel closes | The stdin pump `return`ed from the middle of its loop, skipping reader completion; completing the writer does not release a pending `FlushAsync` | Every exit path of the pump completes the reader |
| Channel opens in progress hang forever when the connection is disposed | Only `Fault` settles `_pendingOpens`, and disposal does not go through `Fault` | Disposal settles them too |
| Cancelling a channel open leaks the channel on the server (using up `MaxSessions`) | The late confirmation after a cancel was dropped | A sent open stays in the ledger; when the confirmation arrives the channel is closed |
| CLOSE could be skipped for good; the channel number was reclaimed before the peer's CLOSE; requests / EOF / data could still be sent after CLOSE | "Sent" was set before sending; the number was returned as soon as the local side finished; sends didn't check state | "Sent" is set under the enqueue lock at the moment of enqueueing; the number is held until both CLOSEs; nothing is sent after CLOSE (`spec/05` §1 rules 2 and 6) |
| Disposing a channel stream drops the tail and sends no EOF; `FlushAsync` is a no-op | The channel was closed directly | Flush stdin and send EOF first, then close; `FlushAsync` waits until the data is handed to the session (`spec/09` §5.1) |
| After a few cancellations every SFTP operation hangs | Requests never sent stayed in the ledger and never returned their in-flight slot; waiters were not woken on failure | Unsent requests are removed and give their slot back immediately; bytes are committed whole-frame and cancellation only interrupts backpressure; waiters are tied to the pipeline's lifetime |
| Under strict KEX the connection drops after a rekey (chacha20 / HMAC suites) | Strict KEX was recomputed from each peer KEXINIT, so when the rekey KEXINIT omitted the marker the sequence numbers were not reset; the test server had the same bug | Fixed by the initial exchange and kept for the whole connection; rule (a) applies only to the initial exchange (`spec/03` §6) |
| SFTP download throughput pinned at "block size ÷ RTT" | Sequential reads sent one `READ` at a time | Read-ahead for sequential reads (`spec/06` §5.5, spec written first) |

#### DoS limits and the window budget

| Problem | Fix |
| --- | --- |
| The session window budget meant nothing: refunded twice on refusal, growth never charged, refunded at the grown size on close | Refund exactly what was charged; growth requests budget first, shrinking gives it back (`spec/05` §3.3) |
| When the peer sends but never reads, the receive loop's replies queue without bound | Hard limit `MaxQueuedReplyBytes` (16 MiB by default), exceeding it is a protocol violation; replies count toward backpressure |
| A peer flooding unknown channel requests grows the event stream without bound (up to 256 KiB each) | At most 64 unread unknown requests are kept |
| Refusing a channel open echoed a 64 KiB type name verbatim | Description truncated to 256 characters |
| Rekeying had no timeout: the connection silently stopped when the peer never answered KEXINIT or stalled midway | 2 minutes by default; timing out disconnects with `Timeout / Rekeying` (`spec/03` §8.2) |
| Starting a rekey while one was in progress sent a second KEXINIT (the "packet-count threshold" test failed about 1 in 4 full runs because of it — **this predates this round**) | "In progress" lasts until the gate reopens; closing the gate and KEXINIT are enqueued under the same lock |

Also: `ReadAllBytesAsync` used to preallocate its initial capacity from the length the server reported (a server claiming 2 GiB got 2 GiB allocated up front); it is now capped at 1 MiB.

#### Verification

626 unit tests (606 pass, 20 interop tests skip without an sshd), 10 consecutive full runs green; the host's `VelaShell.Infrastructure.Tests` 536 pass.
No interop run against real OpenSSH this round — how the strict-KEX fix plays out in practice depends on whether the server includes the marker in its rekey KEXINIT,
so an interop test "chacha20 across a rekey" is worth adding the next time the target server is up.

### 11.2.21 Second batch: the remaining security findings (2026-09-24)

| Problem | Fix |
| --- | --- |
| Rekeying re-ran the whole host key policy (an interactive policy prompted mid-session while the receive loop waited; a lenient one let a swapped key through) | Pin the host key from the initial exchange and stop consulting the policy; if `K_S` changes, disconnect with `HostKeyChanged / Rekeying`; rekeying only negotiates algorithms of the pinned key's type (`spec/03` §8.4 already said so) |
| After disposing the agent forwarder, agent channels already open kept signing for the remote | The forwarder has its own lifetime; disposing it closes the open channels too (`spec/07` §7.2) |
| `DISPLAY=localhost:N` tried the Linux abstract socket first, so another local user binding it first received the real cookie | "Where to connect" and "which cookie" are separated: `localhost:N` is TCP only, cookie selection still treats it as local (`spec/07` §7.5.6 corrected too) |
| Private key KDF parameters were unbounded (Argon2 4 GiB memory, one million bcrypt rounds) and used before the MAC is checked | bcrypt capped at 4096 rounds; Argon2 at 256 MiB, 256 passes, parallelism 1–16, with a cap on memory × passes; also pick the Argon2 variant named by `Key-Derivation` |
| On Windows, connecting to a named-pipe agent didn't check who was on the other end (another user squatting `openssh-ssh-agent` would receive signing requests and, when adding keys, plaintext private keys) | Check the pipe owner: current user / SYSTEM / Administrators; the impersonation level is not lowered — the OpenSSH agent service stores keys as the connecting user (`spec/07` §7.2) |
| Peer text went verbatim into exception messages (terminal escape injection) | `PeerText.Sanitize`: control characters, `DEL`, C1 and bidi controls become `?`, and the text is truncated; the verbatim text stays in `PeerDescription` / `ServerMessage` (`spec/08` §1) |
| `KnownHostsPolicy` treated an unrecorded key type as "never seen"; `!pattern` was ignored; appends didn't add a missing newline; `ProxyCommand` could be injected | `OtherKeyTypesKnown` handled like "changed", and `IHostKeyTypePreference` makes connections prefer recorded types (`spec/03` §5.4); a negation vetoes the whole line; appends add the newline first; values are checked before substitution (`spec/09` §6) |

Verification: 655 unit tests (635 pass, 20 interop tests skipped), 25 consecutive full runs green; of the 29 new tests, 19 were confirmed to fail with their fix removed.
Two things could not be verified on this machine and are recorded as such: the X11 `localhost` difference only shows on Linux/macOS; the named-pipe owner check was only tested with "an agent run by the same user still connects" and the decision function —
neither the real OpenSSH agent pipe owned by SYSTEM nor "another user squatting the name" was exercised.

### 11.2.22 Third batch: correctness, performance and design, plus host certificates (2026-09-25)

The remaining sections 3–5 of the review report (correctness, performance, design) were fixed in this round. The same day the user set the scope for algorithm support —
"compatible with the mainstream algorithms is enough; leave the too-old, insecure ones aside" — and on that basis host certificates were added and legacy encrypted PEM was explicitly placed on the unsupported side.
All behavior is written into the specs; the section noted in each item is where.

#### Correctness: a failure must look like a failure

| Problem | Decision |
| --- | --- |
| The forwarding relay treated errors as EOF: when one direction failed reading, it still sent FIN / `CHANNEL_EOF` to the other side, so truncated data was accepted as complete; the other direction hung | A normal end of reading half-closes only that direction; an error in either direction (including cancellation) **aborts** both ends — TCP gets a linger-0 RST, the channel a `CLOSE` without a preceding EOF; the side that failed first is reported, not the cancellation it caused on the other side (`spec/07` §2.2, §6) |
| When the connection died mid-way, channel readers saw an EOF-like "done" | The read throws the connection's failure; `ObjectDisposedException` when the connection was disposed locally; only the peer's EOF / `CLOSE` counts as done (`spec/05` §4.4) |
| After a channel closed entirely (`CLOSE`), the relay direction writing to it stayed stuck reading the local socket, holding the socket and a forwarding slot | An `SshChannel.Closed` token; the relay stops that direction on it (`spec/05` §4.4) |
| Connection failures leaked to callers as-is: internal parse exception types, raw `IOException` / `SocketException`, both during a session and during setup; a mid-packet close during key exchange was reported as `ProtocolError`; `SshPublicKeyException` and `SshKeyExchangeException` were not under `SshException` | Normalized to public types before being handed out, with the same rules during setup (from a successful dial until authentication finishes) and `Phase` recording the failing step (`spec/08` §2.1); both exceptions now derive from `SshException` (`spec/08` §2). **Known limitation, not changed**: a session fault normalized this way always has `Phase` `Open`, even when it happened during a rekey |

These share one judgement: **"the peer is done" and "the link broke" must look different to the reader.** Treat them alike,
and a half-transferred file or half-run command output is handed out as a complete result, and a terminal cannot tell whether the user typed `exit` or the link broke (only the latter should trigger an automatic reconnect).

#### Performance: control messages must not queue behind data

| Problem | Decision |
| --- | --- |
| During a heavy upload, another channel's `WINDOW_ADJUST` queued behind the outbound backlog (and first waited on backpressure for room): the peer kept waiting for window, and downloads were dragged almost to a stop by the upload | Our window top-ups go through a priority lane that the send pump checks before every item, exempt from backpressure; the lane only moves a top-up earlier, never past `CLOSE` or the rekey gate (`spec/05` §3.2) |
| Keep-alive probes waited on backpressure, or started their clock only once on the wire: a dead link typically looks exactly like the send pump stuck on one write, so the probe got stuck too, and the connection could not be declared dead precisely when it mattered most | Probes bypass backpressure, and their deadline starts when they are enqueued; at most one frame per interval, so this is not unbounded (`spec/05` §6.3) |
| The adaptive window looked only at "ran low": a slow consumer pushed the window all the way to the maximum, piling up tens of MiB of unread data locally | Grow only when it ran low **and** the reader was recently starved; shrink after 3 rounds without running low (`spec/05` §3.3, §5.5 here) |
| On a peer channel open, the receive loop awaited the user's handler in place (it may show a prompt or look up configuration): a slow handler stalled receiving for every channel on the connection | Parsing and handler lookup finish on the receive loop; "ask the handler, create the channel, confirm" runs in the background; at most 64 are being decided at once, and the excess is refused with `RESOURCE_SHORTAGE` (`spec/05` §8.1) |

#### Performance: allocations on the hot path

- **Channel data buffers are rented from a pool**: every block on the upload path used to allocate a new array. When the sender's `await` returns, the send pump no longer references the buffer, so it can go straight back to the pool;
  **it is copied only when the rekey gate stashes it** — the stash holds on to it until the gate reopens, by which time the sender has long returned the buffer. The frame sent directly needs no copy: encryption has already copied it out.
- **SFTP request frames are written directly into the channel's pipe**, with no intermediate buffer. They used to be assembled in a buffer that started at 256 bytes and doubled, so a 256 KiB `WRITE` was reallocated a dozen times and its data moved one extra time.
- **Cipher suite objects live with the suite, not per packet**: the HMAC context is reused; ChaCha20-Poly1305's two engines and its Poly1305 stay resident, with only the nonce changed per packet;
  CTR XORs in vector-width batches and advances the counter by the block count at once rather than one block at a time. A suite serves one direction on one thread only, so reuse is safe.
  It brings one new risk: after a rekey the new set must start fresh from the new keys, and any leftover state makes the first packet under the new keys undecryptable —
  yet when we encrypt and decrypt ourselves both sides are wrong in exactly the same way, and only another implementation shows it. Hence the interop test that lands a rekey **in the middle** of a large output (see verification below).

Measured (Release, 32 KiB payload):

| Suite | Throughput | Allocation per packet |
| --- | --- | --- |
| AES-CTR + HMAC | ~800 → ~1100–1200 MB/s | 288 B → 0 |
| ChaCha20-Poly1305 | ~430–480 MB/s (bounded by BouncyCastle) | ~1.4 KB → 272 B |
| AES-GCM | Unchanged: seal ~6.2–6.9 GB/s, open ~5 GB/s | 0 (already 0) |

#### The link simulator was the bottleneck

`DelayedStream` (`Transport/LinkCharacteristics.cs`) used to sleep the whole one-way latency inside its write lock on every write before handing the data down:
writes were serialized by the latency, at most one write was in flight on the link at any moment, and throughput was capped at "size of one write ÷ one-way latency" —
**part of what the window and pipeline-depth tests measured was the simulator itself**.

It is now pipelined, like a real link: a write returns as soon as bandwidth throttling allows, and the data becomes readable at the other end after the one-way latency;
data in flight sits in a bounded queue that a background task hands down in order when due — when the lower stream cannot keep up the queue fills and writers wait, so backpressure does not disappear with pipelining.
`DisposeAsync` lets the data in flight arrive before closing the lower stream (the FIN goes after the bytes already sent; otherwise the final `DISCONNECT` never reaches the other end),
and if the lower stream is stuck it waits one one-way latency plus 1 second before treating the link as cut; synchronous `Dispose` does not wait, which amounts to cutting the link.

> The test harness is part of the system under test: if its model is wrong, what you measure is the harness.

#### Dialing and `ssh_config`

| Problem | Decision |
| --- | --- |
| Multiple addresses were tried one by one: on networks that advertise IPv6 but don't route it, the first address waited out a ~21-second SYN timeout before IPv4 got a turn | Happy Eyeballs (RFC 8305): interleave address families, stagger attempts 250 ms apart, the first to connect wins, the rest are closed (`spec/09` §2.5) |
| The TCP step was fixed at 30 seconds, and a longer timeout configured on the connection had no effect on it | Dialers default to no limit; the connect timeout governs everything (`spec/09` §2.4) |
| Authentication on a jump host counted against the outer connect timeout; an outer expiry was reported as "TCP connect timed out"; caller cancellation and expiry could not be told apart | The outer timer is paused during jump host authentication; expiry is reported as `Timeout / Dialing` naming the hop; cancellation propagates as-is (`spec/09` §2.4) |
| `Include` contents were appended after the whole file: a later `Host *` in the main file beat the host-specific settings in the included file; the same file included from two blocks was skipped the second time as a "cycle". After switching to in-place expansion, the enclosing condition was lost as soon as the included file opened a block, and settings after the `Include` fell into the included file's last block | Expanded in place as a conditional include: the included file's own blocks are scoped under the enclosing condition (`SshConfigBlock.Enclosing`; every level must hold), and parsing returns to the block containing the `Include` afterwards; cycle detection looks only at the current include chain (`spec/09` §7.1) |
| A `Match` condition that could not be evaluated counted as "not satisfied", so a leading `!` made it satisfied; `Match host` compared against the typed alias | Tri-state: cannot be evaluated means the whole block does not apply, negated or not; `Match host` compares against the name after `HostName` rewriting (`spec/09` §7.1) |
| One unreadable `IdentityFile` failed the whole resolution; jump hosts and the target each read the same key (two KDF runs, two passphrase prompts) | Skip only that key and notify the caller; share by path within one resolution, no caching across calls (`spec/09` §7) |
| The password the caller meant for the target was also handed to jump hosts | Password and keyboard-interactive go only to the final target; jump hosts get only public-key credentials (`spec/09` §7) |
| With `ask` / absent, merely setting `UserKnownHostsFile` in the configuration discarded the caller's policy; `none` / `/dev/null` were treated as file paths | The caller's policy wins; `none` / `/dev/null` mean no `known_hosts` (`spec/09` §7) |

#### Host certificates, and the private key format we do not read

Under the "mainstream algorithms, nothing too old" scope:

- **Host certificates are in** (`spec/03` §5.5). Certificate algorithms come **after** the plain ones: with no CA configured for the host, negotiating a certificate adds no assurance, and ordering them last keeps behavior exactly as before.
  The certificate is validated only when `known_hosts` has a matching `@cert-authority` line; what gets recorded is the **plain key** inside the certificate (the blob changes on every re-signing, so recording the certificate would guarantee a "changed" next time);
  a CA-managed host presenting a key the CA does not vouch for is rejected **without falling back to TOFU** — configuring a CA for a host is precisely about no longer relying on blind first-use trust; SHA-1 `ssh-rsa` CA signatures are refused.
  A CA-vouched but invalid certificate, or a certificate algorithm negotiated while `K_S` is not a certificate, is reported as `HostKeyRejected` (`spec/08` §3).
- **Rekeying negotiates only host key algorithms of the pinned key's type and of the same certificate-ness** (`SshConnection.RestrictToPinnedHostKey`).
  The check used to strip the certificate suffix: with a pinned certificate the plain algorithm stayed in the list, and once it was negotiated the server presented its plain key, which did not match the pinned certificate,
  so the connection was dropped as "host key changed" (`spec/03` §5.5).
- **Legacy encrypted PEM (`Proc-Type: 4,ENCRYPTED`) is not read**; only modern formats are supported (OpenSSH, PuTTY `.ppk`, PKCS#8, and unencrypted PKCS#1 / SEC1).
  The passphrase goes through a single MD5 to become the key, often paired with 3DES; carrying a decryptor for it in a security library is not worth it, and conversion is one command.
  It is refused **before** asking for a passphrase, with conversion guidance (`ssh-keygen -p` to change the passphrase once rewrites the file in OpenSSH format); it used to go to the BCL, which does not understand it, and the result read "the passphrase is probably wrong" (`spec/04` §4.6).

#### Verification

The full suite is 761 tests, 22 of them interop tests against real OpenSSH (`linuxserver/openssh-server`), run this round with the target server up, none skipped.
The two new interop tests cover what previously could only be checked against the in-memory stub: every encryption algorithm in the default list (chacha20-poly1305, AES-GCM, AES-CTR)
keeps working across a rekey that lands **in the middle** of a large output — the to-do left by §11.2.20, and the real acceptance test for the cipher object reuse above;
and a real `ssh-keygen`-signed host certificate validated against a `known_hosts` `@cert-authority` line.

### 11.3 Switch-over strategy with VelaShell

1. VelaShell's `ISshClientWrapper` / `ISftpClientWrapper` / `IShellStreamWrapper`
   **are already library-neutral abstractions** — the most valuable asset left over from the original migration off SSH.NET.
   The blast radius of the switch is the single directory `Infrastructure/Ssh/`.
2. First let `VelaShell.SshClientWrapper` and `TmdsSshClientWrapper` **coexist**, switched by a feature flag.
   Both implementations run the same set of tests and are aligned item by item.
3. Once aligned, delete the Tmds implementation and the patches in the table in §9, and remove the flag along with them.

---

## 12 Risks and what we will not do

### Risks

| Risk | Mitigation |
| --- | --- |
| **Interop long tail** — SSH's pitfalls are all on old devices | The §10.3 matrix runs from M1 onward, not left to the end; old-device algorithm priorities are a separate group |
| **Cryptographic implementation errors** | Principle 6: only assemble, don't build primitives (ChaCha20 / Poly1305 come from BouncyCastle too); the only hand-written one is `bcrypt_pbkdf` (§11.2.18), which does only that one KDF and is verified only end to end against real `ssh-keygen` output; external audit before 1.0 |
| **Schedule overrun** | Milestones are cut by capability, and every M is a usable state; VelaShell keeps using Tmds until M6, so we can stop at any time |
| **Staffing** — 24 weeks of work | Understand it clearly before starting. **The cost of giving up halfway is a half-built SSH stack**, which is far worse than today's 1,200 lines of patches |
| "Still looks similar" | The clean-room procedure from §2.2 is followed from the very first commit of M0 — not added at the end, by which point it's too late |

### Explicitly not doing

- **An SSH server** (except test stubs). A completely different direction, out of scope.
- **SSH1**. Dead, and insecure.
- **Implementing AES / SHA / RSA / ECDH ourselves**. The BCL has them, with hardware acceleration.
- **A synchronous API**. Principle 1.
- **The other end of X11 forwarding** (the X server) is not part of this library. The forwarding channel itself **is already done** (§11.2.14); the X server is a separate library in the host repository, `VelaShell.XServer`, and the two meet through `X11ForwardOptions.LocalConnector` (§11.2.14 addendum).
- **Mosh**. A different protocol stack, consistent with VelaShell's existing decision.

---

## 13 Decisions pending sign-off

| # | Decision | Recommendation |
| :-: | --- | --- |
| 1 | **Do it or not** (24 weeks) | **Decided**: do M0 first (spec + skeleton), and decide at the M0 exit whether to continue with M1–M7 |
| 2 | Library name and namespace | **Decided**: repository `velashell-ssh`, package and namespace `VelaShell.Ssh`. Renaming remains zero-cost until the end of M0 |
| 3 | License | **Decided**: open source under a single MIT license (changed from Apache-2.0 on 2026-09-22). MIT **does not include** a patent grant clause — that is the one thing given up relative to Apache-2.0; VelaShell uses it closed-source as usual |
| 4 | Whether to build a compatibility layer | Yes, only the surface VelaShell uses, marked `[Obsolete]` from day one (§6.3) |
| 5 | TFM and language version | **Decided**: single target `net11.0` + `LangVersion=preview`. Gives up potential net10 users in exchange for zero polyfills and the latest BCL |
| 6 | Whether to open-source it | **Decided**: open source |
| 7 | How detailed the M0 spec should be | Recommended down to the level of "a field table + sequence diagram for every message". **The isolation in §2.2 item 2 rests on it** — if the spec isn't detailed, during implementation you'll be tempted to go back and read the source |

---

*Items 1 / 2 / 3 / 5 / 6 of §13 were signed off on 2026-09-21 (see the notes in the table); 4 and 7 will be decided before the M0 exit.*
