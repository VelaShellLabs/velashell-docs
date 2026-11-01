# 07 · 能力 API(Capability APIs)

插件能做的一切都经由能力域;每个域对应:SDK 代理接口(插件侧)→ RPC
方法族(05)→ 宿主能力服务(权限 Demand → 桥接到既有子系统)。本文档
定义 apiLevel 1 的域清单与关键签名(C# 形式为 SDK 视角)。

设计纪律:

- 每个方法标注所需权限;无权限方法不存在于清单即不存在于协议。
- 全部异步、全部可取消;可能长跑的带 `IProgress<T>`。
- 路径、会话等标识使用宿主颁发的不透明 id,插件不接触内部对象。
- 桥接实现**只依赖 `Core.*` 中立接口**(如 `Core.Ssh.ISftpClient`),
  延续"库类型不出 Infrastructure"的纪律。

## 1. vela.sessions — 会话

权限:`sessions.observe.basic`(脱敏列表)/ `sessions.read`(完整)/
`sessions.create`(发起)。

```csharp
public interface ISessions
{
    Task<SessionInfo[]> ListAsync(CancellationToken ct);          // 字段随权限级别脱敏
    IAsyncEnumerable<SessionEvent> WatchAsync(CancellationToken ct); // 连接/断开/重连事件
    Task<string> ConnectAsync(string savedProfileId, CancellationToken ct); // sessions.create
}
```

## 2. vela.remoteFs — 远程文件(核心场景 S1/S2/S7)

权限:`remote.files.read` / `remote.files.write`,scope 支持会话与路径
前缀。实现桥接现有 SFTP 包装(注意既有语义:GetAttributes 对不存在路
径返回 null,不以异常判存在——SDK 的 `StatAsync` 同样返回可空)。

```csharp
public interface IRemoteFs
{
    Task<RemoteEntry[]> ListAsync(string sessionId, string path, CancellationToken ct);
    Task<RemoteEntry?> StatAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenReadAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string sessionId, string path, WriteMode mode, CancellationToken ct);
    Task DownloadAsync(string sessionId, string remotePath, string localPath,   // localPath 需同时有 fs.local.write
                       IProgress<TransferProgress>? progress, CancellationToken ct);
    Task UploadAsync(...);
    Task DeleteAsync/RenameAsync/MkdirAsync(...);
    IAsyncEnumerable<RemoteChange> PollWatchAsync(string sessionId, string path,
                       TimeSpan interval, CancellationToken ct);  // 轮询式 watch,间隔下限 2s(保护远端)
}
```

宿主侧保护:单插件并发 SFTP 操作上限(默认 4)、带宽占用与用户前台
传输错峰(插件流量入低优先级队列)、进度回调节流(≥100ms;满格(100%)那一帧只在传输成功后补发一次,失败不发)——吸取
大文件传输卡顿的历史教训,插件流量不允许灌爆 UI 调度器。

条目上的链接字段(SDK 2.1.3,宿主已接上):`ListAsync` / `StatAsync` 返回的 `RemoteFileEntry` 带 `IsSymbolicLink`
(按 lstat 判断)与 `LinkTarget`(readlink 原文,可以是相对路径);`IsDirectory` / `Size` / `LastModified` 描述链接**目标**,
断链保留链接自身的属性。进程内与隔离进程(`fs/list`、`fs/stat` 的 JSON)都带上,旧宿主或旧数据缺这两个字段时读成「不是链接」。
见 [SDK §3.3d](../sdk/sdk-reference.md#33d-插件协议的文件能力补口sdk-213)。登记之前就被明确拒绝的传输见 §2.1 的 `PluginTransferRejectedException`。

### 2.1 独立上传任务（SDK 2.1.1）

`IRemoteFsApi` 增加 `SupportsTransferOperations` 和 `StartUploadAsync` / `GetTransferAsync` / `WaitTransferAsync` / `CancelTransferAsync` / `ResumeTransferAsync`，完整契约见 [SDK §3.3b](../sdk/sdk-reference.md#33b-独立上传任务sdk-211)。登记之前就被明确拒绝的（源不存在、Windows 共享/锁冲突、额度满）以 `PluginTransferRejectedException` 交回（契约随 SDK 2.1.3 发布，宿主已接上；见 [SDK §3.3e](../sdk/sdk-reference.md#33e-传输启动前被拒sdk-213)）。它与上方同步传输不同：启动返回任务快照，内容继续由宿主按流上传，单次查询/等待超时不取消传输，不把旧两小时 RPC 时限当执行死线。

当前宿主每插件上传与下载合计最多 4 项实际传输并行、32 项活动/排队任务，保留最多 1024 条生命周期幂等回执；满额明确拒绝新任务，同 id 重取已有回执不受活动额度阻挡。这是资源并发预算，不是文件大小上限。

本地源的父目录别名先逐组件解析，审批与任务快照显示解析后的路径；末级文件链接、不支持的平台句柄验证或不可解析路径明确拒绝。Windows 上只有名字代理类重解析点（符号链接、目录联接、挂载点）算链接，OneDrive 云文件占位符这类过滤器标记不算。打开之后按**文件身份**核验句柄就是这个路径上的文件（Windows 卷序列号 + 文件 id，`FILE_ID_INFO`，取不到时两边一起退到 `BY_HANDLE_FILE_INFORMATION`；Linux statx 的 dev + inode；macOS fstat 的 st_dev + st_ino），核验那一端不经任何链接走到：Windows 逐级钉住祖先目录，不许有名字代理；Linux 从 `/` 起逐级 `openat(O_NOFOLLOW)`；macOS 用 `fstatat(AT_SYMLINK_NOFOLLOW_ANY)`。这是真正的防线。映射盘、subst 盘、硬链接名与大小写 / 规范化变体按同一个文件放行，查不出身份一律拒绝（2026-10-25 起；此前逐字比句柄的最终路径）。重复启动仍以原请求参数去重，不因别名改向重新打开文件。

新上传 API 使用独立的宿主可信确认与单任务授权租约：展示插件、真实会话、本地源、**文件大小**、远端目标及覆盖策略，只有批准当前任务或拒绝，不沿用终端/开会话授权。缺少授权闸/提示器/桌面窗口时拒绝；等待确认不阻塞启动 RPC。撤销插件权限会取消待批准及活动上传、阻止后续提交并隐藏新的任务元数据，已进入提交的不确定结果仍按 `Unknown` 处理，不承诺回滚。AI 的 `Bypass` 只跳插件审批，不能绕过该宿主确认。

进度采用 64 位字节计数、单调快照序号、执行代次和明确终态。已写满仍可能正在远端收尾；只有 `Succeeded` 是成功。远端临时文件是目标同目录下的 `.vela-up-<16 位十六进制>`；开始写之前先做提交预检，要覆盖而服务端没有 `posix-rename`、不覆盖而目标已存在时直接失败（`ErrorCode` 分别为 `not-supported` / `commit-rejected`），不传完一整份才被拒，提交仍是唯一权威。覆盖上传的临时文件一建出来就按被替换普通文件的 0777 权限位新建（服务端照常减 umask；目标是个条目却读不出模式时按 0600；读不到目标就按服务端默认，提交时照旧把关），写入期间不比原文件宽；setuid / setgid / 粘滞位与属主仍由提交补上。覆盖提交把被替换普通文件的权限位带到新文件上，属主尽力而为，setuid / setgid 只在属主属组对上之后才带；带不过去又会放宽权限时拒绝提交、目标不动。取消必须指定执行代次，恢复必须重新审批、核对连接身份/源对象/检查点及最终提交结果；不能自动重传不确定副作用。新方法默认明确不支持，能力位 false 时不退回同步大文件上传。`ErrorCode` `insufficient-space`：Linux / macOS 的源快照所在卷空间不够（详见 [自动化与 AI §1.6](11-automation-and-ai.md)）。停止宽限期 30 秒；断线记 `Failed` / `session-disconnected`，表示确认停写、目标没写入。这里的契约不等于通用权限 Broker 已实现，实际可用权限与生命周期边界必须按宿主实现说明解读。

只撤「还没批准」的传输用条件撤回（SDK 2.1.4）：`SupportsTransferWithdraw`（默认 false，与上传、下载能力各自独立）与 `WithdrawPendingTransferAsync(transferId, expectedGeneration, ct)`。判定与撤下由宿主同一步完成，与批准生效先到者赢，结论为 `PastApproval`（批准已生效，什么都不改）/ `Withdrawn`（撤下了，以 `Cancelled` 收尾、什么都没写）/ `AlreadyEnded`（批准之前已因别的原因收尾）；不支持的宿主明确抛 `NotSupportedException`，不退化成 `CancelTransferAsync`。当前宿主上传与下载都已支持，完整契约见 [SDK §3.3f](../sdk/sdk-reference.md#33f-条件撤回sdk-214)。

### 2.2 独立下载任务（SDK 2.1.2）

`SupportsDownloadOperations`（默认 false）与 `StartDownloadAsync(sessionId, remotePath, localPath, RemoteDownloadOptions, ct)` 为独立下载入口；不支持则拒绝，不降级同步下载。复用 §2.1 的 get/wait/cancel 与原共享资源额度：上传下载合计每插件 4 实际传输、32 活动/排队、1024 生命周期回执；请求/等待取消不取消任务，`Timeout=null` 无默认执行死线，下载 `CanResume=false`。方向/阶段、64 位进度和停止确认见 [SDK §3.3c](../sdk/sdk-reference.md#33c-独立下载任务sdk-212)。

新下载使用**独立于上传的单任务宿主租约**，覆盖远端读和本地写：逐次确认插件、真实会话、源、最终目标及 `Overwrite=false` 默认策略，显式覆盖需明确批准。保存选择器和 AI Bypass 都不能跳过，缺授权闸/提示器/窗口则拒绝。撤权取消待批准及活动下载，提交前重验，隐藏后续元数据，清理取消不被阻挡；不保证已提交文件回滚。此强制点不扩展旧同步 API，也不是通用文件 Broker。

当前宿主在 Windows 固定本地 NTFS 与 Linux / macOS（x64、arm64）的本地文件系统上实现安全提交：审批前钉住目标目录链，独占随机临时文件，并相对钉住的父目录原子提交。Windows 逐段以遍历 + 读属性权限持有祖先目录句柄，共享读写、不共享删除，祖先改不了名、挪不走；只拒名字代理类重解析点，云文件占位符不算链接；临时文件按句柄核验建在钉住的父目录里，提交经 `NtSetInformationFile` 相对父目录句柄改名。Linux / macOS 从 `/` 起逐级 `openat(O_NOFOLLOW)`、只留父目录 fd，链上有符号链接即拒绝，9p、SMB、FUSE、NFS 等非本地文件系统在审批前拒绝；无替换走 `renameat2(RENAME_NOREPLACE)` / `renameatx_np(RENAME_EXCL)`，标志不被支持时退到 `linkat`。默认原子无替换，即使目标中途出现也不覆盖；获准覆盖不先删目标，POSIX 上新文件沿用旧文件的权限位。UNC、其它文件系统/后端明确拒绝（`not-supported`），不直接写最终路径。复用用户下载限速并保留时间戳；长度/前后元数据/可用摘要只能检测部分源变化，不能保证远端原子快照。满字节仍须校验、关闭、提交；不合作后端停止未确认时保留槽/句柄并报告 Unknown/TimedOut，不自动重下。

下载契约已随 SDK 2.1.2 正式发布，宿主中央包、AI 显式依赖与 minSdk 均已升到 2.1.2；提交用例在 Windows 与 Linux 上实际跑过，macOS 只经 CI，真实 SSH / GUI 整链尚未验收。模型及链接双入口见 [自动化与 AI §1.7](11-automation-and-ai.md#17-ai-双入口后台下载sdk-212)。

## 3. vela.remoteExec — 远程执行(S5/S6)

走**独立执行通道**，不进用户终端，不污染用户 shell 历史与环境。实际接口是 `IRemoteExecApi`，完整参考见 [SDK §3.3a](../sdk/sdk-reference.md)。

| 方法 | 实际契约与限制 |
| --- | --- |
| `RunAsync` | 返回 `ExecResult.Output` / `Error` / `ExitCode`，非零退出是结果而非异常；默认 30 秒，最长 10 分钟 |
| `StreamAsync` | `IProgress<ExecOutput>` 按 stdout/stderr 回调，返回 `ExecStreamResult`；默认无执行死线，令牌取消或显式 Timeout 收尾，不提供 stdin |
| `StreamSudoAsync`（SDK **2.1.0**） | 返回 `SudoExecResult`，`IProgress<SudoExecEvent>` 回调无秘密状态与 PTY 合并原始文本片段；仅 Linux sudo，整条命令 root（含管道/重定向），不是通用交互 API |

普通与 sudo 流共享每插件 4 条流上限，接收器应同步快速转发，不用 `Progress<T>` 打乱输出顺序。接收器慢了不会中断执行，只会让远端输出跟着慢下来：隔离插件的通知队列满时宿主等它腾出位置，背压经 SSH 窗口传回远端；进程内插件本就在读行线程上同步回调。普通流无换行长输出按最多 4096 字符分片，CR 可分段，超长行带 `[VelaShell: continued]`，因此不要依赖一条回调等于一条完整应用记录。sudo 输出与普通流一样把非 UTF-8 字节替换为 U+FFFD，GBK 日志、二进制内容或非 UTF-8 文件名不会让在跑的 root 命令变成 Unknown。

**权限现状与设计目标分开。** 本蓝图的 `remote.exec` Broker、通用 Demand 与不可关闭审计是设计目标，不能当作本次已实现的保证；当前 `PluginPermissionGate` 覆盖 `terminal.write` / `session.open`，并为新上传/下载 API 提供各自专用单任务租约（§2.1/§2.2），不是其它远程文件操作的通用许可。此次 sudo 新增的是**宿主逐次可信确认**，而非整套通用权限系统。来源插件从宿主清单绑定，真实目标来自连接；即使 NOPASSWD 也不能跳过确认。确认框弹出后「批准」先灰约 0.6 秒（从窗口打开起算，2026-10-25 起），取消随时可点，回车与初始焦点仍在取消上 —— 防的是用户正打字时顺手一个回车批掉 root 命令。

AI 插件的提权审批与宿主确认是两层：`Ask` / `ReadOnlyAuto` 使用独立 `run_command_elevated` 类型、展示目标及整条 root 范围且不设 RepeatKey；缺少审批 Handler 即拒绝，root `cat` 也不能按只读自动放行。用户明确选择 `Bypass` 只跳过插件审批，绝不跳过宿主确认。MCP 无 Handler 时前两模式拒绝，Bypass 可以请求桌面宿主确认。

密码仅在宿主可信遮罩输入框内收集，不读取登录密码、不记忆、不进入 SDK/RPC/插件/模型/历史/日志；受控 PTY 确认关闭回显并匹配本次随机提示后才发送。命令启动后永久关闭密码应答，stdin 关闭。启动标记的完整原始字节不得直接出现于 sudo 可能回显的 argv，只有真正获准执行的 root 包装进程才能生成；sudoers 拒绝诊断不得被误判为 Running/Exited，也不得作为命令输出泄漏认证尾部。复杂 PAM/OTP、无 UI、PTY 被拒及受限 sudoers 拒绝 root shell 包装时明确失败，不绕过策略、不自动普通执行重试。用户的登录 shell 是 fish / tcsh 也照常可用：交给登录 shell 的只是一条 `/bin/sh` 调用，受控脚本编码在 `printf %b` 载荷里，登录 shell 改写不了它（见 [安全 §1.1](12-security-threat-model.md#11-此次-sudo-与-ai-托管执行的实际边界sdk-210)）。普通命令包含 sudo 不做正则改写或自动提权重跑，避免重放前半段副作用。这不是终端自动填充，不改变旧终端凭据交互决定。

诊断租约与保守异步收尾见 [SSH 规格 §9.1](../ssh/spec/08-failures.md#91-敏感交互的连接级载荷抑制租约)；授权范围缓存/撤销限制和未验证部分见 [安全说明 §1.1](12-security-threat-model.md#11-此次-sudo-与-ai-托管执行的实际边界sdk-210)。

## 4. vela.terminal — 终端(S3/S4)

权限:`terminal.read` / `terminal.write`(高敏)。

```csharp
public interface ITerminal
{
    IAsyncEnumerable<TerminalOutput> SubscribeAsync(string sessionId, CancellationToken ct);
        // 宿主侧 ≥16ms 合帧节流;订阅建立时 Demand 一次,撤销即断流
    Task<TerminalSnapshot> GetSnapshotAsync(string sessionId, CancellationToken ct);  // 屏幕+滚回尾部(行数上限)
    Task<string?> GetSelectionAsync(string sessionId, CancellationToken ct);
    Task WriteAsync(string sessionId, string input, CancellationToken ct);
        // 经现有输入串行化队列(勿直写流);默认单次 ≤4KB
    Task<string> CreateOutputChannelAsync(string title, CancellationToken ct);
    Task AppendToChannelAsync(string channelId, string text, CancellationToken ct);
        // 插件专属只读输出页(挂 VelaDock 文档),不需要 terminal 权限,仅 ui.contributions
}
```

`terminal.write` 附加护栏:注入的输入在终端内以角标短暂标识来源插件
(用户可在设置关闭);单插件写入频率限速(默认 20 次/min,防失控刷屏),
超限熔断并通知用户。

## 5. vela.localFs — 本地文件

权限:`fs.local.read` / `fs.local.write`(scope=目录前缀);
**免权限路径**:经 `PickFileAsync`/`PickFolderAsync`(宿主弹系统选择
器)获得的路径自动获得临时授权(用户亲手选择即授权,Android SAF 思
路)——鼓励插件优先用选择器而非申请宽目录。本蓝图的选择器临时授权不代替新下载API逐任务宿主可信确认（§2.2）；选择最终路径不等于授权覆盖，也不应提前创建/截断目标。

```csharp
public interface ILocalFs
{
    Task<string?> PickFileAsync(FilePickerOptions opts, CancellationToken ct);
    Task<string?> PickFolderAsync(..., CancellationToken ct);
    Task<Stream> OpenReadAsync(string path, CancellationToken ct);
    Task<Stream> OpenWriteAsync(string path, WriteMode mode, CancellationToken ct);
    Task<LocalEntry[]> ListAsync(string path, CancellationToken ct);
    // 常规 Stat/Delete/Move/Copy…
}
```

## 6. vela.storage / vela.secrets / vela.settings

```csharp
public interface IStorage      // 权限:storage.private(普通)
{
    Task<T?> GetAsync<T>(string key);   Task SetAsync<T>(string key, T value);
    Task RemoveAsync(string key);       string DataDirectory { get; }   // 直接文件读写也限于此目录
}
public interface ISecrets      // 权限:secrets(危险);命名空间强制为 <pluginId>/
{
    Task<string?> GetAsync(string name); Task SetAsync(string name, string value); Task DeleteAsync(string name);
}   // 底层走 OS 凭据库(DPAPI/Keychain/libsecret),与宿主凭据不同命名空间,互不可见
public interface IPluginSettings   // 权限:settings.own(普通)
{
    T Get<T>(string key);  Task SetAsync<T>(string key, T value);
    IAsyncEnumerable<SettingChange> WatchAsync(CancellationToken ct);   // 用户在设置页改动时推送
}
```

## 6b. vela.timeSeries — 私有时序库(已落地)

```csharp
public interface ITimeSeriesApi          // 权限:随 storage.private,不单独提权
{
    Task<ITimeSeries> OpenAsync(TimeSeriesDefinition definition, CancellationToken ct);  // 建表(幂等)
    Task<IReadOnlyList<string>> ListAsync(CancellationToken ct);
    Task<bool> DropAsync(string name, CancellationToken ct);
}
public interface ITimeSeries
{
    Task WriteAsync(TimeSeriesPoint point, CancellationToken ct);
    Task WriteManyAsync(IEnumerable<TimeSeriesPoint> points, CancellationToken ct);
    Task<IReadOnlyList<TimeSeriesPoint>> QueryAsync(TimeSeriesQuery query, CancellationToken ct);
    Task<long> CountAsync(string field, TimeSeriesQuery query, CancellationToken ct);
    Task<IReadOnlyList<string>> DistinctTagValuesAsync(string tag, CancellationToken ct);
    Task<int> DeleteAsync(IReadOnlyDictionary<string, string>? tags, CancellationToken ct);
}
```

落宿主同一个 SonnetDB;物理 measurement 名 `pts_<插件命名空间>_<短名>`,命名空间由
插件 id 派生(不可由插件指定),卸载按前缀整体 drop。语义与配额见
[dev-guide §5.2b](../templates/dev-guide.md#52b-timeseries--私有时序库);隔离进程经
`ts/*` 方法路由到同一实现(见 05)。**插件永不直连数据库,也拿不到 SQL 入口** ——
标签值等外来输入一律走参数化,列名限 `[a-z0-9_]`。

## 7. vela.ui — 界面(详见 08)

权限:`ui.contributions`(普通)。包含:命令/菜单/状态栏动态更新、
VelaUI 表面(面板与文档页)、对话框(Message/Confirm/Input/QuickPick)、
应用内通知(频控:每插件 ≤6 条/min)、进度指示(状态栏/通知内)。

> 应用内通知的实际契约(SDK 2.1.3,宿主已接上):`IUiApi.SupportsNotifications` / `PostNotificationAsync(PluginNotification)`,
> 返回 `Posted` / `AlreadyPosted` / `Muted` / `RateLimited`;按 id 先去重后限频(每插件每分钟 6 条),只收要留存、可回看的消息,
> 动作只能是本插件的命令,没有网址字段。详见 [SDK §3.7](../sdk/sdk-reference.md#37-往消息中心发通知sdk-213)。

## 8. vela.events — 宿主事件订阅(S5)

权限:事件本体多数随对应域权限(如会话事件随 `sessions.read`);
`appStartup/appShutdown/themeChanged/localeChanged` 免权限。

```csharp
public interface IEvents
{
    IAsyncEnumerable<HostEvent> SubscribeAsync(EventFilter filter, CancellationToken ct);
    // HostEvent:SessionConnected/Disconnected、TransferCompleted、ThemeChanged、
    //           LocaleChanged、AppShutdown(给插件 flush 机会)、ScheduleFired(见 11)
}
```

## 9. vela.audio — 音频输出(S2)

权限:`audio.playback`。设计取舍:**插件解码、宿主输出**——宿主不内置
编解码(避免编解码器依赖膨胀与授权问题),只提供 PCM 混音输出;MP3
解码由插件自带托管解码器(如 NLayer,官方示例演示)。

```csharp
public interface IAudio
{
    Task<IAudioTrack> CreateTrackAsync(AudioFormat fmt, CancellationToken ct); // fmt: 采样率/位深/声道
    // IAudioTrack: WriteAsync(pcm 分块,背压)、暂停/恢复/停止、音量、position 反馈
}
```

宿主输出后端:待选型 spike(候选:miniaudio 绑定 / OpenAL Soft /
平台原生 WASAPI+CoreAudio+ALSA 三实现);占位接口先行,选型不阻塞
其它域。同一时刻全局混音,插件音量受宿主主音量与静音开关约束;
插件被停用/崩溃时其音轨立即静音回收。

## 10. vela.net — 网络(声明制)

权限:`network` + manifest 声明域名列表。v1 无物理拦截(见 06 前提),
SDK 提供 `ctx.Http`(基于 HttpClient 的便捷封装,自动带插件 UA);
使用 SDK 封装的请求会经宿主审计通道记录目标域名。直接自己创建 HttpClient
在 v1 无法禁止——合规性靠审核与签名,文档如实告知。

## 11. vela.clipboard / 通知 / i18n

- `clipboard.read/write`:读写系统剪贴板(读为危险权限——剪贴板常含
  密码)。
- `notifications.system`:OS 级通知(应用内通知归 vela.ui,免此权限)。
- `i18n.read`:当前 locale、翻译好的宿主常用词条(按钮"确定/取消"等,
  帮插件文案与宿主一致)。

## 12. vela.ai — AI 网关(远期,详见 11-automation-and-ai)

权限:`ai.invoke`。占位于 apiLevel 1(接口预留、宿主可宣告
capability 不可用),完整落地见 [11-automation-and-ai.md](11-automation-and-ai.md)。

## 13. 开发计划(本分项)

批次划分与里程碑对应(见 14):

| 任务 | 内容 | 依赖 | 估算 |
| --- | --- | --- | --- |
| C-1 | 能力服务基座:注册表、Demand 基类、审计挂点、每插件限流器 | B-2, P-3 | 3d |
| C-2 | 第一批:storage / settings / secrets / events(基础域,示例插件最小依赖) | C-1 | 4d |
| C-3 | 第二批:sessions / remoteFs(桥 Core.Ssh、低优先级传输队列、进度节流) | C-1 | 5d |
| C-4 | 第三批:localFs(选择器临时授权模型)+ clipboard + notifications | C-1 | 3d |
| C-5 | 第四批:terminal(订阅合帧、快照、写入限速熔断)+ remoteExec(审计) | C-3 | 5d |
| C-6 | 音频输出 spike(三候选评估报告)→ IAudio 实现 | C-1 | 5d |
| C-7 | vela.net SDK 封装 + 审计 | C-1 | 1d |
| C-8 | 契约测试套件:每个域"权限矩阵 × 正常/边界/取消/断连"用例(接 13) | 各批次 | 持续 |

验收:S1 所需域(C-2/C-3)可支撑图片查看器示例端到端;C-5 完成后
S3/S4/S5 的域全部可用。

---

## protocols —— 自带远程文件协议(apiLevel 1,2026-08 落地)

插件注册一种协议后,它在连接配置页里与内建协议同为一等公民;宿主把该协议的会话
路由到插件的 `IProtocolFileSystem`,而双栏浏览器、传输队列、限速、拖放、冲突策略
全部零改动复用。

| 成员 | 说明 |
| --- | --- |
| `IProtocolsApi.Register(descriptor, fileSystem)` | 注册一种协议;释放返回值即注销。协议 id 必须等于插件 id 或以其为前缀 |
| `IProtocolsApi.GetTransferOptionsAsync()` | 读宿主当前的带宽上限与时间戳策略(全局用户偏好,不该每个协议各配一份) |
| `IProtocolFileSystem` | 连接/断开 + 列举/统计/读写/删除/改名/复制/权限 + 协议动作,全部以宿主分配的不透明会话键为第一参数 |
| `ProtocolDescriptor` | 页签名称、默认端口、三格标签改写、**声明式设置表单**、**声明式右键动作**、能力位、证书指纹回写字段 |
| `ProtocolFeatures` | `Permissions` / `ServerSideCopy` / `ResumeUpload` / `ResumeDownload` / `AnonymousAccess` / `CertificateTrust` |
| 异常族 | `ProtocolAuthenticationException` / `ProtocolCertificateTrustException` / `ProtocolConnectionException` / `ProtocolUnsupportedException` |
| 清单贡献 | `contributes.protocols` + `onProtocol:<协议id>` 惰性激活 |
| 可选兼实现(SDK 2.1.3,宿主已接上) | `IProtocolLastWriteTime`(改修改时间)、`IProtocolChecksum`(只用服务器自己给的摘要,`RemoteChecksumAlgorithm`);`RemoteFileEntry` 加 `IsSymbolicLink` / `LinkTarget`。见 [SDK §3.3d](../sdk/sdk-reference.md#33d-插件协议的文件能力补口sdk-213) |
| `PluginManifestReader.IsValidProtocolId` | 协议 id 的权威判定(全小写 `[a-z0-9.-]`、≤128、须以插件 id 为前缀);清单校验与运行期注册共用,避免两边口径分叉 |

设计取舍:

- **设置表单是声明式 schema,不是控件树。** 连接对话框是宿主的核心界面,布局、主题、
  校验与本地化都由它统一负责;插件只描述"要哪些参数"。这与本蓝图 08 明确不做的
  通用声明式 UI(VelaUI)是两回事 —— 那是任意界面,这只是一张形状封闭的参数表。
- **右键动作也声明式**:菜单要在按下右键那一帧画出来,不能等一次异步往返。
- **仅 `inProcess`**:协议是宿主反向调用插件的高频通道(含流式读),
  隔离进程的 RPC 只承载插件→宿主方向。放开它需要双向请求,属于 05 的后续工作。
  隔离进程里的 `IProtocolsApi` 是一个"注册即报错"的退化实现,不是 null。
- **同 id 重注册 = 替换**:换成**另一个**实现会发一次注销(宿主据此收掉旧实现名下的会话),
  换成同一个实例(插件为刷新本地化文案而重注册)则不发 —— 否则切一次界面语言
  就会把用户正开着的标签页全部断掉。旧句柄在被替换后 `Dispose` 是空操作。
- **注册表不在锁内回调插件**:事件订阅/拆订阅一律在锁外完成。那把锁 UI 线程也会持有,
  插件若在事件访问器里做阻塞式 `Dispatcher.Invoke` 就是死锁。

---

## ui 补记:`IPluginPanel.ActivateAsync`(apiLevel 1,2026-08 新增)

把面板带到前台:窗口形态还原最小化并 `Activate()`,文档形态切到该标签页。幂等、任意线程可调、
面板已关闭时是空操作。

补它的原因:插件对"同一目标重复触发"的惯例是聚焦已开的那一扇而不是再开一扇,
但 SDK 此前只有 `CloseAsync`,表达不了"聚焦"——于是插件只能 `return`,
而同尺寸的 `CenterOwner` 窗口恰好会把前一扇完全盖住,用户看到的是"点了没反应"。
