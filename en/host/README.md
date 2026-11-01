# Host Documentation (English)

Architecture, design specs and research for the VelaShell main application.
中文:[`../../zh/host/`](../../zh/host/)

## Architecture and specs

| Document | Contents |
| --- | --- |
| [architecture.md](architecture.md) | Layering, dependency direction and the SonnetDB persistence strategy |
| [architecture-design.md](architecture-design.md) | Engineering refactor blueprint |
| [dock-replacement-plan.md](dock-replacement-plan.md) | Replacing Dock.Avalonia with VelaDock |
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
| [session-import.md](session-import.md) | Migrating sessions from Xshell / WinSCP / OpenSSH `~/.ssh/config`: dialog behaviour, how each source is located and its passwords recovered, the `ssh_config` parsing rules, and how to add a source; the format, encryption and import rules of VelaShell's own connection files (JSON / CSV) |
| [sftp-dual-pane-winscp-gap-analysis.md](sftp-dual-pane-winscp-gap-analysis.md) | Dual-pane SFTP vs WinSCP, item by item |
| [ftp-client-feasibility-research.md](ftp-client-feasibility-research.md) | Trade-offs behind FTP / FTPS support |
| [telnet-and-serial-feasibility-research.md](telnet-and-serial-feasibility-research.md) | Feasibility and work list for Telnet / serial sessions |
| [redis-client-plugin-research-and-design.md](redis-client-plugin-research-and-design.md) | Redis GUI client: workspace connection type, engine trade-offs, UI design |
| [mongodb-workbench-plugin-design.md](mongodb-workbench-plugin-design.md) | MongoDB workbench: opened from the command palette, connections managed by the plugin (saved connections at the tree root, the board-10 connection dialog, SSH jump through a host SSH session), where each of the 23 boards landed, staging and write guards |
| [s3-protocol-plugin-design.md](s3-protocol-plugin-design.md) | Design of the S3 object-storage plugin |
| [s3-protocol-full-support-implementation-report-2026-08.md](s3-protocol-full-support-implementation-report-2026-08.md) | Implementation log for full S3 support |
| [rdp-plugin-design-and-use.md](rdp-plugin-design-and-use.md) | RDP remote desktop plugin: pure-managed C# protocol stack (no FreeRDP, no native), connection and settings, clipboard, drive mapping, certificate and NLA, build and verification |
| [system-keychain-and-sudo-credential-fill-feasibility-research.md](system-keychain-and-sudo-credential-fill-feasibility-research.md) | Feasibility of the three platforms' system keychains and sudo autofill (handing the local encryption key to the system keychain landed on 2026-10-25; the erratum at the top describes the implementation) |
| [master-password-design.md](master-password-design.md) | Design and implementation of the optional master password (the second step of master password protection, implemented 2026-10-31): Argon2id wrapping of the local key, the `VSMP1` wrapping format and recovery code, unlocking and the locked state, the gate and no dialogs for automatic reconnects / background tasks, turning on / off / changing / forgetting, automatic locking; the maintainers' 2026-10-31 decisions are in §9 |
| [credential-manager-integration-design.md](credential-manager-integration-design.md) | Password-manager integration (1Password / Bitwarden / KeePassXC / system keychain): the `ICredentialProvider` contract, wiring, unlock cache, CLI hardening and phasing; its first source, the built-in shared credentials, shipped with #550 (see §0.1) |

## Engineering logs

| Document | Contents |
| --- | --- |
| [performance-and-memory-optimization-2026-07.md](performance-and-memory-optimization-2026-07.md) | Performance and memory optimisation log |
| [terminal-input-ordering-analysis.md](terminal-input-ordering-analysis.md) | Serialising terminal input |

## Kept in the code repository

| Document | Contents |
| --- | --- |
| [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) | The design system: colour, type and spacing tokens plus component rules. XAML comments and unit tests **cite its section numbers directly**, so it stays next to the code |
| [`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md) | **What already happened**: the progress log, the current architecture, and the reasoning behind every change |
| [`feature-plan.md`](https://github.com/joesdu/VelaShell/blob/main/feature-plan.md) | **What has not happened yet**: the backlog, candidate features, and the won't-do list with reasons |

> ⚠️ Where a research document on this page says "X is not implemented", **trust
> `feature-plan.md` instead** — research documents record the judgement made *at the time* and are
> not updated as things ship.
