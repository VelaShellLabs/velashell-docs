# VelaShell.Ssh 文档

English: [`../../en/ssh/`](../../en/ssh/)

`VelaShell.Ssh` 是宿主自研的 SSH 客户端库(替换了 Tmds.Ssh):远程命令、交互式 shell、
SFTP、端口转发与隧道,全异步、低分配、AOT 友好。它是**独立实现**,不是任何现有库的 fork。

| 文档 | 内容 |
| --- | --- |
| [getting-started.md](getting-started.md) | **上手**:连接、认证、命令与 shell、SFTP、转发、代理与跳板 —— 面向调用方的最短路径 |
| [design/architecture.md](design/architecture.md) | **架构与原理**:净室论证、分层、标识符体系、测试与互操作策略、里程碑与逐条决策记录。先读这个 |
| [spec/](spec/) | **行为规格** 00–09:实现的唯一依据(纯自然语言 + 报文字段表 + 时序图,零代码片段) |

## 代码在哪

2026-09-23 起本库并入宿主仓库,不再单独发 NuGet(原仓库 `VelaShellLabs/velashell-ssh`)。
本区文档里出现的 `src/…`、`tests/…`、`scripts/ssh/…` 都是**宿主仓库**里的路径:

| 路径 | 内容 |
| --- | --- |
| [`src/VelaShell.Ssh/`](https://github.com/joesdu/VelaShell/tree/main/src/VelaShell.Ssh) | 库本体;该目录按 **MIT** 授权(`LICENSE` / `NOTICE.md`),与宿主其余部分不同 |
| [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) | 开发约定,**含净室规程** —— 改这个库之前必读 |
| [`tests/VelaShell.Ssh.Tests/`](https://github.com/joesdu/VelaShell/tree/main/tests/VelaShell.Ssh.Tests) | 单元测试(内存传输)与 `[TestCategory("Interop")]` 互操作用例 |
| [`scripts/ssh/`](https://github.com/joesdu/VelaShell/tree/main/scripts/ssh) | 相似度门禁、互操作靶机脚本、压缩严格校验检查、性能基准 |

**改了库的行为就要同步改这里的规格**,两个 PR 互相引用、一起合(宿主 AGENTS.md 第二节)。

## 相关

宿主怎么用这个库(`ISshClientWrapper` 等中立抽象、异常翻译)见
[宿主架构](../host/architecture.md)。
