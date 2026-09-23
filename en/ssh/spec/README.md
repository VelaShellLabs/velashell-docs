# Behavior Specification

> **This document set is the sole basis for the VelaShell.Ssh implementation.**
>
> While writing the implementation, there should be only two things on your desk: the specifications in this directory, and the RFCs they reference.
> **Do not open the source code of any other SSH implementation** —— [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) §2, Discipline 2.
> 中文：[`../../../zh/ssh/spec/README.md`](../../../zh/ssh/spec/README.md)

| File | Content | Implementation layer |
| --- | --- | --- |
| [`00-overview.md`](00-overview.md) | Terminology, notation, data types, general security baseline, algorithm master table and priorities | — |
| [`01-transport-framing.md`](01-transport-framing.md) | Binary packet protocol, padding, sequence numbers, **the three shapes of cipher suites**, send/receive sequencing and coalescing | `Transport/` `Crypto/` |
| [`02-version-exchange.md`](02-version-exchange.md) | Identification string exchange, banner lines, interoperability details | `Session/` |
| [`03-key-exchange.md`](03-key-exchange.md) | Algorithm negotiation, seven KEX methods, **exchange hash field by field**, key derivation, strict KEX, rekeying and the send gate | `Crypto/` `Session/` |
| [`04-authentication.md`](04-authentication.md) | Method dispatch, **partial success (2FA)**, publickey signature input, **keyboard-interactive**, extension negotiation | `Auth/` |
| [`05-connection.md`](05-connection.md) | Channel lifecycle, **flow control and adaptive windows**, channel requests, pty and pixel dimensions | `Channels/` |
| [`06-sftp.md`](06-sftp.md) | SFTP v3 wire, request pipelining, **write watermark**, extensions and capability queries, symlink conventions | `Sftp/` |
| [`07-forwarding.md`](07-forwarding.md) | Four kinds of forwarding, SOCKS5 subset, half-close, **in-library metering**, agent forwarding | `Forwarding/` |
| [`08-failures.md`](08-failures.md) | Failure taxonomy, three exceptions with structured context, metrics, tracing, packet tap | `Diagnostics/` |
| [`09-dialing.md`](09-dialing.md) | Dialing layer: SOCKS5, HTTP CONNECT, jump hosts, proxy commands, nesting, per-hop failure information, `ssh_config` mapping | `Transport/` `Config/` |

## Which sections to read first

Ordered by "most error-prone and most expensive first":

1. **[03 §4 Exchange hash](03-key-exchange.md)** —— every single byte-order mistake surfaces as the same
   "signature verification failed", and is often **probabilistic** (`mpint` leading zeros).
   The tables in this section should directly become the data source for unit tests.
2. **[04 §3.3 Partial success](04-authentication.md)** —— treat `partial_success = true`
   as a failure and 2FA will never connect.
3. **[01 §1.2 Padding and alignment](01-transport-framing.md)** —— under AEAD the length field does not take part in alignment;
   getting it wrong only shows up at certain packet lengths.
4. **[06 §4.5 SYMLINK argument order](06-sftp.md)** —— the specification has it backwards, and the whole world follows OpenSSH in getting it wrong.
   The code must spell this out clearly, otherwise someone will certainly "fix it in passing" one day.
5. **[05 §1 EOF is not CLOSE](05-connection.md)** —— mishandling half-close silently truncates data.

## How to change this specification

- Where the specification is unclear, **go back to the RFC and fill the gap in the specification**, rather than looking elsewhere for a ready-made answer.
- Every 〔Decision〕 must give its rationale —— people in the future will ask "why is it this way",
  and the answer should not exist only in someone's memory.
- Every 〔Interop〕 must state **which server, which version, and how it was verified**.
  A vague "some servers reportedly do this" without a concrete subject does not count.
- A change in behavior requires a matching change in the specification, both in the same PR.
