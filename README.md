# velashell-docs

[VelaShell](https://github.com/joesdu/VelaShell) 生态的**全部文档**:宿主设计、插件系统蓝图、
以及面向插件作者的 SDK / CLI / 模板文档。

**[中文](zh/) · [English](en/)**

在这之前,这些文档散在五个代码仓库里,跨仓库互相引用只能写绝对 URL,一改路径就断。
现在它们在同一棵树下,互相之间用相对链接,代码仓库只保留 README。

## 目录结构

```
zh/                    en/                    内容
  host/                  host/                宿主(主程序)的架构、设计规格与各类可行性调研
  plugins/               plugins/             插件系统设计蓝图 01–15 + 进度总览 STATUS
  sdk/                   sdk/                 插件契约 SDK 参考、发版流程
  cli/                   cli/                 vela-plugin 命令行手册、发版流程
  templates/             templates/           插件开发指南、打包与发布、发版流程
```

中英两棵树同构。中文是原文,英文是翻译;`release-process.md` 只有中文版。

## 快速入口

| 我想… | 读这个 |
| --- | --- |
| 写一个 VelaShell 插件 | [开发指南](zh/templates/dev-guide.md) · [Dev guide](en/templates/dev-guide.md) |
| 查 `vela-plugin` 命令 | [CLI 手册](zh/cli/cli.md) · [CLI manual](en/cli/cli.md) |
| 把插件打包发布出去 | [打包与发布](zh/templates/publishing.md) · [Publishing](en/templates/publishing.md) |
| 查插件能调用哪些 API | [SDK 参考](zh/sdk/sdk-reference.md) · [SDK reference](en/sdk/sdk-reference.md) |
| 理解插件系统为什么长这样 | [插件蓝图](zh/plugins/) · [Blueprint](en/plugins/) |
| 让团队从 IM 里用 / 让别的 agent 调 | [协作接入](zh/plugins/协作接入.md) · [Collaboration](en/plugins/collaboration.md) |
| 理解宿主的分层与依赖方向 | [架构](zh/host/architecture.md) · [Architecture](en/host/architecture.md) |

## 代码仓库

| 仓库 | 内容 |
| --- | --- |
| [joesdu/VelaShell](https://github.com/joesdu/VelaShell) | 宿主主程序 |
| [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) | `VelaShell.PluginSdk`、`.Testing` |
| [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli) | `vela-plugin`、`VelaShell.PluginSdk.Build` |
| [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates) | `dotnet new velaplugin` 模板 |
| [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins) | 第一方插件(Redis / S3 / Telnet / Serial) |
| [velashell-markets](https://github.com/VelaShellLabs/velashell-markets) | 插件市场:上传、审核、检索与分发 |
| [velashell-identity](https://github.com/joesdu/velashell-identity) | 统一认证服务(OIDC / OpenIddict):生态的信任根,下游只验令牌 |
| [velashell-feeds](https://github.com/joesdu/velashell-feeds) | 资讯服务:CVE 聚合与公告/广告投放,给消息中心供稿 |

## 改文档

改中文时**顺手改英文那份**——两棵树同构,漏掉一边就会开始漂。跨区链接一律走相对路径
(`../templates/dev-guide.md`),不要写回 GitHub 绝对 URL,那正是搬到这里要消掉的东西。

留在代码仓库里的三份文档,是因为它们服务的是"在那个仓库里写代码"这件事,搬出来反而更远:
[`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md)(设计令牌,被 XAML 与测试直接引用)、
[`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md)(进展与待办)、各仓库的 `CONTRIBUTING.md`。

## 许可

MIT,见 [LICENSE](LICENSE)。
