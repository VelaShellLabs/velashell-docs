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

中英两棵树同构:中文是原文,英文是翻译。**当前有 7 篇只有中文版**:

| 只有中文版 | 位置 |
| --- | --- |
| Redis 客户端插件化调研与设计 | [`zh/host/Redis客户端插件化调研与设计.md`](zh/host/Redis客户端插件化调研与设计.md) |
| S3 协议插件化设计 | [`zh/host/S3协议插件化设计.md`](zh/host/S3协议插件化设计.md) |
| S3 完整支持实施报告 | [`zh/host/S3协议完整支持-实施报告-2026-08.md`](zh/host/S3协议完整支持-实施报告-2026-08.md) |
| 系统密钥链与 sudo 凭据填充可行性调研 | [`zh/host/系统密钥链与sudo凭据填充可行性调研.md`](zh/host/系统密钥链与sudo凭据填充可行性调研.md) |
| 三份发版流程 `release-process.md` | [`zh/sdk/`](zh/sdk/release-process.md) · [`zh/cli/`](zh/cli/release-process.md) · [`zh/templates/`](zh/templates/release-process.md) |

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
| 看整个生态怎么拼起来 | 下面这张图 |

## 生态地图

```mermaid
flowchart TB
    subgraph Client["用户机器"]
        VS["<b>VelaShell 宿主</b><br/>joesdu/VelaShell"]
        AI["AI 插件<br/>（同仓构建、同版发布）"]
        P3["第三方 / 第一方插件<br/>Redis · S3 · Telnet · Serial · DockerPanel"]
        VS --- AI
        VS -.装载 .vpx.- P3
    end

    subgraph Author["插件作者"]
        SDK["velashell-plugin-sdk<br/>契约 NuGet 包"]
        CLI["velashell-plugin-cli<br/>vela-plugin · PluginSdk.Build"]
        TPL["velashell-plugin-templates<br/>dotnet new velaplugin"]
    end

    subgraph Services["线上服务"]
        MKT["velashell-markets<br/>插件市场：上传 · 审核 · 分发"]
        IDN["velashell-identity<br/>统一认证（OIDC）—— 生态的信任根"]
        FEED["velashell-feeds<br/>CVE 聚合与公告投放"]
    end

    DOCS["<b>velashell-docs</b><br/>（本仓库）全部文档"]

    SDK --> P3
    CLI --> P3
    TPL --> P3
    P3 -->|发布| MKT
    MKT -->|按需安装| VS
    IDN -.验令牌.-> MKT
    FEED -->|供稿| VS
    DOCS -.描述.-> VS
    DOCS -.描述.-> Author
    DOCS -.描述.-> Services

    style VS fill:#1d3557,color:#fff
    style DOCS fill:#2d6a4f,color:#fff
    style IDN fill:#7c4a03,color:#fff
```

## 代码仓库

| 仓库 | 内容 |
| --- | --- |
| [joesdu/VelaShell](https://github.com/joesdu/VelaShell) | 宿主主程序 |
| [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) | `VelaShell.PluginSdk`、`.Testing` |
| [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli) | `vela-plugin`、`VelaShell.PluginSdk.Build` |
| [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates) | `dotnet new velaplugin` 模板 |
| [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins) | 第一方插件(Redis / S3 / Telnet / Serial) |
| [VelaShell.Plugin.DockerPanel](https://github.com/VelaShellLabs/VelaShell.Plugin.DockerPanel) | Docker 管理面板插件(自成一仓,已发布 `0.3.1`) |
| [velashell-markets](https://github.com/VelaShellLabs/velashell-markets) | 插件市场:上传、审核、检索与分发 |
| [velashell-identity](https://github.com/joesdu/velashell-identity) | 统一认证服务(OIDC / OpenIddict):生态的信任根,下游只验令牌 |
| [velashell-feeds](https://github.com/joesdu/velashell-feeds) | 资讯服务:CVE 聚合与公告/广告投放,给消息中心供稿 |

## 改文档

改中文时**顺手改英文那份**——两棵树同构,漏掉一边就会开始漂。跨区链接一律走相对路径
(`../templates/dev-guide.md`),不要写回 GitHub 绝对 URL,那正是搬到这里要消掉的东西。

留在代码仓库里的几份文档,是因为它们服务的是"在那个仓库里写代码"这件事,搬出来反而更远:

| 文档 | 内容 |
| --- | --- |
| [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) | 设计令牌,被 XAML 注释与单元测试**按章节号直接引用** |
| [`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md) | **已经发生的事**:进展记录、当前架构、每次改动的来龙去脉 |
| [`feature-plan.md`](https://github.com/joesdu/VelaShell/blob/main/feature-plan.md) | **还没发生的事**:待办、候选特性、确认不做的清单(附理由) |
| 各仓库的 `CONTRIBUTING.md` | 分支、提交与测试约定 |

## 许可

MIT,见 [LICENSE](LICENSE)。
