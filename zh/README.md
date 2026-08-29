# VelaShell 文档(中文)

English: [`../en/`](../en/)

| 分区 | 内容 |
| --- | --- |
| [`host/`](host/) | 宿主(主程序)的架构、设计规格、性能记录与各类可行性调研 |
| [`plugins/`](plugins/) | 插件系统设计蓝图 01–15 与[进度总览](plugins/STATUS.md) |
| [`sdk/`](sdk/) | 插件契约 SDK 参考与发版流程 |
| [`cli/`](cli/) | `vela-plugin` 命令行手册与发版流程 |
| [`templates/`](templates/) | 插件开发指南、打包与发布、模板包发版流程 |

## 插件作者按这个顺序读

1. [开发指南](templates/dev-guide.md) —— 快速上手、清单、生命周期、能力 API、测试
2. [CLI 手册](cli/cli.md) —— `vela-plugin` 的全部命令
3. [打包与发布](templates/publishing.md) —— `.vpx`、签名、发到插件商店
4. [SDK 参考](sdk/sdk-reference.md) —— 契约表面与能力域一览(当作字典查)

[插件蓝图](plugins/)描述的是**宿主侧**的设计与实现,读它是为了理解插件为什么长这样,
写插件本身用不到。
