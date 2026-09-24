# VelaShell.XServer 文档

English: [`../../en/xserver/`](../../en/xserver/)

`VelaShell.XServer` 是一个**可嵌入、无原生依赖、跨平台**的 X11 服务端库(rootless,软件绘图)。
宿主把它的顶层窗口画成自己的原生窗口,远端经 SSH X11 转发过来的图形程序就显示在本机 —— 用户不必另装
VcXsrv / XQuartz。它是**独立实现**,不是任何现有 X 服务端的 fork。

| 文档 | 内容 |
| --- | --- |
| [design/architecture.md](design/architecture.md) | **架构与原理**:为什么自己写、目标与非目标、净室规程、分层、线程模型、宿主接口、关键取舍、里程碑、测试策略、决策记录。先读这个 |
| [troubleshooting.md](troubleshooting.md) | **排障**:远端图形程序经 SSH 转发时的 DRI3 警告、GTK4 启动慢 25 秒(桌面门户)、操作卡(GL 整窗推像素),以及已修的两处宿主缺陷 |

## 代码在哪

在宿主仓库里(与 `VelaShell.Ssh` 同样的安排,不单独发包):

| 路径 | 内容 |
| --- | --- |
| [`src/VelaShell.XServer/`](https://github.com/joesdu/VelaShell/tree/main/src/VelaShell.XServer) | 库本体;该目录按 **MIT** 授权(`LICENSE` / `NOTICE.md`),与宿主其余部分不同 |
| [`src/VelaShell.XServer/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.XServer/AGENTS.md) | 开发约定,**含净室规程** —— 改这个库之前必读 |
| [`tests/VelaShell.XServer.Tests/`](https://github.com/joesdu/VelaShell/tree/main/tests/VelaShell.XServer.Tests) | 单元测试(内存双工流)与 `[TestCategory("Interop")]` 真实客户端用例 |
| [`scripts/xserver/interop/`](https://github.com/joesdu/VelaShell/tree/main/scripts/xserver/interop) | 互操作靶场:客户端镜像、起服务端存 PNG / 注入输入的脚本 |

**改了库的行为就要同步改这里**,两个 PR 互相引用、一起合(宿主 AGENTS.md 第二节)。

## 相关

宿主目前怎么显示 X11 转发(拉起用户装好的 VcXsrv)见
[`../host/交互与界面规格.md`](../host/交互与界面规格.md) §4A.2 与 §14「X Server」页。
