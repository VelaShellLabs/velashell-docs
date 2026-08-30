# 宿主文档(中文)

VelaShell 主程序自身的架构、设计规格与调研记录。English: [`../../en/host/`](../../en/host/)

## 架构与规格

| 文档 | 内容 |
| --- | --- |
| [architecture.md](architecture.md) | 分层架构、依赖方向与 SonnetDB 持久化策略 |
| [架构设计.md](架构设计.md) | 工程化重构蓝图 |
| [dock-replacement-plan.md](dock-replacement-plan.md) | VelaDock 自研替换 Dock.Avalonia 的方案 |
| [design-specs.md](design-specs.md) | UI 视觉规格(Pencil 逐帧提取) |
| [交互与界面规格.md](交互与界面规格.md) | 交互逻辑与设计 Token |
| [快捷键参考.md](快捷键参考.md) | 全部键盘快捷键与鼠标手势(与 `ShortcutCatalog` 同源,非手抄) |
| [settings-audit.md](settings-audit.md) | 设置项审计台账与整改记录 |

## 功能设计与调研

| 文档 | 内容 |
| --- | --- |
| [Xshell兼容登录.md](Xshell兼容登录.md) | 堡垒机/SSO 按 Xshell 方式拉起登录的兼容层与安全模型 |
| [隧道功能规划.md](隧道功能规划.md) | 端口转发隧道设计 |
| [消息中心与资讯源.md](消息中心与资讯源.md) | 侧边栏铃铛的设计、资讯源 JSON 契约与投放规则（做推送后台看这篇） |
| [路由追踪设计.md](路由追踪设计.md) | 路由追踪与地理可视化 |
| [SFTP双栏与WinSCP差距分析.md](SFTP双栏与WinSCP差距分析.md) | 双栏 SFTP 与 WinSCP 的逐项差距决策清单 |
| [FTP客户端可行性调研.md](FTP客户端可行性调研.md) | FTP / FTPS 支持的取舍 |
| [Telnet与串口可行性调研.md](Telnet与串口可行性调研.md) | Telnet / 串口会话类型的可行性与改造清单 |
| [Redis客户端插件化调研与设计.md](Redis客户端插件化调研与设计.md) | Redis 界面客户端:工作台连接类型、引擎取舍与界面设计 |
| [S3协议插件化设计.md](S3协议插件化设计.md) | S3 对象存储接入插件的设计 |
| [S3协议完整支持-实施报告-2026-08.md](S3协议完整支持-实施报告-2026-08.md) | S3 完整支持的实施记录 |
| [系统密钥链与sudo凭据填充可行性调研.md](系统密钥链与sudo凭据填充可行性调研.md) | 三平台系统密钥链与 sudo 自动填充的可行性 |

## 工程记录

| 文档 | 内容 |
| --- | --- |
| [性能与内存优化-2026-07.md](性能与内存优化-2026-07.md) | 性能与内存优化批次记录 |
| [终端输入乱序问题分析与架构建议.md](终端输入乱序问题分析与架构建议.md) | 终端输入串行化 |

## 留在代码仓库里的两份

[`DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) —— 设计系统的色彩/字体/间距令牌与组件规范。
它被 XAML 注释与单元测试直接按章节号引用,跟着代码走才不会漂。

[`plan.md`](https://github.com/joesdu/VelaShell/blob/main/plan.md) —— 进展记录、已知问题与后续待办,开发跟进以它为准。
