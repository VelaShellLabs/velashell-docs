# Plugin Contract SDK Documentation

中文:[`../../zh/sdk/`](../../zh/sdk/)

| Document | Contents |
| --- | --- |
| [sdk-reference.md](sdk-reference.md) | **SDK reference**: package layout, entry contract, capability domains, SDK version history, test doubles, load model |

Repository: [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk)
(`VelaShell.PluginSdk`, `VelaShell.PluginSdk.Testing`). Its release process is documented in
Chinese only: [`../../zh/sdk/release-process.md`](../../zh/sdk/release-process.md).

## Related

Do not start here to write a plugin — read the [dev guide](../templates/dev-guide.md) first;
`sdk-reference.md` is the dictionary you look things up in. For the command line see the
[CLI manual](../cli/cli.md), for shipping see [publishing](../templates/publishing.md).

The plugin system's **architecture blueprint** (process model, IPC protocol, permission system, UI
extensions, threat model, roadmap — documents 01–15) is in [`../plugins/`](../plugins/). That is
host-side design; you do not need it to write a plugin.
