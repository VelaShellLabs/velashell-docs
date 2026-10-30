# 插件 SDK 参考

> 适用版本:**SDK 2.1.3 / apiLevel 2**
> 相关文档(在别的仓库):[开发指南](../templates/dev-guide.md)(教程式) · [CLI 手册](../cli/cli.md) · [打包与发布](../templates/publishing.md)

本篇是**面向查阅**的 SDK 全貌:包结构、契约表面、每个能力域能做什么/受什么限制、
版本演进与测试替身。想要按步骤走一遍的,先读[开发指南](../templates/dev-guide.md)。

---

## 1. 包结构

| 包                              | 出自哪个仓库                                                                              | 谁引用                   | 内容                                                                                         |
| ------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------- |
| **`VelaShell.PluginSdk.Build`** | 本仓库(2026-09-11 起;此前在 velashell-plugin-cli)                                         | **插件工程(只引这一个)** | MSBuild props/targets、锁定版本的 Avalonia、随包分发的打包器、清单校验、`PackVpx` 目标       |
| `VelaShell.PluginSdk`           | 本仓库                                                                                    | 由上者传递引入           | 契约程序集:入口接口、`IPluginContext` 与全部能力接口、DTO、清单模型、`.vpx` 容器、宿主注册表 |
| `VelaShell.PluginSdk.Testing`   | 本仓库                                                                                    | 插件的**测试**工程       | `TestPluginContext` 与各能力的内存替身                                                       |
| `VelaShell.Plugin.Cli`          | [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli)             | 开发者机器(dotnet tool)  | `vela-plugin`:开发内环、校验、打包、签名、体检                                               |
| `VelaShell.Plugin.Templates`    | [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates) | 开发者机器               | `dotnet new velaplugin` / `velaplugin-ui` / `velaplugin-protocol` / `velaplugin-workspace` |

插件工程要写的那一行 `PackageReference`(**含当前版本号**)在[开发指南](../templates/dev-guide.md)里 ——
那个版本号属于 `VelaShell.PluginSdk.Build`,与本仓库的 SDK 版本各走各的线,
所以不在这里重复一遍(重复一遍就是等着它过期)。

**不要**再单独引用 `VelaShell.PluginSdk` 或 `Avalonia` —— 版本对不上会在构建期直接报
`VELA1001`,而不是等用户装上后在运行期表现为控件类型转换失败。

契约程序集只依赖 BCL。这是一条纪律:它是宿主与插件**唯一共享的类型来源**,
往里塞第三方依赖等于把那个依赖的版本强加给所有插件。

---

## 2. 入口契约

```csharp
using VelaShell.PluginSdk;

[VelaPlugin]                                  // 恰好一个公开、非抽象、无参构造的类
public sealed class DemoPlugin : IVelaPlugin
{
    public Task ActivateAsync(IPluginContext context, CancellationToken ct)
    {
        context.Log.Info("activated");
        return Task.CompletedTask;            // 必须快速返回(限时 10 秒)
    }

    public Task DeactivateAsync(CancellationToken ct) => Task.CompletedTask;  // 限时约 2 秒
}
```

| 约束     | 细节                                                                                                |
| -------- | --------------------------------------------------------------------------------------------------- |
| 激活时限 | 10 秒;**挂着调试器时自动放宽到 10 分钟**                                                            |
| 停用时限 | 约 2 秒(应用退出路径),超时被放弃                                                                    |
| 长任务   | 自己开后台任务,用 `context.Shutdown` 令牌响应停机                                                   |
| 资源回收 | 经 SDK 注册的命令与事件订阅由宿主自动清理;别把自己的类型塞进宿主静态字段/长命事件,否则 ALC 无法回收 |

---

## 3. `IPluginContext` 能力一览

`IPluginContext` 是插件访问宿主的唯一入口。**全部接口都是传输无关的**(只有异步方法、
DTO 与不透明 id),所以同一份插件源码在 `inProcess` 与 `isolated` 两种模式下都能跑。

### 3.1 身份与基础设施

| 成员                         | 说明                                                                                                                        |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `PluginId` / `PluginVersion` | 来自 `plugin.json`                                                                                                          |
| `DataDirectory`              | 插件私有目录(已创建)。**一切本地写入都应限于此**,卸载时整体删除                                                             |
| `Host` (`IHostInfo`)         | 宿主版本、apiLevel、当前语言。`Host.Theme` 只有 `dark`/`light`/`system` 三个粗粒度值,主题的实际身份与配色看 §3.5 的 `Theme` |
| `Log` (`IPluginLogger`)      | `Debug/Info/Warn/Error`,写进宿主日志管道(带插件 id 前缀),同时落到这个插件自己的日志文件 `~/.velashell/logs/plugins/<id>.log`,用户在插件管理页「查看日志」就能看到 |
| `Shutdown`                   | 停机令牌:触发后能力调用可能开始失败,应尽快收尾                                                                              |

### 3.2 数据

| 能力                            | 关键方法                                                                                                                                | 说明                                                             |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `Storage` (`IPluginStorage`)    | `GetAsync<T>` / `SetAsync<T>` / `RemoveAsync` / `GetKeysAsync`                                                                          | 按插件 id 命名空间化的 KV,落 SonnetDB(headless 时退回 JSON 文件) |
| `Secrets` (`ISecretsApi`)       | `GetAsync` / `SetAsync` / `DeleteAsync`                                                                                                 | 加密落盘的私有键值。**没有明文兜底**:后端缺席时直接报不可用      |
| `TimeSeries` (`ITimeSeriesApi`) | `OpenAsync` / `ListAsync` / `DropAsync`;series 上 `WriteAsync` / `QueryAsync` / `CountAsync` / `DistinctTagValuesAsync` / `DeleteAsync` | 插件私有的嵌入式时序库(按时间追加 + 按标签检索)                  |

### 3.3 会话与远程

| 能力                                | 关键方法                                                                         | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Sessions` (`ISessionsApi`)         | `ListAsync` / `GetAsync`;`ListSavedAsync` / `OpenAsync` / `CloseAsync`(2.0.2 起) | 枚举当前 SSH 会话,**脱敏,不含任何凭据**。后三个是**请求宿主打开一条已保存的会话** —— 只能开已保存的配置(连哪些机器由用户先定)、凭据一个字节都不经过插件、宿主可以拒绝(`PluginPermissionDeniedException`),`SessionOpenOptions.Reason` 会原样显示给用户。连不上另给 `PluginSessionOpenException`:"不让你连"与"没连上"处置不同,不该混成一个类型。`CloseAsync` **只关得掉本插件经 `OpenAsync` 开的那些**(含被复用的现成会话在内,别人的一律拒绝) |
| `RemoteFs` (`IRemoteFsApi`)         | 目录/属性/读写/传输/重命名/删除                                                  | 基于既有会话的 SFTP                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `RemoteExec` (`IRemoteExecApi`)     | `RunAsync`(整段结果)/ `StreamAsync`(按行回调)                                    | 独立通道,**不进用户终端**                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `RemoteTunnel` (`IRemoteTunnelApi`) | `OpenUnixSocketAsync` / `OpenTcpAsync`                                           | 到远端端点的**裸字节双工流**(Docker Engine API、tar 流这类二进制协议)。仅 `inProcess`                                                                                                                                                                                                                                                                                                                                                       |
| `Terminal` (`ITerminalApi`)         | `GetOutputAsync` / `SearchOutputAsync` / `WriteAsync`                            | 读取/搜索会话输出;**回写输入需要用户授权**(管理页可撤销)                                                                                                                                                                                                                                                                                                                                                                                    |

### 3.3a sudo 专用执行（SDK 2.1.0）

专用契约已随 **SDK 2.1.0 正式发布**，由 NuGet.org 恢复；宿主 `src/tests` 中央包与 AI 显式引用已升级为 2.1.0，AI 清单要求 `minSdkVersion: 2.1.0`（`apiLevel: 2`），防止旧宿主误装。没有使用本地包、临时包源或跨仓工程引用。顶部既有全局横幅仍由发版脚本维护，本节单列新增契约所需版本。

`IRemoteExecApi.StreamSudoAsync(string sessionId, string command, SudoExecOptions? options, IProgress<SudoExecEvent> progress, CancellationToken cancellationToken = default)` 返回 `Task<SudoExecResult>`。

- 仅 Linux sudo，经独立 PTY exec 将**整条命令以 root 执行**，包括管道和重定向；传实际命令，不带 sudo 包装。不是 `su`、任意 stdin 或用户终端凭据自动填充，也不改变旧终端凭据交互决定。
- `SudoExecOptions` 只有可空 `Timeout`（执行死线，null 不限时）；没有密码、来源身份、stdin 或环境变量字段。来源插件和真实目标由宿主确定。即使 NOPASSWD/缓存有效，宿主仍逐次可信确认；没有宿主 UI、PTY/认证方式不支持时明确失败，不降级普通执行。
- `SudoExecEvent.Kind` 为 `State` 或 `Output`。State 是 `AwaitingApproval` / `AwaitingAuthentication` / `Running`；Output.Text 是保留换行和 CR 的 PTY **合并文本片段**，不保证行边界，不能当作独立 stderr。认证内容、口令与随机标记不交给插件。
- `SudoExecResult` 含 `Outcome`、可空 `ExitCode`、`OutputChunks`（只计 Output 事件）。Outcome 为 `Unknown = 0`、`Exited`、`AuthenticationCancelled`（含拒绝审批）、`AuthenticationFailed`、`NotSupported`。后三种表示用户命令未启动；非零退出仍是 `Exited`，不是认证失败。缺失退出码的 `Exited` 按 Unknown 处理，不能默认补 0；`IsSuccess` 仅在 `Exited` 且确认退出码为 0 时为真。Unknown 不得自动重跑。
- 调用方取消抛 `OperationCanceledException`，执行死线到期抛 `TimeoutException`；请求取消不等于已确认远端进程停止。状态/输出同步有序回调且返回前排空，接收器须快速直接转发，不使用会异步 Post 的 `Progress<T>`。普通流和 sudo 流共享每插件 4 条流上限。
- 默认接口实现以及未配置 `FakeRemoteExec.SudoHandler` 都明确抛 `NotSupportedException`，不调用普通执行；替身用 `SudoStreamed` 单独记录请求。

隔离进程的新握手能力、真实取消及事件完整性见 [IPC §7.1](../plugins/05-ipc-protocol.md#71-执行操作sdk-210)；AI 任务/审批与恢复边界见 [自动化与 AI §1.4](../plugins/11-automation-and-ai.md#14-ai-插件的托管执行与恢复sdk-210)。

### 3.3b 独立上传任务（SDK 2.1.1）

**独立上传任务契约已随 SDK 2.1.1 正式发布到 NuGet.org。** 宿主中央依赖、测试替身及 AI 插件显式引用统一使用 2.1.1，AI 清单要求 `minSdkVersion: 2.1.1`（`apiLevel: 2`）；不使用本地包或临时包源。契约发布不等于所有宿主具备执行器，调用方仍须检查实际能力并遵守下面的任务生命周期。

`IRemoteFsApi.SupportsTransferOperations` 默认 `false`；新方法的默认实现抛 `NotSupportedException`，不退回长时间的 `UploadFileAsync`。现有接口和 RPC DTO 的构造/解构签名不变，apiLevel 不变。能力开关仅适用于新契约程序集已加载的宿主；新插件仍需声明正式新版本的最低 SDK 依赖，不能假定旧 SDK 程序集能加载新类型。

| 方法 | 语义 |
| --- | --- |
| `StartUploadAsync(sessionId, localPath, remotePath, RemoteUploadOptions, ct)` | 返回 `RemoteTransferSnapshot`，不等待传完；以宿主绑定的插件身份和 `OperationId` 幂等去重，相同 id 参数变化则拒绝 |
| `GetTransferAsync(transferId, ct)` | 读取所属插件且当前有权访问的任务快照，不泄漏越权 id 的信息 |
| `WaitTransferAsync(transferId, afterSequence, waitTimeout, ct)` | 等 `Sequence > afterSequence` 或终态；`afterSequence >= 0`，等待窗口 0–30 秒（`MaxTransferWaitSeconds`），到期返回当前快照 |
| `CancelTransferAsync(transferId, expectedGeneration, ct)` | 幂等取消指定执行代次；不匹配当前 `Generation` 则拒绝，旧取消不能停止恢复后的新代次 |
| `ResumeTransferAsync(transferId, RemoteTransferResumeOptions, ct)` | 显式、重新审批后恢复；选项为非空 `OperationId`、正数 `ExpectedGeneration` 和非空 `TargetSessionId`，不能指定路径/偏移/源文件身份 |

`RemoteUploadOptions` 只有必填非空 `OperationId`、`Overwrite`（默认否）及可空正数 `Timeout`（null 无执行死线）。无固定文件大小上限；64 位字节计数不受 32 MiB 或 `int.MaxValue` 限制，但不消除磁盘、权限、协议和网络边界。实现必须按流传输，不整体载入内存。宿主用不可预测的临时文件，校验和远端收尾成功后原子提交；无法实现所请求的原子提交/覆盖时明确失败，不能先删除目标。当前宿主的临时文件是目标同目录下的短名 `.vela-up-<16 位十六进制>`（接在目标名后面会让长文件名超出 `NAME_MAX`）。开始写之前做一次提交预检：要覆盖而服务端没有 `posix-rename`、不覆盖而目标已存在时直接失败，`ErrorCode` 分别为 `not-supported` 与 `commit-rejected`，与提交时被拒同一口径；预检查不出来不拦，之后才出现的目标照样由提交拒绝，提交仍是唯一权威。覆盖上传的临时文件一建出来就按被替换普通文件的 0777 权限位新建（服务端照常减 umask；目标是个条目却读不出模式时按 0600；读不到目标就按服务端默认，提交时照旧把关），写入期间不比原文件宽；setuid / setgid / 粘滞位与属主仍由提交补上。覆盖提交把被替换普通文件的权限位（07777）带到新文件上：属主尽力而为，setuid / setgid 只在属主属组对上之后才带；读不到目标模式，或改不了临时文件模式而它比目标宽时拒绝提交，目标原样不动（只许收窄，不许放宽）。

`RemoteTransferSnapshot` 包含任务/启动标识、会话与源/目标路径、`State`、从 1 开始的单调 `Sequence`、`TransferredBytes`/`TotalBytes`（未知 -1、空文件 0）、可空速率/ETA、创建/更新/完成时间、取消意图、远端停止确认、恢复检查点及错误类别/信息。`State` 默认 `Unknown = 0`；其余状态为 `Queued`、`Starting`、`Running`、`CancelRequested`、`Succeeded`、`Failed`、`Cancelled`、`TimedOut`。字节满格时可能还在 `Running`，不能据此宣称完成；`IsSuccess` 仅对应 `Succeeded`。`Unknown`/`TimedOut` 属于本地终态，不证明远端写入停止，不得自动重传。当前宿主在停止（取消、到期、断线、停用）之后最多等 30 秒让后端停下，再定终态；断线记 `Failed` / `session-disconnected`（确认停写、目标未写入，重连后重试安全），与 `Cancelled` 区分；Linux / macOS 源快照所在卷空间不够时为 `Failed` / `insufficient-space`。

- 启动、查询、等待、取消和恢复的令牌都只控制**本次请求**；等待到期/请求取消不停止已启动的任务，显式取消意图一经登记不因请求取消而回滚。启动回执丢失时复用同一 `OperationId`，不能另造 id 重传。
- 任务由宿主持有、归属宿主确认的插件身份，所有访问复核授权。插件侧 AI run/对话隔离仍须另行实施，不能把插件归属当成对话隔离。插件停用、授权撤销、SSH 断线和宿主退出须收尾；无停止确认则为 `Unknown`。只保证当前应用/插件生命周期，不承诺应用重启、插件重新激活或 MCP 会话销毁后仍可恢复。
- 当前宿主逐组件解析本地源的父目录别名（如 macOS `/var`），末级文件链接仍拒绝（Windows 上只有名字代理类重解析点 —— 符号链接、目录联接、挂载点 —— 算链接，OneDrive 云文件占位符这类过滤器标记不算）；打开后按文件身份（Windows 卷序列号 + 文件 id，Unix dev + inode）核验实际对象，核验那一端不经链接走到；映射盘、subst、硬链接名、大小写 / 规范化变体视为同一文件；审批与快照显示解析后的路径。幂等回执保留原请求路径，别名后来改向或源路径消失时，同 OperationId 重取不会重新打开文件。路径检查与快照校验不等于对恶意并发写入的原子文件系统快照保证。
- 首次审批绑定规范化源/目标路径、实际源对象和覆盖策略；宿主检查本地读取与远端写入权限，排队开始前及整个传输期间保持源稳定，防止路径替换、符号链接与并发写入改变已批准内容，无法保证即失败。`Overwrite=false` 必须使用原子无替换提交，不能只在上传前检查存在性。
- 启动/恢复的 `OperationId` 共用插件生命周期命名空间，记录请求种类和全部参数；操作串行化，先复核授权与去重记录再检查状态，重复请求返回当前快照但不执行。回执/墓碑保留整个生命周期，额度满拒绝新增，不遗忘 id 后重执行；参数冲突抛 `InvalidOperationException`，不存在或不属于本插件的 id 统一抛 `FileNotFoundException`。
- 快照增加 `Generation`（从 1 开始，每次恢复加一）及 `DestinationCommitConfirmedAbsent`（默认 false）。`CanResume` 还要求确认最终目标未提交；提交回执丢失未消歧时绝不可恢复。重新验证源对象、临时文件归属和检查点；重连通过明确的 `TargetSessionId` 选择连接，宿主核对服务器密钥身份、账户和文件命名空间，仅地址相同不足以恢复。恢复保持 `transferId` 和递增 `Sequence`，清除取消意图与完成时刻，不满足条件拒绝而不从头重传。
- `Timeout` 从每执行代次登记起算，含排队/收尾；显式恢复经重新审批后以原时长重新获得预算，不继承已耗尽死线。null 仍表示无执行死线。
- RPC 新增 `fs/transfer/startUpload|get|wait|cancel|resume|progress`，载荷为 `FsTransferStartUploadRequest`、`FsTransferRef`、`FsTransferWaitRequest`、`FsTransferCancelRequest`、`FsTransferResumeRequest`、`FsTransferProgressNotification`。握手的 `SupportsTransferOperations` 默认 false，旧宿主不能被推断为支持。`progress` 携带快照，按 `TransferId + Sequence` 丢弃旧帧；至少 100ms 节流，状态变化及时发送，通知丢失用 get/wait 补读，通知不具备 exec 专用严格发送保证。
- `FakeRemoteFs` 显式配置能力开关和 start/get/wait/cancel/resume 的 Handler，记录独立请求，转发参数及请求令牌；没有 Handler 就明确不支持，不退回现有整文件内存上传。幂等、权限及真正的任务执行由宿主实现，测试脚本须显式模拟，不由替身伪造成功。

当前宿主在新上传 API 中通过独立单任务租约实施可信确认：读取内容前展示插件、真实会话、本地源、**文件大小**、远端路径和覆盖策略，只批准当前任务，不继承终端/开会话许可；AI `Bypass` 不影响宿主确认。缺少授权闸、提示器或桌面窗口时拒绝。插件管理页撤销同步使待批准及活动租约失效并请求取消，提交前重验，查询/后续进度不再泄漏被撤销的元数据；不承诺回滚已经提交的文件。这不扩展到旧同步文件 API，也不代表通用权限 Broker 已完成。

此契约不改变现有同步上传/下载、AI 对话附件或 IM 平台发文件上限。宿主与 AI 的接线使用独立任务提供可见进度、模型任务查询及脱离长请求的上传；外部 MCP Agent 仍须主动查询，不能假定有服务端模型唤醒。当前宿主连接边界不能证明重连后的服务器密钥、账户与远端命名空间一致，因此 `ResumeTransferAsync` 明确报不支持、`CanResume=false`；接口存在不代表恢复已实现，`Unknown` 不自动重传。

### 3.3c 独立下载任务（SDK 2.1.2）

**独立下载任务契约已随 SDK 2.1.2 正式发布到 NuGet.org。** 宿主 `src/Directory.Packages.props` 与 `tests/Directory.Packages.props` 的中央依赖、测试替身及 AI 插件显式引用统一使用 2.1.2，AI 清单要求 `minSdkVersion: 2.1.2`（`apiLevel: 2`）；不使用本地包或临时包源。契约发布不等于所有宿主具备执行器，调用方仍须检查 `SupportsDownloadOperations`。

- `IRemoteFsApi.SupportsDownloadOperations` 和 `HandshakeResponse.SupportsDownloadOperations` 独立于上传能力，默认 `false`；不支持即明确拒绝，不降级旧同步 `DownloadFileAsync`。
- `StartDownloadAsync(sessionId, remotePath, localPath, RemoteDownloadOptions, ct)` 返回后台任务快照。`RemoteDownloadOptions` 含必填 `OperationId`、`Overwrite=false`、`Timeout=null`（无默认执行死线）；请求令牌只取消本次回执等待，不取消已登记下载。内容按流处理，没有 AI 固定大小上限，继续服从磁盘、权限、用户下载带宽与共享资源预算。
- 复用 `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` 的所有权、序号和执行代次语义；下载 `CanResume=false`，无断点续传或跨重启恢复。`LocalPath` 始终为最终本地目标，`RemotePath` 为远端源，不暴露临时路径。幂等签名包含方向和全部参数，同 id 换方向/异参拒绝；丢失回执复用原 id，不自动重下未知结果。
- 快照附加 `Direction`：`RemoteTransferDirection.Upload=0`、`Download=1`；`Phase`：`RemoteTransferPhase.Unknown=0`、`Approval`、`Transfer`、`Verification`、`Commit`。缺失字段兼容默认 Upload/Unknown，既有构造/解构不变。当前宿主的**上传**快照同样报 `Phase`（2026-10-25 起；此前上传一律 `Unknown`）：回执与等宿主确认时 `Approval`，批准后 `Transfer`，回读 / 摘要校验 `Verification`，提交 `Commit` —— 插件据此只撤「还没批准」的任务；SDK 契约不变，只是宿主开始填这个字段。64 位字节计数、速率、ETA 与 `RemoteStopConfirmed` 描述进度；满字节不等于成功，刷新、关闭、校验和提交确认后才是 `Succeeded`。取消不合作的后端可能先报告本地 `Unknown`/`TimedOut` 且 `StopConfirmed=false`，实际退出前保留槽和句柄。
- RPC 增加 `PluginRpc.FsTransferStartDownload="fs/transfer/startDownload"` 和 `FsTransferStartDownloadRequest`；继续以 get/wait 为权威观察，不新增一套下载推送承诺。`FakeRemoteFs` 分别配置下载能力/处理器并记录启动请求，缺处理器明确不支持。
- 上传下载共用每插件 1024 生命周期回执、32 活动/排队任务、4 实际传输槽，不因增加下载把总体并发翻倍。旧同步 API、模型上下文读取、聊天附件和 IM 平台附件限制不变。

当前宿主下载使用独立单任务许可，检查远端读/本地写并逐次展示插件、真实会话、源、最终目标及覆盖策略；不借用上传、终端或开会话许可，AI Bypass/保存选择器不能代替可信宿主审批，无窗口/提示器则拒绝。撤权使待审批及活动租约失效并请求取消，提交前重验，后续读取脱敏，取消清理仍可进行；不承诺回滚已提交目标。

**当前安全提交支持 Windows 固定本地 NTFS 与 Linux / macOS（x64、arm64）的本地文件系统。** 两边都在审批之前钉住目标目录链，建临时文件、提交与清理都相对钉住的父目录进行。Windows 逐段持有带遍历 + 读属性权限的祖先目录句柄，共享读写、不共享删除（只读属性的句柄不参与共享检查，带上遍历权限才算数），祖先改不了名、挪不走；只拒名字代理类重解析点（符号链接、目录联接、挂载点），云文件占位符等过滤器标记不算链接；在钉住的父目录里独占 `CreateNew` 随机临时文件并按句柄核验其位置，提交经 `NtSetInformationFile` 相对父目录句柄、只给末级名字。Linux / macOS 先把父目录解析成真实路径并写进快照的 `LocalPath`，再从 `/` 起逐级 `openat(O_DIRECTORY | O_NOFOLLOW)`、只留父目录 fd，链上任何一级是符号链接即拒绝；只收本地、按 inode 寻址的文件系统（Linux 按 `statfs.f_type` 白名单，macOS 看 `MNT_LOCAL`），9p / SMB / FUSE / NFS 在审批前拒绝；临时文件 `O_CREAT | O_EXCL | O_NOFOLLOW` 独占新建并持有 flock，无替换提交用 `renameat2(RENAME_NOREPLACE)` / `renameatx_np(RENAME_EXCL)`，标志不被支持时退到 `linkat` + `unlinkat`，覆盖用 `renameat`；新建文件权限按 umask 收窄，覆盖时沿用旧文件的权限位（不带 setuid / setgid / sticky）。默认原子无替换，目标中途出现也不覆盖；只有明确批准覆盖才替换，不先删最终目标。打开目标时清扫同目录崩溃遗留的 `.vela-download-<48 位十六进制>`，只删名字逐字对得上且没有任务在用的，同形链接只删链接本身。UNC、非 NTFS/非固定盘、其它文件系统及不支持后端明确拒绝，`ErrorCode` 为 `not-supported`，不退回直接写最终路径。用户最终路径不交给旧同步下载；时间戳偏好应用于任务持有的临时句柄。

检查实际长度、传输前后远端元数据，并使用可用的摘要能力；不强制纯 SFTP 账号提供 exec，也不默认第二遍整文件下载。检测源变化则失败，但这些检查**不是远端原子快照保证**。提交用例在 Windows 与 Linux（WSL Debian 的 ext4 与 tmpfs）上实际跑过，macOS 只经 CI；这不是真实 SSH/GUI 整链验证。AI 生命周期与终态通知见 [自动化与 AI §1.7](../plugins/11-automation-and-ai.md#17-ai-双入口后台下载sdk-212)。

### 3.3d 插件协议的文件能力补口（SDK 2.1.3）

**契约已随 SDK 2.1.3 正式发布到 NuGet.org，宿主已接上。** 宿主把中央依赖抬到 2.1.4，并在 `PluginProtocolFileService`（链接字段、改修改时间、摘要）与 `RemoteFsCapability.Map` 中使用这些能力。协议插件用到要声明 `minSdkVersion: 2.1.3`。全部只增不改，`apiLevel` 不变。

- `RemoteFileEntry` 新增两个 `init` 属性（八参构造与 `Deconstruct` 不变）：`IsSymbolicLink` 按 lstat 判断；`IsDirectory`、`Size`、`LastModified` 描述链接**目标**，断链保留链接自身的属性；`LinkTarget` 是 readlink 的原文，可以是相对路径。隔离进程的 `fs/list`、`fs/stat` 现有 JSON 自动带上这两个字段，旧数据缺字段时读成「不是链接」。
- `IProtocolLastWriteTime`（可选，与 `IProtocolStreamUpload` 一样由协议实现类兼实现）：`SetLastWriteTimeAsync(sessionId, path, DateTimeOffset lastWriteTime, ct)`。
- `IProtocolChecksum`（可选，兼实现）：`ComputeChecksumsAsync(sessionId, RemoteChecksumAlgorithm algorithm, IReadOnlyList<string> paths, ct)` 返回 `IReadOnlyDictionary<string, string?>` —— 键就是请求里的路径，值是小写十六进制，单个文件算不出时为 null。**只能用服务器自己给出的摘要**，不许为了凑一个值把整个文件读回来算。`RemoteChecksumAlgorithm` 为 `Unknown = 0`、`Sha256`、`Sha512`、`Sha1`、`Md5`。
- 「不支持」怎么报：协议不实现这两个接口，宿主就如实报不支持；实现了但这条会话做不到（服务器没有 MFMT 或 HASH）时抛 `ProtocolUnsupportedException`。这两个接口只给 `inProcess` 的协议用（协议本来就只支持 `inProcess`），没有新增 RPC。
- 范围外：插件协议上**创建**符号链接仍然没有契约，宿主的 `CreateSymbolicLinkAsync` 照旧报不支持。

### 3.3e 传输启动前被拒（SDK 2.1.3）

**契约已随 SDK 2.1.3 正式发布到 NuGet.org，宿主已接上。** 宿主实现了登记前的明确拒绝：源不存在或所在目录不存在时交 `SourceNotFound`，源被占用时交 `SourceInUse`，活动队列或回执额度满时交 `QuotaExceeded`；任务尚未登记、不会生成 transferId，`OperationId` 仍可重用。

- `PluginTransferRejectedException`（派生自 `InvalidOperationException`，刻意不是 `IOException`；sealed）带 `Reason`（`RemoteTransferRejectReason`：`Unknown = 0`、`SourceNotFound`、`SourceInUse`、`QuotaExceeded`、`InsufficientSpace`）。RPC 错误码 `RpcErrorCodes.TransferRejected = "transfer-rejected"`，线上格式 `transfer-rejected:source-in-use`，不认识或缺失的后缀还原成 `Unknown`；`FromException` 里排在 `InvalidOperationException` 之前。
- 宿主抛它就等于保证：任务没有登记、没有生成 transferId、没有读写任何内容、OperationId 也没有记进账本。条件变了之后可以再试（同一个 OperationId 不会撞上「参数不符」），但不要紧接着自动重试。
- 与另外两种结局分开处理：「结果未知」只能用同一个 OperationId 去重取回执；「用户拒绝」（`PluginPermissionDeniedException`）重试没有意义。参数非法、平台不支持、会话不在、OperationId 参数冲突照旧用原来的异常类型。`IRemoteFsApi.StartUploadAsync` / `StartDownloadAsync` 补了 `<exception>` 说明。
- 当前宿主的空间不足发生在审批后的上传源快照阶段，仍返回已登记任务的 `Failed` / `insufficient-space` 快照（§3.3b），不是登记前拒绝。`SourceInUse` 的识别限于 Windows 共享/锁冲突。

### 3.4 界面与扩展点

| 能力                                | 关键方法                                      | 说明                                                                                                                |
| ----------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `Commands` (`ICommandsApi`)         | `Register` / `TryExecute`                     | 命令 id 必须以插件 id 为前缀;清单里声明的占位命令在激活时被真实处理器替换                                           |
| `Ui` (`IUiApi`)                     | `ShowPanelAsync(options, contentFactory)`     | 呈现插件自己创建的 Avalonia 控件:`inProcess` 可停靠成主窗口标签页,`isolated` 为独立卡片窗口                         |
| `TerminalView` (`ITerminalViewApi`) | `Create(...)`                                 | **出借宿主的终端仿真器**(VT 解析、屏幕缓冲、选区、IME、键盘编码),插件拿到一个可嵌进自己界面的真终端。仅 `inProcess` |
| `Protocols` (`IProtocolsApi`)       | 注册协议实现                                  | 插件自带的远程**文件**协议,与 SSH/SFTP/FTP 同为连接配置页的一等公民。仅 `inProcess`                                 |
| `Workspaces` (`IWorkspacesApi`)     | 注册工作区提供者                              | **非文件型**连接类型(Redis、MySQL…),由插件全权渲染会话文档。仅 `inProcess`                                          |
| `Clipboard` (`IClipboardApi`)       | 文本读写                                      | 系统剪贴板                                                                                                          |
| `Events` (`IHostEvents`)            | 会话连接/断开、主题与语言切换                 | 订阅由宿主在停用时自动清理                                                                                          |
| `Theme` (`IHostThemeApi`)           | `Current` / `Colors` / `GetColor` / `Changed` | 主题身份 + 整套已解析的 `Vela*` 配色 + 覆盖全部换肤情形的信号。见 §3.5                                              |

宿主为第一方 `velashell.ai` 的 `DisplayMode.Document` 提供专属右侧区域：宽度与固定/自动隐藏由宿主管理，不使用 `PlacementRatio` 或发出 `PlacementRatioChanged`。隐藏、折叠不关闭句柄，`ActivateAsync` 恢复同一控件。普通插件文档分栏的宽度契约与 SDK 包保持不变；完整宿主策略见 [UI 扩展](../plugins/08-ui-extensions.md)。

> **`inProcess` 专属的四项**(`RemoteTunnel` / `TerminalView` / `Protocols` / `Workspaces`)
> 在隔离模式下调用会抛"能力不可用"。它们交出去的都是活的原生对象或裸流,
> 跨进程边界没有等价物 —— 需要它们就把 `hostMode` 设成 `inProcess`。

### 3.5 主题(SDK 2.0)

**界面取色不需要这个接口。** Avalonia 里一律写 `{DynamicResource VelaXxx}` ——
令牌由宿主逐主题派生并整套换新,进程内与隔离模式都自动跟随。

需要它的是 DynamicResource 到不了的三种地方:

1. 要 `Color` 而不是 `Brush`(语法高亮定义、自绘、导出图片);
2. 代码里一次性取色的地方(转换器、控件模板之外的计算)—— 它们需要一个"该重取了"的信号;
3. 要按**主题身份**分支(给某套主题换一张插画、一组图标)。

```csharp
HostThemeInfo theme = ctx.Theme.Current;
// theme.Id        "tokyo-night"   —— 已解析,永远不是 "system"
// theme.Name      "Tokyo Night"
// theme.IsDark    true
// theme.Variant   "dark"
// theme.FollowsSystem  用户选的是不是"跟随系统"
// theme.Accent    "#7AA2F7"       —— 含用户的自定义强调色覆盖

string? bg = ctx.Theme.GetColor("VelaBgTerminal");   // "#FF1A1B26"

ctx.Theme.Changed += info => { /* Current 与 Colors 都已是新值 */ };
```

> **别用 `Host.Theme` 判断换肤。** `IHostInfo.Theme` 只有 `dark`/`light`/`system` 三个值:
> 宿主内置十几套具名主题,它们在那里全被收敛成自己的明暗名。于是 VelaDark → Tokyo Night
> 这种换肤,`Host.Theme` 与 `IHostEvents.ThemeChanged` 的**参数都不会变化**,
> 按参数判断"变没变"会整整漏掉一次换肤;`Theme == "system"` 时它也没告诉你此刻是明是暗。
> `IHostThemeApi.Changed` 是三种情形的并集 —— 换具名主题、跟随系统翻转、改强调色。

### 3.6 标签页图标(`PluginIcon`,SDK 2.0.4)

宿主的会话标签页按连接类型画图标(SSH 一个字形、SFTP / FTP 一个字形,本地终端不画;口径见[交互与界面规格 §4B.2](../host/交互与界面规格.md#4b2-单个标签页tab内容))。
**插件的标签默认是一个通用插头** —— 宿主不认识你是 Redis 还是别的什么,也不该认识:
宿主里维护一张「插件 id → 图标」的对照表,第三方插件永远进不去。

所以图标由插件自己交出来。**三处填的都是同一个类型** `PluginIcon`:

| 填在哪 | 画的是 | 什么时候给 |
| --- | --- | --- |
| `ProtocolDescriptor.Icon` | 该协议的会话标签 | 注册期,一个协议一份 |
| `WorkspaceDescriptor.Icon` | 该工作台的会话标签 | 注册期,一个工作台一份 |
| `PanelOptions.Icon` | 该面板的标签**以及窗口模式下的标题栏** | 每次开面板一份 |

最后一处是「每次一份」而不是「一个插件一份」:同一个插件开的聊天页与设置页可以各画各的。

`PluginIcon` 只有三个字段(`init`,record 类型):

| 字段 | 类型 / 默认 | 含义 |
| --- | --- | --- |
| `PathData` | `string`,**必填** | SVG 路径数据,多段用空格连写即可。宿主解析失败时当作没给 |
| `ViewBoxSize` | `double`,默认 `24` | `PathData` 的原生视框边长(lucide 那套恒为 24)。宿主收到非正数或非有限值时按 24 处理 |
| `IsFilled` | `bool`,默认 `false` | 是否为实心填充图形;`false` 按描边渲染 |

宿主画的颜色不由插件定:会话标签(协议 / 工作台)上描边与填充都用这台连接的标识色(与标签上的色条同色),
面板标签用强调色。用户在连接配置里自己挑了标签页图标时,它压过插件自报的这一枚。

两个便捷构造覆盖了全部真实用法:

```csharp
// lucide 那套描边字形:抄路径即可,视框恒为 24
Icon = PluginIcon.Stroked("M6 12h12 M6 8h12a4 4 0 0 1 0 8H6a4 4 0 0 1 0-8Z")

// 品牌 logo:实心,而且视框通常不是 24 —— 两件都要说清楚
Icon = PluginIcon.Filled(RedisLogoPathData, viewBoxSize: 1030)
```

三条容易踩的:

1. **视框不报就是 24。** 品牌 logo 导出的视框常是 1024 之类,不报出来宿主会按 24 缩放,
   **把它放大四十多倍** —— 「图标没显示」这类现象,原因通常就在这一条。
2. **描边与填充选错了不会报错,只会看不出是什么。** 用描边去画实心图形得到的是它的轮廓线;
   反过来把描边字形填起来则是一堆色块。
3. **路径畸形不会把界面顶掉。** 宿主解析失败时当作没给,退回通用插头。

> ⚠️ 与 `PanelTitleAction.IconPathData` **不是一回事**。那个画的是**窗口标题栏**上的动作按钮,
> 恒为 24×24 描边,没有填充与视框可言(见[开发指南 §5.9](../templates/dev-guide.md))。

### 3.7 往消息中心发通知（SDK 2.1.3）

**契约已随 SDK 2.1.3 正式发布到 NuGet.org，宿主已实现。** 宿主的进程内 `PluginUiApi` 与隔离模式 `ui/notify` 路由都经过插件身份校验、去重、限频和按插件静音；消息带插件来源，并在卸载时清理。插件用到要声明 `minSdkVersion: 2.1.3`，调用前照样先看 `SupportsNotifications`。

```csharp
if (ctx.Ui.SupportsNotifications)
{
    PluginNotificationOutcome outcome = await ctx.Ui.PostNotificationAsync(new PluginNotification
    {
        Id = "backup-failed:web-01",          // 本插件内稳定的去重键
        Title = "备份失败",
        Body = "web-01 的夜间备份没有完成。",
        Level = PluginNotificationLevel.Warning,
        ActionCommandId = "acme.backup.open", // 只能是本插件的命令
        ActionLabel = "查看",
    });
}
```

- `PluginNotification`（record）：`Id` 必填，本插件内稳定的去重键，1–128 字符、不能有控制字符；`Title` 必填、纯文本，宿主超过 200 字截断；`Body` 纯文本，超过 1000 字截断；`Level`（`Info` / `Warning` / `Critical`，与宿主的轻重一一对应）；
  `ActionCommandId` 只能是本插件的命令（以 `"<pluginId>."` 开头），`ActionLabel` 有命令才能给；`Validate(pluginId)` 是宿主、PluginHost 与测试替身共用的同一套校验。**故意没有网址字段，也没有「发送方」字段**。
- `IUiApi.SupportsNotifications`（默认 false）与 `PostNotificationAsync(notification, ct)`（默认实现抛 `NotSupportedException`）都有默认实现，现有实现照旧能编译。返回 `PluginNotificationOutcome`：`Unknown = 0`、`Posted`、`AlreadyPosted`、`Muted`、`RateLimited`。
- 约定：消息中心只收**要留存、可回看**的消息，进度、运行时状态、操作回执、调试信息不该往这里发（[消息中心与资讯源](../host/消息中心与资讯源.md) §1）。先去重、后限频：同一个 id 重投返回 `AlreadyPosted`，不算频率；每个插件每分钟最多 6 条新消息（宿主可以收得更紧），超出返回 `RateLimited`、不抛异常；用户按插件静音时返回 `Muted`。消息跨重启保留，受消息中心总量上限约束；插件卸载时宿主删掉它发的消息；插件不能改写或撤回已发的消息。动作命令最好写进清单的 `contributes.commands`，插件没激活时点击也能惰性激活。
- 权限：不新增权限项 —— 普通权限 `ui.contributions` 本来就包含「通知（有频控）」（[06 §1](../plugins/06-permission-system.md)）；`notifications.system` 管的是操作系统通知，与这个无关。用户的控制手段在宿主侧：插件管理页的按插件静音。
- 隔离进程：RPC `ui/notify`（`UiNotifyRequest` / `UiNotifyResponse`），握手 `HandshakeResponse.SupportsNotifications` 缺席时为 false（[IPC §7.4](../plugins/05-ipc-protocol.md#74-通知启动前被拒与整帧写入sdk-213)）。
- 测试替身 `FakeUi`：`SupportsNotifications`（默认 true）、`NotificationRequests`（全部投递）、`PostedNotifications`（实际入列的）、`NotificationOutcome` 脚本（模拟限流与静音）、`PluginId`；校验与去重规则与宿主一致，`TestPluginContext` 把自己的 PluginId 同步给它。

### 3.8 版本比较 `SemanticVersion`（SDK 2.1.3）

**已随 SDK 2.1.3 正式发布到 NuGet.org。** 宿主已在更新检查、发布说明、插件装载版本闸、插件更新和资讯源版本定向中使用 `SemanticVersion`。CLI 与商店的迁移状态需分别核对，不由宿主的包升级推断。

- `VelaShell.PluginSdk.Versioning.SemanticVersion`（readonly struct）：`Major` / `Minor` / `Patch` / `Revision`（第四段，没有时为 0）、`PreRelease` / `BuildMetadata`（永不为 null）、`IsPreRelease`；`TryParse` / `Parse`（不合法抛 `FormatException`）。
- `CompareTo` 按完整的 SemVer 2.0 §11 优先级：`beta.2 < beta.10`，纯数字标识符按长度再按序数比、任意长度不溢出，纯数字低于含字母的，含字母的按 ASCII 序数、区分大小写，前面都相同时标识符多的更高；构建元数据保留但不参与比较（`Equals` / `GetHashCode` 同一口径）。
  `CompareCoreTo` 只比数字段，给 `minSdkVersion` / 最低宿主版本这类装载闸用；`TextComparer` 比较版本字符串，解析不出的当最旧、同优先级按原文序数。
- 解析时数字段沿用宿主与 `plugin.json` 的宽松写法（允许前缀 `v`、两段或四段、数字段前导零），后缀严格按 SemVer：标识符不能为空、只含 `[0-9A-Za-z-]`，预发布里的纯数字标识符不能有前导零。
  所以清单正则放行而它判为不合法的写法有 `1.0.0-beta.01`、`1.0.0-a..b`（要不要收紧清单校验另定）。

---

## 4. 清单(`plugin.json`)

字段全表见[开发指南 §3](../templates/dev-guide.md)。发布相关的三个版本闸见[打包与发布 §1.2](../templates/publishing.md)。
这里只强调最容易出事的三条:

1. `id` 发布后不可变(命令前缀 + 数据命名空间)。
2. 用了新 SDK 面就必须声明 `minSdkVersion`,否则老宿主上是运行期 `MissingMethodException`。
3. 声明式贡献点(`contributes.commands` / `protocols` / `workspaces`)在**发现期**就生效,
   不装载任何程序集 —— 这是"启动零开销 + 惰性激活"的基础,别把它们留空然后指望激活后再补。

---

## 5. SDK 版本历史

`apiLevel` 只在**破坏性**变更时才动;只增不改的新面靠 `minSdkVersion` 拦。
纪律是「SDK 主版本 == apiLevel」——1.x 系列全程 `apiLevel 1`,**2.0 起为 `apiLevel 2`**。

| SDK       | 新增                                                                                                                                                                                                                                                                                                                | 插件需要声明 `minSdkVersion` 吗           |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 1.0       | 首版契约                                                                                                                                                                                                                                                                                                            | —                                         |
| 1.1       | `ExecResult` 加标准错误与退出码;远程执行的流式形态                                                                                                                                                                                                                                                                  | 用到就要                                  |
| 1.2       | `IRemoteTunnelApi`(裸字节双工流)                                                                                                                                                                                                                                                                                    | 用到就要(`1.2.0`)                         |
| 1.3       | `ITerminalViewApi`(出借宿主终端控件)                                                                                                                                                                                                                                                                                | 用到就要(`1.3.0`)                         |
| 1.3.1     | 工作区**变体**:`WorkspaceVariant`、`VariantKey`/`Variants`、`NoCredentials`/`NoEndpoint`                                                                                                                                                                                                                            | 用到就要(`1.3.1`)                         |
| 1.4       | `HostRegistry`(宿主自我登记,供 `vela-plugin` 定位安装与核对版本)                                                                                                                                                                                                                                                    | **不需要** —— 这是工具链面,插件代码不调用 |
| **1.5**   | 协议连接表单三件套:`ProtocolFeatures.NoEndpoint`(收起端口栏)、`ProtocolSettingKind.DynamicChoice` + `IProtocolChoiceSource`(候选项在表单打开时现取)、`AllowsCustomValue` / `HostKind` / `HostChoices` / `HostAllowsCustomValue`(可编辑下拉;主机栏也能做成下拉)。由串口插件驱动 —— 端口是热插拔设备,波特率有非标值。 | 用到就要(`1.5.0`)                         |
| **2.0**   | **代际变更(`apiLevel` 1 → 2)**,详见下方「2.0 迁移」。内容上带的是 `IHostThemeApi`(`ctx.Theme`,见 §3.5):主题身份、整套已解析配色、覆盖全部换肤情形的 `Changed`。在这之前插件只看得到 `IHostInfo.Theme` 那三个值,认不出具名主题与强调色,而且同明暗内部换肤时那个值根本不变。                                          | 不需要 —— 用 `apiLevel: 2` 表达即可       |
| **2.0.2** | 会话能力的「打开已保存的会话」三件套:`ISessionsApi.ListSavedAsync` / `OpenAsync` / `CloseAsync`,配套 `SavedSessionInfo`、`SessionOpenOptions` 与 `PluginSessionOpenException`。在这之前插件只能操作用户**已经手动连上**的机器,任何无人值守的用法都塌了半边。                                                        | 用到就要(`2.0.2`)                         |
| **2.0.3** | 标签页图标的第一版:`ProtocolDescriptor` / `WorkspaceDescriptor` 上各一组 `IconPathData` / `IconViewBoxSize` / `IconIsFilled`。⚠️ **这一版的三对字段已被 2.0.4 版的 `Icon` 取代,不要再用** —— 当时面板那条路还漏着,而同一个概念在两处各写三个平行属性本身就是设计错误。发布次日即改,零插件受影响。 | ❌ 已废弃,直接用 2.0.4 版                   |
| **2.0.4**   | 标签页图标收成一个入口 `PluginIcon`(见 §3.6),三处共用:`ProtocolDescriptor.Icon`、`WorkspaceDescriptor.Icon`,以及**新增的** `PanelOptions.Icon` —— 面板标签走的不是描述符那条路,2.0.3 漏了它。带 `PluginIcon.Stroked` / `PluginIcon.Filled` 两个便捷构造。 | 用到就要(`2.0.4`)                           |
| **2.0.6** | `IProtocolStreamUpload`:`IProtocolFileSystem` 可**选择**兼实现的接口 —— 把一条顺序读的流写成远端文件(覆盖)。宿主据此把文件在两个远程会话之间经内存中转,而不必先落一份临时文件(「远程 + 远程」双栏文件标签)。流归调用方;可能不可 Seek,长度单独给出;不续传。不实现完全合法:宿主就把这个协议当作不支持从流上传。只增不改,`apiLevel` 不变。 | 用到就要(`2.0.6`) |
| **2.1.0** | `StreamSudoAsync`、sudo 非敏感事件/结果与专用测试替身；执行操作握手、取消及有序完整通知。已正式发布，见 §3.3a。 | 用到就要（`2.1.0`）；AI 清单已声明 |
| **2.1.1** | 独立上传任务：`SupportsTransferOperations`、`StartUploadAsync` / `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` / `ResumeTransferAsync`、`RemoteTransferSnapshot` 与 `fs/transfer/*` RPC。已正式发布，见 §3.3b。 | 用到就要（`2.1.1`） |
| **2.1.2** | 独立下载任务：`SupportsDownloadOperations`、`StartDownloadAsync` / `RemoteDownloadOptions`，快照加 `Direction` / `Phase`，`fs/transfer/startDownload` RPC。已正式发布，见 §3.3c。 | 用到就要（`2.1.2`）；AI 清单已声明 |
| **2.1.3** | 插件协议的文件能力补口：`RemoteFileEntry.IsSymbolicLink` / `LinkTarget`、`IProtocolLastWriteTime`、`IProtocolChecksum` 与 `RemoteChecksumAlgorithm`（§3.3d）；传输启动前被拒 `PluginTransferRejectedException` 与 `transfer-rejected` 错误码（§3.3e）；往消息中心发通知 `IUiApi.PostNotificationAsync` 与 `ui/notify`（§3.7）；版本比较 `SemanticVersion`（§3.8）；`RpcConnection` 一帧要么整帧写完、要么不开始（[IPC §7.4](../plugins/05-ipc-protocol.md#74-通知启动前被拒与整帧写入sdk-213)）。已正式发布，宿主已接上；宿主当前中央依赖为 2.1.4。只增不改，`apiLevel` 不变。 | 用到就要（`2.1.3`）；整帧写入的修正不涉及插件代码，抬包即生效 |

### 2.0 迁移

2.0 是本系列**第一次**动 `apiLevel`。主版本跳变把契约程序集的 `AssemblyVersion` 从
`1.0.0.0` 带到 `2.0.0.0`(见 SDK 仓库 `src/Directory.Build.props` 的 `$(VelaSdkMajor).0.0.0`),
所以**已编译的插件必须重新编译**才能被 2.x 宿主装载。

插件作者要做两件事:

1. 把 `VelaShell.PluginSdk.Build` 抬到对应版本并重新编译、重新打 `.vpx`;
2. 把 `plugin.json` 的 `apiLevel` 改成 `2`。

第 2 条不做也能在 2.x 宿主上跑(宿主接受不高于自身的代际),但**装到 1.x 宿主上时**
你要的是发现期那句「Plugin targets apiLevel 2, host supports up to 1. Update VelaShell.」,
而不是一个看不懂的程序集绑定异常 —— 这正是 `apiLevel` 存在的理由。

新增的能力面本身仍是只增不改的:没有删除任何成员,没有改任何签名。唯一的源码级不兼容是
`IPluginContext` 多了 `Theme` 成员 —— 插件**消费**这个接口,不受影响;自己写了
`IPluginContext` 实现(多半是测试替身)的要补上,或者改用 `TestPluginContext`。

---

## 6. 测试:不启动宿主也能跑

`VelaShell.PluginSdk.Testing` 提供 `TestPluginContext` 与各能力的内存替身:

```csharp
using VelaShell.PluginSdk.Testing;

[TestMethod]
public async Task Activate_RegistersCommand()
{
    using var context = new TestPluginContext();
    var plugin = new DemoPlugin();

    await plugin.ActivateAsync(context, CancellationToken.None);

    Assert.Contains("acme.demo.run", context.RecordingCommands.Registered);
    await plugin.DeactivateAsync(CancellationToken.None);
}
```

可用替身:`CollectingLogger`、`InMemoryStorage`、`InMemoryTimeSeries`、`FakeSessions`、
`FakeRemoteFs`、`FakeRemoteExec`、`FakeRemoteTunnel`、`FakeTerminal`、`FakeTerminalViewApi`、
`FakeUi`（SDK 2.1.3 起另有通知成员，见 §3.7）、`FakeSecrets`、`FakeClipboard`、`RecordingCommands`、`RecordingProtocols`、
`RecordingWorkspaces`、`TestHostEvents`、`TestHostInfo`。

`FakeSessions` 除了 `AddConnected` 还有 `AddSaved`(造一条已保存配置)与三个钩子:
`DenyOpen` / `OpenFailure` 模拟"用户点了不"与"连不上",`LastOpenReason` 用来断言那句给用户看的
理由确实传上去了 —— 一个传了"插件需要连接"的实现功能上完全正常,却把确认框变成了盲点按钮。

单测覆盖不了的部分(真实 UI、真实会话、真实协议页签)走开发内环:
`vela-plugin dev init` → F5,见 [CLI 手册](../cli/cli.md)。

---

## 7. 装载模型:三条必须知道的规则

1. **每插件一个可收集 ALC**,插件自带依赖按其 `deps.json` 在插件目录内解析 ——
   你可以引任意 NuGet 包,与宿主的版本互不干扰。
2. **两类程序集强制共享**:`VelaShell.PluginSdk` 与 `Avalonia*` 一律回落到装载方那一份。
   所以插件的 Avalonia 版本必须与宿主一致(SDK 包已锁定),而且名字以 `Avalonia` 开头的
   第三方包用不了(会被当成共享程序集去宿主里找,找不到)。
3. **开发期插件从影子副本装载**(`~/.velashell/dev-shadow/<id>/gen-N`),因此宿主运行时
   不锁你的 `bin`,可以边跑边重编,改完在管理页点"重新加载"即可。生产路径不走影子拷贝。

---

## 8. 宿主注册表(`HostRegistry`,SDK 1.4)

`VelaShell.PluginSdk.Hosting.HostRegistry` 读写 `~/.velashell/host.json`:宿主每次启动登记
自己的可执行文件路径、版本、apiLevel、内置 SDK 版本、Avalonia 版本与数据根。

面向的是**工具链**(`vela-plugin dev init` / `doctor` / `hosts`),插件运行时用不到它。
如果你要写自己的构建脚本或 IDE 插件,可以这样拿到本机宿主:

```csharp
HostRegistryEntry? host = HostRegistry.Resolve();          // 最近启动过的那份
HostRegistryEntry? preview = HostRegistry.Resolve("1.5");  // 按版本挑
IReadOnlyList<HostRegistryEntry> all = HostRegistry.List();
```

所有读路径对文件缺失/损坏都返回空表而不是抛异常 —— 这个文件是加速用的缓存,
坏掉的后果应当是"让你手动指一下路径",而不是任何一侧起不来。
