# VelaShell.Ssh Documentation

中文:[`../../zh/ssh/`](../../zh/ssh/)

`VelaShell.Ssh` is the host's in-house SSH client library (it replaced Tmds.Ssh): remote commands,
interactive shells, SFTP, port forwarding and tunnels — fully async, low-allocation, AOT-friendly.
It is an **independent implementation**, not a fork of any existing library.

| Document | Contents |
| --- | --- |
| [getting-started.md](getting-started.md) | **Getting started**: connecting, authentication, commands and shells, SFTP, forwarding, proxies and jump hosts — the shortest path for callers |
| [design/architecture.md](design/architecture.md) | **Architecture and rationale**: the clean-room argument, layering, identifier scheme, testing and interop strategy, milestones and the per-item decision log. Read this first |
| [spec/](spec/) | **Behavioural specs** 00–09: the sole basis for the implementation (plain prose + packet field tables + sequence diagrams, zero code) |

## Where the code lives

Since 2026-09-23 the library lives in the host repository and is no longer published to NuGet on its
own (it used to be `VelaShellLabs/velashell-ssh`). Paths such as `src/…`, `tests/…` and
`scripts/ssh/…` in these documents refer to the **host repository**:

| Path | Contents |
| --- | --- |
| [`src/VelaShell.Ssh/`](https://github.com/joesdu/VelaShell/tree/main/src/VelaShell.Ssh) | The library; this directory is licensed under **MIT** (`LICENSE` / `NOTICE.md`), unlike the rest of the host |
| [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) | Working rules, **including the clean-room discipline** — required reading before touching the library |
| [`tests/VelaShell.Ssh.Tests/`](https://github.com/joesdu/VelaShell/tree/main/tests/VelaShell.Ssh.Tests) | Unit tests (in-memory transport) and the `[TestCategory("Interop")]` interop cases |
| [`scripts/ssh/`](https://github.com/joesdu/VelaShell/tree/main/scripts/ssh) | Similarity gate, interop test-server scripts, compression strict-validation check, benchmarks |

**A behaviour change in the library must update the specs here too**, in two PRs that reference each
other and merge together (section 2 of the host's AGENTS.md).

## Related

How the host consumes the library (the neutral `ISshClientWrapper` abstractions, exception
translation) is covered in the [host architecture](../host/architecture.md).
