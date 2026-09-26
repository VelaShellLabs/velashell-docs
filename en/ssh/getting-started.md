# Getting Started

> The shortest path for users. The reasoning behind the design and its trade-offs is in [`design/architecture.md`](design/architecture.md);
> the field-by-field protocol-level specifications are in [`spec/`](spec/).
>
> 中文：[`../../zh/ssh/getting-started.md`](../../zh/ssh/getting-started.md)

---

## 1. Connecting

```csharp
using VelaShell.Ssh.Auth;
using VelaShell.Ssh.HostKeys;
using VelaShell.Ssh.Session;

var options = new SshConnectionOptions("joe@10.0.0.1:22")
{
    Credentials   = [ new PasswordCredential("hunter2") ],
    HostKeyPolicy = new KnownHostsPolicy(),   // reads ~/.ssh/known_hosts by default
    KeepAlive     = new SshKeepAlivePolicy(TimeSpan.FromSeconds(30)),
};

await using SshConnection conn = await SshConnection.ConnectAsync(options, ct);
```

`SshConnection.ConnectAsync` does dialing → version exchange → key exchange → **host key decision** → authentication in one go,
and the connection it returns is ready to use — `SshConnection` has no public constructor and no separate `Start` to call; this is the only way to connect.

**The three timers are separate**, and this is no small thing:

| Timer | Default | What it covers |
| --- | --- | --- |
| `ConnectTimeout` | 30 seconds | Dialing + version exchange + key exchange |
| `HostKeyDecisionTimeout` | **Infinite** | Host key decision (a dialog asks the user) |
| `AuthenticationTimeout` | 2 minutes | Authentication (the user may need to dig out their phone for a one-time code) |

The decision time is **not counted against the connect timeout** — otherwise, by the time the user has read the fingerprint and clicked "Trust", this round would already have been declared dead,
and a reconnect would have to be done outside, which would ask the same question all over again.

On timeout, the exception makes clear **which step** it was:

```
Timed out at the "key exchange" step while connecting to 10.0.0.1:22 (limit 30 seconds).
```

### Via proxies, jump hosts and proxy commands

```csharp
using VelaShell.Ssh.Transport;

// SOCKS5 (the host name is resolved by the proxy, not locally)
Dialer = DialerChain.Socks5("127.0.0.1", 1080, new SshProxyCredentials("alice", "pw")),

// HTTP proxy CONNECT (Basic authentication, sent with the very first request)
Dialer = DialerChain.HttpConnect("proxy.corp", 3128),

// Nesting: reach the SOCKS5 proxy via the HTTP proxy, then let it connect to the target (via = how to reach the proxy itself)
Dialer = DialerChain.Socks5("socks.internal", 1080, via: DialerChain.HttpConnect("proxy.corp", 3128)),

// Jump host (ssh -J): the jump host is itself a full connection, with its own credentials and host key policy
Dialer = DialerChain.Jump(new SshConnectionOptions("ops@bastion.example.com") { Credentials = [...] }),

// Multi-level jump (ProxyJump a,b): a is nearest, b connects directly to the target
Dialer = DialerChain.Jumps(bastionOptions, innerOptions),

// ProxyCommand: the external program's standard input and output are the stream (%h %p %r %n %%)
Dialer = DialerChain.Command("cloudflared access ssh --hostname %h"),
```

On failure, `SshConnectException.Hops` lists the result of **every hop** (nearest to farthest) —
"cannot reach the proxy", "the proxy refused to forward" and "jump host authentication failed" are three different problems, and this table keeps them apart.
Reason codes: a proxy refusal is `ProxyRefused`; authentication required with no credentials (or credentials rejected) is `ProxyAuthRequired`.

> [!NOTE]
> On Windows, `ProxyCommand` goes through anonymous pipes, and anonymous pipes do not support overlapped IO —
> its asynchronous reads and writes are completed by the runtime by blocking on thread-pool threads (a platform limitation). For a purely asynchronous path, use one of the other three.

During the host key decision (the dialog asking the user), the connect timer is **paused**; when going through a jump host, the outer timer is also paused while the inner hop is deciding.

---

## 2. Host keys

`HostKeyPolicy` is the one setting you **must think through**. The default is
`KnownHostsPolicy { UnknownHost = Reject }` — hosts never seen before are rejected outright.
Interactive clients should switch to the variant with an ask callback:

```csharp
HostKeyPolicy = new KnownHostsPolicy(askUnknownHost: async (ctx, ct) =>
{
    // ctx.Key.Sha256Fingerprint is the same string the ssh command line shows
    return await ui.ConfirmAsync(
        $"The host key fingerprint of {ctx.Target} is {ctx.Key.Sha256Fingerprint}. Trust it?", ct);
})
```

**"Never seen" and "key changed" take two different paths**:

- Never seen → may ask, may remember (TOFU).
- **Changed → rejected outright, with no "remember the new one" shortcut.**
  It may be a man-in-the-middle, or the server may simply have been reinstalled, and the two cannot be distinguished at the protocol level.
  To accept the new key, first manually delete the old line from `known_hosts` —
  that manual step is exactly the point that makes people stop and think. The exception message tells you which line it is.

For automation, pin the fingerprint so that not even the first connection is trusted blindly:

```csharp
HostKeyPolicy = new PinnedFingerprintHostKeyPolicy(["SHA256:abc..."])
```

---

## 3. Credentials

**The library does no implicit fallback** — it does not automatically read `~/.ssh/id_*`, and does not automatically connect to ssh-agent.
If you want them, add them explicitly:

```csharp
using VelaShell.Ssh.Keys;

// The path goes to the file system as is; ~ is not expanded
string keyPath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.UserProfile), ".ssh", "id_ed25519");
using InMemorySshSigner key = await SshPrivateKeyFile.LoadAsync(keyPath, passphrase: null, ct);

await using SshAgentClient agent = await SshAgentClient.ConnectAsync(cancellationToken: ct);
IReadOnlyList<SshCredential> fromAgent = await agent.GetCredentialsAsync(ct);

Credentials =
[
    new PublicKeyCredential(key, "~/.ssh/id_ed25519"),   // the second argument is only the label shown in the attempt record
    .. fromAgent,
    new KeyboardInteractiveCredential(PromptAsync),   // 2FA / one-time code
    new PasswordCredential(AskPasswordAsync),
];
```

`LoadAsync` returns an `InMemorySshSigner`: it holds the private key in memory and **zeroes it on dispose**, so take it with `using`.
The connection only needs it during authentication; once connected it can be disposed.

The order is the order of attempts. Methods the server does not accept are skipped and faithfully recorded in the attempt record —
on failure, `SshAuthenticationException.DescribeAttempts()` tells you what happened with each one:

```
· none (probing available methods): failed — server accepts: publickey, keyboard-interactive
· ~/.ssh/id_ed25519: partial success (server requires more)
· keyboard-interactive: skipped (server does not accept this method)
```

> **Multi-factor (2FA) is supported automatically.** There is no "2FA message" in SSH —
> it is simply "this step passed, but one more kind is needed". The library treats it as success and continues,
> rather than treating it as failure and stopping.

### Encrypted private keys

**All of them can be read; just pass `passphrase`**: PKCS#8 (`BEGIN ENCRYPTED PRIVATE KEY`),
the OpenSSH format (`BEGIN OPENSSH PRIVATE KEY` + `cipher != none`, i.e. the default output of
`ssh-keygen` with a passphrase), and `.ppk` v2/v3.

```csharp
using InMemorySshSigner key = await SshPrivateKeyFile.LoadAsync(keyPath, "passphrase", ct);
```

Algorithms supported for OpenSSH encrypted private keys: `aes{128,192,256}-{ctr,cbc}`,
`aes{128,256}-gcm@openssh.com`, `chacha20-poly1305@openssh.com`.
**No `3des-cbc`** — it can only be selected via `ssh-keygen -Z`, and carrying 3DES in a security library
to read a format nobody uses is not worth it. When encountered, the exception message tells you how to convert.

A wrong passphrase throws `SshPrivateKeyException` with `NeedsPassphrase` set to `true` —
the UI uses this flag to decide whether to show the input box again. **Algorithms with and without an authentication tag give the same verdict**:
`aes256-gcm` fails when verifying the tag, `aes256-ctr` only gives itself away at the check bytes, but to the caller it is the same.

> This is the **only** cryptographic code this library implements itself (`bcrypt_pbkdf`).
> Why the exception was made, where its scope is pinned, and how it was verified: see [architecture document §11.2.18](design/architecture.md).

### OpenSSH user certificates

Certificate authentication **has no separate authentication method** — it still goes through `publickey`, only presenting the whole certificate:

```csharp
using VelaShell.Ssh.Keys;

using InMemorySshSigner key = await SshPrivateKeyFile.LoadAsync(keyPath, null, ct);
OpenSshCertificate cert = await OpenSshCertificate.LoadAsync(keyPath + "-cert.pub", ct);

Credentials = [ new PublicKeyCredential(SshCertificateSigner.Create(cert, key)) ];
```

`Create` **checks on the spot that the certificate and private key are a pair**, and reports a mismatch immediately — without that check,
what you would see is a single `Permission denied (publickey)` from the server, which looks exactly the same as
"CA not trusted", "principal mismatch" or "certificate expired".

The facts in the certificate are all exposed and can be used directly for display or troubleshooting:

```csharp
Console.WriteLine(cert.KeyId);                      // the identifier written by the CA; appears in server logs
Console.WriteLine(string.Join(", ", cert.ValidPrincipals));   // empty = valid for all principals
Console.WriteLine(cert.ValidBeforeTime);            // null = never expires
Console.WriteLine(cert.SignatureKey?.Sha256Fingerprint);      // the CA that issued it
if (!cert.IsTimeValid(DateTimeOffset.UtcNow))
    // say "the certificate has expired" in the UI plainly, instead of leaving the user to guess at Permission denied
```

> **This library does not verify the CA signature**, and that is deliberate: verification is the server's job, and client-side verification would not change the outcome —
> the server has to verify it again anyway, and the client has no list of "which CAs are trusted" in the first place.

### PuTTY `.ppk`

Both `.ppk` v2 and v3 can be read, **including encrypted ones**:

```csharp
using InMemorySshSigner key = await SshPrivateKeyFile.LoadAsync("key.ppk", passphrase: "passphrase", ct);
```

> The `.ppk` KDFs (SHA-1 concatenation for v2, Argon2id for v3) are provided directly by BouncyCastle,
> so it could always be read; nobody provides OpenSSH's `bcrypt_pbkdf`, so we wrote that one ourselves
> — the only instance in the whole library, see the note above.

### Adding a private key to the agent (`ssh-add`)

Decrypt an encrypted private key once and hand it to the agent; authentication and forwarding are then signed through the agent:

```csharp
using InMemorySshSigner key = await SshPrivateKeyFile.LoadAsync(keyPath, passphrase: "passphrase", ct);

await using SshAgentClient agent = await SshAgentClient.ConnectAsync(cancellationToken: ct);
await agent.AddIdentityAsync(key, "~/.ssh/id_ed25519", cancellationToken: ct);

// ssh-add -t 3600 -c: deleted automatically after an hour, every signature must be confirmed
await agent.AddIdentityAsync(key, "~/.ssh/id_ed25519",
    new SshAgentKeyConstraints { Lifetime = TimeSpan.FromHours(1), ConfirmEachUse = true }, ct);
```

- Only in-process private keys (`InMemorySshSigner`) are accepted; adding certificates is not supported yet.
- **The library never adds keys on its own**; when to put something into the user's agent is the caller's decision.
- Some agents do not support constraints and reject the whole request (`SshAgentException`). How long an added key lives is up to the agent —
  the Windows OpenSSH agent stores it in the registry, so it survives a reboot. Specification: [spec/07 §7.3](spec/07-forwarding.md).

---

## 4. Running commands

```csharp
SshCommandResult r = await conn.RunAsync("uname -a", cancellationToken: ct);
Console.WriteLine(r.StandardOutput);
r.EnsureSuccess("uname -a");   // on failure, throws SshCommandFailedException (Reason = CommandFailed) carrying stderr
```

`SshCommandResult` is the **complete** result of one run: `StandardOutput`, `StandardError`, and the exit status `ExitStatus` (`SshExitStatus`).
`ExitCode` is **`int?`**, not `int`:

| Situation | `ExitCode` | `ExitSignalName` |
| --- | :-: | :-: |
| Normal exit | Has a value | `null` |
| Killed by a signal | **`null`** | `"KILL"` |
| Connection interrupted | **`null`** | `null` |

**Signals are not encoded as pseudo exit codes like 128+n** — that is a shell convention, not an SSH one;
faking it would make "the process returned 137" and "the process was KILLed" indistinguishable.

To process output as a stream, take the channel yourself:

```csharp
await using SshCommand cmd = await conn.ExecuteAsync("tail -F /var/log/x", cancellationToken: ct);
await foreach (var line in ReadLinesAsync(cmd.StandardOutput, ct)) { }
await cmd.SendSignalAsync("TERM", ct);    // note: no SIG prefix
SshExitStatus status = await cmd.WaitAsync(ct);
```

When you are done writing to `cmd.StandardInput`, **completing it is EOF**: `await cmd.StandardInput.CompleteAsync()` and
`await cmd.CompleteStandardInputAsync(ct)` do the same thing — both flush what was written and then send `CHANNEL_EOF`,
so remote programs waiting for end of input (`cat`, `sort`) can finish. EOF does not close the channel; output keeps arriving.

---

## 5. Interactive shell

```csharp
await using SshShell shell = await conn.OpenShellAsync(new SshShellOptions
{
    TerminalType = "xterm-256color",
    Size  = new SshTerminalSize(cols, rows, pixelWidth, pixelHeight),
    Modes = SshTerminalModes.Empty.With(SshTerminalModeOpcode.Utf8Input, 1),
}, ct);

await shell.ResizeAsync(new SshTerminalSize(cols, rows, pixelWidth, pixelHeight), ct);
```

`SshTerminalModes` is immutable: `With` returns a new copy, so `SshTerminalModes.Empty` is safe to share everywhere.

**Pixel dimensions are first-class citizens**, not always 0 — things like sixel and the kitty graphics protocol rely on them for layout.
If you don't know, pass 0; that is also a meaningful answer.

`SshShell` **does not expose `StandardError`**: with a pty, stderr is merged into stdout by the pseudo-terminal,
and exposing a stream that is always empty would only leave people waiting on it forever.

To enable X11 / agent forwarding on a shell, specify it directly in the options (the request order
`pty-req → x11-req → auth-agent-req → env → shell` is handled by the library):

```csharp
await using SshShell shell = await conn.OpenShellAsync(new SshShellOptions
{
    X11Forwarding   = new X11ForwardOptions(),        // ssh -X
    AgentForwarding = AgentForwardOptions.Default,    // ssh -A
    BeforeStart     = (channel, ct) => SendMyCustomRequestAsync(channel, ct),   // requests the library has no built-in support for
}, ct);
```

`SshCommandOptions` (one-shot commands) has the same three options — they live on the shared base class `SshSessionRequestOptions`.

---

## 6. SFTP

```csharp
using VelaShell.Ssh.Sftp;

await using SftpFileSystem fs = await SftpFileSystem.ConnectAsync(conn, cancellationToken: ct);

Console.WriteLine(fs.WorkingDirectory);   // comes from REALPATH ".", more reliable than assembling /home/{user}

await foreach (SftpDirectoryEntry e in fs.EnumerateDirectoryAsync("/etc", ct))
{
    Console.WriteLine($"{e.Name}\t{e.Length}\t{(e.IsSymbolicLink ? "→ " + e.LinkTarget : "")}");
}

byte[] content = await fs.ReadAllBytesAsync("/etc/hostname", ct);
await fs.WriteAllBytesAsync("/tmp/x", content, cancellationToken: ct);
```

### Resuming transfers — use `DurableLength`, not the file length

```csharp
await using SftpFileStream s = await fs.OpenWriteAsync("/tmp/big.bin", cancellationToken: ct);
try
{
    await s.WriteAsync(data, ct);
    await s.FlushAsync(ct);
}
catch (SftpTransferInterruptedException ex)
{
    // resume from here, precisely — no need to blindly back off 2 MiB from the file length as is common practice
    long resumeFrom = ex.DurableLength;
}
```

With pipelined writes, the order of replies is not guaranteed, so **the file length reported by the server is only "the highest acknowledged offset"**;
there may be holes before it that read as 0. `DurableLength` is the length that has been acknowledged **contiguously** starting from 0.

If you need "the file is a complete prefix at every moment", use `SftpWriteMode.Sequential` (trading throughput for determinism).

### Ask about capabilities first

```csharp
if (fs.Capabilities.HasPosixRename)
    await fs.RenameAsync(a, b, overwrite: true, ct);   // atomic overwrite
else
    // this server cannot do an atomic overwrite — say so in the UI rather than quietly switching to a non-atomic approach
```

### Async only

The synchronous `Read` / `Write` / `SetLength` of `SftpFileStream` throw `NotSupportedException` —
synchronous versions could only be implemented by blocking a thread while waiting for network round trips. Synchronous `Flush` is a non-blocking no-op (there is no local buffer),
and synchronous `Dispose` hands teardown to the background and returns immediately; to see errors during teardown, use `await using`.

---

## 7. Compression

**Off by default.** To turn it on, say so explicitly:

```csharp
var options = new SshConnectionOptions("root@example.com")
{
    Algorithms = SshAlgorithmSet.Default.WithCompression(),   // zlib@openssh.com
};
```

Two things worth knowing first:

1. **Turning it on for interactive sessions is basically a net loss.** Terminal output is small to begin with, and the bytes saved by compression
   do not pay for the CPU on both ends. **Incompressible content (already-compressed files, encrypted data) gets bigger after compression.**
   Scenarios worth enabling it for are SFTP transfers of text, or links billed by traffic.
2. **`zlib@openssh.com` only starts compressing after authentication**, which is its only difference from the old `zlib`.
   The reason is that compressed length leaks the compressibility of the plaintext — during authentication, that amounts to opening a
   distinguishing channel for the password (the CRIME family). So we offer only the former.

**Whether compression was actually negotiated is something you have to check yourself** — when the server does not support it, it silently falls back to `none`,
**without an error**:

```csharp
Console.WriteLine(conn.Algorithms.CompressionServerToClient);   // zlib@openssh.com or none
```

`Algorithms` exposes all algorithms actually negotiated for this session (key exchange, host key,
encryption and integrity for both directions, compression for both directions, whether strict KEX was enabled).
Showing "what we connected with" on a status bar, or answering "what exactly did this connection negotiate" when troubleshooting,
both come from here — no need to probe again yourself.

---

## 8. Forwarding

```csharp
using VelaShell.Ssh.Forwarding;

// -L: local 127.0.0.1:8080 → remote 10.0.0.9:80
await using LocalPortForwarder local = LocalPortForwarder.Start(
    conn, "10.0.0.9", 80, new LocalPortForwardOptions { BindPort = 8080 });

// -D: local SOCKS5
await using LocalPortForwarder socks = LocalPortForwarder.StartDynamic(conn);

// -R: server listens → connects back to the local machine
await using RemotePortForwarder remote = await RemotePortForwarder.StartAsync(
    conn, "127.0.0.1", 3000,
    new RemotePortForwardOptions { BindAddress = "localhost", BindPort = 0 }, ct);
Console.WriteLine(remote.BoundPort);   // when 0 was requested, the actual port assigned by the server is here

// metering is in the library (on PortForwarder, the base class shared by all three)
Console.WriteLine($"{local.ActiveConnections} connections · sent {local.BytesSent} B · received {local.BytesReceived} B");
local.ConnectionClosed += (_, e) => log.Info($"{e.Target} transferred {e.BytesSent + e.BytesReceived} bytes");
```

The local forwarder's `Start` is synchronous — it only binds a port on this machine; a remote forward has to wait for the server to answer `tcpip-forward`, hence `StartAsync`.

**Binds to loopback by default.** To open it to the outside you must explicitly write `BindAddress = IPAddress.Any` —
the other end of a tunnel is often an internal database, and binding to `0.0.0.0` by default would expose it to everyone on the same network segment.

### Unix sockets in the reverse direction (`ssh -R /remote:/local`)

```csharp
await using RemotePortForwarder sock = await RemotePortForwarder.StartUnixSocketAsync(
    conn,
    targetSocketPath: "/var/run/docker.sock",   // the local one
    remoteSocketPath: "/tmp/docker.sock",       // the one the server should create
    cancellationToken: ct);

Console.WriteLine(sock.RemoteEndpointName);   // a unified way to describe both forms
```

**With a socket instead of a port, other users on the remote machine can neither see nor connect to it** — who can connect is decided by file permissions,
whereas a `-R 2375:...` port can be connected to by everyone on that machine.

> The server creates that socket file on its own file system. OpenSSH refuses when the path already exists,
> so a failure here most likely means "the last run was not cleaned up".

### X11 forwarding (`ssh -X` / `-Y`)

```csharp
await using SshChannel session = await conn.OpenSessionChannelAsync(null, ct);

await using X11Forwarder x11 = await X11Forwarder.RequestAsync(conn, session,
    new X11ForwardOptions
    {
        // Display = X11Display.Parse(":0"),   // reads DISPLAY by default
        Trusted = false,                        // default: corresponds to ssh -X
        Timeout = TimeSpan.FromMinutes(20),     // Zero = never expires
    },
    ct);

Console.WriteLine($"{x11.AcceptedChannels} accepted · {x11.RejectedChannels} rejected");
```

> [!WARNING]
> **X11 has no client isolation.** Any client connected to the same display can read others' keystrokes,
> capture others' windows, and inject events into others' windows — so handing the local display to the remote
> means handing the input and output of **every** local graphical session to the remote.
> Off by default and untrusted by default are both because of this.

Three things worth knowing:

1. **What is sent to the server is always a random fake cookie**; the real cookie never leaves the local machine.
   The remote X client connects with the fake cookie; we check it (constant-time comparison), replace it with the real cookie,
   and only then forward to the local X server. If it does not match, it is rejected, and the X server **is not even touched**.
2. **Untrusted mode (the default) requires `xauth` locally**, and an X server that supports the SECURITY extension.
   Windows usually has neither — there you can only use `Trusted = true`,
   but be clear that this amounts to handing the local display entirely to the remote.
3. **`Timeout` applies in both modes** (OpenSSH only applies it to untrusted).
   Trusted mode is precisely the more dangerous one, so having no time limit there makes no sense. For long sessions, set `TimeSpan.Zero` explicitly.

On the server side, `X11Forwarding yes` in `sshd_config` and an installed `xauth` are required;
without them the request is rejected, and the exception message states these two conditions directly.

Two more points:

- **The restricted cookie generated in untrusted mode is written to a temporary file** (`xauth -f`) and deleted after use;
  it is **never written into your `.Xauthority`** — otherwise it would overwrite the fully authorized cookie for the local display,
  and after it expires your own local X programs could no longer connect to your own display.
- **Multiple sessions on the same connection can each have their own X11 forwarding**: `x11` channels are dispatched to the right one by fake cookie,
  and releasing one does not affect the others. The most common usage is to specify it directly in the shell options (see §5).

### Tunnels without a listening port


When connecting to `/var/run/docker.sock` or an internal API, **the local machine does not need a listening port**:

```csharp
await using SshChannel tunnel = await conn.OpenUnixSocketTunnelAsync("/var/run/docker.sock", cancellationToken: ct);
// tunnel.StandardInput / tunnel.StandardOutput are the stream
```

Opening a `-L` listener means any process on the same machine can connect to it. For a root-equivalent endpoint,
this difference is not an optimization but a prerequisite.

### Agent forwarding (`ssh -A`)

**Off by default, and there is no "turn it on globally" switch** — it is a per-connection, per-channel decision:

```csharp
await using SshChannel session = await conn.OpenSessionChannelAsync(null, ct);

await using AgentForwarder fwd = await AgentForwarder.RequestAsync(
    conn, session,
    new AgentForwardOptions
    {
        AllowedKeys = [deployKey],              // forward only this one; the rest are invisible to the remote
        ConfirmEachSignature = AskUserAsync,    // ask a human for every signature
        MaxConnections = 4,
    },
    cancellationToken: ct);
```

`ssh -A` has a bad reputation for a reason: **remote root can borrow your agent to log in anywhere as you**,
and you cannot see it. So here the agent channel is not simply wired through as a byte pipe;
instead **the agent protocol is parsed and then forwarded** — only by parsing it can the following be done:

- When `AllowedKeys` is non-empty, the remote **cannot see** keys outside the list when listing keys (`fwd.KeysHidden` counts them),
  and a request to sign with a key outside the list gets `FAILURE` directly;
- `ConfirmEachSignature` receives "which key, and what its comment is", so it can show a dialog to ask a human; if declined, `FAILURE` is returned;
- Messages that **would change the local agent's state**, such as `ADD_IDENTITY` / `LOCK` / `UNLOCK`, are always rejected
  and not forwarded — a remote server has no reason whatsoever to change our local keyring.

`fwd.SignatureRequests` / `SignaturesDenied` / `IdentityListings` can be used directly for display or auditing.

---

## 9. Reading `ssh_config`

```csharp
// LoadAsync expands Include; Parse only parses text and does not touch the file system
IReadOnlyList<SshConfigBlock> blocks = await SshConfigFile.LoadAsync(cancellationToken: ct);

SshHostConfig cfg = SshConfigFile.Resolve(blocks, new SshConfigMatchContext
{
    Host = "prod-web-1",
    User = "deploy",
    LocalUser = Environment.UserName,
});

Console.WriteLine(cfg.HostName);       // after HostName rewriting
Console.WriteLine(cfg.Port);           // 22 if not specified
Console.WriteLine(cfg.IdentityFiles);  // there may be several
```

**The library only parses; it does not decide for you.** The parsed result is handed to the caller — if the library took it upon itself to read
`~/.ssh/config`, "why didn't it connect to the machine I wrote" would become a very hard problem to track down.

If you decide to use it, it turns into ready-to-connect parameters in one step:

```csharp
SshConnectionOptions options = await SshConfigFile.CreateConnectionOptionsAsync(
    blocks, "prod-web-1",
    new SshConfigConnectOptions
    {
        Credentials        = [new KeyboardInteractiveCredential(PromptAsync)],   // placed after IdentityFile
        PassphraseProvider = (path, ct) => AskPassphraseAsync(path, ct),         // for encrypted IdentityFile
        AskUnknownHost     = ConfirmFingerprintAsync,
    }, ct);

await using SshConnection conn = await SshConnection.ConnectAsync(options, ct);

// ForwardAgent / ForwardX11 are session items, not connection items:
await using SshShell shell = await conn.OpenShellAsync(SshConfigFile.Resolve(blocks, "prod-web-1").ApplyToShell(), ct);
```

`HostName` / `Port` / `User` / `IdentityFile` / `Compression` / `ServerAlive*` / `ConnectTimeout` /
`StrictHostKeyChecking` / `UserKnownHostsFile` / `ProxyJump` / `ProxyCommand` all map onto connection parameters;
each jump host in `ProxyJump` is **resolved against the same configuration**, and the jump chain has a depth limit and cycle detection.
Item-by-item rules are in [`spec/09-dialing.md`](spec/09-dialing.md) §7.

Two security constraints worth knowing:

1. **`Match exec` executes no commands by default.** It means *parsing a configuration file can run arbitrary programs on the local machine*,
   and configuration files are often copied from elsewhere, synced in, or given by someone else. Without an evaluator,
   blocks with `exec` **never match**. If you really need it, pass one yourself:

   ```csharp
   new SshConfigMatchContext { Host = h, ExecEvaluator = cmd => RunAndCheck(cmd) }
   ```

   This way the decision "whether to run external commands" clearly rests with you, rather than hiding in the library's default behavior.
2. **`Include` has a depth limit (16) and cycle detection.** `a` including `b` and `b` including `a` again
   is easy to write, and without cycle detection the symptom is *the whole process freezes while reading the configuration*.

Other details: `Match` supports `all` / `host` / `originalhost` / `user` / `localuser`;
conditions are ANDed, and `!` negation is supported; `canonical` / `final` never match (we do no host name canonicalization).
**Insufficient information means no match** — when the user is unknown, `Match user` will not guess one.

---

## 10. Rekeying


**It is on by default, and usually you don't need to touch it.** It is documented here because it occasionally needs tuning.

```csharp
var options = new SshConnectionOptions("root@example.com")
{
    // default: 1 GiB / 1 hour / 2³¹ packets, rekey when any one is reached
    Rekey = SshRekeyPolicy.Default,
};

// you can also pick the moment yourself
await conn.StartRekeyAsync(ct);

Console.WriteLine(conn.RekeyCount);        // how many times it has rekeyed
Console.WriteLine(conn.LastRekeyReason);   // which threshold triggered the last one
```

Three things worth knowing:

1. **Handling peer-initiated rekeying is always on and cannot be turned off.** `SshRekeyPolicy.Disabled`
   only turns off "rekeying initiated by us". This is not an omission — OpenSSH's `RekeyLimit` defaults to
   1 GiB / 1 hour, and when the limit is reached it sends `KEXINIT` itself; the symptom of not responding is
   **a shell that has been open all afternoon suddenly drops**, **a large file transfer drops halfway through**.
2. **Do not lower the packet-count threshold.** SSH sequence numbers are 32-bit, and the AES-GCM nonce
   advances once per packet — nonce reuse is catastrophic for GCM (the authentication key can be recovered).
   It is a hard cryptographic constraint, not a tuning knob.
3. **The thresholds have lower bounds** (64 MiB / 1 minute / 1024 packets); going below them throws
   `ArgumentOutOfRangeException` immediately. Every rekey involves an asymmetric operation,
   and tuning it too frequently opens a denial-of-service surface against yourself.

**Channels are not interrupted** during rekeying: the receive direction keeps receiving data as usual, and outgoing channel data is buffered
and flows out in the original order once negotiation completes (RFC 4253 §7.1 only allows transport-layer messages during this period).

---

## 11. Reading failures


All exceptions derive from `SshException` and carry `Reason` (a decidable reason code) and `Phase` (which step).

| Exception | When | Key fields |
| --- | --- | --- |
| `SshConnectException` | Dialing / handshake phase | `Reason`: `DnsFailure` / `TcpRefused` / `TcpTimeout` / …; a configuration that cannot work (`ProxyJump` loop, invalid `ProxyCommand` template) is `InvalidConfiguration` |
| `SshAuthenticationException` | Authentication failed | `Attempts`, `ServerOffered`, `PartialSuccessAchieved` |
| `SshPrivateKeyException` | Private key unreadable or cannot be decrypted | `Reason`: `KeyFileUnreadable` / `KeyFormatInvalid` / `KeyPassphraseRequired` / `KeyPassphraseIncorrect`; `NeedsPassphrase` |
| `SshCertificateException` | Certificate is wrong | `Reason`: `KeyFormatInvalid` / `KeyMismatch` (not a pair with the private key, or a host certificate used to log in) |
| `SshAgentException` | ssh-agent error | `Reason`: `AgentUnavailable` / `AgentRefused` |
| `SshChannelException` | Channel could not be opened, or a request on it was refused | not opened: `ChannelOpenFailed` + `OpenFailureReason` + an actionable hint; exec / pty-req / shell / subsystem refused: `ChannelRequestRejected` |
| `SftpException` | SFTP operation failed | **`ServerMessage` (the server's own words)**, `Operation` (`SftpOperation`), `Path` |
| `SftpTransferInterruptedException` | Transfer interrupted | `DurableLength` |
| `SshForwardException` | Forwarder failed to start | `Reason`: `ForwardRejected` / `ForwardBindFailed` / `ForwardSetupFailed` / `LimitExceeded` |
| `SshCommandFailedException` | `EnsureSuccess` found the command did not succeed | `Reason` = `CommandFailed`; `Result` (the complete `SshCommandResult`) |

`Reason` tells the truth: the same exception type carries different reason codes in different situations, so a UI can branch and translate on it
without parsing message sentences — **messages are diagnostic text for developers**, not UI copy.

`SftpException.ServerMessage` deserves a separate word: SFTP v3 has only 9 status codes,
and code `4` carries the vast majority of real errors — "directory not empty", "file already exists", "disk full", "quota exceeded"
**are all the same code**. The text given by the server is the only information that can tell them apart.

**Channels are independent failure domains**: if a channel cannot be opened or errors out, the session remains usable.
The same goes for forwarders — a single connection failure only raises the `Error` event, and the forwarder keeps running.
