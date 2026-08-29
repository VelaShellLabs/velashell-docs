# Plugin Development Documentation

**This is the main documentation for writing VelaShell plugins.**
中文:[`../../zh/templates/`](../../zh/templates/)

| Document | Contents |
| --- | --- |
| [dev-guide.md](dev-guide.md) | **Dev guide**: getting started, manifest, lifecycle, capability APIs, isolation modes, testing, deployment, performance discipline |
| [publishing.md](publishing.md) | **Packaging and publishing**: Release builds, `.vpx`, signing and trust, shipping to the [marketplace](http://market.easilynet.top), CI packaging |

Repository: [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates)
(`dotnet new velaplugin` / `velaplugin-ui`). Its release process is documented in Chinese only:
[`../../zh/templates/release-process.md`](../../zh/templates/release-process.md).

Writing your first plugin, read in this order: [dev-guide.md](dev-guide.md) →
[CLI manual](../cli/cli.md) → [publishing.md](publishing.md), with the
[SDK reference](../sdk/sdk-reference.md) as your dictionary.

The plugin system's **architecture blueprint** (process model, IPC protocol, permission system, UI
extensions, threat model, roadmap — documents 01–15) is in [`../plugins/`](../plugins/). Those
documents describe the **host-side** design; read them to understand why plugins look the way they
do, not to write one.
