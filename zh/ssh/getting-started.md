# 上手

> 面向使用者的最短路径。设计与取舍的理由在 [`design/architecture.md`](design/architecture.md)，
> 协议层的逐字段规格在 [`spec/`](spec/)。

---

## 一 连上去

```csharp
using VelaShell.Ssh.Auth;
using VelaShell.Ssh.HostKeys;
using VelaShell.Ssh.Session;

var options = new SshConnectionOptions("joe@10.0.0.1:22")
{
    Credentials   = [ new PasswordCredential("hunter2") ],
    HostKeyPolicy = new KnownHostsPolicy(),   // 默认读 ~/.ssh/known_hosts
    KeepAlive     = new KeepAlivePolicy(TimeSpan.FromSeconds(30)),
};

await using SshConnection conn = await options.ConnectAsync(ct);
```

`ConnectAsync` 一口气做完拨号 → 版本交换 → 密钥交换 → **主机密钥裁决** → 认证。

**三把计时器是分开的**，这不是小事：

| 计时器 | 默认 | 管什么 |
| --- | --- | --- |
| `ConnectTimeout` | 30 秒 | 拨号 + 版本交换 + 密钥交换 |
| `HostKeyDecisionTimeout` | **无限** | 主机密钥裁决（要弹窗问用户） |
| `AuthenticationTimeout` | 2 分钟 | 认证（用户可能要去掏手机看动态码） |

裁决的时间**不计入连接超时** —— 否则用户看完指纹点「信任」，这一轮已经被判死，
只能在外面补一次重连，而那次重连会再问一遍同样的问题。

超时时异常会说清**是哪一步**：

```
连 10.0.0.1:22 时在「密钥交换」这一步超时（限 30 秒）。
```

### 经代理、跳板、代理命令

```csharp
using VelaShell.Ssh.Transport;

// SOCKS5（主机名交给代理解析，不在本机解析）
Dialer = DialerChain.Socks5("127.0.0.1", 1080, new SshProxyCredentials("alice", "pw")),

// HTTP 代理的 CONNECT（Basic 认证，第一次请求就带上）
Dialer = DialerChain.HttpConnect("proxy.corp", 3128),

// 嵌套：经 HTTP 代理到达 SOCKS5 代理，再由它连目标
Dialer = DialerChain.Socks5("socks.internal", 1080).Via(DialerChain.HttpConnect("proxy.corp", 3128)),

// 跳板（ssh -J）：跳板本身是一条完整连接，有自己的凭据与主机密钥策略
Dialer = DialerChain.Jump(new SshConnectionOptions("ops@bastion.example.com") { Credentials = [...] }),

// 多级跳板（ProxyJump a,b）：a 最近，b 直接连目标
Dialer = DialerChain.Jumps(bastionOptions, innerOptions),

// ProxyCommand：外部程序的标准输入输出就是那条流（%h %p %r %n %%）
Dialer = DialerChain.Command("cloudflared access ssh --hostname %h"),
```

失败时 `SshConnectException.Hops` 列出**每一跳**的结果（从近到远）——
「连不上代理」「代理拒绝转发」「跳板认证失败」是三个不同的问题，这张表让它们分得开。
原因码：代理拒绝是 `ProxyRefused`，要认证而没凭据（或凭据被拒）是 `ProxyAuthRequired`。

> [!NOTE]
> `ProxyCommand` 在 Windows 上走匿名管道，而匿名管道不支持重叠 IO ——
> 它的异步读写由运行时在线程池线程上阻塞完成（平台限制）。要纯异步的链路，用另外三种。

主机密钥裁决（弹窗问用户）期间，连接计时器**停表**；经跳板时，里面那一跳在裁决时外层也停表。

---

## 二 主机密钥

`HostKeyPolicy` 是**必须想清楚**的一项。默认值是
`KnownHostsPolicy { UnknownHost = Reject }` —— 没见过的主机直接拒。
交互式客户端要换成带询问回调的那一种：

```csharp
HostKeyPolicy = new KnownHostsPolicy(askUser: async (ctx, ct) =>
{
    // ctx.Key.Sha256Fingerprint 就是 ssh 命令行显示的那一串
    return await ui.ConfirmAsync(
        $"{ctx.Target} 的主机密钥指纹是 {ctx.Key.Sha256Fingerprint}，信任吗？", ct);
})
```

**「没见过」与「密钥变了」走两条不同的路**：

- 没见过 → 可以问、可以记（TOFU）。
- **变了 → 直接拒绝，而且没有「记住新的」这条捷径。**
  它可能是中间人，也可能只是服务器重装了，而这两者在协议层无法区分。
  要接受新密钥，得先手动把 `known_hosts` 里旧的那一行删掉 ——
  那一下手动操作正是让人停下来想一想的地方。异常消息里会指出是第几行。

自动化场景用指纹钉死，连第一次都不盲信：

```csharp
HostKeyPolicy = new PinnedFingerprintHostKeyPolicy(["SHA256:abc..."])
```

---

## 三 凭据

**库不做任何隐式回退** —— 不自动读 `~/.ssh/id_*`，不自动连 ssh-agent。
要用就显式加进来：

```csharp
using VelaShell.Ssh.Keys;

ISshSigner key = await SshPrivateKeyFile.LoadAsync("~/.ssh/id_ed25519", passphrase: null, ct);

await using SshAgentClient agent = await SshAgentClient.ConnectAsync(cancellationToken: ct);
IReadOnlyList<SshCredential> fromAgent = await agent.GetCredentialsAsync(ct);

Credentials =
[
    new PublicKeyCredential(key, "~/.ssh/id_ed25519"),
    .. fromAgent,
    new KeyboardInteractiveCredential(PromptAsync),   // 2FA / 动态码
    new PasswordCredential(AskPasswordAsync),
];
```

顺序就是尝试顺序。服务端不接受的方法会被跳过并如实记进尝试记录 ——
失败时 `SshAuthenticationException.DescribeAttempts()` 会告诉你每一条发生了什么：

```
· none（探测可用方法）：失败 —— 服务端接受：publickey, keyboard-interactive
· ~/.ssh/id_ed25519：部分成功（服务端要求继续）
· keyboard-interactive：跳过（服务端不接受这种方法）
```

> **多因素（2FA）是自动支持的。**SSH 里没有「2FA 报文」——
> 它就是「这一步过了，但还要再来一种」。库会把它当成成功并继续，
> 而不是当成失败停下来。

### 加密的私钥

**都能读，传 `passphrase` 就行**：PKCS#8（`BEGIN ENCRYPTED PRIVATE KEY`）、
OpenSSH 格式（`BEGIN OPENSSH PRIVATE KEY` + `cipher != none`，也就是
`ssh-keygen` 带口令时的默认产物）、以及 `.ppk` v2/v3。

```csharp
ISshSigner key = await SshPrivateKeyFile.LoadAsync("~/.ssh/id_ed25519", "口令", ct);
```

OpenSSH 加密私钥支持的算法：`aes{128,192,256}-{ctr,cbc}`、
`aes{128,256}-gcm@openssh.com`、`chacha20-poly1305@openssh.com`。
**没有 `3des-cbc`** —— 它只有 `ssh-keygen -Z` 才选得到，为读一种没人用的格式
在安全库里带上 3DES 不划算。遇到时异常消息里会告诉你怎么换。

口令不对时抛 `SshPrivateKeyException` 且 `NeedsPassphrase` 为 `true` ——
界面靠这个标志决定要不要再弹一次输入框。**有没有认证标签的算法给的是同一种结论**：
`aes256-gcm` 在验标签时就判死，`aes256-ctr` 要到校验字才露馅，但对调用方是一样的。

> 这一处是本库**唯一**自己实现的密码学代码（`bcrypt_pbkdf`）。
> 为什么破例、范围钉在哪、怎么验的，见 [架构文档 §11.2.18](design/architecture.md)。

### OpenSSH 用户证书

证书认证**没有单独的认证方法** —— 它走的还是 `publickey`，只是出示的是整张证书：

```csharp
using VelaShell.Ssh.Keys;

ISshSigner key = await SshPrivateKeyFile.LoadAsync("~/.ssh/id_ed25519", null, ct);
OpenSshCertificate cert = await OpenSshCertificate.LoadAsync("~/.ssh/id_ed25519-cert.pub", ct);

Credentials = [ new PublicKeyCredential(SshCertificateSigner.Create(cert, key)) ];
```

`Create` **当场核对证书与私钥是不是一对**，配错了立刻报 —— 不核对的话，
表现是服务端一句 `Permission denied (publickey)`，而那句话与
「CA 不被信任」「主体不匹配」「证书过期」长得一模一样。

证书里的事实都交出来了，可以直接拿去显示或排障：

```csharp
Console.WriteLine(cert.KeyId);                      // CA 写的标识串，出现在服务端日志里
Console.WriteLine(string.Join(", ", cert.ValidPrincipals));   // 空 = 对所有主体有效
Console.WriteLine(cert.ValidBeforeTime);            // null = 不过期
Console.WriteLine(cert.SignatureKey?.Sha256Fingerprint);      // 签发它的 CA
if (!cert.IsTimeValid(DateTimeOffset.UtcNow))
    // 在界面上直说「证书过期了」，而不是让用户对着 Permission denied 猜
```

> **本库不验证 CA 签名**，这是有意的：验证是服务端的事，客户端验了也不改变结果 ——
> 服务端照样要自己验一遍，而客户端这边根本没有「哪些 CA 可信」那份名单。

### PuTTY 的 `.ppk`

`.ppk` v2 与 v3 都能读，**包括加密的**：

```csharp
ISshSigner key = await SshPrivateKeyFile.LoadAsync("key.ppk", passphrase: "口令", ct);
```

> `.ppk` 的 KDF（v2 是 SHA-1 的拼接、v3 是 Argon2id）BouncyCastle 直接给，
> 所以它一直就能读；OpenSSH 的 `bcrypt_pbkdf` 谁都不给，所以那一条是自己写的
> —— 全库唯一的一处，见上面那个注。

---

## 四 跑命令

```csharp
SshCommandOutput r = await conn.RunAsync("uname -a", cancellationToken: ct);
Console.WriteLine(r.StandardOutput);
r.EnsureSuccess("uname -a");   // 失败时异常里带着 stderr
```

`ExitCode` 是 **`int?`**，不是 `int`：

| 情况 | `ExitCode` | `ExitSignalName` |
| --- | :-: | :-: |
| 正常退出 | 有值 | `null` |
| 被信号杀死 | **`null`** | `"KILL"` |
| 连接中断 | **`null`** | `null` |

**不把信号编成 128+n 这样的伪退出码** —— 那是 shell 的约定，不是 SSH 的；
伪造它会让「进程返回 137」和「进程被 KILL」无法区分。

要流式处理输出就自己拿通道：

```csharp
await using SshCommand cmd = await conn.ExecuteAsync("tail -F /var/log/x", cancellationToken: ct);
await foreach (var line in ReadLinesAsync(cmd.StandardOutput, ct)) { }
await cmd.SendSignalAsync("TERM", ct);    // 注意：不带 SIG 前缀
```

---

## 五 交互式 shell

```csharp
await using SshShell shell = await conn.OpenShellAsync(new SshShellOptions
{
    TerminalType = "xterm-256color",
    Size  = new TerminalSize(cols, rows, pixelWidth, pixelHeight),
    Modes = TerminalModes.Empty.Set(TerminalModeOpcode.Utf8Input, 1),
}, ct);

await shell.ResizeAsync(new TerminalSize(cols, rows, pixelWidth, pixelHeight), ct);
```

**像素尺寸是一等公民**，不恒为 0 —— sixel、kitty 图形协议这类东西要靠它排版。
不知道就给 0，那也是一个有意义的回答。

`SshShell` **不暴露 `StandardError`**：有 pty 时 stderr 由伪终端合并进 stdout，
暴露一条永远空的流只会让人对着它干等。

要在 shell 上开 X11 / agent 转发，直接写在选项里（请求顺序
`pty-req → x11-req → auth-agent-req → env → shell` 由库负责）：

```csharp
await using SshShell shell = await conn.OpenShellAsync(new SshShellOptions
{
    X11             = new X11ForwardOptions(),        // ssh -X
    AgentForwarding = AgentForwardPolicy.Default,     // ssh -A
    BeforeStart     = (channel, ct) => SendMyCustomRequestAsync(channel, ct),   // 库没内置的请求
}, ct);
```

`SshExecutionOptions`（一次性命令）有同样的三项。

---

## 六 SFTP

```csharp
using VelaShell.Ssh.Sftp;

await using SftpFileSystem fs = await SftpFileSystem.ConnectAsync(conn, cancellationToken: ct);

Console.WriteLine(fs.WorkingDirectory);   // 来自 REALPATH "."，比拼 /home/{user} 靠谱

await foreach (SftpDirectoryEntry e in fs.EnumerateDirectoryAsync("/etc", ct))
{
    Console.WriteLine($"{e.Name}\t{e.Length}\t{(e.IsSymbolicLink ? "→ " + e.LinkTarget : "")}");
}

byte[] content = await fs.ReadAllBytesAsync("/etc/hostname", ct);
await fs.WriteAllBytesAsync("/tmp/x", content, cancellationToken: ct);
```

### 断点续传 —— 用 `DurableLength`，不要用文件长度

```csharp
await using SftpFileStream s = await fs.OpenWriteAsync("/tmp/big.bin", cancellationToken: ct);
try
{
    await s.WriteAsync(data, ct);
    await s.FlushAsync(ct);
}
catch (SftpTransferInterruptedException ex)
{
    // 从这里续，精确 —— 不用像常见做法那样从文件长度盲退 2 MiB
    long resumeFrom = ex.DurableLength;
}
```

流水线写入时应答顺序不保证，所以**服务端报告的文件长度只是「已确认的最高偏移」**，
它前面可能留着读作 0 的空洞。`DurableLength` 是「从 0 开始**连续**已确认」的长度。

需要「任何时刻文件都是完整前缀」就用 `SftpWriteMode.Sequential`（牺牲吞吐换确定性）。

### 能力要先问

```csharp
if (fs.Capabilities.HasPosixRename)
    await fs.RenameAsync(a, b, overwrite: true, ct);   // 原子覆盖
else
    // 这台服务器做不到原子覆盖 —— 在界面上说出来，而不是悄悄换成非原子的做法
```

### 只有异步

`SftpFileStream` 的同步 `Read` / `Write` / `SetLength` 会抛 `NotSupportedException` ——
同步版本只能靠阻塞线程等网络往返实现。同步 `Flush` 是不阻塞的空操作（本端没有缓冲），
同步 `Dispose` 把收尾交给后台、立刻返回；要看到收尾时的错误就用 `await using`。

---

## 七 压缩

**默认不开。**要开就显式写：

```csharp
var options = new SshConnectionOptions("root@example.com")
{
    Algorithms = SshAlgorithmSet.Default.WithCompression(),   // zlib@openssh.com
};
```

两件事值得先知道：

1. **交互式会话开它基本是净亏。** 终端输出本来就小，压缩省下的字节
   抵不上两端的 CPU。**压不动的内容（已压缩的文件、加密数据）压完还会变大。**
   值得开的场景是 SFTP 传文本、或者链路按流量计费。
2. **`zlib@openssh.com` 等认证过了才开始压缩**，这是它与老的 `zlib` 唯一的区别。
   原因是压缩后的长度会泄露明文的可压缩性 —— 在认证阶段，这等于给口令
   开了一条区分信道（CRIME 那一类）。所以我们只提供前者。

**压缩是否真的谈成了，要自己查** —— 服务端不支持时会静默落到 `none`，
**不会报错**：

```csharp
Console.WriteLine(conn.Algorithms?.CompressionServerToClient);   // zlib@openssh.com 或 none
```

`Algorithms` 交出的是这条会话实际协商出来的全部算法（密钥交换、主机密钥、
两个方向的加密与完整性、两个方向的压缩、是否启用了严格 KEX）。
状态栏上要显示「连上了什么」、排障时要回答「这条连接到底谈成了什么」，
都从这里取，不用再自己探一次。

---

## 八 转发

```csharp
using VelaShell.Ssh.Forwarding;

// -L：本机 127.0.0.1:8080 → 远端 10.0.0.9:80
await using PortForwarder local = PortForwarder.StartLocal(
    conn, "10.0.0.9", 80, new PortForwardOptions { BindPort = 8080 });

// -D：本机 SOCKS5
await using PortForwarder socks = PortForwarder.StartDynamic(conn);

// -R：服务端监听 → 回连到本机
await using RemoteForwarder remote = await RemoteForwarder.StartAsync(
    conn, "127.0.0.1", 3000,
    new RemoteForwardOptions { BindAddress = "localhost", BindPort = 0 }, ct);
Console.WriteLine(remote.BoundPort);   // 请求 0 时，服务端分配的实际端口在这里

// 计量在库里
Console.WriteLine($"{local.ActiveConnections} 条 · 上行 {local.BytesUp} B · 下行 {local.BytesDown} B");
local.ConnectionClosed += (_, e) => log.Info($"{e.Target} 传了 {e.BytesUp + e.BytesDown} 字节");
```

**默认绑环回。**要对外开放必须显式写 `BindAddress = IPAddress.Any` ——
一条隧道的另一端往往是内网数据库，默认绑 `0.0.0.0` 等于把它暴露给同网段所有人。

### 反方向的 Unix 套接字（`ssh -R /远端:/本机`）

```csharp
await using RemoteForwarder sock = await RemoteForwarder.StartUnixSocketAsync(
    conn,
    targetSocketPath: "/var/run/docker.sock",   // 本机的
    remoteSocketPath: "/tmp/docker.sock",       // 服务端要建的
    cancellationToken: ct);

Console.WriteLine(sock.RemoteEndpointName);   // 两种形态统一的说法
```

**走套接字而不是端口，远端机器上的其它用户就看不到也连不上** —— 谁能连由文件权限说了算，
而一个 `-R 2375:...` 的端口是同机所有人都能连的。

> 服务端会在它自己的文件系统上创建那个套接字文件。路径已存在时 OpenSSH 会拒绝，
> 所以这里的失败多半是「上一次没清干净」。

### X11 转发（`ssh -X` / `-Y`）

```csharp
await using SshChannel session = await conn.OpenSessionChannelAsync(null, ct);

await using X11Forwarder x11 = await X11Forwarder.RequestAsync(conn, session,
    new X11ForwardOptions
    {
        // Display = X11Display.Parse(":0"),   // 默认读 DISPLAY
        Trusted = false,                        // 默认：对应 ssh -X
        Timeout = TimeSpan.FromMinutes(20),     // Zero = 不过期
    },
    ct);

Console.WriteLine($"{x11.AcceptedChannels} 条接受 · {x11.RejectedChannels} 条拒绝");
```

> [!WARNING]
> **X11 没有客户端隔离。** 连上同一个显示的任何客户端都能读别人的按键、
> 抓别人的窗口、往别人的窗口里塞事件 —— 所以把本机显示交给远端，
> 等于把本机**所有**图形会话的输入输出交给远端。
> 默认关、默认非受信，都是因为这一条。

三件事值得知道：

1. **发给服务端的永远是一个随机的假 cookie**，真 cookie 一步都不离开本机。
   远端 X 客户端拿假 cookie 连过来，我们核对（常数时间比较）、换成真 cookie，
   才转给本机 X server。对不上就拒绝，而且**连碰都不碰** X server。
2. **非受信模式（默认）要本机有 `xauth`**，且 X server 支持 SECURITY 扩展。
   Windows 上通常两者都没有 —— 那里只能用 `Trusted = true`，
   但要清楚那等于把本机显示完全交给远端。
3. **`Timeout` 两种模式都生效**（OpenSSH 只管非受信）。
   受信模式恰恰更危险，却反而没期限，说不通。长会话显式设 `TimeSpan.Zero`。

服务端那边需要 `sshd_config` 里 `X11Forwarding yes` 且装了 `xauth`；
没有的话请求会被拒，异常消息里直接写着这两条。

另外两点：

- **非受信模式生成的受限 cookie 写进一个临时文件**（`xauth -f`），用完即删，
  **绝不写进你的 `.Xauthority`** —— 否则它会覆盖本机显示的完全授权 cookie，
  到期之后你自己本机的 X 程序就连不上自己的显示了。
- **同一条连接上可以有多个会话各开各的 X11 转发**：`x11` 通道按假 cookie 分给对应的那个，
  释放一个不影响其它的。最常见的用法是直接写在 shell 选项里（见 §五）。

### 不开监听端口的隧道


接 `/var/run/docker.sock` 或内网 API 时，**本机不需要一个监听端口**：

```csharp
await using SshChannel tunnel = await conn.OpenUnixSocketTunnelAsync("/var/run/docker.sock", ct);
// tunnel.StandardInput / tunnel.StandardOutput 就是那条流
```

开一个 `-L` 监听意味着同机任何进程都能连上去。对 root 等价的端点，
这个区别不是优化而是前提。

### Agent 转发（`ssh -A`）

**默认是关的，而且没有「全局打开」的开关** —— 它是逐连接、逐通道的决定：

```csharp
await using SshChannel session = await conn.OpenSessionChannelAsync(null, ct);

await using AgentForwarder fwd = await AgentForwarder.RequestAsync(
    conn, session,
    new AgentForwardPolicy
    {
        AllowedKeys = [deployKey],              // 只转发这一把，其余的对远端不可见
        ConfirmEachSignature = AskUserAsync,    // 每次签名都问一下人
        MaxConcurrentChannels = 4,
    },
    cancellationToken: ct);
```

`ssh -A` 的名声不好是有原因的：**远端 root 能借你的 agent 以你的身份登录
任何地方**，而你看不见。所以这里不是把 agent 通道当字节管子对接过去，
而是**把 agent 协议解析一遍再转发**——只有解析了，才谈得上：

- `AllowedKeys` 非空时，远端列钥时**看不到**名单外的钥（`fwd.KeysHidden` 会计数），
  要名单外的钥签名直接回 `FAILURE`；
- `ConfirmEachSignature` 拿到的是「哪把钥、注释是什么」，可以弹窗问人，拒了就回 `FAILURE`；
- `ADD_IDENTITY` / `LOCK` / `UNLOCK` 这类**会改本机 agent 状态**的消息一律拒绝，
  不转发 —— 远端服务器没有任何理由改我们本机的钥圈。

`fwd.SignatureRequests` / `SignaturesDenied` / `IdentityListings` 可以直接拿去显示或审计。

---

## 九 读 `ssh_config`

```csharp
// LoadAsync 会展开 Include；Parse 只解析文本，不碰文件系统
IReadOnlyList<SshConfigBlock> blocks = await SshConfigFile.LoadAsync(cancellationToken: ct);

SshHostConfig cfg = SshConfigFile.Resolve(blocks, new SshConfigMatchContext
{
    Host = "prod-web-1",
    User = "deploy",
    LocalUser = Environment.UserName,
});

Console.WriteLine(cfg.HostName);       // HostName 改写之后的
Console.WriteLine(cfg.Port);           // 没写就是 22
Console.WriteLine(cfg.IdentityFiles);  // 可能有多条
```

**库只解析，不替你决定。** 解出来的结果交给调用方 —— 库自作主张去读
`~/.ssh/config` 会让「为什么连的不是我写的那台机器」变成一个很难查的问题。

决定用它的话，一步变成可以直接拿去连的参数：

```csharp
SshConnectionOptions options = await SshConfigFile.CreateConnectionOptionsAsync(
    blocks, "prod-web-1",
    new SshConfigConnectSettings
    {
        Credentials        = [new KeyboardInteractiveCredential(PromptAsync)],   // 排在 IdentityFile 之后
        PassphraseProvider = (path, ct) => AskPassphraseAsync(path, ct),         // 加密的 IdentityFile
        AskUnknownHost     = ConfirmFingerprintAsync,
    }, ct);

await using SshConnection conn = await options.ConnectAsync(ct);

// ForwardAgent / ForwardX11 是会话项，不是连接项：
await using SshShell shell = await conn.OpenShellAsync(SshConfigFile.Resolve(blocks, "prod-web-1").ApplyToShell(), ct);
```

`HostName` / `Port` / `User` / `IdentityFile` / `Compression` / `ServerAlive*` / `ConnectTimeout` /
`StrictHostKeyChecking` / `UserKnownHostsFile` / `ProxyJump` / `ProxyCommand` 都会落到连接参数上；
`ProxyJump` 上的每个跳板**按同一份配置解析**，跳板链有深度上限并检测环。
逐项规则见 [`spec/09-dialing.md`](spec/09-dialing.md) §7。

两条安全约束值得知道：

1. **`Match exec` 默认不执行任何命令。** 它意味着*解析一份配置文件就能在本机跑任意程序*，
   而配置文件常常是从别处拷来的、同步过来的、别人给的。没有求值器时，
   带 `exec` 的块**一律不匹配**。真要用就自己传：

   ```csharp
   new SshConfigMatchContext { Host = h, ExecEvaluator = cmd => RunAndCheck(cmd) }
   ```

   这样「要不要跑外部命令」这个决定明确地落在你身上，而不是藏在库的默认行为里。
2. **`Include` 有深度上限（16）与环检测。** `a` include `b`、`b` 又 include `a`
   很容易写出来，而没有环检测的表现是*读配置的时候整个进程不动了*。

其余细节：`Match` 支持 `all` / `host` / `originalhost` / `user` / `localuser`，
条件之间是与，支持 `!` 取反；`canonical` / `final` 永远不匹配（我们不做主机名规范化）。
**信息不足就不匹配** —— 不知道用户时 `Match user` 不会猜一个。

---

## 十 密钥重协商


**默认就是开着的，通常不用管它。** 写在这里是因为它偶尔要调。

```csharp
var options = new SshConnectionOptions("root@example.com")
{
    // 默认：1 GiB / 1 小时 / 2³¹ 个报文，任一条到了就换
    Rekey = SshRekeyPolicy.Default,
};

// 也可以自己挑时机
await conn.StartRekeyAsync(ct);

Console.WriteLine(conn.RekeyCount);        // 换过几次
Console.WriteLine(conn.LastRekeyReason);   // 上次是哪条阈值触发的
```

三件事值得知道：

1. **接住对端发起的重协商永远开着，关不掉。** `SshRekeyPolicy.Disabled`
   只关「我们主动发起」。这不是遗漏 —— OpenSSH 的 `RekeyLimit` 默认
   1 GiB / 1 小时，到点它自己发 `KEXINIT`，不应答的表现是
   **挂了一下午的 shell 忽然断了**、**传到一半的大文件断了**。
2. **报文数那条阈值别调低。** SSH 的序号是 32 位的，AES-GCM 的 nonce
   每个报文推进一次 —— nonce 重用对 GCM 是灾难性的（可恢复认证密钥）。
   它是密码学硬约束，不是调优旋钮。
3. **阈值有下限**（64 MiB / 1 分钟 / 1024 个报文），低于下限会当场抛
   `ArgumentOutOfRangeException`。每次重协商都要做一次非对称运算，
   调得太频繁就是自己给自己开的拒绝服务面。

重协商期间**通道不中断**：收方向照常收数据，发方向的通道数据会暂存，
谈完之后按原顺序流出（RFC 4253 §7.1 只许这期间发传输层消息）。

---

## 十一 失败怎么读


所有异常都从 `SshException` 派生，带 `Reason`（可判定的原因码）与 `Phase`（哪一步）。

| 异常 | 什么时候 | 关键字段 |
| --- | --- | --- |
| `SshConnectException` | 拨号 / 握手阶段 | `Reason`：`DnsFailure` / `TcpRefused` / `TcpTimeout` / … |
| `SshAuthenticationException` | 认证失败 | `Attempts`、`ServerOffered`、`PartialSuccessAchieved` |
| `SshChannelException` | 通道打不开 | `OpenFailureReason` + 可操作的提示 |
| `SftpException` | SFTP 操作失败 | **`ServerMessage`（服务端原话）** |
| `SftpTransferInterruptedException` | 传输中断 | `DurableLength` |
| `SshForwardException` | 转发器起不来 | — |

`SftpException.ServerMessage` 要单独说一句：SFTP v3 只有 9 个状态码，
而码 `4` 承载了绝大多数真实错误 ——「目录非空」「文件已存在」「磁盘满」「配额超限」
**全是同一个码**。服务端给的那段文本是唯一能区分它们的信息。

**通道是独立的失败域**：一条通道打不开或出错，会话仍然可用。
转发器同理 —— 单条连接失败只触发 `Error` 事件，转发器继续跑。
