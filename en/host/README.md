# Host Documentation (English)

Architecture, design specs and research for the VelaShell main application.
中文:[`../../zh/host/`](../../zh/host/)

## Architecture and specs

| Document | Contents |
| --- | --- |
| [architecture.md](architecture.md) | Layering, dependency direction and the SonnetDB persistence strategy |
| [architecture-design.md](architecture-design.md) | Engineering refactor blueprint |
| [dock-replacement-plan.md](dock-replacement-plan.md) | Replacing Dock.Avalonia with the in-house VelaDock |
| [design-specs.md](design-specs.md) | UI visual specs (extracted frame by frame from Pencil) |
| [interaction-and-ui-specs.md](interaction-and-ui-specs.md) | Interaction logic and design tokens |
| [keyboard-shortcuts.md](keyboard-shortcuts.md) | Every keyboard shortcut and mouse gesture (generated from `ShortcutCatalog`, not hand-copied) |
| [settings-audit.md](settings-audit.md) | Settings audit ledger and remediation log |

## Feature design and research

| Document | Contents |
| --- | --- |
| [xshell-compatible-login.md](xshell-compatible-login.md) | Xshell-compatible external launch for jump servers, and its security model |
| [tunnel-feature-planning.md](tunnel-feature-planning.md) | Port-forwarding tunnel design |
| [route-tracing-design.md](route-tracing-design.md) | Traceroute and geographic visualisation |
| [sftp-dual-pane-winscp-gap-analysis.md](sftp-dual-pane-winscp-gap-analysis.md) | Dual-pane SFTP vs WinSCP, item by item |
| [ftp-client-feasibility-research.md](ftp-client-feasibility-research.md) | Trade-offs behind FTP / FTPS support |
| [telnet-and-serial-feasibility-research.md](telnet-and-serial-feasibility-research.md) | Feasibility and work list for Telnet / serial sessions |

## Engineering logs

| Document | Contents |
| --- | --- |
| [performance-and-memory-optimization-2026-07.md](performance-and-memory-optimization-2026-07.md) | Performance and memory optimisation log |
| [terminal-input-ordering-analysis.md](terminal-input-ordering-analysis.md) | Serialising terminal input |

## Chinese-only

Three research documents have no English translation yet — Redis client design, S3 protocol design
and its implementation report, and the system-keychain / sudo credential research. See
[`../../zh/host/`](../../zh/host/).

## Kept in the code repository

[`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) — the design system: colour,
type and spacing tokens plus component rules. XAML comments and unit tests cite its section numbers
directly, so it stays next to the code.

[`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md) — progress log, known issues and
the backlog; the source of truth for day-to-day work.
