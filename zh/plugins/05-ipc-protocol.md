# 05 · IPC 协议

> **实现注记(2026-08)**:隔离进程模式已按本篇思路落地,但协议选型有一处
> 有意偏离:未引入 StreamJsonRpc + MessagePack(依赖树过重,与本仓库零依赖
> 纪律冲突;双进程同库同版,暂无跨语言协商需求),改为
> **长度前缀 + JSON 轻量双向 RPC**(`plugin-sdk/VelaShell.PluginSdk/Rpc/`:
> req/res/evt 三型、并发不排队、统一错误码空间与类型化异常还原)。
> 传输为命名管道(.NET 在 macOS/Linux 底层即 UDS),随机名 + 一次性令牌 +
> `CurrentUserOnly`。握手、错误模型、"凭据不出主进程"均与本篇一致;
> 大块数据走 base64 内联(≤16MB)与同机文件路径中转,流式子通道与共享内存
> 表面(§6-2/3)未做。若未来需要非 .NET SDK 或流式通道,再回到本篇评估
> StreamJsonRpc/编码协商。方法清单见 `Rpc/PluginRpc.cs`。

## 1. 分层

```text
应用层   IHostEndpoint / IPluginEndpoint(强类型接口,定义于 PluginProtocol)
协议层   JSON-RPC 2.0(StreamJsonRpc),MessagePack 二进制编码
帧层     StreamJsonRpc 内建 length-prefixed framing
传输层   Windows: NamedPipeServerStream(每插件一条,双工,异步)
         macOS/Linux: UnixDomainSocket(socket 文件位于用户运行时目录,0600)
侧通道   ① 流式数据通道(同一 RPC 连接上的 marshaled stream / 分块协议)
         ② 共享内存(MemoryMappedFile,仅图像表面等高带宽场景,见 08)
```

选型理由(D3 的展开):

- StreamJsonRpc:双向调用、通知、`CancellationToken` 与 `IProgress<T>`
  自动跨进程、异常序列化、成熟稳定(VS/VSCode 家族在用);MessagePack
  编码避免 JSON 文本膨胀。
- 不用 gRPC:不需要 HTTP/2、TLS、跨机;不想引 protoc 代码生成进插件
  开发者的内环。协议本身仍是语言中立的(JSON-RPC over pipe),远期非
  .NET SDK 可自行实现帧层与编码(届时提供 JSON 编码协商选项)。

## 2. 连接建立与认证

```text
1. 宿主创建管道/socket(随机名)→ 拉起 PluginHost(环境变量携带名字与令牌)
2. PluginHost 连接;10s 未连上 → 宿主判启动失败
3. 首帧(认证,原始帧,不走 JSON-RPC):magic "VELA" + authToken
   校验失败 → 立即断开并记安全日志
4. JSON-RPC 挂接,进入握手
```

## 3. 握手(handshake)

```jsonc
// PluginHost → 宿主  host/hello
{ "protocolVersion": 1,
  "apiLevels": [1],                  // PluginHost 侧支持的 apiLevel 集合
  "pluginId": "acme.image-viewer",
  "pluginVersion": "1.2.0",
  "sdkVersion": "1.3.0",
  "encodings": ["messagepack", "json"] }

// 宿主 → PluginHost  应答
{ "apiLevel": 1,                     // 协商结果(交集内取最高;空集 → 拒绝并给出双方版本)
  "encoding": "messagepack",
  "hostVersion": "0.3.0",
  "locale": "zh-Hans",
  "theme": "dark",
  "capabilities": ["remoteFs","terminal","ui","storage", ...] }  // 宿主实际具备的能力域
```

握手完成前除认证与 hello 外一切调用拒绝。之后宿主调
`plugin/activate(reason)` 进入插件生命周期。

## 4. 端点接口(应用层,节选)

```csharp
// 宿主暴露给插件(经 PluginHost 转发)。方法名 = "<域>/<动作>"。
public interface IHostEndpoint
{
    // —— 能力域调用(完整清单见 07,全部第一参数为隐式插件身份,由连接绑定,不可伪造)——
    Task<RemoteEntry[]> RemoteFsListAsync(string sessionId, string path, CancellationToken ct);
    Task<Stream> RemoteFsOpenReadAsync(string sessionId, string path, CancellationToken ct);
    Task TerminalWriteAsync(string sessionId, string input, CancellationToken ct);
    // 时序域(ts/open|list|drop|write|query|count|distinct|delete):句柄按 measurement 短名
    // 寻址,宿主侧记住已打开的实例;未 open 就用 → 明确报错,不隐式建表。
    Task<TimeSeriesPoint[]> TimeSeriesQueryAsync(string name, TimeSeriesQuery query, CancellationToken ct);
    Task UiPatchAsync(string surfaceId, UiPatch[] patches, CancellationToken ct);
    Task<PermissionState> PermissionQueryAsync(string permissionId);
    // ... 通知(宿主→插件,无应答):事件推送
}

// 插件侧暴露给宿主
public interface IPluginEndpoint
{
    Task ActivateAsync(ActivationReason reason, CancellationToken ct);
    Task DeactivateAsync(string reason, CancellationToken ct);
    Task ExecuteCommandAsync(string commandId, JsonElement? args, CancellationToken ct);
    Task UiEventAsync(string surfaceId, UiEvent e, CancellationToken ct);   // VelaUI 事件回传
    Task OnHostEventAsync(HostEvent e);                                     // 订阅的事件(通知)
    Task<string> PingAsync();                                               // 心跳
    Task OnEnvironmentChangedAsync(EnvChange e);                            // 语言/主题切换
}
```

纪律:

- 接口只出现在 `PluginProtocol`;DTO 全部为不可变记录,MessagePack 键
  用整数 key 显式标注(改名安全),**新增字段只能追加可选项**(apiLevel
  兼容承诺的机械保障,CI 里跑二进制兼容检查)。
- 插件身份绑定在连接上:能力实现读连接元数据获得 pluginId,**协议里
  没有"以某插件身份调用"的参数**,从根上防冒充。

## 5. 错误模型

统一错误码空间(JSON-RPC error.code):

| 段 | 含义 | 例 |
| --- | --- | --- |
| -32xxx | JSON-RPC 保留 | 方法不存在、参数错误 |
| 1xxx | 通用宿主错误 | 1001 HostShuttingDown、1002 CapabilityUnavailable |
| 2xxx | 权限 | 2001 PermissionDenied(含 permissionId)、2002 PermissionPromptDismissed |
| 3xxx | 远程会话 | 3001 SessionNotFound、3002 SessionDisconnected、3003 SftpError(内嵌 VelaSsh 错误分类) |
| 4xxx | 文件系统 | 4001 NotFound、4002 AccessOutsideScope |
| 5xxx | UI | 5001 SurfaceClosed、5002 InvalidPatch |

SDK 把错误码还原为类型化异常(`PermissionDeniedException` 等);未知码
落到 `VelaPluginException` 基类。宿主内部异常细节(堆栈、路径)**不跨
进程下发**,只给分类码与安全的 message(防信息泄漏)。

## 6. 大块数据与流

三档策略,按数据量与形态选择:

1. **小(≤256KB)**:直接作为 MessagePack `byte[]` 内联在应答里
   (如读取小配置文件)。
2. **流式(文件传输、终端输出订阅)**:StreamJsonRpc 的 stream
   marshaling(管道上多路复用的子流),天然背压;取消即断流。
   终端输出订阅额外做**宿主侧节流合并**(≥16ms 窗口合帧),吸取
   perf-pass 的教训——高频上报必带节流,防止插件订阅拖慢终端热路径。
3. **高带宽表面(图像帧)**:MemoryMappedFile 共享内存 + RPC 只传帧
   元数据(mmf 名、尺寸、stride、序号),见 08 §4。共享内存段由宿主
   创建并限定当前用户访问。

## 7. 排序、并发与重入

- 同一连接上请求并发执行(StreamJsonRpc 默认),**不做全局串行**;
  需要顺序语义的域(如 `terminal/write`)在宿主能力实现内部按
  sessionId 排队——与现有"终端输入只准入队勿直写流"的纪律对齐。
- 宿主 → 插件的调用一律带超时(默认 5s,UI 事件 2s);插件慢不阻塞
  宿主任何路径(fire-and-forget + 超时观察)。
- 取消:`CancellationToken` 跨进程传播;连接断开等价于该连接全部
  未完成调用取消。单帧的写入是原子的(SDK 2.1.3 起):排队等写闸时取消,这一帧就不发;
  已经开写之后取消,调用方只是不再等待,帧照样写完整、对端照样执行(§7.4)。

### 7.1 执行操作（SDK 2.1.0）

以下是 SDK 2.1.0 正式契约与当前长度前缀 JSON RPC 的执行操作行为，不代表上方 StreamJsonRpc/MessagePack 设计全部落地；正式包恢复后的执行 RPC 回归已运行，不能据此扩大成真实 GUI/插件/远端整链验收。

- `HandshakeResponse.SupportsExecutionOperations` 为附加 init 属性，缺席/default 为 false；新宿主置 true，表示执行 operation、取消与有序通知能力。false 时 PluginHost 在发 sudo RPC 前抛 `NotSupportedException`，无弹窗、无普通降级。普通流兼容旧宿主保留两小时 RPC 兜底；只有新协议无限流才不再靠两小时死线代替取消。
- `exec/stream` 的 `ExecStreamRequest.OperationId` 与 `exec/output` 的 `ExecOutputNotification.Sequence` 是附加 init 属性，保留原构造和 Deconstruct 签名。新增 `exec/sudoStream` 请求只含 `SessionId` / `Command` / `OperationId` / `TimeoutSeconds`；`exec/sudoEvent` 含 `OperationId` / `Sequence` / 非敏感 Event；`exec/cancel` 只含 OperationId，是插件到宿主的幂等通知。来源身份绑定该插件连接，没有密码或冒用身份参数。
- 每插件连接独立登记操作/CTS，链接连接与插件停机；取消早于登记也必须生效，退出/断开取消并异步收尾。早到取消最多 128 个，超限 fail-closed 停止登记和活动执行，不丢弃取消后重新启动。带 OperationId 的请求帧不随调用方取消：在写闸前排队时取消只让调用方不再等待，请求照发，随后的 `exec/cancel` 要么早于登记到达（登记即取消），要么取消已登记的操作；否则请求帧没发出、取消通知却照发，这些永远等不到请求的 id 会白白占满早到取消的上限。此处不泛化其它 RPC 能力。
- 输出与状态进 256 项有界通知队列、由单异步写泵按序发送，序号从 1 连续递增；新协议拒绝非正序号、缺号及乱序。SSH 读泵（普通流逐行）与 sudo 执行器（逐事件）逐条等待入队：队列满就等写泵腾出位置，背压经 SSH 窗口传回远端，一段上千个短行的输出也不会冲爆队列；只有只能同步上报的调用方在队列满时明确失败并取消操作，不静默丢输出。写泵失败时立即唤醒正在等待的生产者并报出原始失败。终态返回前排空；普通 Lines 与实际回调数、sudo OutputChunks 与仅 Output 事件数须一致，尾事件丢失也不能成功。回调不得阻塞或同步等待同一连接 RPC；不用 `Progress<T>` 或未观察通知冒充保序。
- 执行通知专用严格发送路径 `NotifyExecutionAsync` 不吞写入失败。若 IO/已释放等错误可能已写出半个长度头/帧体，立即终止该 RPC 连接、失败所有在途请求并取消其执行，不再在坏帧连接上发送错误终态。它不是普通 `NotifyAsync` 的尽力而为语义。SDK 2.1.3 起「写坏半帧就作废整条连接」扩到了全部写入（请求与普通 `NotifyAsync` 也一样），见 §7.4。
- `cancelled` / `timeout` / `not-supported` 错误分别还原取消、超时、不支持；不能把用户停止变成普通失败或把缺失结果当成功。远端进程是否真的停止仍须另行确认，不能由取消通知投递成功推出。

SDK 契约、严格发送/半帧失败回归，以及正式 SDK 2.1.0 装配后的宿主 `ExecutionRpcTests` 已实际通过。发布前缺 sudo 类型的编译阻塞已经解除；RPC 初轮超时定位为测试夹具未启动读泵，修复夹具后重跑通过，不是把超时忽略。**RPC 回归不等于真实 GUI 到远端整链验收。** 当前验证边界见 [自动化与 AI §1.4](11-automation-and-ai.md#14-ai-插件的托管执行与恢复sdk-210)。

### 7.2 独立上传任务（SDK 2.1.1）

文件内容仍由宿主按路径流式读取，不塞进 JSON 正文。新 `fs/transfer/startUpload|get|wait|cancel|resume` 请求只交换任务参数和不可变快照，实际传输不再绑定一次长时间 RPC。旧同步 `fs/upload` / `fs/download` 不改变；插件不得把旧同步上传的两小时 RPC 兜底当成无限长传输支持。

- 握手附加 `SupportsTransferOperations`，缺席为 false；只在宿主实际实现时宣告支持，不支持就明确失败，不静默退回旧上传。
- 启动的 `OperationId` 与恢复操作共用插件生命周期命名空间；宿主按绑定的插件身份和参数去重，重复请求不得第二次上传。请求取消或回执丢失不证明未启动；复用同 id 查询回执，不另造 id 重传。
- `wait` 的等待窗口 0–30 秒，仅等待单调 `Sequence` 更新或终态；到期返回当前快照，取消等待不会杀后台任务。状态查询是权威来源，进度通知可以节流/补读，不要求日志流式通知逐帧完整性。
- `cancel` 携带 `ExpectedGeneration`；恢复产生新代次后，旧取消请求不能作用于它。恢复选项携带明确的 `TargetSessionId`，由宿主验证连接身份及检查点，不能由调用方指定偏移绕过验证。
- `Succeeded` 只在远端写入收尾、校验和原子提交成功后产生；字节满格、已发取消请求和连接已关闭都不是成功或停止证明。提交结果未消歧的 `Unknown` 不自动重传，也不可恢复。

完整 DTO、所有权、恢复与超时预算约定见 [SDK §3.3b](../sdk/sdk-reference.md#33b-独立上传任务sdk-211)。该契约不提供跨应用重启恢复，也不消除远端空间、权限及网络边界。

### 7.3 独立下载任务（SDK 2.1.2）

下载新增 `fs/transfer/startDownload`（`PluginRpc.FsTransferStartDownload`、`FsTransferStartDownloadRequest`），只交换会话/源/最终目标和 `RemoteDownloadOptions`，内容不进 RPC。`HandshakeResponse.SupportsDownloadOperations` 默认 false，独立于上传能力；缺席明确不支持，不降级旧同步下载。`RemoteTransferSnapshot.Direction` 缺席为 Upload，`Phase` 缺席为 Unknown；下载明确携带 Download 与 Approval/Transfer/Verification/Commit 阶段。

共用 get/wait/cancel、插件身份绑定、`OperationId` 生命周期账本、单调 Sequence 和执行代次；方向也是幂等参数，同 id 换方向拒绝。启动请求取消只停止回执等待，不能推出任务没启动；短 wait 到期不取消下载。查询为权威来源，不承诺独立下载推送。默认不覆盖，只有校验、关闭和安全提交确认后才 Succeeded；Unknown/TimedOut 且未确认停止不自动重下。下载不支持 resume。上传下载合用原 1024 回执/32 活动排队/4 实际传输预算。

下载契约已随 SDK 2.1.2 正式发布到 NuGet.org，宿主中央包、AI 显式依赖与 minSdk 均为 2.1.2；下载有自己的回归用例，不拿上节的上传回归结论代替。安全提交支持 Windows 固定本地 NTFS 与 Linux/macOS（x64、arm64）的本地文件系统，UNC/9p/SMB/FUSE/NFS 等其它文件系统明确拒绝，不回退旧 API。完整边界见 [SDK §3.3c](../sdk/sdk-reference.md#33c-独立下载任务sdk-212)。

### 7.4 通知、启动前被拒与整帧写入（SDK 2.1.3）

**以下契约已随 SDK 2.1.3 正式发布到 NuGet.org。** `ui/notify` 与 `transfer-rejected` 已由宿主接上；整帧写入不用改代码，抬包即生效。

- **`ui/notify`**（`PluginRpc.UiNotify`，`UiNotifyRequest(PluginNotification)` / `UiNotifyResponse(PluginNotificationOutcome)`）：隔离插件往消息中心发通知。握手 `HandshakeResponse.SupportsNotifications` 缺席时为 false，
  这时 PluginHost 在插件进程里直接抛 `NotSupportedException`；支持时先本地校验再发请求。语义见 [SDK §3.7](../sdk/sdk-reference.md#37-往消息中心发通知sdk-213)。
- **错误码 `transfer-rejected:<原因>`**（`RpcErrorCodes.TransferRejected`）：传输在登记之前就被明确拒绝（源不存在、被占用、额度满、空间不够），还原成 `PluginTransferRejectedException`，后缀认不出时原因为 `Unknown`。见 [SDK §3.3e](../sdk/sdk-reference.md#33e-传输启动前被拒sdk-213)。
- **一帧要么整帧写完、要么不开始**（`RpcConnection.WriteAsync`，公开 API 不变）：调用方的取消令牌只管排队等写闸 —— 拿到写闸之前取消，这一帧一个字节都不写；拿到之后流写入一律不再看调用方的令牌，
  这时取消，调用方立即收到 `OperationCanceledException`，帧在后台照样写完整，对端仍会执行这条请求（与 §7.1「请求帧不随调用方取消」同一口径）。写到一半失败，整条连接作废：排在后面的写入一个字节都不再写，
  在途请求全部失败、触发断开、释放连接 —— 普通的 `NotifyAsync` 对单条通知仍是尽力而为，但不再带着半帧继续用这条连接（此前只有 `NotifyExecutionAsync` 这样做，§7.1）。排队等写闸的调用也响应连接关闭，以 `ObjectDisposedException` 退出
  （`RequestAsync` 里表现为 `RpcDisconnectedException`），不再一直挂着。宿主与 PluginHost 不必改代码，抬了包版本就生效。

### 7.5 条件撤回（SDK 2.1.4）

**契约已随 SDK 2.1.4 正式发布到 NuGet.org，宿主已接上。**

- **`fs/transfer/withdraw`**（`PluginRpc.FsTransferWithdraw`，载荷 `FsTransferWithdrawRequest(TransferId, ExpectedGeneration)`，应答 `RemoteTransferWithdrawResult`）：只在这一代的批准还没生效时撤下传输，判定与撤下在宿主那边同一步完成，与批准生效先到者赢 —— 没有「插件 `get` 看到 `Approval`、再发 `cancel`」之间那一个往返的空当。结论 `RemoteTransferWithdrawOutcome` 线上按数值传，不认识的值读成 `Unknown`，不算撤下。
- 握手 `HandshakeResponse.SupportsTransferWithdraw` 缺席时为 false。宿主按能力实例的实际支持宣告；没宣告时 PluginHost 不发这条，在插件进程里直接抛 `NotSupportedException`，也**不**改发 `fs/transfer/cancel`。
- 与 `cancel` 同一套所有权、执行代次与撤权口径：代次不符拒绝，同一代重复请求结论不变，撤权后仍可用于清理、快照脱敏；请求帧不随调用方取消，已登记的撤回不回滚。无效或越权的 id 在进程内抛 `FileNotFoundException`；隔离进程里这类错误目前还原成 `RpcRemoteException`（`RpcErrorCodes` 不认 `FileNotFoundException`，`fs/transfer/*` 其余请求同样如此）。

完整语义见 [SDK §3.3f](../sdk/sdk-reference.md#33f-条件撤回sdk-214)。

## 8. 开发计划(本分项)

| 任务 | 说明 | 依赖 | 估算 |
| --- | --- | --- | --- |
| P-1 | 传输层封装:管道/UDS 双实现 + 认证帧 + 连接生命周期(断开事件、半开检测) | A-2 | 3d |
| P-2 | 握手与 apiLevel 协商 + 编码协商;拒绝路径的可读错误 | P-1 | 2d |
| P-3 | 端点接口与 DTO 首批定稿(activate/deactivate/command/ping/事件);MessagePack 整数键纪律 + CI 二进制兼容检查 | P-2 | 3d |
| P-4 | 错误码空间 + SDK 异常映射 | P-3 | 1d |
| P-5 | 流式通道:stream marshaling 封装、终端订阅节流合并、背压测试 | P-3 | 3d |
| P-6 | 共享内存表面通道(创建/授权/回收协议) | P-3 | 2d |
| P-7 | 协议模糊测试:乱序帧、超长帧、非法 MessagePack、握手期调用(接 H-8 混沌组) | P-1..P-4 | 2d |

验收:双进程回环基准——空调用 P95 < 1ms,1MB 流吞吐 ≥ 500MB/s(本机
管道);协议模糊测试宿主侧零崩溃零挂起。
