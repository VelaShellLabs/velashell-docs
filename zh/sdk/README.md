# 插件契约 SDK 文档

English: [`../../en/sdk/`](../../en/sdk/)

| 文档 | 内容 |
| --- | --- |
| [sdk-reference.md](sdk-reference.md) | **SDK 参考**:包结构、入口契约、能力域一览、SDK 版本历史、测试替身、装载模型 |
| [release-process.md](release-process.md) | **`velashell-plugin-sdk` 怎么发版**:Release 流程、NuGet 可信发布配置、apiLevel 与 Avalonia 版本纪律 |

对应仓库:[velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk)
(`VelaShell.PluginSdk`、`VelaShell.PluginSdk.Testing`)。

## 相关

写插件不从这里开始——先读[开发指南](../templates/dev-guide.md),
`sdk-reference.md` 是查用的字典。命令行见 [CLI 手册](../cli/cli.md),
打包发布见[打包与发布](../templates/publishing.md)。

插件系统的**架构蓝图**(进程模型、IPC 协议、权限系统、UI 扩展、威胁模型、路线图,
编号 01–15 的那批)在 [`../plugins/`](../plugins/) —— 那是宿主侧的设计,写插件用不到。
