# VelaShell Documentation (English)

中文:[`../zh/`](../zh/)

| Area | Contents |
| --- | --- |
| [`host/`](host/) | Host (main app) architecture, design specs, performance logs and feasibility research |
| [`plugins/`](plugins/) | Plugin system design blueprint 01–15 and the [status overview](plugins/STATUS.md) |
| [`sdk/`](sdk/) | Plugin contract SDK reference |
| [`cli/`](cli/) | `vela-plugin` command-line manual |
| [`templates/`](templates/) | Plugin dev guide, packaging and publishing |

## Read in this order to write a plugin

1. [Dev guide](templates/dev-guide.md) — getting started, manifest, lifecycle, capability APIs, testing
2. [CLI manual](cli/cli.md) — every `vela-plugin` command
3. [Publishing](templates/publishing.md) — `.vpx`, signing, shipping to the marketplace
4. [SDK reference](sdk/sdk-reference.md) — contract surface and capability domains (look things up here)

The [plugin blueprint](plugins/) describes the **host-side** design and implementation. Read it to
understand why plugins look the way they do; you do not need it to write one.

> Release-process documents (`release-process.md`) exist in Chinese only, under
> [`../zh/sdk/`](../zh/sdk/release-process.md), [`../zh/cli/`](../zh/cli/release-process.md) and
> [`../zh/templates/`](../zh/templates/release-process.md).
