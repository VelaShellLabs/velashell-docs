# 08 · 失败分类与可观测性

> 对应实现：`Diagnostics/`（L9）。
>
> 本文件定义的是**契约**，不是实现细节：
> 使用者据此写 `catch` 与 UI 文案，所以这里的每一个取值一旦发布就不能随意改。

---

## 一 原则：异常里装数据，不装拼好的句子

一个失败必须能回答四个问题：

1. **是什么失败了？** → `SshFailureReason`（强类型枚举）
2. **在哪一步失败的？** → `SshPhase`
3. **对端说了什么 / 我方提供了什么？** → 结构化上下文（算法名单、尝试记录、状态码）
4. **下一步能做什么？** → `IsRetryable` + 上下文里的可操作信息

**禁止**把这些信息只写进 `Message` 然后让调用方去切字符串。
理由很直接：那正是我们要消灭的东西之一 ——
底层库把失败原因藏在 internal 异常里、只在消息前缀露一截，
上层为了区分「认证失败 / 超时 / 协商失败」只能去 `IndexOf(" - ")`。
那种代码在库升级改了一个字的措辞时会静默失效。

`Message` 是给人看的，**不是** API。

〔决策〕**对端给的文本进 `Message` 之前先清一遍**：控制字符（`ESC`、`CR`、`BEL`…）、`DEL`、C1 控制码与双向文本控制符
一律换成 `?`，并截到固定长度。版本标识串、`DISCONNECT` 的描述、拒绝开通道的理由、SFTP 的状态消息、
代理的应答都来自对端，很多时候来自一个还没被认证的对端 —— 而 `Message` 最后会被打到终端或界面上，
原样拼进去就是一个转义序列注入面（改剪贴板、清屏伪造提示、盖掉前半行）。
**原话照样放在专门的属性里**（`PeerDescription`、`ServerMessage`），那些属性按不可信文本对待。

---

## 二 异常层级

```
SshException                          抽象基类；带 Reason / Phase / IsRetryable
├── SshConnectException               建连阶段（拨号、版本、协商、主机密钥）—— 带 Hops
│   └── SshNegotiationException       算法协商失败 —— 带双方名单
├── SshKeyExchangeException           密钥交换的计算失败（对端公开值不合法）
├── SshAuthenticationException        认证失败 —— 带逐方法尝试记录
├── SshProtocolException              对端违反协议
├── SshConnectionClosedException      连接已断（对端关闭 / 保活判死 / 收到 DISCONNECT / 本端中止 / 重协商时换了主机密钥）
├── SshChannelException               通道打不开，或通道上的请求被拒 —— 带原因码与对端原文
├── SshCommandFailedException         远端命令没有成功结束（EnsureSuccess）—— 带完整结果
├── SshForwardException               转发器起不来
├── SshPublicKeyException             公钥 blob / 文本解析失败、类型不支持
├── SshPrivateKeyException            私钥文件读不出来、解不开 —— 带 NeedsPassphrase
├── SshCertificateException           证书读不出来，或与私钥不是一对
├── SshAgentException                 连不上 agent，或 agent 拒绝了
├── SftpException                     SFTP 操作失败 —— 带 StatusCode 与服务端原文 ServerMessage
├── SftpTransferInterruptedException  传输中断 —— **带 DurableLength**
└── SftpUnavailableException          SFTP 子系统起不来
```

下面这些类型只对应一种失败，`Reason` 与 `Phase` 是固定的：

| 类型 | `Reason` | `Phase` |
| --- | --- | --- |
| `SshNegotiationException` | `NegotiationFailed` | `KeyExchange` |
| `SshKeyExchangeException` | `ProtocolError` | `KeyExchange` |
| `SftpUnavailableException` | `Unsupported` | `Open` |
| `SftpTransferInterruptedException` | `ClosedByPeer` | `Open` |
| `SshCommandFailedException` | `CommandFailed` | `Open` |

下面这些 `Phase` 固定，`Reason` **按实情报**：

| 类型 | `Reason` | `Phase` |
| --- | --- | --- |
| `SshPublicKeyException` | `KeyFormatInvalid`、`Unsupported`（类型不支持） | `None` |
| `SshPrivateKeyException` | `KeyFileUnreadable`、`KeyFormatInvalid`、`KeyPassphraseRequired`、`KeyPassphraseIncorrect`、`Unsupported` | `None` |
| `SshCertificateException` | `KeyFileUnreadable`、`KeyFormatInvalid`、`KeyMismatch`、`Unsupported` | `None` |
| `SshAgentException` | `AgentUnavailable`、`AgentRefused`、`LimitExceeded`（要加的钥超出报文上限）、`ProtocolError` | `Authenticating` |
| `SshChannelException` | `ChannelOpenFailed`（带 `OpenFailureReason`）、`ChannelRequestRejected` | `Open` |
| `SshForwardException` | `ForwardRejected`、`ForwardBindFailed`、`ForwardSetupFailed`、`LimitExceeded`、`ProtocolError` | `Open` |

读私钥、读证书、解析公钥发生在连接之外（也可能根本没有连接），所以 `Phase` 是 `None`。
密钥交换里解析对端的主机密钥失败时，由密钥交换把它包成 `Reason` 为 `HostKeyRejected` 的 `SshConnectException`。

〔决策〕**`Reason` 必须说真话，不许写死。**曾经私钥、证书、agent、转发的异常一律报 `Unsupported`，
exec / pty-req / shell 被拒报成 `ChannelOpenFailed`（通道其实开成功了），跳板成环报成可重试的 `ProxyRefused`。
调用方于是只能去解析消息句子 —— 而消息是写给开发者的诊断文本，不是界面文案，也不承诺措辞稳定。
没有合适的值就加一个，不借一个相近的；只有异常类型本身只对应一种失败时才在类型里固定（上面第一张表）。

主机密钥被拒没有专门的类型：它是 `Reason` 为 `HostKeyRejected` 的 `SshConnectException`，策略给的原因就是它的 `Message`（§3）。

〔决策〕**通道级失败不派生自连接级失败。** 一条通道打不开
（服务端 `MaxSessions` 满了）与整条连接断了是两件事，
上层的重连策略只该对后者生效。把它们放进同一条继承链，
调用方 `catch (SshConnectionClosedException)` 就会把前者也吞进去。

〔决策〕**`SshPublicKeyException` 与 `SshKeyExchangeException` 都派生自 `SshException`。**
两者都曾经直接派生自 `Exception`：调用方用 `catch (SshException)` 兜库的失败时漏掉它们，宿主的异常翻译也认不出；
`SshKeyExchangeException` 在建连时还会原样漏给调用方。
`SshPublicKeyException` 的 `Phase` 记 `None`（见上一段）；它曾经记 `KeyExchange`，读本地 `.pub` 时那个阶段并不成立。
`SshKeyExchangeException` 的 `Reason` 记 `ProtocolError`：它报的几乎总是对端给的公开值不合法（长度不对、不在曲线上、弱值）；
「算法没实现」那一类在连接之前就被挡住了（`SshConnection.ConnectAsync` 先核对算法清单）。
清单与实现表万一对不上，那是库自己的编程错误，报 `InvalidOperationException`，不借 `SshException` 的名义。

### 2.1 连接中途断掉：原因先归成公开类型

一条**已经建好**的连接被什么终结了 —— 接收循环、发送泵、保活、重协商里出的任何错 ——
那个原因会被原样交给这条连接上的每一个调用者，以及每一条通道的读者（读会抛出它，而不是像对端发了 EOF 那样读完，
见 `05-connection.md` §4.4）。〔决策〕**交出去之前先归成本库的公开类型：**

| 终结连接的原因 | 调用方拿到的 | `Reason` |
| --- | --- | --- |
| 本来就是 `SshException` | 原样 | 原样 |
| `OperationCanceledException`、`ObjectDisposedException` | 原样 —— 那是本端在收工（取消、释放），不是故障 | — |
| 对端在一个报文的中途关闭了连接 | `SshConnectionClosedException` | `ClosedByPeer` |
| `IOException`、`SocketException`（套接字断了、被重置） | `SshConnectionClosedException` | `ClosedByPeer` |
| 帧格式或完整性校验失败、报文解析失败 | `SshProtocolException` | `ProtocolError` |
| 其它任何意外 | `SshConnectionClosedException`，原异常在 `InnerException` 里 | `Unknown` |

新包出来的这几种 `Phase` 一律记为 `Open`，即使故障发生在重协商期间（已知局限）；原样交出的保留自己的 `Phase`（比如重协商超时是 `Rekeying`）。

理由：这个原因会落到使用者的 `catch` 与重连策略上，而那两者都按 `SshException` 与它的 `Reason` 分流
（§3：自动重连只该对「断了」生效）。曾经原样抛出：内部的解析异常类型，使用者按类型 `catch` 不到；
裸的套接字异常绕开了 `Reason`，重连策略分不出它是「断了」；宿主的异常翻译也认不出它们。

**建连期间**（拨通之后，到认证结束）用的是同一套口径，只是 `Phase` 记失败发生的那一步：

| 建连期间出了什么事 | 调用方拿到的 | `Reason` |
| --- | --- | --- |
| 版本交换、密钥交换、认证期间流上的 `IOException` / `SocketException` | `SshConnectionClosedException`，原异常在 `InnerException` 里 | `ClosedByPeer` |
| 对端在一个报文的中途关闭（密钥交换、认证期间） | `SshConnectionClosedException` | `ClosedByPeer` |
| 对端在版本交换期间关闭 | `SshConnectException` | `ClosedByPeer` |
| 帧格式或完整性校验失败（密钥交换、认证期间） | `SshProtocolException` | `ProtocolError` |
| 密钥交换的计算失败（对端公开值不合法） | `SshKeyExchangeException` | `ProtocolError` |

同一个「断了」按在哪一步、由谁察觉，可能是 `SshConnectException` 也可能是 `SshConnectionClosedException`，
`Reason` 却总是 `ClosedByPeer` —— 判断「是不是断了」看 `Reason`，不看类型。
〔决策〕曾经密钥交换期间的报文中途断开报成 `ProtocolError`，调用方会以为不值得重连；拨通之后的套接字异常原样漏出。现在与会话期间一致。

拨号阶段不在此列：各个拨号器自己把失败归成带原因的 `SshConnectException`（§3 的 `DnsFailure`、`TcpRefused`、`ProxyRefused`……，见 `09-dialing.md`）。
建连期间的超时、协商失败与主机密钥被拒也由各步自己报（§3）。

---

## 三 `SshFailureReason`

| 取值 | 含义 | 可重试 | 典型下一步 |
| --- | --- | :-: | --- |
| `DnsFailure` | 主机名解析不了 | ✔ | 检查主机名/DNS |
| `TcpRefused` | 连接被拒 | ✔ | 检查端口/服务是否在跑 |
| `TcpTimeout` | 连接超时 | ✔ | 检查防火墙/网络 |
| `TcpUnreachable` | 网络不可达 | ✔ | |
| `ProxyRefused` | 代理拒绝转发 | ✔ | **见 §5.2** |
| `ProxyAuthRequired` | 代理要求认证：没配凭据，或凭据被拒 | ✘ | 配置（或改对）代理凭据 |
| `NotAnSshServer` | 对端不说 SSH | ✘ | 端口连错了 |
| `VersionMismatch` | 协议版本不是 2.0 | ✘ | |
| `NegotiationFailed` | 算法无交集 | ✘ | **见 §5.1** |
| `HostKeyRejected` | 主机密钥被拒（`SshConnectException`，`Phase` 为 `KeyExchange`）：策略拒绝（`SshHostKeyVerdict.Reject`）—— 没见过而不许问、使用者拒绝、被 `@revoked`、指纹不在白名单；`K_S` 解析不了、签名验不过、RSA 太短、与协商出的算法对不上（含协商出证书算法而 `K_S` 不是证书，或反过来）；有 CA 担保的主机证书不合格（`03-key-exchange.md` §5.5） | ✘ | 看 `Message`：策略给的原因（`SshHostKeyVerdict.Message`）原样放在里面，附指纹与 `known_hosts` 行号 |
| `HostKeyChanged` | 主机密钥**变了**，两种处境：① 首次交换时策略用 `SshHostKeyVerdict.RejectChanged` 拒绝 —— `KnownHostsPolicy` 在记着的密钥变了、或只记着别的类型时这么报（`SshConnectException`，`Phase` 为 `KeyExchange`，消息里有新旧指纹与行号）；② 重协商时对端出示的主机密钥与首次交换时钉住的不同（`03-key-exchange.md` §8.4），连接以 `SshConnectionClosedException`（`Phase` 为 `Rekeying`）断开 | ✘ | 可能是中间人：不要自动重连，也不要给「信任并记住」的捷径 —— 服务器确实重装了的话，让人去 `known_hosts` 删掉旧的那一行 |
| `AuthenticationFailed` | 某次认证尝试失败 | ✔ | |
| `AuthenticationMethodExhausted` | 所有方法试完 | ✘ | **见 §5.3** |
| `TwoFactorRequired` | 服务端要 keyboard-interactive 而我们没配 | ✘ | 提示「这台机器需要动态码」 |
| `PasswordExpired` | 服务端要求改密码 | ✘ | |
| `KeyFileUnreadable` | 私钥 / 证书 / 公钥文件读不出来（不存在、没有权限、IO 错误） | ✘ | 消息里有路径 |
| `KeyFormatInvalid` | 私钥 / 证书 / 公钥的内容格式不对（损坏、截断、参数不成立） | ✘ | |
| `KeyPassphraseRequired` | 加密的私钥需要口令，而没有给 | ✘ | 弹口令输入框（`SshPrivateKeyException.NeedsPassphrase`） |
| `KeyPassphraseIncorrect` | 给了口令，但解不开这把私钥 | ✘ | 再问一次口令 |
| `KeyMismatch` | 凭据材料对不上：证书里的公钥与私钥不是一对、拿主机证书去登录 | ✘ | 证书要与签发时用的那把私钥一起用 |
| `AgentUnavailable` | 找不到或连不上 ssh-agent，或它的端点不可信 | ✘ | 检查 agent 是否在跑 |
| `AgentRefused` | ssh-agent 拒绝了请求（`ssh-add -c` 的确认被拒、agent 被锁、钥已不在） | ✘ | |
| `Timeout` | 某阶段超时 | ✔ | |
| `KeepAliveTimeout` | 保活判死 | ✔ | **自动重连只该对这一类生效** |
| `ClosedByPeer` | 对端主动关闭 | ✔ | |
| `Disconnected` | 收到 `SSH_MSG_DISCONNECT` | ✘ 〔未实现：本想按 `DisconnectReason` 细分，目前一律不可重试〕 | 原因码在 `SshConnectionClosedException.DisconnectReason`，对端原话在 `PeerDescription` |
| `ProtocolError` | 对端违反协议 | ✘ | |
| `ChannelOpenFailed` | 通道打不开（`CHANNEL_OPEN_FAILURE`，原因码在 `OpenFailureReason`） | ✘ 〔未实现：本想按原因码细分，目前一律不可重试〕 | |
| `ChannelRequestRejected` | 通道开成了，但通道上的 `exec` / `pty-req` / `shell` / `subsystem` 被拒 | ✘ | 看服务端的 `ForceCommand`、`PermitTTY`、`Subsystem` 配置 |
| `ForwardRejected` | 服务端不接受转发请求（`AllowTcpForwarding no`、`AllowAgentForwarding no` 之类），或本端拒绝了对不上任何转发的入站通道 | ✘ | |
| `ForwardBindFailed` | 本机的监听端口开不了（被占用、没有权限） | ✘ | 换一个端口 |
| `ForwardSetupFailed` | 本机一侧准备转发失败：拿不到 X 显示、`xauth` 跑不起来或失败 | ✘ | |
| `LimitExceeded` | 本端的某个并发上限到了（转发连接数、agent / X11 通道数） | ✘ | |
| `CommandFailed` | 远端命令没有以退出码 0 结束（`SshCommandResult.EnsureSuccess` 抛的 `SshCommandFailedException`） | ✘ | 看 `Result`：stderr、退出码或信号 |
| `InvalidConfiguration` | 配置本身不成立：`ProxyJump` 成环、跳数超限、`ProxyCommand` 模板非法 | ✘ | 改配置 —— 不改的话下一次还是一样 |
| `Aborted` | 本端中止（Dispose / 取消） | ✘ | |
| `Unsupported` | 请求的能力（算法、密钥类型、格式版本）对端或本库不支持 | ✘ | |
| `Unknown` | 未分类：连接因意外错误中断（§2.1） | ✘ | 看 `InnerException` |

〔决策〕**`IsRetryable` 是库给的建议，不是承诺。** 它表达的是
「这个失败是否可能因为重试而消失」，不表达「应该重试」——
重试策略是使用者的事（他们才知道用户在等还是在跑批）。

---

## 四 `SshPhase`

```
Dialing → VersionExchange → KeyExchange → Authenticating → Open → Closing
                                 ↑                          │
                                 └──── Rekeying ←───────────┘
```

| 取值 | 说明 |
| --- | --- |
| `Dialing` | DNS、TCP、代理握手、跳板链 |
| `VersionExchange` | 标识串交换 |
| `KeyExchange` | 首次 KEX（含主机密钥验证） |
| `Authenticating` | 认证 |
| `Open` | 正常运行 |
| `Rekeying` | 重协商中 |
| `Closing` | 收尾 |

**失败必须带 Phase。** 同一个 `Timeout` 出现在 `Dialing` 与 `Authenticating`
是完全不同的两件事：前者是网络问题，后者很可能是用户在找手机上的动态码。

---

## 五 三个带结构化上下文的异常

### 5.1 `SshNegotiationException`

```
NegotiationCategory   Category       // KeyExchange / HostKey / EncryptionC2S /
                                     // EncryptionS2C / MacC2S / MacS2C / CompressionC2S / CompressionS2C
IReadOnlyList<string> OfferedByPeer  // 对端 KEXINIT 的该类名单，原样
IReadOnlyList<string> OfferedByUs    // 我们发出去的该类名单，原样
string                PeerVersion    // "SSH-2.0-OpenSSH_9.6p1 ..."
```

**这一条直接替代了「协商失败后再开一条 TCP 明文探对端 KEXINIT」的做法。**
那份名单我们在握手时本来就收到了，只是以前没交出来。

〔决策〕**`Message` 里给一段可读的差异说明**，但**名单必须同时以结构化形式提供** ——
UI 要按类别分行排版、要高亮交集为空的那一类，靠解析 `Message` 做不到。

〔决策〕**展示时过滤掉 `*-cert-v01@openssh.com` 变体**（在使用者一侧做，库照常给全量）。
它们只在对端出示证书时才用得上，列出来会把六个可用算法淹没在十二行里。

### 5.2 `SshConnectException` 的代理上下文

```
IReadOnlyList<SshHopInfo> Hops   // 跳板链/代理链上每一跳：类型、地址、是否成功、耗时
```

〔决策〕失败在链上哪一跳**必须能看出来**。
「连不上 10.0.0.9」和「连不上跳板机 jump.example.com」对用户是两个完全不同的问题，
而现在它们长得一模一样。

`SshHopInfo` 至少有 `Kind`（`Tcp` / `Socks5` / `HttpConnect` / `SshJump` / `ProxyCommand`）、
`Target`、`Succeeded`、`Elapsed`、`Detail`。
填写规则（从近到远、内层原样保留）见 `09-dialing.md` §2.2。

〔决策〕HTTP CONNECT 代理拒绝 22 端口时，**在消息里直接给出建议**
（「代理可能只放行 80/443，请改用 SOCKS5 或直连」）。
这是一个高频且用户完全猜不到的失败。

### 5.3 `SshAuthenticationException`

```
IReadOnlyList<AuthAttempt> Attempts      // 见 04-authentication.md §3.4
IReadOnlyList<string>      ServerOffered // 服务端最后给出的可用方法
bool                       PartialSuccessAchieved
```

`AuthAttempt.Outcome` 里的 `SkippedNotOffered` / `SkippedNoMaterial` 是关键：
没有它们，「私钥文件读不出来」和「服务端不接受公钥认证」在 UI 上是同一句
「用户名或密码不正确」。

---

## 六 `SSH_MSG_DISCONNECT`

| # | 类型 | 字段 |
| :-: | --- | --- |
| 1 | `byte` | 1 |
| 2 | `uint32` | 原因码 |
| 3 | `string` | 描述（UTF-8） |
| 4 | `string` | 语言标记 |

| 码 | 名称 |
| :-: | --- |
| 1 | `HOST_NOT_ALLOWED_TO_CONNECT` |
| 2 | `PROTOCOL_ERROR` |
| 3 | `KEY_EXCHANGE_FAILED` |
| 4 | `RESERVED` |
| 5 | `MAC_ERROR` |
| 6 | `COMPRESSION_ERROR` |
| 7 | `SERVICE_NOT_AVAILABLE` |
| 8 | `PROTOCOL_VERSION_NOT_SUPPORTED` |
| 9 | `HOST_KEY_NOT_VERIFIABLE` |
| 10 | `CONNECTION_LOST` |
| 11 | `BY_APPLICATION` |
| 12 | `TOO_MANY_CONNECTIONS` |
| 13 | `AUTH_CANCELLED_BY_USER` |
| 14 | `NO_MORE_AUTH_METHODS_AVAILABLE` |
| 15 | `ILLEGAL_USER_NAME` |

〔决策〕**服务端给的描述文本必须原样保留**（`SshDisconnectException.PeerDescription`）。
它常常是唯一有用的信息（`"Too many authentication failures"`、
`"No supported authentication methods available"`）。
同时它是**不可信文本**，文档里要写明展示时按不可信内容处理。

〔决策〕**我们断开时也发 `DISCONNECT`**，尽力而为。
不发会让服务端日志里只看到一个 TCP reset，管理员无从判断是网络问题还是客户端主动。

---

## 七 度量（`System.Diagnostics.Metrics`）

Meter 名：`VelaShell.Ssh`

| 仪表 | 类型 | 标签 |
| --- | --- | --- |
| `velashell.ssh.connections.active` | UpDownCounter | `host` |
| `velashell.ssh.connect.duration` | Histogram (ms) | `host`、`outcome`、`phase` |
| `velashell.ssh.bytes` | Counter | `host`、`direction` |
| `velashell.ssh.packets` | Counter | `host`、`direction` |
| `velashell.ssh.rekeys` | Counter | `host` |
| `velashell.ssh.channels.active` | UpDownCounter | `host`、`type` |
| `velashell.ssh.channel.window` | Histogram (bytes) | `host`、`type` —— 自适应窗口的实际取值 |
| `velashell.ssh.sftp.inflight` | Histogram | `host` —— 管线深度的实际取值 |
| `velashell.ssh.forward.*` | 见 [`07-forwarding.md`](07-forwarding.md) §5 |

〔决策〕**`host` 标签用「逻辑目标」而不是 IP。** 跳板链上的最终目标才是用户认识的东西。

〔决策〕**不给密钥指纹、用户名之类打标签。** 标签会进时序数据库，基数爆炸是一方面，
把用户名写进可被广泛查询的指标里是另一方面。

---

## 八 追踪（`ActivitySource`）

Source 名：`VelaShell.Ssh`

| Activity | 何时 |
| --- | --- |
| `ssh.connect` | 整个建连，子 span 为各 Phase 与各跳 |
| `ssh.command` | 一次 `RunAsync` |
| `ssh.sftp.<op>` | 单个 SFTP 操作（只在采样命中时建，见下） |

〔决策〕**SFTP 的单操作 span 默认不建。** 一次目录传输会产生上万个操作，
逐个建 Activity 的开销本身就会改变性能特征。默认只在
`SftpFileSystem` 级别建一个聚合 span；需要细粒度时显式打开。

---

## 九 报文旁路 `IPacketTap`

```
interface IPacketTap
{
    void OnPacket(in PacketTapRecord record);
}

readonly struct PacketTapRecord
{
    PacketDirection Direction;     // Inbound / Outbound
    byte            MessageNumber;
    int             Length;        // 载荷长度
    uint            SequenceNumber;
    uint?           ChannelNumber; // 通道消息才有
    ReadOnlySpan<byte> Payload;    // **默认为空**，见下
}
```

**三条硬规则**：

1. **默认不启用，且零开销。** 字段为 `null` 时整段调用被 JIT 消掉。
2. **载荷默认不给。** 要给必须显式设 `TapOptions.IncludePayload = true`，
   且该选项的文档里**必须**写明它会带出密码、密钥与文件内容。
3. **认证阶段的载荷永远不给**，即使 `IncludePayload = true`。
   〔决策〕这一条不提供开关 —— 没有任何排错场景值得把密码打进日志，
   而提供了开关就一定会有人在生产上打开它。

用途：连接诊断面板、协议级排错、录制回放。

---

## 十 日志

用 `ILogger`，但**日志不是 API**：任何使用者需要程序化消费的东西
都必须同时以结构化形式出现在异常、事件或度量里。

| 级别 | 内容 |
| --- | --- |
| `Trace` | 每个报文的元信息（等价于 `IPacketTap` 不含载荷） |
| `Debug` | 状态迁移、协商结果、窗口调整、管线深度变化 |
| `Information` | 连接建立/断开、认证成功、转发启停 |
| `Warning` | 降级（SHA-1 签名、无严格 KEX）、重试、单条转发连接失败 |
| `Error` | 连接级失败 |

〔决策〕**「对端不支持严格 KEX」必须记 Warning。** 它不阻断连接（§03 6），
但使用者应当能看见 —— 这是一条实打实的安全属性缺失。

〔决策〕**热路径的日志用 `LoggerMessage` 源生成器。**
每报文一条 Trace 日志，若走字符串插值会在满速传输时成为主要开销。
