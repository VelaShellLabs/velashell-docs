# 插件开发文档

**写 VelaShell 插件的主文档在这里。** English: [`../../en/templates/`](../../en/templates/)

| 文档 | 内容 |
| --- | --- |
| [dev-guide.md](dev-guide.md) | **开发指南**:快速上手、清单、生命周期、能力 API、隔离模式、测试、部署、性能纪律 |
| [publishing.md](publishing.md) | **打包与发布**:Release 构建、`.vpx`、签名与信任、发布到[插件商店](http://market.easilynet.top)、CI 出包 |
| [release-process.md](release-process.md) | **`velashell-plugin-templates` 怎么发版**:模板包的 Release 流程、两个版本号的分工、NuGet 可信发布配置 |

对应仓库:[velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates)
(`dotnet new velaplugin` / `velaplugin-ui` 模板)。

第一次写插件的话,按 [dev-guide.md](dev-guide.md) → [CLI 手册](../cli/cli.md) →
[publishing.md](publishing.md) 的顺序读,[SDK 参考](../sdk/sdk-reference.md)当字典查。

插件系统的**架构蓝图**(进程模型、IPC 协议、权限系统、UI 扩展、威胁模型、路线图,
编号 01–15 的那批)在 [`../plugins/`](../plugins/) —— 那些文档描述的是**宿主侧**的设计与实现,
读它是为了理解插件为什么长这样,写插件本身用不到。
