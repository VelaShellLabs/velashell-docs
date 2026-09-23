# VelaShell.XServer documentation

中文:[`../../zh/xserver/`](../../zh/xserver/)

`VelaShell.XServer` is an **embeddable, native-dependency-free, cross-platform** X11 server library (rootless, software
rendering). The host draws its top-level windows as native windows, so GUI programs forwarded from remote hosts over
SSH X11 appear locally — without users installing VcXsrv / XQuartz. It is an **independent implementation**, not a fork
of any existing X server.

| Document | Content |
| --- | --- |
| [design/architecture.md](design/architecture.md) | **Architecture and rationale**: why we build it, goals and non-goals, clean-room rules, layering, threading model, host interface, key trade-offs, milestones, test strategy, decision log. Read this first |

## Where the code lives

In the host repository (the same arrangement as `VelaShell.Ssh`; not published as a separate package):

| Path | Content |
| --- | --- |
| [`src/VelaShell.XServer/`](https://github.com/joesdu/VelaShell/tree/main/src/VelaShell.XServer) | The library; this directory is **MIT** licensed (`LICENSE` / `NOTICE.md`), unlike the rest of the host |
| [`src/VelaShell.XServer/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.XServer/AGENTS.md) | Development rules, **including the clean-room rules** — read before changing the library |
| [`tests/VelaShell.XServer.Tests/`](https://github.com/joesdu/VelaShell/tree/main/tests/VelaShell.XServer.Tests) | Unit tests (in-memory duplex streams) and `[TestCategory("Interop")]` real-client cases |
| [`scripts/xserver/interop/`](https://github.com/joesdu/VelaShell/tree/main/scripts/xserver/interop) | Interop range: client image, a script that runs the server, saves PNGs and injects input |

**Behaviour changes in the library must be reflected here**; the two PRs reference each other and merge together
(host AGENTS.md, section 2).

## Related

How the host currently shows X11 forwarding (launching the VcXsrv the user installed): see
[`../host/interaction-and-ui-specs.md`](../host/interaction-and-ui-specs.md) §4A.2 and the "X Server" page in §14.
