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
| [message-center-and-feed.md](message-center-and-feed.md) | The sidebar bell, the news-feed JSON contract and targeting rules (read this to build the push backend) |
| [route-tracing-design.md](route-tracing-design.md) | Traceroute and geographic visualisation |
| [session-import.md](session-import.md) | Migrating sessions from Xshell / WinSCP / OpenSSH `~/.ssh/config`: dialog behaviour, how each source is located and its passwords recovered, the `ssh_config` parsing rules, and how to add a source |
| [sftp-dual-pane-winscp-gap-analysis.md](sftp-dual-pane-winscp-gap-analysis.md) | Dual-pane SFTP vs WinSCP, item by item |
| [ftp-client-feasibility-research.md](ftp-client-feasibility-research.md) | Trade-offs behind FTP / FTPS support |
| [telnet-and-serial-feasibility-research.md](telnet-and-serial-feasibility-research.md) | Feasibility and work list for Telnet / serial sessions |

## Engineering logs

| Document | Contents |
| --- | --- |
| [performance-and-memory-optimization-2026-07.md](performance-and-memory-optimization-2026-07.md) | Performance and memory optimisation log |
| [terminal-input-ordering-analysis.md](terminal-input-ordering-analysis.md) | Serialising terminal input |

## Chinese-only

Four research documents have no English translation yet:

| Document | Contents |
| --- | --- |
| [Redis客户端插件化调研与设计.md](../../zh/host/Redis客户端插件化调研与设计.md) | Redis GUI client: workspace connection type, engine trade-offs, UI design |
| [S3协议插件化设计.md](../../zh/host/S3协议插件化设计.md) | Design of the S3 object-storage plugin |
| [S3协议完整支持-实施报告-2026-08.md](../../zh/host/S3协议完整支持-实施报告-2026-08.md) | Implementation log for full S3 support |
| [系统密钥链与sudo凭据填充可行性调研.md](../../zh/host/系统密钥链与sudo凭据填充可行性调研.md) | Feasibility of the three platforms' system keychains and sudo autofill |

## Kept in the code repository

| Document | Contents |
| --- | --- |
| [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) | The design system: colour, type and spacing tokens plus component rules. XAML comments and unit tests **cite its section numbers directly**, so it stays next to the code |
| [`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md) | **What already happened**: the progress log, the current architecture, and the reasoning behind every change |
| [`feature-plan.md`](https://github.com/joesdu/VelaShell/blob/main/feature-plan.md) | **What has not happened yet**: the backlog, candidate features, and the won't-do list with reasons |

> ⚠️ Where a research document on this page says "X is not implemented", **trust
> `feature-plan.md` instead** — research documents record the judgement made *at the time* and are
> not updated as things ship.
