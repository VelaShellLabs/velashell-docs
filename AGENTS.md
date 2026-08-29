# AGENTS.md

> 给 AI 代理与新加入者的操作约定。**动手之前先读完本文件,以及它指向的文档。**

## 一、开工前必读:velashell-docs

VelaShell 生态的**全部文档**集中在一个仓库:
**[VelaShellLabs/velashell-docs](https://github.com/VelaShellLabs/velashell-docs)**。
本仓库**不放** `docs/`、`docs-en/` —— 设计手册、开发规范与开发文档都在那边。

**在动任何代码之前**,先把下表中与你要改的部分相关的几篇读掉。跳过这一步直接改,
结果通常是两种:与既有设计冲突,或者重复实现一个已经存在的能力。

| 位置 | 内容 |
| --- | --- |
| [`zh/host/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/zh/host) | 宿主分层架构与依赖方向、工程化重构蓝图、交互与界面规格、快捷键参考、设置项审计,以及 SFTP / FTP / Telnet / 串口 / Redis / S3 / 系统密钥链等可行性调研 |
| [`zh/plugins/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/zh/plugins) | 插件系统设计蓝图 01–15(进程模型、IPC 协议、权限系统、UI 扩展、威胁模型、路线图)与[进度总览 STATUS](https://github.com/VelaShellLabs/velashell-docs/blob/main/zh/plugins/STATUS.md) |
| [`zh/sdk/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/zh/sdk) | 插件契约 SDK 参考、SDK 仓库的发版流程 |
| [`zh/cli/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/zh/cli) | `vela-plugin` 命令行手册、CLI 仓库的发版流程 |
| [`zh/templates/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/zh/templates) | 插件开发指南、打包与发布、模板仓库的发版流程 |

英文镜像在 [`en/`](https://github.com/VelaShellLabs/velashell-docs/tree/main/en),与 `zh/` 同构。
[仓库首页](https://github.com/VelaShellLabs/velashell-docs)有按「我想做什么」组织的快速入口表。

## 二、涉及文档的改动一律同步到 velashell-docs

**这是本文件最重要的一条。**

- 本仓库里**不新建** `docs/`、`docs-en/` 或任何成体系的文档目录。要写文档,去 velashell-docs 开 PR。
- 改了代码,而**行为、接口、配置项、命令行、构建流程或版本纪律**与现有文档对不上时,
  必须**同时**在 velashell-docs 提一个 PR 把文档改过来。两个 PR 在正文里互相引用,一起合。
  只改代码不改文档,等于让文档开始骗人 —— 而文档是别人照抄的。
- velashell-docs 的 `zh/` 与 `en/` 是**互为镜像**的两棵树,文件一一对应。改了中文就要改英文,
  反之亦然。漏一边,两棵树就开始漂。
- velashell-docs 内部的互相引用**一律走相对路径**(如 `../templates/dev-guide.md`),
  不要写回 GitHub 绝对 URL —— 文档集中到一个仓库,消掉的正是那种一改路径就断的跨仓库链接。
- **例外**:留在代码仓库里的少数几份文件不适用上述规则,因为它们服务的是「在这个仓库里写代码」
  这件事,搬走只会离使用场景更远。各仓库的例外清单见下面第三节。

## 三、本仓库:velashell-docs(文档本体)

**你正站在第一、二节说的那个仓库里。** 这里是 VelaShell 生态全部文档的所在,
其余仓库的所有文档改动最终都落到这棵树上。

### 目录结构

```
zh/            en/
  host/          host/        宿主(joesdu/VelaShell)的架构、规格与调研
  plugins/       plugins/     插件系统设计蓝图 01–15 与 STATUS
  sdk/           sdk/         插件契约 SDK 参考、发版流程
  cli/           cli/         vela-plugin 手册、发版流程
  templates/     templates/   插件开发指南、打包与发布、发版流程
```

`zh/` 与 `en/` **同构**,文件一一对应。`release-process.md` 目前只有中文版,
`en/` 各区索引页里注明了这一点。

### 改这里的规矩

- **改中文顺手改英文**,反之亦然。漏一边,两棵树立刻开始漂 —— 这是本仓库最容易犯的错。
- **跨区引用一律相对路径**(`../templates/dev-guide.md`、`../../en/sdk/`),
  不要写 GitHub 绝对 URL。文档集中到一起,消掉的正是那种跨仓库绝对链接。
  指向**代码仓库**里的文件(如宿主的 `DESIGN.md`、`plan.md`)才用绝对 URL。
- **新增文件要挂进索引**:所在分区的 `README.md`,必要时还有 `zh/README.md` / `en/README.md`
  与仓库首页的快速入口表。只丢一个孤儿文件进去,没人会找到它。
- 提交前跑一遍相对链接可达性检查,别留死链。

### 版本横幅由别的仓库的脚本来改

`zh|en/sdk/sdk-reference.md`、`zh|en/cli/cli.md`、`zh|en/templates/dev-guide.md` 里的
版本号/`PackageReference` 片段,由**各自代码仓库**的 `scripts/Set-Version.ps1` 在发版时写入
(它们按 `-DocsRoot` / `$env:VELASHELL_DOCS` / 同级 `../velashell-docs` 找本仓库)。
**手工改这几处版本号通常是错的** —— 应该去对应仓库跑那个脚本,否则下次发版会被覆盖回去。

### 不在这里的文档

留在代码仓库的:各仓库的 `README.md` / `CONTRIBUTING.md` / `SECURITY.md`,
宿主的 [`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md)(被 XAML 注释与
单元测试按章节号引用)与 [`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md)(进展记录),
以及 velashell-markets 的 `docs/`(**尚未并入,待迁移**)。
