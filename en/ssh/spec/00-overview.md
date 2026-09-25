# 00 · Behavior Specification: General Provisions

> **This document set is the sole basis for the VelaShell.Ssh implementation.**
>
> While writing the implementation, there should be only two things on your desk: the specifications in this directory, and the RFCs they reference.
> **Do not open the source code of any other SSH implementation** —— this is [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) §2, Discipline 2,
> and it is the entire substance of this project's founding premise of being "a provably independent implementation".
>
> Where the specification is unclear, go back to the RFC and **fill the gap in the specification**, rather than looking elsewhere for a ready-made answer.
> 中文：[`../../../zh/ssh/spec/00-overview.md`](../../../zh/ssh/spec/00-overview.md)

---

## 1 How to read this document set

| File | Content | Corresponding implementation layer |
| --- | --- | --- |
| `00-overview.md` | This document: terminology, notation, general conventions | — |
| [`01-transport-framing.md`](01-transport-framing.md) | Binary packet protocol, framing, padding, encryption shapes | L2 framing layer / L3 cipher suites |
| [`02-version-exchange.md`](02-version-exchange.md) | Version identification string exchange | L4 state machine |
| [`03-key-exchange.md`](03-key-exchange.md) | Algorithm negotiation, key exchange, exchange hash, key derivation, rekeying, strict KEX | L3 / L4 |
| [`04-authentication.md`](04-authentication.md) | Service request, authentication methods, partial success, extension negotiation | L7 authentication |
| [`05-connection.md`](05-connection.md) | Channels, flow-control windows, channel requests, exec / shell / pty | L5 channels |
| [`06-sftp.md`](06-sftp.md) | SFTP v3 wire format and extensions | L7 SFTP |
| [`07-forwarding.md`](07-forwarding.md) | Local / remote / dynamic forwarding, agent forwarding | L8 forwarding |
| [`08-failures.md`](08-failures.md) | Failure taxonomy, observability contract | L9 |

## 2 The three kinds of statements in the specification

The keywords of RFC 2119 / RFC 8174 are used. The Chinese original always renders them with the following fixed
translations; this English version uses the RFC keywords directly:

| English | Chinese rendering | Meaning |
| --- | --- | --- |
| MUST / REQUIRED / SHALL | **必须** | Not doing so is a protocol violation |
| MUST NOT / SHALL NOT | **禁止** | Same as above |
| SHOULD / RECOMMENDED | **应当** | May be deviated from only with good reason; the deviation must state its rationale |
| SHOULD NOT | **不应** | Same as above |
| MAY / OPTIONAL | **可以** | Left to the implementation |

In addition there are two annotations specific to this project:

- **〔Decision〕** —— behavior the RFC leaves open and that we decide. **Every one of them must give its rationale**,
  because people in the future will ask "why is it this way", and the answer should not exist only in someone's memory.
- **〔Interop〕** —— the actual behavior of some servers deviates from the RFC and we must accommodate it.
  It must state **which server, which version, and how it was verified**. A vague "some servers reportedly do this" without a concrete subject does not count.

## 3 Data types (RFC 4251 §5)

The SSH wire format has only seven types. All are **big-endian**.

| Type | Encoding | Notes |
| --- | --- | --- |
| `byte` | 1 byte | |
| `boolean` | 1 byte | 0 is false, **any non-zero value is true** (a sender MUST send 1) |
| `uint32` | 4 bytes, big-endian | |
| `uint64` | 8 bytes, big-endian | |
| `string` | `uint32` length + that many raw bytes | **It is a byte string, not text**; may contain `\0`; the length does not include its own 4 bytes |
| `mpint` | Two's-complement big integer encoded as a `string` | See below |
| `name-list` | Comma-separated US-ASCII names encoded as a `string` | See below |

### 3.1 The two pitfalls of `mpint`

1. **When the most significant bit of a positive number is 1, a `0x00` byte MUST be prepended.** Otherwise it would be read as negative under two's complement.
2. **Zero is encoded as a string of length 0** (i.e. the four bytes `00 00 00 00`), not as a single `0x00` byte.

Both rules must be pinned down by unit tests in the implementation —— they are the two most classic off-by-one errors in SSH implementations.

### 3.2 Constraints on `name-list`

- Each name: non-empty, only printable US-ASCII, **no commas**.
- Names are case-sensitive.
- An empty list is encoded as a string of length 0.
- A name containing `@` is a vendor extension (`name@domain`), where `domain` is a domain name owned by the definer.
  **A name without `@` MUST be IANA-registered** —— we do not invent algorithm names without a domain suffix.

### 3.3 Treating `string` as text

At the protocol level a `string` is a byte string. When it needs to be used as text (user names, error messages, banners),
it is **decoded as UTF-8**, as recommended by RFC 4251 §6.

〔Decision〕**On a decoding failure, do not throw; use replacement characters instead.**
Rationale: fields that fail to decode are usually display-only content such as banners, error messages and file names;
killing an entire connection over one garbled byte is pure loss for the user.
But **user names and algorithm names are exceptions** —— they take part in protocol decisions and signature inputs, so a decoding failure MUST be treated as a protocol error.

## 4 How the message tables in this document are written

Every message is described in a uniform three-part form:

**① Field table** —— in wire order, giving the type and meaning of each field:

| # | Type | Field | Description |
| :-: | --- | --- | --- |
| 1 | `byte` | `SSH_MSG_XXX` | Message number |
| 2 | `uint32` | `recipient channel` | Channel number allocated by the peer |

**② Sequence** —— a mermaid sequence diagram making clear who sends first, who must reply, and what may happen concurrently.

**③ Boundaries and errors** —— value ranges of fields, upper limits, and how illegal values are handled.
**This is the part of the specification most easily skipped, and the one most likely to cause incidents in the implementation**; it must not be left empty.

## 5 General security baseline

The following rules apply to all message processing without exception and are not repeated in the individual files:

1. **Every length field MUST be validated before use.** The length of a `string` MUST fall within
   "the number of bytes remaining to be read", and MUST NOT exceed the upper limit this specification sets for that field.
   For fields where the specification gives no limit, the implementation MUST define one itself and write it into the specification ——
   **"this field will never be that large" is not a valid argument**; messages come from an untrusted peer.
2. **Memory MUST NOT be preallocated based on a length field.** First confirm the data is actually readable, then allocate.
3. **Every parse failure disconnects**; there is no "skip this field and keep reading" tolerance.
   SSH messages are fixed concatenations; after a single misalignment, nothing that follows can be trusted.
4. **Keys, signatures and MACs MUST be compared with a timing-safe comparison** (`CryptographicOperations.FixedTimeEquals`).
5. **Key material MUST NOT be written to logs**, including at DEBUG level.
   The packet tap (`IPacketTap`) exposes only metadata by default; payloads must be enabled explicitly, with the risk documented.

## 6 Algorithms we support (master table)

Details of each algorithm are in [`03-key-exchange.md`](03-key-exchange.md); this section only gives the overall picture and the trade-offs.

### 6.1 Key exchange

| Algorithm | Basis | Enabled by default | Notes |
| --- | --- | :-: | --- |
| `mlkem768x25519-sha256` | draft-kampanakis-curdle-ssh-pq-ke | ✅ highest priority | Post-quantum hybrid; the BCL has ML-KEM |
| `sntrup761x25519-sha512` | OpenSSH `PROTOCOL` | ✅ | Post-quantum hybrid; default since OpenSSH 8.5+ |
| `sntrup761x25519-sha512@openssh.com` | Same as above | ✅ | Old name of the same algorithm, for compatibility with OpenSSH < 9.9 |
| `curve25519-sha256` | RFC 8731 | ✅ | |
| `curve25519-sha256@libssh.org` | Same as above | ✅ | Old name of the same algorithm |
| `ecdh-sha2-nistp256/384/521` | RFC 5656 | ✅ | |
| `diffie-hellman-group14-sha256` | RFC 8268 | ✅ | |
| `diffie-hellman-group16-sha512` | RFC 8268 | ✅ | |
| `diffie-hellman-group-exchange-sha256` | RFC 4419 | ❌ not implemented yet | 〔Interop〕old devices often offer only this. Specified in 03 §3.5; until it is implemented, putting it in the list is rejected before connecting (03 §2.2) |
| `diffie-hellman-group14-sha1` | RFC 4253 | ❌ off by default | 〔Interop〕Cisco IOS / old VRP have only this. **MUST be explicitly enabled by the user** (§6.6) |
| `ext-info-c` / `kex-strict-c-v00@openssh.com` | RFC 8308 / OpenSSH | ✅ | Not real algorithms but **indicators**; see 03 |

### 6.2 Host keys

| Algorithm | Basis | Default |
| --- | --- | :-: |
| `ssh-ed25519` | RFC 8709 | ✅ |
| `ecdsa-sha2-nistp256/384/521` | RFC 5656 | ✅ |
| `rsa-sha2-512` / `rsa-sha2-256` | RFC 8332 | ✅ |
| The `-cert-v01@openssh.com` variants of the three rows above (host certificates) | OpenSSH `PROTOCOL.certkeys` | ✅ after all the plain algorithms |
| `ssh-rsa` (SHA-1 signatures) | RFC 4253 | ❌ off by default (§6.6) |
| `ssh-rsa-cert-v01@openssh.com` (certificate with SHA-1 signatures) | OpenSSH `PROTOCOL.certkeys` | ❌ not in the default list, and the §6.6 switch does not add it either |
| `ssh-dss` | RFC 4253 | ❌ **Not implemented**. Fixed at 1024 bits, no longer acceptable |

The default list is ordered `ssh-ed25519`, `ecdsa-sha2-nistp256/384/521`, `rsa-sha2-512`, `rsa-sha2-256`,
followed by the six certificate variants in the same order. Why the certificate variants come last, and when they are moved forward, is in 03 §5.5.

### 6.3 Encryption

| Algorithm | Shape | Default |
| --- | --- | :-: |
| `chacha20-poly1305@openssh.com` | AEAD, length field **encrypted separately** | ✅ highest priority |
| `aes256-gcm@openssh.com` / `aes128-gcm@openssh.com` | AEAD, length field is plaintext AAD | ✅ |
| `aes256-ctr` / `aes192-ctr` / `aes128-ctr` | Stream + separate MAC | ✅ |
| `aes256-cbc` / `aes128-cbc` | Block + separate MAC | ❌ **Not implemented**. The §6.6 switch does not include it either |
| `3des-cbc` / `arcfour*` | — | ❌ **Not implemented** |

〔Decision〕**CBC is not implemented, and the legacy switch does not offer it to the peer either.**
Once a name we do not implement appears in our KEXINIT, it only gets "negotiated" against a device that has nothing but CBC left,
and then fails during key derivation —— with a "not implemented" message that is much harder to understand than "no common encryption algorithm",
and that shows up only when connecting to that one device. Not offering it makes such devices fail on the spot during negotiation, with both sides' complete lists in the exception (03 §2.2).
The name constants for `aes256-cbc` / `aes128-cbc` are kept and marked as not implemented; they only serve to recognize names in the peer's list and must not be put in our list.

### 6.4 MAC (used only with non-AEAD encryption)

| Algorithm | Default |
| --- | :-: |
| `hmac-sha2-256-etm@openssh.com` / `hmac-sha2-512-etm@openssh.com` | ✅ highest priority (Encrypt-then-MAC) |
| `hmac-sha2-256` / `hmac-sha2-512` | ✅ |
| `hmac-sha1-etm@openssh.com` / `hmac-sha1` | ❌ off by default. 〔Interop〕old devices |
| `hmac-md5*` / `*-96` truncated variants | ❌ **Not implemented** |

### 6.5 Compression

| Algorithm | Default |
| --- | :-: |
| `none` | ✅ |
| `zlib@openssh.com` (compression starts only after authentication) | Optional, off by default |
| `zlib` (RFC 4253, compression starts during the handshake) | ❌ Not implemented |

〔Decision〕**Only `zlib@openssh.com` is implemented; plain `zlib` is not.** Plain `zlib` compresses from the first NEWKEYS,
so authentication packets (passwords, public-key signatures) are in the compressed stream too —— ciphertext length leaks the
compressibility of the plaintext, and an unauthenticated party can mount a CRIME-style compression side channel.
OpenSSH servers have long compressed only after authentication, so leaving out plain `zlib` does not hurt real-world interop.
Even if a caller adds `zlib` to the list by hand, the pre-connect list check rejects it (§6.6) rather than it being silently treated as no compression (see 01 §6).

〔Decision〕**Compression is off by default**, consistent with OpenSSH.
Rationale: on modern links compression is usually not worth it (trading CPU for bandwidth), and the combination of compression + encryption carries
the historical lesson of CRIME-style side channels. Those who need it (weak networks, high latency) turn it on explicitly with `WithCompression()`.

### 6.6 The legacy switch and list validation

The old algorithms outside the default list are enabled in one go by `SshAlgorithmSet.WithLegacyInterop()`: it **appends**
`diffie-hellman-group14-sha1`, `ssh-rsa` (host keys with SHA-1 signatures), `hmac-sha1-etm@openssh.com` and `hmac-sha1` to the **end** of the respective lists.
Appended rather than prepended —— as long as the peer still supports one modern algorithm, the negotiation result is the same as without the switch.
It does **not** include CBC (§6.3).

Callers may also build their own lists, but no category may be empty, and every name in the key exchange, encryption, MAC and compression categories
must be one this library implements. This is checked **when connecting starts, before dialing**; a bad list throws `ArgumentException` right away,
without dialing. The rule and its rationale are in 03 §2.2.

## 7 How algorithm priority is ordered

〔Decision〕**The order of the client list is our order of preference**; the choice is "the first entry in the client list that both sides support"
(the rule of RFC 4253 §7.1). Ordering principles:

1. Security over performance: post-quantum hybrid > elliptic curve > finite-field DH.
2. AEAD over "encryption + MAC": one fewer pass over the data, and none of the historical pitfalls around MAC-vs-encryption ordering.
3. At equal security, prefer what has hardware acceleration: AES-GCM beats ChaCha20 on machines with AES-NI;
   **but ChaCha20-Poly1305 is still first by default** —— it is fast on every platform,
   while AES drops to very ugly numbers on devices without AES-NI.
   〔Decision〕At runtime the relative order of these two is adjusted dynamically according to `System.Runtime.Intrinsics.X86.Aes.IsSupported` (together with `Pclmulqdq.IsSupported`) /
   `System.Runtime.Intrinsics.Arm.Aes.IsSupported`: with AES hardware acceleration, AES-GCM is placed before ChaCha20-Poly1305.
4. Algorithms that are off by default (old algorithms) are not in the default list; they are added only when the user configures them explicitly.

## 8 Glossary

| Term | Meaning |
| --- | --- |
| **Frame** | One complete SSH binary packet (length + padding length + payload + padding + MAC) |
| **Payload** | The part of a frame left after removing the length, the padding length and the padding; its first byte is the message number |
| **Sequence number** | One `uint32` per direction, starting at 0, incremented per frame, **wrapping around on overflow**. Takes part in MAC computation |
| **Cipher suite** | The combination of "encryption algorithm + MAC algorithm". AEAD algorithms provide integrity themselves and are not paired with a MAC |
| **Shape** | How cipher suites differ in framing: whether the length field is encrypted, how long the AAD is, how long the tag is, whether it is EtM |
| **Send gate** | The gate that only lets transport-layer messages through during rekeying. See 03 and architecture.md §5.4 |
| **Watermark** | In SFTP writes, "the highest offset acknowledged **contiguously**". See 06 |
