# VelaShell.Ssh —— 架构与原理

> 状态：**已决策，M0–M5 完成 + 面向使用者的连接层**（2026-09-21）。本文定方向、架构与原理。
> 实现依据不是本文，是 [`spec/`](../spec/) 下的行为规格 —— 见 §2.2 第 2 条纪律。
>
> 依据：对 `Tmds.Ssh`（MIT，2026-09 快照，28,769 行）与 VelaShell 当前 SSH 栈
> （`Infrastructure/Ssh/` + `Core/Ssh/` + `Core/Sftp/`，约 8,600 行）的逐文件通读。
> 文中每一条「现状」都指得出文件与行号，不写没查过的话。
>
> **2026-09-23 更新**：本库已并入宿主仓库（`src/VelaShell.Ssh`，测试在 `tests/VelaShell.Ssh.Tests`，
> 工程脚本在 `scripts/ssh/`），不再单独发 NuGet。下文关于独立仓库、NuGet 包、打包冒烟的段落
> 是当时的决策记录，保留原样；文中的路径已改成宿主仓库里的新位置。
> 原 §2.4 的 CI 相似度门禁随之移除，相关段落已删去。

---

## 〇 一句话结论

**值得做，但理由不是「想自己写一个」，而是 VelaShell 已经为绕开底层库写了 1,200 余行补丁，
并且有两个能力（2FA、PTY 像素）被死死卡在上游。**

做法是：**全新 API + 独立实现 + 一个可选的兼容垫片包**。
不 fork、不逐文件改写、不沿用它的类型体系 —— 这样「抄袭嫌疑」不是靠辩解消掉的，
而是**根本不存在**：两边的内部模型不是同一套东西。

---

## 一 为什么值得自己实现：VelaShell 已经付出的代价

不是感觉，是清单。下面每一行都是**为了绕开底层库的限制而存在的宿主代码**：

| # | 宿主代码 | 行数 | 存在的唯一原因 |
| :-: | --- | :-: | --- |
| 1 | `Infrastructure/Net/LoopbackProxyRelay.cs` | ~200 | Tmds 只认 `host:port`，没有传输层扩展点。走 HTTP/SOCKS5 代理时只能**在本机起一个环回中继**骗它 |
| 2 | `Infrastructure/Ssh/SshAlgorithmProbe.cs` | 199 | 协商失败时库不交出对端的 KEXINIT 名单，只好**再开一条 TCP 明文探一次** |
| 3 | `Infrastructure/Ssh/SshAlgorithmDiagnostics.cs` | 105 | 同上，把探回来的名单和本端名单求差集拼成人话 |
| 4 | `Infrastructure/Ssh/MeteredPortForwardHandle.cs` | 376 | 库把转发的数据面做在内部、不暴露任何计数。隧道面板要显示「3 连接 · 1.4 MB」，只能**自己重写一遍转发** |
| 5 | `Infrastructure/Ssh/OpenSshPrivateKey.cs` | 303 | 库的私钥解析器只认 OpenSSH 封装。用户手上的 PKCS#1 / PKCS#8 / 加密 PKCS#8 会被**静默跳过**，只好先在宿主里转一道 |
| 6 | `Infrastructure/Ssh/TmdsSshInterop.cs` 的 `ExtractConnectFailedReason` | ~40 | `ConnectFailedException` 是 `internal`。要区分「认证失败 / 超时 / 协商失败」，只能**按消息前缀切字符串** |
| 7 | `TmdsSftpClientWrapper.ResumeSafetyMargin` | — | 库不保证写入连续。断点续传前必须**盲退 2 MB**（64 缓冲 × 32 KB），因为文件长度只代表「已确认的最高偏移」，之前可能有空洞 |
| 8 | `TmdsSftpClientWrapper.PosixRenameFileAsync` | — | 库没暴露 `posix-rename@openssh.com`，只能退化成普通 rename —— 而那正是某些服务器会拒的那条路 |
| 9 | `InfrastructureServiceCollectionExtensions.AddCredential` 的整段注释 | — | 必须**整体替换**默认凭据列表而不是追加，否则 Windows 上每次连接都先撞一发 `SSH_AUTH_SOCK` 的 `ArgumentException` |

再加上**四个被卡住的能力**：

| 能力 | 状态 |
| --- | --- |
| **keyboard-interactive（2FA / OTP）** | 上游根本没实现。`KeyboardInteractiveSupportTests` 是一条「引信」用例，断言的是**现在还不支持** —— 堡垒机上的 Google Authenticator / Duo 一律连不上，文案只能写「本版无法连接」 |
| **PTY 像素尺寸** | 卡 [tmds/Tmds.Ssh#519](https://github.com/tmds/Tmds.Ssh/pull/519)，未合并未发版。`pty-req` / `window-change` 的像素字段恒为 0 |
| **压缩 zlib@openssh.com** | 我们自己给上游提了 [#513](https://github.com/tmds/Tmds.Ssh/pull/513)，卡合并 |
| **SSH Agent 转发** | 对标矩阵里六家全有、我们全无的那一格 |

> **这就是全部理由。** 不是「库写得不好」—— Tmds.Ssh 写得相当好，async-first、AOT 友好、
> 没有历史包袱，当初从 SSH.NET 迁过去是对的。是**边界不对**：
> 它是一个「SSH 客户端库」，而 VelaShell 需要的是一个**终端产品的 SSH 引擎** ——
> 要能观测、能插拔、能报出人话、能被产品的 UI 拿去显示。这两者的 API 取舍不一样。

---

## 二 「抄袭嫌疑」这件事怎么真正解决

### 2.1 先把法律事实摆清楚

Tmds.Ssh 是 **MIT**。MIT 允许 fork、修改、闭源、再分发、商业销售，
**唯一的义务是保留版权声明与许可证全文**。所以：

- **fork 它并不违法**，哪怕一行不改直接卖也不违法（只要带上 LICENSE）。
- 真正的风险**不在法律，在三处别的地方**：
  1. **声誉** —— 被人 diff 出来「这是 Tmds.Ssh 改个名」，对一个要卖授权的产品是硬伤。
  2. **双许可的摩擦** —— VelaShell 走双许可。MIT 代码混进闭源分发物里要持续带 attribution，
     一旦文件级混杂，以后每次合规审查都要重新梳理「哪几行是谁的」。
  3. **升级绝缘** —— fork 之后上游每次改动你都要手工 rebase，几个月就漂到无法合并，
     最后变成「既没有自己实现的自由，也没有上游的红利」。**这是 fork 最贵的隐性成本。**

所以目标不是「合法」，是**可证明的独立**：任何人拿两边的源码做 diff / 相似度扫描，
结论都应该是「这是两个不同的实现」。

### 2.2 净室规程（六条，可执行、可审计）

| # | 纪律 | 怎么做 |
| :-: | --- | --- |
| 1 | **规范优先，源码不作依据** | 实现依据只能是 RFC 4250–4254 / 4256 / 4419 / 5656 / 8308 / 8332 / 8709、OpenSSH 仓库的 `PROTOCOL*` 文件、draft-ietf-sshm-*。**每个协议实现文件头写明它实现的是哪份文档的哪一节**，做不到就说明当时抄的是别人的代码 |
| 2 | **两阶段隔离** | 分析阶段产出的是**行为规格**（纯自然语言 + 报文时序表，零代码片段）；实现阶段只看规格与 RFC。用 AI 辅助时这一条尤其可操作：分析会话与实现会话**不共享上下文** |
| 3 | **标识符体系整体另起** | 不复用它的类名 / 方法名 / 字段名 / 枚举成员名的**组合**。§6.1 给了对照表。注意是「组合」——`SshClient` 这种通名撞上无所谓，一整套撞就是证据 |
| 4 | **架构真的不同，不是改名** | §4–§5 给的是**另一套内部模型**（Pipelines 替 Tmds 手写的 Sequence、状态机替信号量握手、统一账本替三套 pending、IDuplexPipe 替双 buffer 读）。这一条是前几条的地基：**只要内部模型真的不同，两边的代码自然就不像** |
| 5 | **测试向量只取公开来源** | RFC 测试向量、NIST CAVP、OpenSSH regress 的用例**思路**。**不复制它的测试文件**，一个都不 |
| 6 | **NOTICE 如实写** | 见 §2.4 |

### 2.3 什么**不**算抄袭（别过度紧张）

这几类东西**必然相同**，相同也不构成侵权 —— 它们是协议规定的事实，
受 merger doctrine / scènes à faire 保护（表达方式唯一时不受版权保护）：

- 消息号常量：`SSH_MSG_KEXINIT = 20`、`SSH_MSG_CHANNEL_DATA = 94` …
- 算法名字符串：`"curve25519-sha256"`、`"chacha20-poly1305@openssh.com"` …
- 报文字段顺序与 wire 布局（RFC 规定死的）
- 交换哈希 `H` 的输入拼接顺序（RFC 4253 §8）
- SFTP 包类型号、`SSH_FX_*` 错误码
- 「35000 字节最大包长」这类 RFC 写死的上限

**不要为了「看起来不一样」去改这些** —— 改了就是协议不兼容，那才是真正的灾难。

### 2.4 NOTICE 的写法

诚实比隐瞒安全得多。仓库根放一份 `NOTICE.md`：

```
VelaShell.Ssh 是一套独立实现的 SSH 客户端库。

它不是任何现有库的 fork，也不包含来自其它 SSH 实现的代码。
实现依据是 IETF RFC 与 OpenSSH 的协议文档（见各文件头的引用）。

设计上，我们研究过并受益于以下项目所展示的思路 ——
它们的源码未被复制或改写：
  - OpenSSH（BSD）—— 协议扩展与互操作行为的事实标准
  - Tmds.Ssh（MIT）—— .NET 上 async-first SSH 客户端的先行者
  - libssh2 / golang.org/x/crypto/ssh —— 报文处理的公开参考实现
```

**写上 Tmds.Ssh 这一行。** 一个诚实的致谢，比一份被人扒出来的沉默强十倍；
而且它同时说清了「受益于思路」和「未复制代码」这两件不同的事。

---

## 三 设计原则（六条，用来裁决后面所有分歧）

1. **异步是唯一形态。** 没有同步重载，没有 `.Result`。热路径一律 `ValueTask` +
   池化的 `IValueTaskSource`。
2. **背压是结构性的，不是加出来的。** 缓冲能力由 `PipeReader`/`PipeWriter` 与 SSH 窗口
   共同表达，任何一层都不许开无界队列**兜住**对端的产出速度。
3. **失败必须是结构化的。** 每个可能失败的点都要能回答「是什么失败了、对端说了什么、
   我方提供了什么、下一步该做什么」。**异常里带数据，不带拼好的英文句子。**
4. **能观测的东西一律暴露。** 字节数、往返、窗口、在途请求、协商结果、每一跳耗时 ——
   产品要显示的东西，库必须已经知道；库知道而不说，宿主就只能重写一遍
   （这正是 §1 第 4 条的来历）。
5. **可插拔优先于可配置。** 与其加第 41 个布尔开关，不如开一个接口。
   拨号、密码套件、认证方法、主机密钥策略、SFTP 扩展 —— 全部是接口（§8）。
6. **不自己写密码学原语。一个例外，写明在案。** 只做「协议装配」。
   BCL 有的走 BCL（AES / SHA-2 / ECDH / ECDSA / RSA / ML-KEM —— 还能吃到硬件加速）；
   BCL 缺的走 BouncyCastle（raw ChaCha20、独立 Poly1305、Ed25519 曲线运算、sntrup761、
   Argon2id）。
   **唯一的例外是 `Keys/BcryptPbkdf.cs`**（加密的 OpenSSH 私钥所需的 `bcrypt_pbkdf`）——
   没有任何现成原语能凑出来，而不做它等于「绝大多数人手里的那把钥读不了」。
   例外的范围、理由与验证方式见 §11.2.18。
   〔2026-09-21 收紧〕原先给 raw ChaCha20 留了一个「自己写」的例外，实现时撤销了：
   手写的密码学原语对一个要被审计的库是实打实的负债，而它的错误不会报错，
   只会在特定输入上静默产出错误结果。

---

## 四 架构总览

```mermaid
flowchart TB
    subgraph API["VelaShell.Ssh —— 面向使用者"]
        CONN["SshConnection<br/>状态 · 生命周期 · 事件"]
        CMD["SshCommand · SshShell"]
        SFTP["SftpFileSystem · SftpFileStream"]
        FWD["PortForwarder · SshTunnelStream"]
    end

    subgraph CORE["协议核心（同一程序集，按文件夹分层）"]
        L7A["L7 认证方法链 IAuthMethod"]
        L7B["L7 SFTP 管线 SftpRequestPipeline"]
        L6["L6 请求账本 RequestLedger"]
        L5["L5 通道层 IDuplexPipe · 自适应窗口"]
        L4["L4 会话状态机 SessionMachine · SendGate"]
        L3["L3 密码套件 ISshCipherSuite"]
        L2["L2 帧层 PipeReader · PipeWriter"]
        L1["L1 拨号 ISshTransportDialer"]
    end

    subgraph EXT["扩展面"]
        DIAL["Socks5 · HttpConnect · SshJump · InMemory"]
        ALGO["算法注册表 KEX·HostKey·Cipher·MAC·Compression"]
        SIGN["ISshSigner 文件·Agent·PKCS#11·KeyVault"]
        POL["IHostKeyPolicy known_hosts·CA·TOFU"]
        OBS["Metrics · ActivitySource · IPacketTap"]
    end

    API --> CORE
    L7A --> L6
    L7B --> L6
    L6 --> L5
    L5 --> L4
    L4 --> L3
    L3 --> L2
    L2 --> L1
    L1 -.-> DIAL
    L3 -.-> ALGO
    L7A -.-> SIGN
    L4 -.-> POL
    CORE -.-> OBS
```

**一个包，一件事。**（2026-09-21 决定，原方案是 Core + 主包两个）

| 包 | 内容 | 运行时依赖 |
| --- | --- | --- |
| `VelaShell.Ssh` | L1–L9 全部，按**文件夹**分层（`Protocol/` `Transport/` `Crypto/` `Session/` `Channels/` `Threading/` `Auth/` `HostKeys/` `Sftp/` `Forwarding/` `Diagnostics/` `Client/` `Config/`） | `BouncyCastle.Cryptography`（仅 BCL 缺失的密码学原语） |

不拆成 `Core` + 主包，是因为拆了以后使用者第一件要纠结的事就是「我该引哪个」——
而答案永远是「引主包」。真正只需要协议核心的场景至今一个都没有。
**扩展点靠接口开放（§8），不靠拆程序集** —— 拆程序集换来的是版本对齐负担，
不是扩展性。

> 兼容垫片 `VelaShell.Ssh.Compat.Tmds`（§6.3）如果真要做，会是**另一个包** ——
> 它的存在期是有限的（VelaShell 迁完即废），不该和主包绑同一条发布线。
> 这件事在 M6 决定，M0–M5 不为它留任何接口。

---

## 五 逐层设计与原理

> 每一节都先说**它跟 Tmds 那层有什么本质不同**，再说为什么这个不同是对的。
> 这既是设计说明，也是 §2.2 第 4 条纪律的兑现记录。

### 5.1 L1 拨号层 —— `ISshTransportDialer`

```
ValueTask<Stream> DialAsync(SshDialTarget target, CancellationToken ct)
```

**不同点：Tmds 没有这一层。** 它的 `ConnectCallback` / `ConnectContext` 是 `internal`，
`Proxy` 类虽然 public，但唯一的成员 `ConnectToProxyAndForward` 是 `internal abstract`
—— 等于一个**外部无法实现的抽象类**。VelaShell 的 `LoopbackProxyRelay` 就是被这一点逼出来的。

内置实现：`TcpTransportDialer`、`Socks5Dialer`、`HttpConnectDialer`、`SshJumpDialer`（跳板）、
`ProxyCommandDialer`（`ProxyCommand`）、`InMemoryTransport.CreateDialer`（测试用，见 §10.2）。
入口是 `DialerChain`；**嵌套靠 `Via`**：每个代理类拨号器都有一个「怎么到达代理本身」的内层拨号器，
默认直连 TCP。跳板的嵌套写在跳板自己的连接参数里（它本来就是一条有自己拨号方式的完整连接），
`DialerChain.Jumps(a, b, c)` 对应 `ProxyJump a,b,c`。
**跳板链与网络代理是同一套机制的两次应用** —— 而不是像现在这样，跳板走库内的 `SshProxy`、
代理走宿主的环回中继，两条完全不同的路。行为规格见 `spec/09-dialing.md`；
`ssh_config` 的 `ProxyJump` / `ProxyCommand` 经 `SshConfigFile.CreateConnectionOptionsAsync` 直接落到这里。

> **VelaShell 收益**：`LoopbackProxyRelay.cs` 整个删掉；`TmdsSshClientWrapper` 里
> `PrepareProxyRelay` / `DisposeRelay` / `DescribeProxyError` 三段一并删掉。

### 5.2 L2 帧层 —— 交给 `System.IO.Pipelines`

**不同点：Tmds 自己写了 `Sequence` + `Sequence.Segment` + `SequencePool`
（250 + 100 + 60 行），本质上是重新实现了 Pipelines 的一半**，
外加一个 `Packet` struct 用 `Move()` / `Clone()` 手工管理所有权。

我们直接用 `PipeReader` / `PipeWriter`：

- 读：`ReadResult.Buffer` 就是 `ReadOnlySequence<byte>`，**零拷贝**切出帧；`AdvanceTo` 表达消费进度。
- 写：`GetSpan` / `Advance` / `FlushAsync`，背压由 `PipeOptions.PauseWriterThreshold` 表达。
- 所有权：**没有所有权问题** —— 缓冲归 Pipe 管，不存在「谁该 Dispose 这个 Packet」。

两个可量化的收益（都是读源码读出来的事实，不是推断）：

| | Tmds 现状 | VelaShell.Ssh |
| --- | --- | --- |
| **收包** | `StreamSshConnection.ReceiveAsync` 每次只 `AllocGetMemory(4096)`，一个 32 KiB 的 SFTP 数据包要 **8 次以上 `stream.ReadAsync`** | PipeReader 单次 read 上限自适应（默认 64 KiB），一次读完 |
| **发包** | `SendLoopAsync` 每个包一次 `stream.WriteAsync`。SFTP 64 个在途 32 KB 写 = **一轮 64 次 write** | 发送闸门批量取包 → 一次 `FlushAsync`，**一轮 1–2 次 write** |

### 5.3 L3 密码套件 —— 单一接口 `ISshCipherSuite`

**不同点：Tmds 把它拆成 `IPacketEncryptor` / `IPacketDecryptor` 两族，
外加 `TransformAndHMacPacketEncryptor` / `EncryptionCryptoTransform` / `IHMac` 的组合装配**，
AEAD（GCM、ChaCha）与「Transform + HMac」是两条平行的类树。

我们做成**一个接口描述一整套**：

```
interface ISshCipherSuite : IDisposable
{
    CipherSuiteShape Shape { get; }   // 长度字段是否加密 · AAD 长度 · tag 长度 · 块大小 · 是否 EtM
    int  Seal(ReadOnlySpan<byte> payload, ulong seq, IBufferWriter<byte> output);
    OpenResult TryOpen(ref SequenceReader<byte> input, ulong seq, IBufferWriter<byte> output);
}
```

`CipherSuiteShape` 把「chacha20-poly1305 的长度字段用另一把密钥单独加密」
「AES-GCM 的长度字段是明文 AAD」「EtM 的 MAC 覆盖密文」这三种**形状差异**
变成了**数据**，帧层只看 `Shape` 就知道该怎么切包 —— 而不是让每个 decryptor
各自实现一遍「先读 4 字节还是先解密」。

> 加密算法本身不自己写：AES-GCM 走 BCL 的 `AesGcm`（AES-NI 硬件加速）。
> `chacha20-poly1305@openssh.com` 是 OpenSSH 的自定义构造（**两把密钥，长度字段单独加密**），
> BCL 的 `ChaCha20Poly1305` 是 RFC 8439 的那一个，用不上 —— 这里需要的是 raw ChaCha20 块函数
> 与独立的 Poly1305，两样都走 **BouncyCastle**。
> AES-CTR 则用 BCL 的 AES-ECB 逐块生成计数器块（CTR 模式的定义，NIST SP 800-38A §6.5），
> 以吃到 AES-NI 硬件加速；计数器与异或是簿记，不是密码学。

### 5.4 L4 会话状态机 —— 显式状态 + 发送闸门

**不同点，也是本设计最关键的一处。**

Tmds 的重协商是这样做的：收包循环发现 `SSH_MSG_KEXINIT` → 造一个 `SemaphoreSlim` 赋给
`_keyReExchangeSemaphore` → 把自己的 KEXINIT 塞进发送队列 → `await` 那个信号量；
发送循环取到 KEXINIT → **把 `_keyReExchangeSemaphore` 重新赋值成另一个新信号量** →
释放旧的那个 → `await` 新的那个；收包循环醒来 → 取走新信号量 → 做完 kex → 释放它。
**一个字段，两次赋值，两个循环交叉持有** —— 能跑，但没人敢动它。

我们换成**一条单向的状态机 + 一个发送闸门**：

```
Dialing → VersionExchange → KeyExchange → Authenticating → Open
                                 ↑                           │
                                 └────── Rekeying ←──────────┘
                                                   Closing → Closed
```

- **发送闸门（SendGate）**：出站帧分两类 —— *传输层消息*（msg id 1–49）与*其余*。
  闸门只有一个布尔：`Rekeying` 期间只放行传输层消息，其余**留在队列里**
  （不是丢弃、不是阻塞调用方）。kex 完成 → 开闸 → 积压的包按序流出。
- 这恰好就是 RFC 4253 §7.1 的原话（"…MUST NOT send any messages other than…"），
  **规范怎么说，代码就怎么写**，不需要两个循环互相递信号量。
- 状态迁移集中在一个 `SessionMachine.Advance(Event)` 里，非法迁移直接抛
  —— 而不是散落在两个 loop 的 `if` 里。

发送路径是**单写者 + 批量合并**：

```
多个生产者 → Channel<OutboundItem>(SingleReader) → SendPump
SendPump: 一次 TryRead 尽量多取（上限按 64 KiB 或 32 项）
          → 逐项处理（报文过闸门后 Seal 进 PipeWriter / 关闸 / 换密钥 / 开闸排空）
          → 一次 FlushAsync → 通知这一轮里等着的发送方（池化的 IValueTaskSource）
```

> 这就是 §5.2 那张表里「一轮 1–2 次 write」的来源。

落地时定下的几条（2026-09-22，§11.2.19）：

- **关闸、NEWKEYS + 换密钥、开闸排空都是队列里的一项**，与报文在同一条队列里 ——
  「关闸之前入队的照常发、之后的被暂存」「NEWKEYS 与换密钥之间插不进任何一帧」
  都由入队顺序保证，不需要锁。
- **背压在入队这一步**（排在泵前面、还没封装上线的字节，含暂存的，上限 16 MiB），
  只对数据面的发送方生效。**接收循环与密钥交换绝不在背压上等** ——
  背压要靠接收循环读到 `WINDOW_ADJUST`、读到对端的 NEWKEYS 才能解除，让它等就是它等它自己。
- **接收循环要回的报文一律「投递」不等待**（CLOSE 应答、请求应答、OPEN_CONFIRMATION、
  UNIMPLEMENTED）：TCP 发送缓冲一满就停下的接收循环，正是两端互相等着对方读的那种死锁。

### 5.5 L5 通道层 —— `IDuplexPipe` 语义 + 自适应窗口

**不同点一：读 API 的形状。** Tmds 的
`SshChannel.ReadAsync(Memory<byte>? stdout, Memory<byte>? stderr, …)`
要调用方同时传两个可空缓冲，并用 `_skippingStdout` / `_skippingStderr` 两个字段
记住「上次是不是跳过了」，传错组合就抛 `InvalidOperationException`。
而且数据是从内部 `Sequence` **拷贝**进用户 `Memory` 的。

我们给每条通道：

```
PipeReader StandardOutput;   // 零拷贝：ReadOnlySequence<byte>
PipeReader StandardError;    // 独立的一条，不会互相饿死
PipeWriter StandardInput;
ValueTask<SshChannelEvent> ReadEventAsync(…);  // Eof / Closed / ExitStatus / ExitSignal / 自定义请求
```

数据面与控制面分开 —— `ExitStatus` 不再和字节流挤在同一个 `ReadAsync` 的返回值里。

**不同点二（性能头条）：自适应窗口。**
Tmds 的 `DefaultWindowSize` 固定 2 MB，窗口在消费到一半时补满（这一点本身是对的，
内存也因此有界）。问题在于**固定窗口同时决定了吞吐上限**：

```
吞吐上限 ≈ 窗口 / RTT
2 MB / 200 ms ≈ 10 MB/s   ← 跨洋链路上 SFTP 怎么也跑不过这个数
```

我们让窗口在 `[256 KiB, 64 MiB]` 间自适应，但**不估 RTT，也不估 BDP**。判据只看本端看得见的两件事：
这一轮（两次回补之间）窗口有没有**见底**（剩余 ≤ 1/8），以及读的一方最近（这一轮或上一轮）有没有**读空了在等**。
两者同时成立才 ×2 —— 窗口太小时，读得快的一方会读空、空等一个往返；读得慢时见底是读的一方造成的，扩窗换不来吞吐。
连续 3 轮没见底就 ×0.75。不用「两次回补隔了多久」这类时间判据：它在局域网上永远成立，会让窗口白涨到上限。
策略经 `SshChannelOptions.WindowPolicy` 暴露（`SshWindowPolicy.Fixed(n)` / `Adaptive(min, max)`），
逐条规则见 `spec/05-connection.md` §3.3。
**局域网上省内存，跨洋链路上跑满带宽。**

> 窗口回补的触发点挂在 `PipeReader.AdvanceTo` 上 —— 消费者不读，窗口就不补，
> 背压是结构性的（原则 2），不需要额外的限流器。

### 5.6 L6 请求账本 —— `RequestLedger<TKey,TResult>`

**不同点：Tmds 有三套各写一遍的 pending 机制** ——
global request 用 `Queue<(TaskCompletionSource, bool)>`（FIFO，靠顺序对齐，没有 id）、
channel request 用 success/failure 两个消息、
SFTP 用 `ConcurrentDictionary<int, PendingOperation>` + 手写的 `IValueTaskSource` 池。
三处的取消语义、超时语义、连接断开时的收尾各写各的。

我们做一个：池化的 `IValueTaskSource<TResult>` + 一张 id 表 + 一条统一的
「连接断开 → 全部以同一个异常收尾」的路径，三处共用。
少三套并发代码，取消与断线的正确性**只需要证明一次**。

〔2026-09-22 落地时的修正〕协议里其实是**两种**账本，硬合成一个反而要引入假的 id：

| 账本 | 用在哪 | 对齐方式 |
| --- | --- | --- |
| `FifoRequestLedger<TResult>` | 全局请求、通道请求 | 没有 id，先发先答；**登记与入队在同一把锁里**（`spec/05` §6.1） |
| `SftpRequestPipeline` 的 id 表 | SFTP | 有 request-id；取消后迟到的应答要**善后**（关掉服务端已经打开的句柄） |

前两处共用一个实现，收尾语义（断线 → 在途的一律以「失败」收尾）只写一次；
SFTP 的「迟到应答要善后」是它独有的，留在它自己那里。

### 5.7 L7 认证 —— 可插拔方法链

```
interface IAuthMethod
{
    string Name { get; }        // "publickey" / "password" / "keyboard-interactive" / ...
    ValueTask<AuthOutcome> RunAsync(IAuthChannel ch, AuthContext ctx, CancellationToken ct);
}
```

内置：`none`、`password`、`publickey`（含证书）、**`keyboard-interactive`**、
`gssapi-with-mic`、`hostbased`、ssh-agent 签名。

`keyboard-interactive` 的回调形状（**直接解锁 VelaShell 的 2FA**）：

```csharp
new KeyboardInteractiveCredential(async (challenge, ct) =>
{
    // challenge.Name / .Instruction / .Prompts[i].Text / .Prompts[i].Echo
    return responses;   // 宿主弹一个对话框，或从 TOTP 生成器取
});
```

签名与密钥解耦为 `ISshSigner`：

```
interface ISshSigner
{
    SshPublicKey PublicKey { get; }
    ValueTask<byte[]> SignAsync(ReadOnlyMemory<byte> data, string algorithm, CancellationToken ct);
}
```

内置：文件私钥、ssh-agent、PKCS#11、Azure Key Vault —— **私钥可以从不进程内**。
这同时是 VelaShell「凭据管理器集成」那条线的落点。

私钥读取支持：OpenSSH v1（含 bcrypt_pbkdf 加密）、PKCS#1、PKCS#8（含加密）、PuTTY `.ppk` v2/v3。

> **VelaShell 收益**：`OpenSshPrivateKey.cs`（303 行 PEM→OpenSSH 转换）整个删掉。

### 5.8 L7 SFTP —— 三段式管线

**不同点：Tmds 的 `SftpChannel.cs` 是 2,291 行的 partial class**，
编解码、流水线、文件语义、目录枚举、上传下载全在里面。

拆成三层，每层可独立单测：

| 层 | 职责 | 可测性 |
| --- | --- | --- |
| `SftpWire` | 纯编解码：`bytes ↔ SftpMessage`。无状态、无 I/O | **纯函数**，可用报文样本逐字节断言 |
| `SftpRequestPipeline` | 流水线深度、in-flight 窗口、重排、取消、扩展协商 | 用 `InMemoryDialer` + 报文脚本测，无需服务器 |
| `SftpFileSystem` | 面向使用者：路径、属性、目录枚举、上传下载、进度 | 集成测 |

**三处能力上的改进：**

1. **写入水位线（write watermark）—— 直接消灭 `ResumeSafetyMargin`。**
   管线记录「已**连续**确认的最高偏移」（contiguous acked offset，而不是「最高确认偏移」），
   `SftpFileStream.DurableLength` 暴露它。断点续传从这个数续，**不用盲退 2 MB**；
   通道断开时把它写进异常，宿主可以直接用。
2. **OpenSSH 扩展一等公民**：`posix-rename@openssh.com`、`hardlink@openssh.com`、
   `fsync@openssh.com`、`statvfs@openssh.com`、`limits@openssh.com`、`copy-data`、
   `home-directory`。协商结果暴露为 `SftpCapabilities`，**宿主能查「这台服务器支不支持」**
   —— 而不是像现在这样，`PosixRenameFileAsync` 只能静默退化成普通 rename。
3. **块大小按 `limits@openssh.com` 定，管线深度按「有没有等过」自适应**，而不是写死 64 × 32 KB：
   服务器宣告的 `max-read-length` / `max-write-length` 给出单个请求的块大小上限；
   在途请求数从 64 起步，每 32 个请求看一次其中有多少是**等着才拿到在途额度**的 ——
   过半在等说明深度是瓶颈，翻倍（上限 256）；一次都没等过就收回四分之一（不低于起步值）。
   不按 RTT 估 BDP：那要先估出带宽，而带宽估计在一条还有别的流量的链路上很不稳；「有没有撞到上限」更直接，也更难估错。

### 5.9 L8 转发与隧道 —— 库内计量

**不同点：Tmds 把 `LocalForward` / `SocksForward` / `RemoteForward` 的搬运做在内部，
一个计数都不给。** VelaShell 只好写 376 行 `MeteredPortForwardHandle` 自己重做一遍转发
（本地/动态自己监听；远程转发更是让库转到本机一个临时监听、再接力一次，多一次环回拷贝）。

我们让 `PortForwarder` 自带：

```
int  ActiveConnections { get; }
long TotalConnections  { get; }
long BytesUp { get; }  long BytesDown { get; }
event EventHandler<ForwardConnectionEventArgs> ConnectionOpened, ConnectionClosed;
event EventHandler<ForwardErrorEventArgs>      Error;
```

并且这些数同时走 `System.Diagnostics.Metrics`（`velashell.ssh.forward.bytes` 等），
宿主要么订阅事件、要么接 OpenTelemetry，两条路都通。

> **VelaShell 收益**：`MeteredPortForwardHandle.cs` 从 376 行缩到一个薄适配（约 60 行），
> 远程转发那条「环回接力」整条消失。

### 5.10 L9 可观测性 —— 失败必须带数据

**① 协商报告随异常送达。**

```
class SshNegotiationException : SshException
{
    NegotiationCategory   Category      { get; }  // KeyExchange / HostKey / Cipher / Mac / Compression
    IReadOnlyList<string> OfferedByPeer { get; }  // 对端 KEXINIT 原样
    IReadOnlyList<string> OfferedByUs   { get; }
    string                PeerVersion   { get; }  // "SSH-2.0-OpenSSH_9.5"
}
```

> **VelaShell 收益**：`SshAlgorithmProbe.cs`（199 行，**为了拿这份名单要再开一条 TCP**）
> 与 `SshAlgorithmDiagnostics.cs`（105 行）合起来缩成一个纯格式化函数。
> 顺带消掉一个副作用：失败后再连一次，对端日志里会多一条建立后立刻断开的连接。

**② 失败原因是强类型，不是字符串前缀。**

```
enum SshFailureReason {
    DnsFailure, TcpRefused, TcpTimeout, ProxyRefused, ProxyAuthRequired,
    VersionMismatch, NegotiationFailed, HostKeyRejected, HostKeyChanged,
    AuthenticationFailed, AuthenticationMethodExhausted, TwoFactorRequired,
    Timeout, ClosedByPeer, KeepAliveTimeout, ProtocolError, Aborted
}
```

每个失败都带 `Reason` + `Phase`（在哪一步）+ `Attempts`（试过哪些认证方法、各自结果）。

> **VelaShell 收益**：`TmdsSshInterop.ExtractConnectFailedReason` 那段
> 「按 `"The connection could not be established - "` 前缀切字符串」删掉。
> 顺带：`TwoFactorRequired` 让「这台机器要 OTP」变成一个可判定的状态，
> 而不是笼统的「用户名或密码不正确」。

**③ 报文级旁路（`IPacketTap`）。**
一个可选接口，拿到每个收发报文的**元信息**（方向、msg id、长度、通道号、序号），
默认不启用、零开销（接口为 null 时整段被 JIT 消掉）。
用于：连接诊断面板、协议级排错、录制回放。**载荷默认不给**，要给必须显式开
`TapOptions.IncludePayload`，并在文档里写明它会带出凭据。

---

## 六 公共 API 形态

### 6.1 命名对照（同时是 §2.2 第 3 条纪律的兑现）

| Tmds.Ssh | VelaShell.Ssh | 为什么这么改 |
| --- | --- | --- |
| `SshClient` | **`SshConnection`** | 换一个隐喻：ADO.NET 的 `SqlConnection` / SignalR 的 `HubConnection` 那套 —— 有 `State`、`StateChanged`、`OpenAsync`/`CloseAsync`。**「客户端」是个物件，「连接」是个有生命周期的东西**，后者才是 VelaShell 要管的 |
| `SshClientSettings` | `SshConnectionOptions` | .NET 的 `*Options` 惯例 |
| `Credential` | `SshCredential` | — |
| `RemoteProcess` | **`SshCommand` / `SshShell`** | **拆成两个。** 一次性命令与交互式 shell 的生命周期、读写形状、退出语义都不一样，现在挤在一个 1,198 行的类里，`HasTerminal` 这种属性就是挤出来的 |
| `SftpClient` | `SftpFileSystem` | 它不是一个「客户端」，它是一个文件系统视图 |
| `SftpFile` | `SftpFileStream` | 它是 `Stream` 的子类，名字就该说这件事 |
| `SftpDirectory` / `ISftpDirectory` | `SftpDirectoryHandle` | — |
| `SshDataStream` | `SshTunnelStream` | — |
| `LocalForward`/`RemoteForward`/`SocksForward` | `PortForwarder` + `ForwardKind` | 三个类的公开面几乎一样，差别只在「谁监听」 |
| `HostAuthentication`（委托） | `IHostKeyPolicy`（接口） | 策略要能带状态（known_hosts 句柄、CA 信任链、本次运行的临时信任），委托做不了 |
| `SftpProgressHandler`（抽象类） | `IProgress<SftpTransferProgress>` | BCL 已有的惯例，不另造 |
| `SshChannel`（internal） | `SshChannelCore` + `SshChannelPipe` | 内部模型不同，见 §5.5 |
| `SshSession`（internal） | `SessionMachine` + `SendPump` + `ReceivePump` | 同上，见 §5.4 |
| `Sequence` / `SequencePool` / `Packet` | **（不存在）** | 用 Pipelines，见 §5.2 |

### 6.2 主线用法（对照 VelaShell 现有调用点）

```csharp
var options = new SshConnectionOptions("root@10.0.0.1:22")
{
    ConnectTimeout = TimeSpan.FromSeconds(15),
    KeepAlive      = new KeepAlivePolicy(TimeSpan.FromSeconds(30), maxMissed: 3),
    Credentials    = [ new PrivateKeyCredential(path, passphrase),
                       new KeyboardInteractiveCredential(PromptAsync) ],
    HostKeyPolicy  = new TofuHostKeyPolicy(store, onDecision: AskUserAsync),
    Dialer         = DialerChain.Socks5("127.0.0.1", 10808),   // 到代理本身默认直连；嵌套用 .Via(...)
    Algorithms     = SshAlgorithmSet.Default.WithCompression(Compression.ZlibOpenSsh),
    AutoConnect    = false,   // 显式连接（VelaShell 现在就是这么配的，而且是对的）
    AutoReconnect  = false,
};

await using var conn = new SshConnection(options, loggerFactory);
await conn.OpenAsync(ct);

// 交互式 shell —— 像素尺寸一等公民
await using SshShell shell = await conn.OpenShellAsync(new ShellOptions {
    TerminalType = "xterm-256color",
    Size  = new TerminalSize(cols, rows, widthPx, heightPx),
    Modes = modes,            // pty-req 的 encoded terminal modes
}, ct);
shell.Resize(new TerminalSize(cols, rows, widthPx, heightPx));

// 一次性命令 —— 三样东西一次拿全
SshCommandResult r = await conn.RunAsync("uname -a", ct);  // StdOut · StdErr · ExitCode · ExitSignal

// 长驻命令 —— 逐行，取消时先 TERM
await foreach (var line in conn.StreamAsync("tail -F /var/log/x", StreamOptions.IncludeStderr, ct)) { }

// SFTP
await using SftpFileSystem fs = await conn.OpenSftpAsync(ct);
if (fs.Capabilities.HasPosixRename) await fs.PosixRenameAsync(a, b, ct);

// 转发 —— 计量在库里
await using PortForwarder fwd = await conn.ForwardAsync(
    ForwardKind.Local, bind: "127.0.0.1:8080", target: "10.0.0.9:80", ct);
Console.WriteLine($"{fwd.ActiveConnections} conn · {fwd.BytesUp + fwd.BytesDown} B");
```

### 6.3 兼容层 `VelaShell.Ssh.Compat.Tmds`（可选）

一个**薄垫片**：用 `Tmds.Ssh` 的类型名与方法签名转发到 VelaShell.Ssh。

- 目的不是长期维护，是**让 VelaShell 的切换分两步走**：
  先换引擎跑通全部 3,194 条测试，再逐个调用点迁到新 API。
- 只覆盖 VelaShell 实际用到的那一面（`SshClient`、`SshClientSettings`、`RemoteProcess`、
  `SftpClient`、`Credential` 族、异常族），**不追求全量兼容**。
- **它自己不含任何 Tmds 的代码** —— 只是同名的类，转发到我们的实现。
  （类型名与方法签名不受版权保护；受保护的是实现。）
- 标 `[Obsolete]`，在 VelaShell 迁完后随大版本移除。

> **决策点**：兼容层是「省事」还是「拖累」有争议。建议**做，但只做 VelaShell 用到的那一面，
> 并从第一天就标 Obsolete** —— 它的价值在于把「换引擎」和「换 API」拆成两次可独立回滚的改动。

---

## 七 性能设计：七个手段与预期收益

| # | 手段 | 原理 | 预期收益 |
| :-: | --- | --- | --- |
| 1 | **发送合并** | 单写者一次取尽队列，逐帧 Seal 进 PipeWriter，一次 Flush | SFTP 高并发写：每轮 syscall **64 → 1~2** |
| 2 | **大块收包** | PipeReader 按 64 KiB 读，替代固定 4 KiB | 32 KiB 包：read **8 次 → 1 次** |
| 3 | **自适应通道窗口** | 见底且读的一方读空过 → ×2，连续 3 轮没见底 → ×0.75，`[256 KiB, 64 MiB]`；不估 RTT / BDP（§5.5） | 200 ms RTT 链路 SFTP：**~10 MB/s → 接近带宽上限** |
| 4 | **SFTP 管线自适应** | 块大小按 `limits@openssh.com`；深度按「请求等在途额度的次数」伸缩（64 起步，上限 256），而非写死 64×32 KB（§5.8） | 高延迟链路吞吐显著提升；小服务器不再被打爆 |
| 5 | **零拷贝读路径** | `PipeReader` 交 `ReadOnlySequence<byte>`，消费者可直接解析 | 去掉 Sequence→Memory 那一次拷贝（每包一次） |
| 6 | **按硬件排默认加密算法** | 有 AES-NI / ARM Crypto 扩展时 AES-GCM 排第一，没有时 chacha20-poly1305 排第一（`SshAlgorithmSet.Default`）；ChaCha20 与 Poly1305 走 BouncyCastle 的 `ChaChaEngine` / `Poly1305`，不自己写，引擎常驻、每个报文只换 nonce（§11.2.22） | 无 AES 硬件加速的设备（部分 ARM）上不落进慢一个数量级的软件 AES |
| 7 | **全路径池化** | `ArrayPool` + `IValueTaskSource`（统一账本 §5.6）+ `PoolingAsyncValueTaskMethodBuilder` | 稳态传输接近零 GC 分配（目标：1 GB SFTP 传输 Gen0 < 50 次） |

**验收方式**：BenchmarkDotNet 套件，与 `Tmds.Ssh` / `SSH.NET` 在同一台靶机上跑同一组场景
（1 GB 上传 / 下载 / 10k 小文件 / 交互式回显延迟 / 建连耗时 / 稳态分配量），
并用 `tc netem` 注入 5 ms / 50 ms / 200 ms RTT 与 0.1% 丢包三档。
**数据进 README，不藏。**

---

## 八 扩展点清单（这一条决定五年后好不好加东西）

| # | 扩展点 | 用来加什么 |
| :-: | --- | --- |
| 1 | `ISshTransportDialer` | 代理、跳板、TUN、内存传输（测试）、异构承载 |
| 2 | `ISshCipherSuite` + `CipherRegistry` | 新加密算法。**含国密 SM4-GCM / SM3**（国内政企的现实需求） |
| 3 | `IKeyExchange` + `KexRegistry` | 新 KEX。后量子（ML-KEM、sntrup761）内置，将来的混合方案照此加 |
| 4 | `IHostKeyAlgorithm` | 新主机密钥类型，含 **CA 签发的主机证书**（`*-cert-v01@openssh.com`） |
| 5 | `ISshSigner` | 私钥从哪来：文件 / Agent / PKCS#11 / HSM / KeyVault / 系统密钥链 |
| 6 | `IAuthMethod` | 新认证方式，含堡垒机的私有扩展 |
| 7 | `IHostKeyPolicy` | 信任模型：known_hosts / CA / TOFU / 企业白名单 |
| 8 | `IIncomingChannelHandler` | 服务端发起的通道：**agent 转发**、X11、`forwarded-tcpip` |
| 9 | `IGlobalRequestHandler` | 服务端全局请求，如 `hostkeys-00@openssh.com`（主机密钥轮换） |
| 10 | `ISftpExtension` | 厂商 SFTP 扩展 |
| 11 | `IPacketTap` / Metrics / ActivitySource | 诊断、录制、APM |
| 12 | `ISshConfigSource` | 配置来源：`~/.ssh/config`、企业下发、UI |

> 这张表就是「后期扩展性」的全部答案。判据很简单：
> **§1 那九条宿主补丁，逐条都能对应到这张表的某一行。**
> 也就是说 —— 如果当初有这十二个扩展点，那九条补丁一条都不用写。

---

## 九 VelaShell 能删掉 / 解锁什么

**能删的（约 1,200 行）：**

| 文件 | 行数 | 结局 |
| --- | :-: | --- |
| `Infrastructure/Net/LoopbackProxyRelay.cs` | ~200 | **删** → `Socks5Dialer` / `HttpConnectDialer` |
| `Infrastructure/Ssh/SshAlgorithmProbe.cs` | 199 | **删** → 协商异常自带名单 |
| `Infrastructure/Ssh/SshAlgorithmDiagnostics.cs` | 105 | 缩成一个格式化函数（~30 行） |
| `Infrastructure/Ssh/MeteredPortForwardHandle.cs` | 376 | 缩成薄适配（~60 行） |
| `Infrastructure/Ssh/OpenSshPrivateKey.cs` | 303 | **删** → 库直接吃 PKCS#1 / PKCS#8 / ppk |
| `TmdsSshInterop.ExtractConnectFailedReason` + 两处调用 | ~60 | **删** → 强类型 `SshFailureReason` |
| `TmdsSftpClientWrapper.ResumeSafetyMargin`（2 MB 盲退） | — | **删** → `DurableLength` 水位线 |
| `TmdsSshClientWrapper` 的「弹窗超时补连一次」那段 | ~40 | **删** → 主机密钥裁决不计入连接超时（见下） |

> 最后一条值得单说：现在主机指纹弹窗摆在那儿的时间**算进连接超时**，
> 用户点完「信任」这一轮已经被判死，只能原地补连一次。
> VelaShell.Ssh 里 `IHostKeyPolicy` 的裁决**独立计时**（`HostKeyDecisionTimeout`，默认无限），
> 这个补丁连同它那段解释性注释一起消失。

**能解锁的：**

| 能力 | 现状 | 之后 |
| --- | --- | --- |
| **2FA / OTP（keyboard-interactive）** | 连不上，文案写「本版无法连接」 | 原生支持，`KeyboardInteractiveSupportTests` 那条引信可以拆 |
| **PTY 像素尺寸** | 恒为 0，卡上游 PR #519 | 一等公民，`TerminalSize` 四个字段 |
| **压缩 zlib@openssh.com** | 卡上游 PR #513 | 内置 |
| **SSH Agent 转发** | 无（对标矩阵里最扎眼的一格） | `IIncomingChannelHandler` + `auth-agent-req@openssh.com` |
| **算法协商可配** | 一半（能诊断，不能配） | `SshAlgorithmSet` 完整可配，且能从 UI 直接列 |
| **posix-rename** | 退化成普通 rename | 显式支持 + 能力查询 |
| **known_hosts 与 OpenSSH 互通** | 只能看和删 | 读写 OpenSSH 格式（含 hashed、`@cert-authority`、`@revoked`） |
| **每条隧道的实时计量** | 自己重写了转发 | 库内直出 + Metrics |

---

## 十 测试与验收

### 10.1 分层测试

| 层 | 手段 |
| --- | --- |
| 编解码 | 报文样本逐字节断言；**SharpFuzz 模糊测试**（解码器是唯一直面不可信输入的地方，必须 fuzz） |
| 密码套件 | RFC 8439 / NIST CAVP 向量逐条；与 BCL 交叉验证 |
| 状态机 | 属性测试：随机事件序列不得进入非法状态、不得死锁 |
| 通道 / 窗口 | 内存传输 + 报文脚本，断言窗口回补时机与背压 |
| SFTP | `SftpWire` 纯函数测 + 管线脚本测 |

### 10.2 内存传输（`InMemoryDialer`）—— 这一条很关键

Tmds 的测试基本都要起 Docker 里的 sshd。我们让 `ISshTransportDialer` 可以返回
**一对内存双工流**，另一端接我们自己写的**最小测试服务端桩**（只为测试存在，不发布）。
于是：**协议层的绝大部分测试不需要网络、不需要容器、毫秒级跑完**，
可以进每次提交的门禁 —— 而不是像现在这样排除掉 `DockerIntegration` 再跑。

### 10.3 互操作矩阵（这个才是 SSH 库的真正验收）

| 对端 | 为什么必须测 |
| --- | --- |
| OpenSSH 8.x / 9.x / 10.x | 事实标准；8.x 还很常见 |
| OpenSSH for Windows | 行为与 POSIX 版有差异 |
| Dropbear | 嵌入式设备、路由器 |
| Cisco IOS / 华为 VRP / H3C | **老网络设备**：只有 `aes128-ctr` + `hmac-sha1`，甚至 `diffie-hellman-group14-sha1`。VelaShell 的用户里一定有人要连这个 |
| CentOS 7 的 OpenSSH 7.4 | 同上 |
| AWS Transfer / Azure SFTP / 各家托管 SFTP | SFTP 扩展支持面差异极大 |
| GitHub / GitLab（仅 exec） | 最常见的公开端点 |

**已落地并实测通过**（见 §11.2.9、§11.2.13）：互操作矩阵跑 OpenSSH
`latest` 与 `9.3` 两个版本，13 条 `[TestCategory("Interop")]` 用例。
**2026-09-21 本机实测：OpenSSH 10.3 与 9.3 各 13/13 通过** ——
而第一次跑的时候是 **0/13**，逮到了一个密钥派生的真 bug（§11.2.13）。
本机可用 `scripts/ssh/interop/Start-TestServer.ps1 -X11` 起同一套环境。
**表里其余对端还没接** —— 它们要么要买设备（网络设备），要么要账号
（托管 SFTP），不是写几行 YAML 能解决的。

### 10.4 安全验收

- 门禁：`CodeQL` + `dotnet list package --vulnerable`
- **必须实现**：`strict KEX`（`kex-strict-c-v00@openssh.com`，Terrapin 缓解）、
  RSA 最小长度检查、算法降级防护、认证失败计数与退避
- 第三方审计：发 1.0 前请一次外部审计（这是可卖授权的产品，不是玩具）

---

## 十一 工程与里程碑

### 11.1 仓库形态

新仓库 `VelaShellLabs/velashell-ssh`，与 VelaShell 主仓平级。
NuGet 包名 **VelaShell.Ssh（单包）**;解决方案共两个项目(库 + 测试),
内部分层用文件夹,见 §4。
TFM：**`net11.0` 单目标**（不做多目标 —— 省掉 polyfill、条件包引用与 `#if` 分支，
并且可以直接吃 .NET 11 的 BCL：`AesGcm`、`ChaCha20Poly1305`、ML-KEM、Pipelines 的新 API）。
`LangVersion` 取 `preview`（跟进 C# 最新语法，库自己就是最早的试用场）。
`IsAotCompatible=true`，零反射，`PublicApiAnalyzers` 钉住公开面。

### 11.2 里程碑

| M | 内容 | 出口判据 | 估算 |
| :-: | --- | --- | :-: |
| **M0** | 行为规格（§2.2 第 2 条的「规格」）+ 骨架 + 内存传输 + 测试服务端桩 | 规格评审通过；空跑的 CI 全绿 | 2 周 |

| **M1** | L1–L4：拨号、帧、密码套件、版本交换、KEX、认证（password / publickey / kbdint） | 能连上 OpenSSH 并认证 | 4 周 |
| **M2** | L5：通道、exec、shell、pty、signal、exit-status | 能跑交互式 shell 与一次性命令 | 3 周 |
| **M3** | L7 SFTP：三段式管线 + 扩展协商 + 水位线 | SFTP 上传 / 下载 / 枚举 / 属性 / 链接全通 | 4 周 |
| **M4** | L8 转发：`-L` / `-R` / `-D` + 计量 + agent 转发 | 三种转发通，计量准 | 2 周 |
| **M5** | 周边：known_hosts、ssh_config、私钥全格式、ppk、Agent 客户端、压缩 | 与 OpenSSH 共用一份 known_hosts | 3 周 |
| **M6** | 兼容层 + **VelaShell 整体切换** | VelaShell 3,194 条测试全绿 | 2 周 |
| **M7** | 性能与互操作：Benchmark、窗口调优、互操作矩阵、fuzz、审计 | §7 指标达成；§10.3 矩阵全绿 | 4 周 |

### 11.2.1 M0 进度（2026-09-21）

| 项 | 状态 |
| --- | --- |
| 仓库骨架（MIT / NOTICE / net11.0 单目标 / LangVersion=preview / 中央包管理） | ✅ |
| **行为规格 9 份**（`spec/00`–`08`） | ✅ 待评审 |
| CI（构建测试 · 公开面门禁 · 打包冒烟 · interop） | ✅ 已写，未在真实 runner 上跑过。interop 当时是占位，现已接真容器（§11.2.9） |
| 公开面门禁 + `scripts/ssh/Update-PublicApi.ps1` | ✅ |
| 骨架代码：wire 原语、发送闸门、失败分类、拨号抽象 | ✅ **72 个用例全绿** |
| **内存传输**（`InMemoryTransport` + `ISshTransportDialer`） | ✅ 含半关闭、背压、取消 |
| 测试服务端桩（会说 SSH 的那一端） | ✅ 随 M1 落地（`TestSshServer` + `TestAuthServer`） |
| 链路特征模拟（单向时延 / 带宽 / 丢包） | ⏳ 自适应窗口与 SFTP 管线深度要靠它才能真正测到 |
| 依赖面 | ✅ **运行时零依赖**（`Microsoft.Extensions.Logging.Abstractions` 在 net11 随框架自带） |

**一处与原计划的偏差，已记录：**

1. **收敛成两个项目**（库 + 测试），内部用文件夹分层 —— 原计划是 Core + 主包两个包。
   理由见 §4。

### 11.2.2 M1 进度（2026-09-21）

L1–L4 全部落地，**197 个用例全绿**。完整握手与认证在内存里跑通：
版本交换 → 算法协商 → 密钥交换 → 验签 → 主机密钥策略 → 换密钥 → 加密收发 → 用户认证。

| 项 | 状态 |
| --- | --- |
| L1 传输：`SshPacketTransport`（Pipelines 行式 + 帧式 IO、发送合并） | ✅ |
| L2 密码套件：`ISshCipherSuite` + `CipherSuiteShape` | ✅ 明文 / AES-GCM / ChaCha20-Poly1305 / AES-CTR+HMAC（EtM 与 MtE 两条路径） |
| L3 密钥交换：curve25519 · ECDH（3 条曲线）· DH group14/16 · 混合 PQ（ML-KEM-768 / sntrup761） | ✅ 7 种 |
| 交换哈希与密钥派生（含扩展轮） | ✅ 两侧独立算出的 H 逐字节比对 |
| 严格 KEX（Terrapin 缓解，CVE-2023-48795） | ✅ 含「KEX 期间注入 IGNORE 必须断开」的攻击用例 |
| 主机密钥：ed25519 / ecdsa×3 / rsa 解析、指纹、验签（算法名不符直接拒） | ✅ 5 种 |
| L4 认证：`SshAuthenticator` + 四种凭据 + `ISshSigner` | ✅ none / password / publickey / keyboard-interactive |
| **部分成功（2FA）** | ✅ 公钥 + 动态码两步认证有端到端用例 |
| `server-sig-algs`（RFC 8308）决定 RSA 签名算法 | ✅ 默认拒绝 SHA-1 降级，需显式开关 |
| 认证失败诊断（逐条 `SshAuthAttempt`） | ✅ 「私钥读不出来」「服务端不认这把钥」「需要动态码」三者可区分 |
| 覆盖面 | 7 种 KEX × 5 种主机密钥 × 6 种加密 × 4 种 MAC，以及失败路径 |

**M1 期间抓到的三个真 bug**（都不是测试写错，记下来是因为它们都只在真实时序下出现）：

1. **ECDH 无效点在 Windows 上没被拦住** —— CNG 把 `CryptographicException` 包在另一层里，
   原先的 `catch (CryptographicException)` 漏过去了。改成按「不是我们自己的异常就算失败」拦截。
2. **`DisposeAsync` 抛 `IOException`** —— 对端已走时 `PipeWriter.CompleteAsync` 会抛，
   把每条错误路径上真正的失败原因盖住。释放路径现在不抛。
3. **版本交换只包了读没包写** —— 对端在我们发标识串时就消失的话，抛的是裸 `IOException`
   而不是 `SshConnectException(ClosedByPeer)`。

**一条设计缺口在写测试时才暴露**：认证步骤的「说明」（例如「服务端要求先改密码」）
原先在 `ReadAuthOutcomeAsync` 里被构造出来又丢掉，只有结果传了出去。
现在 `AuthStepResult` 把结果与说明一起往上传 —— 否则那条失败在界面上会退化成
一句没有内容的「认证失败」，而这正是用户最需要知道的那一句。

**M1 未做（推到后续里程碑，已记录）**：

- 私钥文件解析（OpenSSH v1 / PKCS#1 / PKCS#8 / ppk）与 ssh-agent 客户端 → M5。
  目前只有 `InMemorySshSigner`（进程内持有私钥）。
- `known_hosts` 与 `ssh_config` → M5。目前只有 `IHostKeyPolicy` 抽象。
- 改密码流程（`PASSWD_CHANGEREQ`）—— **明确不做**，但给出可读的失败原因。
- 与真实 OpenSSH 的互操作矩阵 —— 需要 CI runner，本地跑不了。
  **→ 已补，见 §11.2.9。**

### 11.2.3 M2 进度（2026-09-21）

L5 连接协议落地，**228 个用例全绿**。会话、通道、流控、请求全部跑通。

| 项 | 状态 |
| --- | --- |
| `SshConnection`：接收循环 + 发送锁的多路复用器 | ✅ |
| `SshChannel`：三条管子 + 一条有序事件流 | ✅ stdout / stderr 是**两条独立的 PipeReader** |
| 流控窗口，**回补挂在 `AdvanceTo` 上而不是报文到达上** | ✅ `WindowedPipeReader` |
| 自适应窗口策略（`Fixed` / `Adaptive`） | ✅ 反馈环已接上，见 §11.2.8 |
| 会话窗口总预算 + 通道数上限 | ✅ 撞满时拒绝开新通道，**不断开会话** |
| 通道号延迟复用（防串话） | ✅ 默认 30 秒 |
| EOF 单向半关闭 / CLOSE 双向 | ✅ 有「发完 EOF 仍然收得到输出」的端到端用例 |
| 通道请求 FIFO（**没有 id，靠顺序对齐**） | ✅ 失步时报 `ProtocolError` |
| 全局请求 FIFO + 保活 | ✅ 未知请求**必回** `REQUEST_FAILURE`（沉默会让对端 FIFO 永久错位） |
| `exec` / `shell` / `subsystem` / `pty-req` / `window-change` / `env` / `signal` | ✅ |
| `exit-status` / `exit-signal`，`ExitCode` 为 `int?` | ✅ 三种情况（有码 / 被信号杀 / 什么都没有）各有用例 |
| 像素级终端尺寸 | ✅ 一路贯通到 `window-change` |
| 服务端发起的通道 | ⏳ 明确回 `ADMINISTRATIVELY_PROHIBITED`，M4 再实现 |

**M2 抓到的两个真 bug**，都只在特定时序下出现：

1. **`exit-status` 被读成别的数。** 通道拿到的是整段报文载荷，却只跳过了一个字节的
   消息编号 —— 而 `CHANNEL_REQUEST` 的头部还有通道号与一个**变长**的类型串。
   现在由 `SshConnection` 按 `reader.Consumed` 切好「类型相关的那一段」再交给通道：
   让下游自己去跳过变长头部，迟早会有人按固定长度跳。
2. **窗口一点一点漏光。** 窗口回补泵攒不够阈值时，把这一轮已消费的字节**丢掉**了。
   对端视角的窗口会逐步缩小到零，症状是传了一阵子之后通道永久停住，
   而本地账面上看窗口明明是满的。消费者逐行读（每次几十字节）时最容易撞上。
   已改成跨轮累加，并补了一条「每次只读 64 字节、总量 6 个窗口」的用例 ——
   把 bug 注回去，这条用例和另一条窗口用例都会挂到超时。

**M2 未做（已记录）**：

- **自适应窗口的 RTT 反馈环**。`SshWindowPolicy.Adaptive` 目前只提供上下限，
  按 RTT 自动伸缩要等 `InMemoryTransport` 能模拟单向时延与带宽 ——
  没有链路特征模拟，「窗口翻倍」这类逻辑没有任何东西可验证，写了也是空转。
  这条和 M0 记下的那条待办是同一件事。
- 服务端发起的通道（`forwarded-tcpip`、`auth-agent@openssh.com`）→ M4。
- `hostkeys-00@openssh.com` 全局请求 → M5。

### 11.2.4 M3 进度（2026-09-21）

SFTP 三层落地，**287 个用例全绿**（其中 23 条是 `SftpWire` 的逐字节断言，不需要对端）。

| 项 | 状态 |
| --- | --- |
| `SftpWire`：纯函数编解码（bytes ↔ 报文，无状态无 I/O） | ✅ 跨段 `ReadOnlySequence` 也能解 |
| `SftpRequestPipeline`：**按 request-id 对齐，不靠顺序** | ✅ 服务端故意倒着回也能对上 |
| `SftpFileSystem`：面向使用者的完整 API | ✅ |
| **写入水位线 `DurableLength`** | ✅ 精确续传点，不用盲退在途窗口 |
| `AckedRangeSet`：有序数组 + 尾部快路径 | ✅ 顺序写时集合大小恒为 1 |
| `SftpWriteMode.Sequential` | ✅ 牺牲吞吐换「文件长度就是可信长度」 |
| 块大小按 `limits@openssh.com` 定 | ✅ 没有扩展时退回 32 KiB |
| **能力可查**（`SftpCapabilities`） | ✅ `posix-rename` 不支持时**不静默降级** |
| 符号链接口径：`LSTAT` + 补跟随的 `STAT`/`READLINK`，**并发发出** | ✅ 含断链用例 |
| `SSH_FXP_SYMLINK` 的参数顺序（OpenSSH 而非 draft） | ✅ 两条用例钉住它 |
| ATTRS 的三个坑（共用标志位 ×2、有符号时间、类型藏在权限高位） | ✅ 各有用例 |
| `EOF` 不是错误；`READ` 可以短读 | ✅ 服务端每次只回 100 字节也能读全 |
| 取消后迟到应答的句柄善后 | ✅ `onLateResponse` 关掉泄漏的句柄 |
| 句柄泄漏检查 | ✅ 读写列目录之后服务端句柄数归零 |

**M3 抓到的两个自身缺陷**（都在写测试时暴露）：

1. **`Order` 的乱序桩会挂死。**测试服务端「攒够两条应答再倒着发」，
   而握手（`INIT` → `VERSION`）是严格的一问一答 —— 唯一那条应答被永远压在手里。
   这是桩的 bug，不是库的，但它暴露的是同一类问题：
   **任何「攒够 N 个再处理」的逻辑都必须有一条「再也等不到了」的出路**。
   现在按「客户端是否还有别的在途请求」决定压不压。
2. `HighestAckedOffset` 的断言我自己写错了（写成区间起点而非终点），实现是对的。

**M3 未做（已记录）**：

- `statvfs@openssh.com`、`copy-data`、`home-directory`、`expand-path@openssh.com`
  的封装 —— 能力位已经能查到，但还没给出便捷方法。它们都不在 VelaShell 的既有用法里。
- `ISftpExtension` 公开扩展点（架构 §8 第 10 项）——
  目前加厂商扩展要直接用 `SftpRequestPipeline.SendAsync` + `SftpWire.WriteExtended`，
  可用但不够体面。
- SFTP 层的 BDP 自适应（在途请求数按 RTT 调整）—— 与 M2 的自适应窗口是同一条待办，
  都要等链路特征模拟。**→ 已补，见 §11.2.9。**

### 11.2.5 M4 进度（2026-09-21）

转发四种形态全部落地，**311 个用例全绿**。

| 项 | 状态 |
| --- | --- |
| `DuplexRelay`：**与 SSH 无关**的搬运循环 | ✅ 5 条用例不碰任何 SSH |
| 逐方向半关闭（TCP `shutdown(SEND)` ↔ `CHANNEL_EOF`） | ✅ 有「一端发完之后另一端仍能继续发」的用例 |
| 计量：事件 + `System.Diagnostics.Metrics` **两条路都给** | ✅ 字节数在搬运循环里算，不含协议开销 |
| 本地转发 `-L` | ✅ 端到端：真 socket → 真 SSH 会话 → 回来 |
| 动态转发 `-D`（SOCKS5 子集） | ✅ **域名不本地解析**，失败码如实映射 |
| 远程转发 `-R` | ✅ 含「端口 0 时从应答载荷取实际端口」 |
| 直连隧道（无监听） | ✅ TCP 与 Unix 套接字两种 |
| 入站通道（`IIncomingChannelHandler`） | ✅ 未登记的类型**明确拒绝**，不沉默 |
| 单条连接失败不影响转发器 | ✅ 拒绝之后还能接下一条 |
| 端口被占用时不留半挂的监听 | ✅ |
| 默认绑环回 | ✅ 要对外开放必须显式写出来 |

**M4 补上的一个真缺口**：`SendGlobalRequestAsync` 原本只返回成不成，
而 `tcpip-forward` 请求端口 `0` 时，**服务端分配的实际端口就在
`REQUEST_SUCCESS` 的载荷里**。只回布尔值的话那个端口号永远拿不到，
后面按 `(bind_addr, 0)` 去路由回连一条都对不上 ——
症状是「转发看起来建好了，但连过来的全被拒」。
现在加了 `SendGlobalRequestWithReplyAsync`，返回 `SshGlobalRequestReply`（含载荷）。

**M4 期间的两条工具教训**（记下来是因为它们都浪费了时间）：

1. 用 Perl 的 `s/\Q…\E/…/` 做代码替换时，**模式里不能直接写 XML 文档注释** ——
   `///` 里的 `/` 会提前终止正则。一律改用 heredoc 取出字面量再替换。
2. `q{…}` 作为 Perl 的字面量分隔符要求花括号配对，
   而 C# 片段经常是不配对的（例如从某个块的中间切一刀）。同样改用 heredoc。

**M4 未做（已记录）**：

- **Agent 转发**（`auth-agent-req@openssh.com`）。机制上只差一个
  `IIncomingChannelHandler` 实现 + 本机 agent 的连接，但 §7.2 的三条安全约束
  （默认关闭、只转发指定密钥、逐次签名确认）要和 M5 的 agent 客户端一起做才完整 ——
  先有「连本机 agent」的能力，才谈得上「只转发其中某几把钥」。
  **→ 已补，见 §11.2.9。**
- `streamlocal-forward@openssh.com`（远程 Unix 套接字转发）——
  直连方向的 `direct-streamlocal` 已经有了，反方向随 agent 转发一起补。
- 取消远程转发后的宽限期目前是写死的 2 秒，没有做成可配置。

### 11.2.6 M5 进度（2026-09-21）

配置与密钥这一层落地，**356 个用例全绿**，源码 78 个文件 / 约 17,700 行。

| 项 | 状态 |
| --- | --- |
| `known_hosts` 解析（明文 / `[host]:port` / **散列形式**） | ✅ 写坏的行跳过，不让整个文件失效 |
| `KnownHostsPolicy`：TOFU / 严格 / 指纹钉死 | ✅ 「没见过」与「变了」走两条路 |
| 密钥变了时的消息 | ✅ 说清是哪台、新指纹、**该去删第几行** |
| `ssh_config` 解析 | ✅ **先出现的值赢**（与多数配置格式相反） |
| 私钥：OpenSSH v1（未加密） | ✅ ed25519 / rsa / ecdsa×3，**签了再验**才算过 |
| 私钥：PKCS#8（含带口令）、PKCS#1、SEC1 | ✅ 走 BCL |
| `ssh-agent` 客户端（Windows 命名管道 / Unix 套接字） | ✅ 私钥从不进本进程 |
| `SshPublicKey.SignatureAlgorithms` | ✅ agent 签名要用它挑算法 |

**M5 抓到的一个真 bug —— 而且是安全相关的**：

`KnownHostsFile.Lookup` 原先一碰到「主机 + 密钥都对上」就立刻返回 `Known`。
但 `@revoked` 那一行通常是**追加**在文件后面的（撤销一把密钥最自然的动作
就是往末尾加一行）。于是「撤销了等于没撤销」——
旧的信任行排在前面，吊销永远轮不到被看见。
现在把整份表扫完才下结论，吊销一旦匹配立刻赢。

**M5 有意留下的一个缺口** —— **已于 2026-09-22 补上，见 §11.2.18**：

**读不了加密的 OpenSSH 私钥**（`-----BEGIN OPENSSH PRIVATE KEY-----` 且
`cipher != none`）。它的口令派生用 `bcrypt_pbkdf`，而那要拿到 Blowfish 的
**密钥编排内部**（`EksBlowfishSetup` / `expandstate`）——
BCL 没有，BouncyCastle 的 `BlowfishEngine` 也只暴露 `Init` + `ProcessBlock`。

> 这个判断到今天仍然成立。变的是取舍：`ssh-keygen` 带口令时的默认产物就是
> 这种格式，所以「读不了」等于「绝大多数人手里的那把钥读不了」。
> 最终给「不自己写密码学原语」开了一个**有记录、范围钉死**的例外，
> 见 §11.2.18。

**M5 未做（已记录）**：

- PuTTY 的 `.ppk` —— 异常消息里提示用 PuTTYgen 转换。**→ 已补，见 §11.2.9。**
- 压缩（`zlib@openssh.com`）—— 算法协商里已经有位置，编解码还没接。
  它对交互式会话几乎没有收益（终端输出本来就小），对 SFTP 才有意义。
  **→ 已补，见 §11.2.9。**
- `ssh_config` 的 `Include` 与 `Match` —— 当时判为「有意不做」。
  **→ 已补，见 §11.2.12**（`Match exec` 默认仍然不执行）。
- agent 转发（与 M4 那条是同一件事，现在 agent 客户端有了，可以接上了）。
  **→ 已补，见 §11.2.9。**

### 11.2.7 连接层（面向使用者的那一层，2026-09-21）

M0–M5 把每一层都做出来了，但调用方还得自己把
拨号 → 版本交换 → 密钥交换 → 裁决 → 认证 拼起来 —— 那不叫可用。
这一步把 §6.2 里写的那个形状补上，**369 个用例全绿**。

| 项 | 状态 |
| --- | --- |
| `SshConnectionOptions`（`user@host:port` 解析，含 IPv6 方括号） | ✅ |
| `SshConnectionFactory.ConnectAsync` / `options.ConnectAsync()` | ✅ 一步到可用 |
| `TcpTransportDialer` | ✅ socket 错误翻成 `DnsFailure` / `TcpRefused` / `TcpTimeout` / `TcpUnreachable` |
| **三把独立的计时器** | ✅ 连接 / 裁决（默认无限）/ 认证（默认 2 分钟） |
| 保活：**基准是「上次收到任何报文」** | ✅ 忙的时候一次都不发 |
| 判死抛 `KeepAliveTimeout` 而不是笼统的 `Timeout` | ✅ 上层的重连策略只该对这一类生效 |
| `conn.RunAsync()` → `SshCommandOutput`（三样一次拿全） | ✅ `EnsureSuccess()` 把 stderr 带进异常 |
| 默认主机密钥策略**不是**「接受任何密钥」 | ✅ 有用例钉住 |
| [`getting-started.md`](../getting-started.md) | ✅ |

**这一步改掉的一个设计缺陷**：工厂自己的超时原先一律报
`SshFailureReason.Timeout`，不管是卡在拨号、版本交换、密钥交换还是认证。
可是这四件事的下一步完全不同 ——
「DNS 慢」「端口被防火墙丢包」「对端不是 SSH 服务」「用户没输动态码」
在界面上该说四句不同的话。现在按阶段给原因码与消息：

```
连 10.0.0.1:22 时在「密钥交换」这一步超时（限 30 秒）。
```

它也解释了一次偶发的测试失败：连 `127.0.0.1:1` 时，
若 socket 被静默丢包，先到的是工厂的计时器而不是拨号器的，
于是原因码从 `TcpTimeout` 退化成 `Timeout`。修好之后连跑 6 轮全绿。

### 11.2.8 链路模拟与自适应窗口（2026-09-21）

这是 M0 就记下、一直拖到现在的那条待办：**没有链路特征模拟，
「自适应」那段代码没有任何东西可验证** —— 在一条零延迟、无限带宽的
内存链路上，窗口是 32 KiB 还是 64 MiB 跑出来一样快。

**376 个用例全绿。**

| 项 | 状态 |
| --- | --- |
| `LinkCharacteristics`（单向时延 + 带宽） | ✅ 含 `LocalNetwork` / `Intercontinental` 预设 |
| `DelayedStream`：给任意流套上时延与限速 | ✅ 限速用「下一次允许发送的时刻」，不累加调度抖动 |
| 通道接收窗口的自适应反馈环 | ✅ |
| 对比用例：自适应 vs 固定 | ✅ 比**往返次数**，不比墙钟 |

**两个判断上的修正**（第一版写错了，都是在测试里暴露的）：

1. **扩窗必须当场把多出来的额度授予对端。**
   第一版只改本地的 `Size`，结果是：回补阈值（`Size/2`）涨了，
   而对端手上的窗口没涨 —— 它发不出更多数据，我们也攒不够下一次回补的量。
   **两边一起停住，谁都不动。**两个窗口用例直接挂到 60 秒超时。

2. **扩窗的判据是「窗口有没有被吃到见底」，不是「两次回补隔了多久」。**
   第一版用间隔 < 250 ms 当信号 —— 但局域网上这永远成立，
   会让窗口一路涨到 64 MiB 白占内存，而那条链路上窗口根本不是瓶颈。
   现在在 `OnData` 里记录「剩余掉到 1/8 以下」这个事实，只在真吃紧时翻倍。

**还有一处是测试方法上的修正**：对比用例原先比的是墙钟时间，
四轮里挂了一轮 —— 忙碌的机器上那种断言本来就不可靠，
而**一个偶尔失败的测试比没有测试更糟**。改成比服务端收到的
`WINDOW_ADJUST` 次数：每一次回补都意味着对端先把窗口用光、等了一个 RTT，
往返少吞吐就高。

**但「这个数是确定性的」这句话，当时说早了。**
它后来又在一次满跑里挂了一次（单独跑必过），原因有两个，都在测试侧：

1. **消费粒度跟着调度走。** 回补泵是按「消费了多少」触发的，
   所以消费的粒度直接决定回补条数；而原先每轮 `AdvanceTo(buffer.End)`
   一次吃掉「此刻恰好到了多少」—— 机器一忙，几个包并成一次读，
   回补就少发几条。量到的是调度噪声，不是窗口策略。
   **改成每轮固定消费 8 KiB**（`AdvanceTo(consumed, consumed)`，
   examined 必须等于 consumed，写成 `buffer.End` 会把剩余数据锁在管道里）。
2. **计数器是在回补还在飞的时候读的。** 单向 20 ms 的链路上，
   数据收完那一刻最后几条 `WINDOW_ADJUST` 还没到服务端，
   此时读到的是「到目前为止飞到的条数」—— 一个随机数。
   **改成轮询到它不再变**。这不是「等够久就算过」的性能断言，
   它等的是一个确定会发生的终止事件：数据收完了就不会再有新的回补。

顺带把观测计数器改成 `Interlocked.Increment` + `Volatile.Read` ——
加它的是服务端循环的线程，读它的是测试线程，原先两边都没有同步。

**但上面两条都不是那次失败的真正原因。** 把失败信息抓全之后，
断言是 `预期 524288，实际 294912` —— 不是条数对不上，是**数据只收到一半就 EOF 了**。
再复现一次，换了个面孔：消费者那边抛出
`Reading is not allowed after reader was completed`。

同一个 bug 的两副面孔，而它在库里，不在桩里：

#### 接收管道的水位是按**起步窗口**算的，而窗口会长过它

```csharp
// 老代码
int window = options.WindowPolicy.InitialBytes;
PipeOptions pipeOptions = new(
    pauseWriterThreshold: window * 2L,   // ← 起步值的两倍
    resumeWriterThreshold: window, …);
```

这行代码背后有一句注释写着「对端**不可能**发来超过窗口的未消费数据，
所以写这一侧永远不该阻塞」。那句话是对的 —— **但只对固定窗口成立**。
自适应策略下窗口会一路长到 `MaximumBytes`（默认 64 MiB），
**一旦长过起步值的两倍，前提就没了。**

而接收循环正是靠这个前提才敢不等 flush：

```csharp
ValueTask<FlushResult> flush = writer.FlushAsync();
if (flush.IsCompletedSuccessfully) { … return; }   // 正常路径
_ = ObserveFlushAsync(flush);                      // 老代码：丢后台，接着写
```

顶到水位之后 flush 不再同步完成，于是下一个报文到达时，
**上一次 flush 还没完成就又写了同一个 `PipeWriter`** —— 那是对 `Pipe` 的误用。
它不会当场报错，只会在之后某个时刻让消费者拿到一句
「Reading is not allowed after reader was completed」，或者干脆少收一半数据。

两处改动：

1. **水位按 `MaximumBytes` 算**，不是 `InitialBytes`。
   固定策略下两者相等，对它毫无影响；自适应下前提重新成立。
   水位只是水线，不预分配内存，而实际堆积量本来就受窗口限制。
2. **那条「丢后台接着写」的路直接抛。** 按新水位它已经不可达了 ——
   真走到就说明我们自己的窗口记账坏了，那时候
   **指着真正原因报错，远好过继续把管道写坏**。

回归用例 `窗口长满之后数据依然一字节不差` 把上限设成起步值的 16 倍，
逼窗口长过老水位，然后只断言「数据一字节不差」。
把 bug 注回去，它**每次**都挂（不再是八轮一次）。

> 三个教训，按分量排：
> 1. **别急着相信自己对失败的第一个猜测。** 我一开始笃定是回补条数的统计噪声，
>    还据此改了两处 —— 那两处改得没错（确实是噪声源），但都不是原因。
>    是**把失败信息抓全**才找到真凶的。
> 2. **写下「不可能」的时候，要写清它依赖什么。** 那句注释没错，
>    错在它的前提后来被自适应窗口改掉了，而注释和代码都没跟着动。
> 3. **偶发失败别用重跑糊过去。** 它当时只有八分之一的概率 ——
>    而它在真实链路上的对应物是「大文件偶尔传丢一半」。

#### 顺带：让桩学会「挂了要说话」

排查过程里最贵的一段，花在一个**没有任何信息的 30 秒超时**上。
原因是测试服务端的**三条**后台路径都在静静地吞异常：

- `RunAsync`（收包循环）没有兜底的 catch —— 一旦抛异常，循环就停了，
  服务端从此不再应答；
- `PlayScriptAsync`（剧本回放）只接 `OperationCanceledException` ——
  别的异常让 Task 悄悄 faulted，EOF 与 CLOSE 于是永远发不出去；
- `PumpHandlerOutputAsync`（子系统输出搬运）写着
  `catch (Exception) { /* 通道没了。测试桩不为此喧哗。*/ }` ——
  注释把「不喧哗」当成了美德。SFTP 那一类用例里搬运一挂，
  客户端就在等一个再也不会来的响应。**这是同一个毛病的第三处**，
  也是最后被揪出来的一处。

三种情况下客户端看到的都不是错误，而是**一个永远不返回的 `await`**，
最后由 `test.runsettings` 里的全局超时收场。
`Harness.DisposeAsync` 还把 `await _serverChannels` 的异常吞掉了 ——
于是真正的原因连一次露面的机会都没有。

现在三条路径都会：**记下原因**
（`Observation.ServerFault` / `ScriptFault` / `SubsystemFault`）、
**主动把对面叫醒**（收掉连接，或补发 EOF + CLOSE ——
让客户端立刻读到「对端不干了」而不是挂着），
并在 `Harness.DisposeAsync` 里把它抬成一条指名道姓的失败。

> **桩的诊断质量决定排查成本。** 一个会说「我为什么挂了」的桩，
> 和一个只会超时的桩，差的不是几行代码，是几个小时。
>
> 而「同一个毛病连着出现三次」本身就是个信号：
> **静默的 `catch` 不是稳，是把账记到了将来。**
> 后台路径上的每一个 catch 都该回答一句「谁会发现这件事」——
> 答不上来的，就是下一个 30 秒超时。

排查途中还顺手修了测试桩里一处真实的竞态（不是这次的原因，但迟早会咬人）：
服务端用一个 `Dictionary<uint, long>` 记「还能往客户端发多少」，
收包循环处理 `WINDOW_ADJUST` 时加、发送泵扣额度时减，两边都是
「读旧值 → 算新值 → 写回」且没有同步 —— 教科书式的丢失更新。
现在所有读写都在同一把锁里，且**查与扣在同一个临界区**（分开写等于没锁）。

**当时仍未做**：SFTP 管线深度的自适应（在途请求数按 RTT 调整）。
链路模拟现在有了，它具备了可验证的前提 —— 但 SFTP 层的在途窗口
（在途请求数 × 块大小）默认 64 × 32 KiB = 2 MiB，
在 200 ms RTT 上同样是 10 MB/s 封顶，这条还立着。**→ 已补，见 §11.2.9。**

**合计约 24 周**（单人全职口径）。规模估算 **15,000–22,000 行**（不含测试；测试约再 1.5 倍）。

> 这不是一个周末项目。**决定做之前请先认下这个量。**
> 如果只是想解决 2FA 和像素尺寸两件事，更省的路是给上游提 PR —— 我们已经提过两个
> （#513、#519 那条线），这条路是通的。
> **自己实现的理由必须是 §1 整张表，不是其中一两行。**

### 11.2.9 收尾六件事（2026-09-21）

§11.2.1–11.2.8 每一节末尾都挂着一张「未做」清单。这一节把上面那些
「→ 已补，见 §11.2.9」一次性结清：**压缩、agent 转发、SFTP 管线深度自适应、
`.ppk`、性能基准、与真实 OpenSSH 的互操作矩阵。**

**427 个用例：414 绿、13 条互操作用例在没有服务端时自报 Inconclusive。**

| 项 | 状态 |
| --- | --- |
| 压缩 `zlib@openssh.com` / `zlib` | ✅ 逐方向持久流 + `Z_PARTIAL_FLUSH`；解压有上限 |
| Agent 转发 `auth-agent-req@openssh.com` | ✅ 解析转发（非字节透传），三条安全约束都落地 |
| SFTP 管线深度自适应 | ✅ 按「有没有等过信号量」伸缩，不猜 RTT |
| PuTTY `.ppk` v2 / v3 | ✅ 含加密私钥（v2 走 SHA-1 KDF，v3 走 Argon2id） |
| 性能基准（BenchmarkDotNet） | ✅ 单文件应用，`scripts/ssh/benchmarks/benchmarks.cs` |
| 互操作矩阵（真 sshd 容器） | ✅ 13 条用例 × 2 个 OpenSSH 版本，本地按需跑（`scripts/ssh/interop/`） |

#### 压缩：难点不在 zlib，在 flush 语义

SSH 的压缩不是「每个包独立压一遍」—— 那样几乎压不动。它是**一条跨包的持久
zlib 流**，每个包压完做一次 flush 把字节挤出来，**但不重置字典**：
第 100 个包仍然能引用第 1 个包里出现过的字符串。对应到 zlib 就是
`Z_PARTIAL_FLUSH`。

第一版据此得出了一个**错误结论**：「BCL 的 `ZLibStream` 不暴露 flush 模式，
所以只能自己接一份 zlib」—— 于是用了 BouncyCastle 的 `ZStream`，
并把它记成「BouncyCastle 唯一一处非密码学用途」。

**这个结论是错的**，纠正见 §11.2.10。

两个判断：

1. **`zlib@openssh.com` 与 `zlib` 不是同一个东西**，差别只有一个：前者
   **等认证通过之后才启用**。原因是压缩率本身就是一条旁路 ——
   压缩后的长度泄露明文的可压缩性，在认证阶段这意味着对口令做区分攻击
   （CRIME 那一类）。`SshCompressorFactory.IsDelayed` 就是这条，
   启用点在 `SshConnectionFactory` 认证成功之后。
2. **解压必须有上限，而且要在写出之前检查。**
   一个 32 KiB 的包能解出几百 MiB —— 对端只要愿意，就能用一个包打爆我们的内存。
   先算长度再写，不是写着写着发现超了再回滚。

基准的结论很直白。实测（22,800 字节的合成日志行）：

```
可压缩文本  22,800 → 159 字节      （约 1/143 —— 这是合成输入的最好情况，
                                    真实日志一般在 1/5 ~ 1/10）
随机数据    22,800 → 22,813 字节   **变大了**
```

所以默认仍然不开压缩 ——
`Compression` 要显式设。交互式会话开它基本是净亏，SFTP 传文本才值。

#### Agent 转发：为什么必须解析，不能透传

把 agent 通道当成一根字节管子对接到本机 agent，是最省事的做法 ——
也是我们**明确没有选的**做法。一旦透传，§7.2 那三条安全约束就全都无从实现：
你不知道对端在要哪把钥的签名，自然也就谈不上「只转发指定密钥」
或者「逐次确认」。

所以 `AgentForwarder` 把 agent 协议**解析一遍再转发**：

- `AllowedKeys` 非空时，`REQUEST_IDENTITIES` 的应答里**过滤掉**不在名单里的钥
  （`KeysHidden` 会计数），`SIGN_REQUEST` 要的钥不在名单里直接回 `FAILURE`。
- `ConfirmEachSignature` 给宿主一个异步回调，可以弹窗问人。拒了回 `FAILURE`。
- `ADD_IDENTITY` / `LOCK` / `UNLOCK` 一类会**改本机 agent 状态**的消息
  **一律回 `FAILURE`**，不转发。远端服务器没有任何理由改我们本机的钥圈。
- `MaxConcurrentChannels` 封住通道数，默认 8。

默认整件事是**关的**：不调 `AgentForwarder.RequestAsync` 就没有转发。

#### SFTP 管线深度：用「有没有等过」而不是 RTT

M2 的自适应窗口踩过一次坑（§11.2.8 第 2 条）：拿「两次回补隔了多久」当信号，
在局域网上永远成立，窗口一路白涨。这次没有重犯 —— **信号是「有没有真的被挡住」。**

`SftpRequestPipeline` 的在途请求数是一个 `SemaphoreSlim(initial, ceiling)`。
发请求时**先试 `Wait(0)`**：拿到了说明管线还没满；没拿到就计一次
`_saturationHits` 再去正常等。每 32 个请求评估一次：

- 超过一半的请求等过 → 深度**翻倍**（并 `Release` 出多的额度），封顶 `ceiling`；
- 一次都没等过 → 收回 25%，但不低于初始值。

这个信号的好处是它**自带链路信息**：高 RTT 长肥管道上，在途请求会迅速填满
信号量，于是它涨；局域网上一个请求还没发完下一个就有位置，它就不涨。
不用去估 RTT，也就不会估错。

#### `.ppk`：能做，是因为 PuTTY 用的 KDF 拿得到

〔当时〕M5 那条「加密的 OpenSSH 私钥读不了」（§11.2.6）还立着 ——
`bcrypt_pbkdf` 要 Blowfish 的密钥编排内部，BC 只给 `Init` + `ProcessBlock`，
而架构原则 6 不允许自己写原语。**这一条已于 2026-09-22 补上，见 §11.2.18。**

`.ppk` 当时就能做：v2 的 KDF 是 SHA-1 的拼接，v3 是 Argon2id，
**两个 BC 都直接给**。所以 `.ppk` 连加密的一起支持 ——
这不是双标，是「有没有现成原语」这一条的直接结果。

实现上唯一容易写错的地方：**MAC 是对解密后的私钥明文算的，不是密文。**
写反了的表现是「口令对也报 MAC 不匹配」，而这句话会把人引向完全错误的方向。
`VerifyMac` 另外容忍两种长度候选（补齐后的和 `NaturalLength` 的），
因为历史上的生成器在这里不完全一致。

#### 性能基准：声称了就要能量出来

README 里那几条性能主张（合并写、零拷贝分帧）**不能只是话术**。
`scripts/ssh/benchmarks/benchmarks.cs` 是一个**单文件应用**（src/VelaShell.Ssh/AGENTS.md §3.2：
脚本走 PowerShell 或 C# 单文件，不新建项目），量四组：
密码套件吞吐、分帧与发送合并、压缩、SFTP wire 编解码。

```
dotnet run -c Release -p:SignAssembly=false scripts/ssh/benchmarks/benchmarks.cs
dotnet run -c Release -p:SignAssembly=false scripts/ssh/benchmarks/benchmarks.cs -- --filter '*Wire*'
```

两个工程上的坑，都写进了脚本头部的注释：

1. **必须用 `InProcessNoEmitToolchain`。** BDN 默认会为每组基准重新生成一个
   项目再编译，而单文件应用根本没有 `.csproj` 给它找。`Emit` 那一版在构建
   包装代码时直接抛异常（只报一句 `Build Error: Exception!`）。
2. **不要在命令行传 `--job`** —— 它会覆盖掉上面那个工具链，于是又回到第 1 条。

它当场逮到了一处真问题：`SftpWire.WriteWrite` 原先和其它 `Write*` 方法一样，
**先把整个报文拼进一个中转缓冲，再整体写出** —— 对 32 KiB 的数据块来说，
那等于每块多搬一遍。改成「一次 `GetSpan` 写完定长头部，
再把数据体直接 `Write` 到 output」之后，同一台机器上的 A/B：

```
| 方法       | 平均      | 比值  | 分配      |
|-----------|----------|------|----------|
| 直写       |  2.53 us | 1.00 |  32.1 KB |
| 中转缓冲    | 11.06 us | 4.37 | 160.1 KB |
```

**快 4.4 倍，分配少 5 倍。**上传路径上每个块都要走一次，这个位置值得。

> 表里的「中转缓冲」是按老写法重写的对照实现，不是当时那份代码的逐字复原
> （它的分配比当时记录的 98 KB 更高一些，因为中转缓冲从默认容量开始扩）。
> 结论的方向和量级是稳的，**但别把 4.37 这个数当成历史事实引用**。

> 这些数字**不进 CI 门禁**。BDN 的结果受机器负载影响太大，当门禁只会天天误报；
> 它的用处是**改动前后在同一台机器上对比**。

#### 互操作矩阵：这一条才是真正的验收

前面 414 个用例都跑在 `InMemoryTransport` 加自写的测试服务端上。
它们能验证的只有一件事：**我们对规范的理解和自己一致。**
真正的坑在「OpenSSH 实际怎么做」与「规范怎么写」的**差值**里 ——
SFTP 的 `SYMLINK` 参数顺序就是最有名的一个（`SftpWire.WriteSymLink`
头上那段警告注释专门拦「好心修正」）。

13 条用例（`[TestCategory("Interop")]`），覆盖：握手与跑命令、
**每一种 KEX 各握一次**、**每一种加密算法各收发一次**、公钥认证、
4 MiB 输出的完整性、stdin、退出码 vs 信号、PTY、SFTP 1 MiB 往返、
真实目录里的符号链接、压缩协商、本地转发、保活应答。

两处刻意的设计：

1. **没有服务端时自报 `Assert.Inconclusive`，不是红。**
   一个在本机永远红着的测试，很快就会被所有人忽略 ——
   然后它在真的坏掉的那天也不会有人看。
2. **测试用的 ed25519 密钥有两份：一份不加口令，一份加。**
   〔2026-09-22〕加密的 OpenSSH 私钥现在读得了（§11.2.18），
   而那条路径只有对着真 `ssh-keygen` 写出来的文件才验得了，
   所以脚本另做一份带口令的同一把钥。

本地跑：

```powershell
pwsh scripts/ssh/interop/Start-TestServer.ps1 -X11   # docker 起 sshd、装 xauth、等 banner
. artifacts/interop/env.ps1
dotnet test tests/VelaShell.Ssh.Tests/VelaShell.Ssh.Tests.csproj --filter "TestCategory=Interop"
pwsh scripts/ssh/interop/Stop-TestServer.ps1
```

互操作矩阵跑**两个** OpenSSH 版本：`latest`（后量子 KEX 在这里才有）
与 `9.3`（验证我们没有默默依赖新算法 —— 真实世界里有大量停在几年前的设备）。
它**不在 CI 里**：要起容器，慢且依赖网络。原先的 CI `interop` job 只在推 main 与手动触发时跑，
在 PR 的检查列表里永远显示为一项 Skipped，2026-09-23 移除，改为按上面的命令在本地按需跑。

**到这里，§11.2 各节列出的待办只剩下这些**（都是有意留的，不是忘了）：

- 加密的 OpenSSH 私钥 —— `bcrypt_pbkdf` 拿不到。**→ 已补，见 §11.2.18。**
- `streamlocal-forward@openssh.com` **→ 已补，见 §11.2.12。**
- `ssh_config` 的 `Include` 与 `Match` **→ 已补，见 §11.2.12。**
- `ISftpExtension` 公开扩展点；`statvfs@openssh.com` 等扩展的便捷封装。
- 取消远程转发后的宽限期写死 2 秒，未做成可配置。

### 11.2.10 压缩改走原生 zlib（2026-09-21）

§11.2.9 里写过一句结论：

> 这一条决定了不能用 BCL 的 `ZLibStream` —— 它不暴露 flush 模式。

**这句话错了，而且错得很贵**：它让我们为一件 BCL 本来就能做的事，
引了一个第三方的托管 zlib 实现，还把「依赖面小」这个卖点上划了一道口子。

#### 错在哪

前提没错：SSH 要的是「flush 但不重置字典」，OpenSSH 用 `Z_PARTIAL_FLUSH`，
而 BCL 确实不让你选 flush 模式。

漏掉的是下一步：**`Stream.Flush()` 在 deflate 流上做的是 `Z_SYNC_FLUSH`，
它同样不重置字典。** `Z_SYNC_FLUSH` 与 `Z_PARTIAL_FLUSH` 的差别只有一个 ——
前者会多吐一个空的存储块（`00 00 FF FF`），**每个报文多 4 个字节**，
而两边的解压器都认。也就是说：要的语义 BCL 一直都有，只是没有那个名字。

> 教训：**「API 没暴露这个选项」不等于「做不到这件事」。**
> 要问的是「我要的语义有没有别的名字」，而不是「有没有这个参数」。

#### 换了之后

`ZlibCompressor` 现在用 `ZLibStream` 跑运行时自带的原生 zlib，
两侧各一条持久流，每个报文 `Flush()` 一次。
配两个很薄的 `Stream` 适配器把 BCL 的流式 API 接到我们的
`IBufferWriter<byte>` / `ReadOnlySequence<byte>` 上 ——
**压缩器吐多少就直接写进调用方的 writer，解压直接按段读，两边都没有中间缓冲。**
（旧实现在解压时要先把载荷 `ToArray()` 一份。）

同一台机器上的 A/B（一条流连压 8 个报文）：

```
| 载荷    | BCL 原生   | BouncyCastle | 快      | 分配                      |
|--------|-----------|--------------|--------|--------------------------|
| 4 KiB  |  17.0 us  |     97.3 us  |  5.7×  |  1.43 KB vs 311 KB (218×) |
| 32 KiB |  57.9 us  |    850.9 us  | 14.7×  |  4.48 KB vs 536 KB (120×) |
```

差距这么大不奇怪：BouncyCastle 的 zlib 是一份 **JZlib 的托管移植**，
而 BCL 走的是运行时里的原生 zlib。

**顺带把依赖的故事也理干净了**：BouncyCastle 现在**只做密码学原语**，
没有第二种用途 —— `Directory.Packages.props` 与 `NOTICE.md` 都已经改过来。

#### 一个必须处理的坑：`UseStrictValidation`

`System.IO.Compression.UseStrictValidation` 是一个 AppContext 开关（默认关）。
打开之后，「读到没有更多数据」会被判成流被截断并抛 `InvalidDataException`。
而 **SSH 的 zlib 流是一直 flush、永不结束的** —— 于是
**每一个报文解完都会撞上它**。不处理的话，开了这个开关的应用一连上就全线报错。

处理方式：只在「这一次读本身没有产出任何数据」时把它当作载荷结尾
（数据已经由前面几次读取回来了）。**解出来什么都没有的载荷不豁免** ——
那仍然报错，否则非法的 zlib 数据会被静静放过去。

这件事没法放进常规单元测试：开关只能在进程启动时设一次，
而读它的静态字段一个进程只读一次，同进程并行的用例没法各设各的。
所以它是一个**单独的单文件脚本**，CI 里单跑一条：

```bash
dotnet run scripts/ssh/compression/verify-strict-validation.cs
```

它验三件事：开关打开时报文能往返、字典确实跨报文保留住（线上字节逐个变小）、
非法数据仍然被拦下。

#### 这一段代码的来源

实现思路来自 VelaShell Labs 自己提给上游的
[tmds/Tmds.Ssh#513](https://github.com/tmds/Tmds.Ssh/pull/513)（作者 `joesdu`，
即本项目作者），**不是第三方的代码**。从那里取的是三条**事实**：
`Z_SYNC_FLUSH` 与 `Z_PARTIAL_FLUSH` 在 SSH 场景下等价（代价 4 字节/报文）、
`CompressionLevel.Optimal` 对应 zlib level 6（OpenSSH 用的级别）、
以及 `UseStrictValidation` 的这个坑。

代码本身是照我们自己的接口（`ISshCompressor` + `IBufferWriter`/`ReadOnlySequence`）
重写的 —— 上游那份是包在它们的 `IPacketEncryptor`/`IPacketDecryptor` 上的，
结构完全不同。

> 记一句：**净室纪律挡的是「照抄别人的结构」，不是「不许知道事实」。**
> `Z_SYNC_FLUSH` 不重置字典是 RFC 1951 的事实，从哪儿知道的都一样。
> 区别在于知道之后是自己写，还是把别人的文件拷过来。

### 11.2.11 密钥重协商（2026-09-21）

这一条不是「补个协议细节」，是**修一个会让长连接断掉的洞**。

OpenSSH 的 `RekeyLimit` 默认 1 GiB 或 1 小时，到点它自己发 `SSH_MSG_KEXINIT`。
客户端不应答的表现不是「少个功能」，而是：

- 挂了一下午的 shell 忽然断了；
- 传到一半的大文件断了。

VelaShell 是终端产品，长连接和大文件正是它的日常 —— 所以这条是硬需求。
另外两个理由：AES-GCM 的 nonce 是确定性推进的，同一把密钥下 nonce 重用会泄漏
认证密钥并直接导致可伪造（所以规格里写的是「接近 2⁶⁴ 时**强制**换」）；
以及每次重协商都是一次新的 DH / ML-KEM，一把会话密钥泄露只暴露那一个窗口。

**437 个用例：424 绿 + 13 条互操作自报 Inconclusive。**

| 项 | 状态 |
| --- | --- |
| 接住服务端发起的重协商 | ✅ 收包循环上原地跑完 |
| 发送闸门真的接进发送路径 | ✅ **之前它只是个有用例的摆件** |
| 压缩上下文随密钥重置 | ✅ 与密钥在同一个临界区里换 |
| `session_id` 跨重协商不变 | ✅ 有用例钉住 |
| 重协商期间的通道数据 | ✅ 收方向照常派发；发方向暂存后按序流出 |
| 连续多次重协商 | ✅ |

#### 发送闸门原来没接上

`SendGate` 是 §5.4 讲得最细的一个组件，有一整个 `SendGateTests.cs`，
**但它从来没有被接进 `SshConnection.SendAsync`。** 也就是说在这次改动之前，
即使我们能应答重协商，也会在 `KEXINIT` 与 `NEWKEYS` 之间继续发通道数据 ——
那是明确的 RFC 4253 §7.1 违规。

> 教训：**「有实现 + 有单元测试」不等于「接上了」。**
> 一个组件的单元测试全绿，只说明它自己是对的，不说明有人在用它。
> 这种洞在架构文档里尤其容易长出来 —— 文档把它写成核心，读文档的人
> （包括三小时前的我）就默认它已经在工作了。

接上之后 `SendAsync` 的形状是：取票 → 拿发送锁 → 问闸门
（`Send` / `Stashed` / `StashFull`）→ 满了就松锁去等再重试。
**满了要等，不能丢**：对端迟迟不完成重协商时，宁可让发送方背压，
也不能无界地攒。

#### 两处「必须原子」

1. **发 `NEWKEYS` 与换发送侧状态。** 发出 `NEWKEYS` 之后我们发的下一个报文
   就要用新密钥；中间被别的发送者插进来，那一帧会用旧密钥发出而对端已经在用
   新密钥解。所以它们在同一个 `_sendLock` 临界区里完成。
2. **压缩上下文和密钥同时换。** 压缩在加密之前，两者都以 `NEWKEYS` 为界。
   为此 `ISshKexTransport` 的两个换档方法都接一个 `ISshCompressor?` ——
   **首次交换传 `null`**，因为 `zlib@openssh.com` 要等认证成功之后才启用
   （传 non-null 就把那条 CRIME 防护破了）。这个「可为 null」不是图省事，
   它编码的正是首次与重协商的语义差别。

#### 为什么在接收循环上原地跑

`ISshKexTransport` 这个抽象的存在理由：首次交换时传输上只有一个读者一个写者，
直接读写就行；重协商时读归接收循环、写归发送锁，密钥交换必须借道。

而重协商的密钥交换**就在接收循环上跑完**，不另起任务 —— 因为
「读到对端的 `NEWKEYS`」与「换上新的接收密钥」之间**一个报文都不能插进来**。
放到别的任务上，这个顺序就得靠额外的同步去保证，那是白找麻烦。

代价是重协商期间接收循环被占着，所以 `RekeyKexTransport.ReadPacketAsync`
会把**非传输层报文就地派发掉**，只把密钥交换自己的报文交出去 ——
否则一条正在跑的 SFTP 会在重协商的那一两个 RTT 里整个停住。

#### 顺带逮到一个更严重的 bug：压缩的中转缓冲被两个方向共用

写重协商 + 压缩的用例时，它挂在了**重协商之前** —— 8 KB 的载荷收到 0 字节。
而同样的连接、同样的压缩、1.9 KB 的载荷次次都过。

原因在 `SshPacketTransport`：

```csharp
// 老代码 —— 一个缓冲，两个方向
private readonly ArrayBufferWriter<byte> _compressionBuffer = new(4096);

WritePacket:      _compressionBuffer.ResetWrittenCount(); … Compress(payload, _compressionBuffer);
DecompressPayload: _compressionBuffer.ResetWrittenCount(); … return _compressionBuffer.WrittenMemory;
```

**收与发是并发的**：接收循环在读，N 条通道的泵在写。解压刚把载荷写进缓冲、
调用方还没用完，一个并发的 `WritePacket` 就 `ResetWrittenCount()` 把它清了 ——
调用方拿到一段长度为 0 或者半截的载荷。

症状为什么这么迷惑：载荷小的时候一发一收之间几乎没有交错的机会，撞不上；
数据量一上来就开始「收到 0 字节」或者「报文载荷为空，没有消息编号」，
**而这两句话都指不到压缩**。

修法就是两个方向各一个缓冲。但值得记下的是**为什么单元测试没抓到它**：
`CompressionTests` 里连「大载荷能往返」都有，可它们都是
**先压完再解压**的顺序驱动 —— 而这个 bug 只在「压和解压交错」时出现。

> 教训：**并发的东西要用并发的方式测。**
> 顺序驱动的用例能验协议理解，验不了共享状态。
> 现在补了一条 `整条连接开着压缩也能跑通`：走完整的连接建立流程、
> 真的并发收发 —— 这一条能抓住它，把 bug 注回去它立刻挂。

#### 主动发起（同日补完）

阈值落在 `SshRekeyPolicy`（默认 **1 GiB / 1 小时 / 2³¹ 个报文**，默认**开着**）。
`SshConnection.StartRekeyAsync()` 也公开出来，宿主可以自己挑时机。

**报文数那一条才是硬线。** SSH 的序号是 32 位的，而 AES-GCM 的 nonce
每个报文推进一次 —— 两者都在 2³² 处出事，而 nonce 重用对 GCM 是**灾难性**的
（可以恢复认证密钥，进而伪造）。字节数与时长只是 RFC 4253 §9 的建议。
顺带说一句：**不需要把 GCM 的 invocation counter 暴露出来** ——
它每个报文推进一次，所以「这把密钥下发过多少报文」就是它的精确值，
而那个数每种密码套件都有，ChaCha20-Poly1305 的 32 位序号同理。

三条路径在 `OnPeerKexInitAsync` 里汇成一条：

| 谁发起 | `_ourPendingKexInit` | 行为 |
| --- | --- | --- |
| 对端 | `null` | runner 去发我们的 `KEXINIT` |
| 我们 | 已发出的那份 | 交给 runner，**别再发第二个**（重复发是协议违规） |
| 同时 | 已发出的那份 | 与「我们发起」完全一样 —— RFC 4253 §7.1 说这合法且只做一次 |

所以「两边同时发起」这个看起来要特殊处理的情况，**一行额外代码都不需要**。

`StartRekeyAsync` **发完 `KEXINIT` 就返回**，不等谈完：密钥交换要读对端的报文，
而这条传输唯一的读者是接收循环。正在谈的时候再调是空操作。

诊断面：`LastRekeyReason` 说清是哪条阈值触发的（「单向报文数达到 1024（阈值 1024）」），
`RekeyCount` 是完成次数。排障时要能回答「这条连接刚才为什么换了密钥」。

##### 一个自己踩出来的 API 陷阱

第一版的 `SshRekeyPolicy` 里，`MaxInterval` 的 `default` 被翻译成「1 小时」
（因为 `TimeSpan` 不能做 record struct 的默认参数值）。后果是
**`Disabled` 里的时长那一条根本关不掉** —— `Disabled.IsEnabled` 居然是 `true`。

是 `阈值低于下限会被当场拒绝` 里那句 `Assert.IsFalse(Disabled.IsEnabled)` 把它揪出来的。

改法是把语义摆平：**三条阈值都是「0 表示不看这一条」**，于是
`default(SshRekeyPolicy)` 与 `Disabled` 是同一个东西 ——
一个全零的结构体就该是「什么都不做」。有主张的那一组值写在 `Default` 里，
写全了，不靠参数默认值去暗示。用例里补了一句
`Assert.AreEqual(Disabled, zeroed)` 钉住它。

> 教训：**「默认参数值」不该承载语义。**
> 它看起来省事，实际是在两个地方各写了一半的真相 ——
> 而那两半一旦对不上（比如 `Disabled` 想关掉一条被默认值打开的阈值），
> 编译器一句话都不会说。

##### 怎么在用例里攒够一千个报文（试错两次）

验「监视循环真的在盯」要让报文数越过 1024。试了两条路都不行：

1. **反复跑命令** —— 每条命令是开通道 + 请求 + 数据 + 退出状态 + 关闭
   一整套往返。单独跑没问题，满跑并行时慢到撞 25 秒超时。
2. **往 stdin 灌 4000 个一字节的块** —— stdin 泵会把它们合并，
   实测 4000 次写只产生了**一两个**报文（整条连接总共才 10 个）。
   这一条是被 `PacketsSent` 的诊断打印直接量出来的。

真正可控的旋钮是**对端的分片粒度**：服务端按我们宣告的 max packet 切分。
把 `ReceiveMaxPacketBytes` 调到 256 字节、让服务端吐 300 KB，
就是一千多个报文 —— 便宜、确定，用例从 6～25 秒降到 **1.2 秒**，
而且顺带验了「重协商发生在传输中途也不会弄丢数据」。

> 教训：**要一个量上去，先找那个真正决定它的旋钮。**
> 前两次都是在「多做几次」上使劲，而决定报文数的根本不是次数，是分片粒度。

#### 还没做

- 客户端发起时对端的行为差异（比如某些实现会拒绝或延迟应答）——
  真实互操作上没验过，互操作用例里还没有一条跨 1 GiB 的用例
  （跑一次要搬 1 GiB，不适合放进常规 CI）。

### 11.2.12 收掉「有意留的缺口」（2026-09-21）

三个缺口，两个关掉了，一个关不掉 —— 而**关不掉的那个，根因是一行写错的文档**。

**449 个用例：436 绿 + 13 条互操作自报 Inconclusive。**

#### `streamlocal-forward@openssh.com`

反方向的 Unix 套接字转发（`ssh -R /远端/路径:/本机/路径`）。
典型用途是把本机的 `docker.sock`、数据库套接字交到远端 ——
走套接字而不是端口，远端机器上的其它用户**看不到也连不上**（文件权限说了算）。

和 `tcpip-forward` 的差别只有三处：全局请求的名字、把 `addr ‖ port` 换成一个
`string socket_path`、回连走 `forwarded-streamlocal@openssh.com`。
**计量、并发槽、搬运、事件、收尾全都一模一样** —— 所以没有另开一个类，
而是在 `RemoteForwarder` 上加了一个可空的 `RemoteSocketPath`，
非空即 Unix 套接字模式。三处分流，其余共用。

顺带补了一个公开面：`RemoteEndpointName` —— TCP 是 `bind:port`，
套接字是路径。隧道面板要显示「这条转发开在哪」，两种形态得有统一的说法，
否则调用方就得自己写那个三目运算（架构原则 4）。

用例 4 条，其中一条是**真的端到端**：起一个真的 Unix 域套接字，
让服务端开一条 `forwarded-streamlocal` 回来，两边对搬。
（`Socket.OSSupportsUnixDomainSockets` 在 Windows 10+ 上是 true，所以不用跳过。）

#### `ssh_config` 的 `Include` 与 `Match`

原来的判断是「有意不做」，理由是 `Match exec` 意味着解析一份配置文件就能跑任意程序。
**这个理由只对 `exec` 那一条成立**，却把整个 `Match` 和无辜的 `Include` 一起毙了。

现在两个都做，各带一条约束：

- **`Include` 只在 `LoadAsync` 里展开。** `Parse` 是纯文本解析，没有基准目录，
  也不该碰文件系统。展开时有**深度上限（16，与 OpenSSH 一致）与环检测** ——
  `a` include `b`、`b` 又 include `a` 很容易写出来，而没有环检测的表现是
  **读配置的时候整个进程不动了**。有一条用例专门钉这个。
  通配结果**排序**：目录枚举顺序在不同文件系统上不一样，
  而 `ssh_config` 是「先出现的值赢」—— 顺序不定就意味着结果不定。
- **`Match exec` 默认不执行。** 没有求值器时，带 `exec` 条件的块**一律不匹配**。
  要用就自己传一个 `SshConfigMatchContext.ExecEvaluator` 进来 ——
  于是「要不要在本机跑外部命令」这个决定**明确地落在调用方身上**，
  而不是藏在库的默认行为里。

`Match` 支持 `all` / `host` / `originalhost` / `user` / `localuser` / `exec`，
条件之间是与，支持 `!` 取反。`canonical` / `final` **永远不匹配** ——
我们不做主机名规范化，静默当成真会让一份为规范化写的配置产生完全不同的结果。

两个判断：**信息不足就不匹配**（不知道用户时 `Match user` 不猜一个），
**不认识的条件不匹配**（认错了会让一个本不该生效的块生效）。

一个解析上的坑：`Match exec "test -f /etc/special"` 里那条命令是**一个**参数。
不认引号的话它被切成三段，后两段变成两个不认识的条件，于是整个块静默失效 ——
是那条 exec 用例把它揪出来的。`exec` 的参数也**不能按逗号切**（命令里可以有逗号）。

#### 加密的 OpenSSH 私钥：当时关不掉，根因找到了

> **〔2026-09-22 后记〕这一条已经补上了，见 §11.2.18。**
> 下面的技术判断全部仍然成立 —— 变的不是事实，是那张表最后一行的取舍：
> 「维持现状」的代价被重新估过，结论是它比「开一个有记录的例外」更贵。

`src/VelaShell.Ssh/AGENTS.md` §3.3 里写着「BCL 缺的走 BouncyCastle（…… `bcrypt_pbkdf` ……）」。

**那一行是错的，而这个错误直接造成了这个缺口。**

实际查了一遍 BouncyCastle 2.7.0：`Org.BouncyCastle.Crypto.Generators.BCrypt`
确实有 `DeriveRawKey` / `EncryptMagicString` / `ProcessTableWithSalt`，
但它们**全是 `internal`**，而且公开的 `Generate` 做的是
**2^cost 轮的标准 bcrypt**，不是 bcrypt_pbkdf 的
「64 轮 `expand0state` + 不同的魔术串 + 字节序交换」那个变体。
`BlowfishEngine` 也只给 `Init` + `ProcessBlock`，拿不到密钥编排内部。

所以结论没变 —— **但理由从「我记得不行」变成了「查过，确实不行」**，
`src/VelaShell.Ssh/AGENTS.md` 那一行也已经改正。

剩下的三条路，都不是我该替使用者选的：

| 路 | 代价 |
| --- | --- |
| 自己写 `bcrypt_pbkdf` | 违反 §3.3「不自己写密码学原语。**没有例外。**」 ← **最终选了这条，并把那句改成「一个例外，写明在案」（§11.2.18）** |
| 再引一个依赖 | 依赖面是这个库的卖点之一；而且没找到合适的（许可 + 维护度 + 只为这一件事） |
| 维持现状 | 两条可照做的路已经在错误消息里：`ssh-keygen -p -m PKCS8`，或者走 ssh-agent |

> 这里值得记一句：**「有意不做」和「当时以为做不了」是两回事。**
> 三个缺口里有两个属于后者 —— 一个是把 `exec` 的风险扩大到了整个 `Match`，
> 一个是一行写错的依赖清单。
> 缺口清单该定期**重新验证**，而不是一直抄下去。

### 11.2.13 接上真实 OpenSSH：一个自己永远测不出来的 bug（2026-09-21）

互操作用例写好之后一直没跑过 —— 没有 Docker。这次跑起来了，
**第一次连真实的 OpenSSH 就全线失败**。

而本地的 449 个用例，全绿。

#### 症状

```
✘ kex=curve25519-sha256          -> 报文完整性校验失败
✘ kex=ecdh-sha2-nistp256         -> 报文完整性校验失败
✘ kex=mlkem768x25519-sha256      -> 报文完整性校验失败
✘ cipher=aes256-gcm@openssh.com  -> 报文完整性校验失败
✘ cipher=aes128-gcm@openssh.com  -> 报文完整性校验失败
✘ cipher=chacha20-poly1305       -> 帧头非法：packet_length=231406730
✘ cipher=aes256-ctr              -> 报文完整性校验失败
```

**每一个 KEX 都失败，每一个密码套件都失败** —— 所以问题不在任何一个具体算法里，
在它们共用的那一段。而 chacha20 那条尤其说明问题：它解出来的长度字段是纯噪声，
说明**我们手上的密钥根本不对**。

#### 根因：密钥派生里多写了两个长度前缀

RFC 4253 §7.2：

```
K1 = HASH(K || H || "A" || session_id)
```

那个 `||` 是**直接拼接**。`K` 看起来「带长度前缀」，但那是因为它本身
以 mpint 编码 —— 前缀是 mpint 的一部分。**`H` 与 `session_id` 是裸字节。**

我们写成了：

```csharp
WriteValue(ref prefixWriter, sharedSecret, secretEncoding);
prefixWriter.WriteString(exchangeHash);   // ← 多了 4 字节长度前缀
...
roundWriter.WriteByte((byte)letter);
roundWriter.WriteString(sessionId);       // ← 又多了 4 字节
```

改成 `WriteRaw` 之后，七条组合全部连上，**包括后量子的 `mlkem768x25519-sha256`**。

#### 为什么 449 个用例一个都没抓到

这是整个项目里最值得记下的一件事。

1. **签名照样验得过。** `H` 的计算走的是另一条代码路径
   （`SshExchangeHash.Compute`，那里 `string` 前缀是对的）。
   所以双方对 `K` 和 `H` 的看法**完全一致**，服务端的签名验证顺利通过 ——
   握手看上去一切正常，直到第一个加密报文解不开。
2. **自写的测试服务端用的是同一个 `DeriveKey`。** 两边一样错，于是完全互通。
   `TestSshServer` 从来没有说过「你算错了」，因为它自己也这么算。
3. **单元测试把错误抄了一遍。** `派生的第一轮就是公式本身` 那条用例里，
   期望值是用 `w.String(h)` 拼出来的 —— 它断言的正是那个 bug。
   修完实现之后，是这两条用例先红的。

> §10.3 开头那句话本来就写着：
> 「我们自己的测试桩再全，也只能验证**我们对规范的理解与自己一致**。」
> 这次是它第一次被真正兑现 —— 而代价是，在此之前我一直以为这个库是能用的。

#### 顺带修掉的两条**用例自己的** bug

真服务端也把测试写得不严的地方照了出来：

| 用例 | 问题 |
| --- | --- |
| `压缩能与OpenSSH协商上` | 断言 `StandardOutput.Length > 50000` —— 那是**字符数**，而每行 13 个中日韩字符 = 14 字符 / 40 字节。改成按行数与字节数断言 |
| `本地转发能穿过真实服务端` | 隧道连 `127.0.0.1:22`，而容器里 sshd 听的是 **2222** —— 服务端回「Connection refused」，看上去像转发坏了 |

还有一条环境问题：13 条用例**并行**连同一台 sshd，撞上 OpenSSH 的
`MaxStartups`（默认 10 个未认证并发连接），多出来的在**发出版本标识串之前**
就被丢掉 —— 症状是「对端在发出版本标识串之前关闭了连接」，看上去像我们的 bug。
给那个类加了 `[DoNotParallelize]`。

#### 现在的状态

```
OpenSSH 10.3（latest）   13/13 ✔
OpenSSH 9.3              13/13 ✔
本地 449 个用例           436 绿 + 13 条互操作（无服务端时自报 Inconclusive）
```

压缩另外单独验过：走真实的 `zlib@openssh.com`，100 / 2000 / 20000 行
（最大 800 KB）**字节数一字不差**，开与不开压缩都对。

> 教训，按分量排：
> 1. **「测试全绿」和「能用」是两回事。** 差的那一段叫「对端不是你自己」。
> 2. **一个 bug 能同时骗过实现、测试桩和单元测试，只要它们出自同一个脑子。**
>    互操作测试不是「更多的测试」，它是**唯一一类能引入外部事实的测试**。
> 3. **它早该跑。** 用例两周前就写好了，缺的只是一个容器。
>    「等有环境再说」的代价，是在那之前对整个库的信心都是虚的。

### 11.2.14 X11 转发（2026-09-21）

〔范围〕**只做转发那一端，不做 X server**（§12「明确不做」里本来就是这么写的）。
服务端开回来的 `x11` 通道，我们接到本机已有的 X 显示上。

**479 个用例：464 绿 + 15 条互操作（有服务端时全跑，无服务端时自报 Inconclusive）。**

#### 安全核心：假 cookie

X11 没有客户端隔离 —— **连上同一个显示的任何客户端都能读别人的按键、
抓别人的窗口、往别人的窗口里塞事件**。所以把本机显示交给远端，
等于把本机所有图形会话的输入输出交给远端。

因此：

1. 默认**不请求**；要开必须显式调 `X11Forwarder.RequestAsync`。
2. 默认**非受信**模式（对应 `ssh -X`）。
3. **发给服务端的永远是一个随机的假 cookie。** 真 cookie 一步都不离开本机。
   远端 X 客户端拿假 cookie 连过来 → 我们核对 → 换成真 cookie → 才转给 X server。
   核对不过就拒绝，而且**连碰都不碰**本机 X server。

比较走 `CryptographicOperations.FixedTimeEquals`。逐字节短路比较会泄漏
「前几个字节对了几个」，而攻击者可以一条条开通道慢慢试。

#### 三个容易写错的地方

| 地方 | 写错的症状 |
| --- | --- |
| `x11-req` 的 cookie 字段是**十六进制文本**，不是原始字节 | 远端 `xauth` 存进去的和我们校验的对不上，而错误只会说「连接被拒绝」 |
| X11 建立报文的长度字段**两种字节序都要认**（首字节 `'B'`/`'l'` 说了算） | 「某些客户端能连，某些连不上」—— 看上去完全像随机故障 |
| 建立报文可能**分几次到达** | 小 MTU 或慢链路上随机失败 |

#### 一处与 OpenSSH 有意不同

OpenSSH 的 `ForwardX11Timeout` 只管非受信模式。**我们两种模式都管。**

理由：「有效期只在某一种模式下起作用」是一个会让人栽跟头的 API ——
而受信模式恰恰是危险得多的那个，却反而没有期限，说不通。
长会话要一直用就显式设 `Timeout = TimeSpan.Zero`。

这一条是写用例时发现的：`过期之后不再接受新的x11通道` 在受信模式下挂了，
一看才意识到「跟着 OpenSSH 抄」在这里抄出了一个反直觉的行为。

#### 验到哪一步了

| 层面 | 状态 |
| --- | --- |
| 纯函数（DISPLAY 解析 / `.Xauthority` / 建立报文） | ✅ 17 条 |
| 转发器（假 cookie、cookie 替换、拒绝路径、过期、并发上限） | ✅ 7 条，含**端到端**：起一个假 X server，验证落到它上面的是**真** cookie |
| 真实 OpenSSH 上的**拒绝路径** | ✅ `X11Forwarding no` 时明确拒绝，消息指得到 `sshd_config` |
| 真实 OpenSSH 上的**成功路径** | ✅ **OpenSSH 10.3 与 9.3 各 15/15**（见下） |

#### 跟真实 OpenSSH 对过了

写完的当天只验到「拒绝路径」，成功路径还没跟真实服务端对过 ——
而 §11.2.13 刚刚证明过那个差距有多要命。所以补上了。

`Start-TestServer.ps1 -X11` 在容器里装 `xauth`、打开 `X11Forwarding`，
两条互操作用例于是能跑：

| 用例 | 验的是 |
| --- | --- |
| `真实服务端接受x11_req并把假cookie存进xauth` | 真实 `xauth list` 里存的**正好是我们发的假 cookie** —— 这一条直接钉住了「cookie 字段是十六进制文本」那个坑 |
| `真实服务端开回的x11通道会被接受并换成真cookie` | 容器里的进程连上 sshd 配的 `DISPLAY`，sshd 开一条 `x11` 通道回来，我们核对假 cookie、换成真 cookie，落到本机一个假 X server 上 |

第二条里，X11 建立报文是**从 stdin 喂进去**的，不是拼进命令行 ——
报文里的假 cookie 只有在 `x11-req` 发完之后才知道，而命令行必须在那之前定下来。

##### 两个踩到的坑（都在脚本里，不在库里）

1. **sshd 的配置文件位置不能写死。**
   `linuxserver` 的 `latest` 镜像跑的是 `sshd -f /config/sshd/sshd_config`，
   而 `9.3` 那个用的是 `/etc/ssh/sshd_config`。改错文件的表现极具迷惑性：
   文件里明明写着 `X11Forwarding yes`，服务端却照样拒绝 —— 因为那份根本没被读。
   现在从 `/proc/<pid>/cmdline` 里把 `-f` 问出来。
2. **`apk` 要重试，而且失败时要把输出打出来。**
   容器刚起来那一两秒网络还没通。第一版把输出丢进 `/dev/null`，
   错误就只剩一句「装 xauth 失败」—— 指不到任何地方。

#### 为什么没有用 Dockerfile

`scripts/ssh/interop/Dockerfile` 写了，但这台机器上 **Docker Hub 拉不动**
（`auth.docker.io` 超时），而 `lscr.io` 是通的。所以退一步：在已经起来的容器里
装包改配置。CI 上 Docker Hub 通，那边可以改回构建镜像。

#### 独立性

实现思路来自本项目作者自己提给上游的
[tmds/Tmds.Ssh#514](https://github.com/tmds/Tmds.Ssh/pull/514)（作者 `joesdu`）。
从那里取的是**行为事实**：`x11-req` 的字段与时序、假 cookie 的替换流程、
两种字节序、`xauth generate` 的用法、以及失败该抛还是该降级。

代码按我们自己的结构重写（`IIncomingChannelHandler` + 既有的
`DuplexRelay` / `ChannelRelayEndpoint` 搬运层），
而且是先写规格（`spec/07` §7.5）再实现。

#### 补记（2026-09-24）：本机连接器

宿主内置了 X server（`VelaShell.XServer`，与本库同仓、互不引用）。本机显示就在同一个进程里时，再去连一个本机端口只是绕路。
`X11ForwardOptions.LocalConnector` 让调用方给一个连接器：每条 `x11` 通道调它一次拿一条双工流，建立报文里的 cookie 换成
`LocalCookie`（没给就是空的）后写进去（`spec/07` §7.5.9）。**假 cookie 的核对不变** —— 那一层防的是远端。
只支持受信模式：非受信模式要 `xauth` 连本机显示签受限 cookie，与连接器同时设时请求直接失败，不静默退回受信。
连接器那一端不可用时按「本机显示连不上」处理。用例 3 条（对搬与 cookie 替换、连接器不可用、与非受信同设）。

### 11.2.15 那个偶发挂死：通道登记得太晚（2026-09-22）

满跑二十轮红三轮，每次红的都不是同一条用例 ——
`深度不够时会自己长大`、`窗口长满之后数据依然一字节不差`、`带宽确实被限住了`、
再往前还有 `多条通道互不干扰`、`动态转发的目标由SOCKS握手给出`。
表现要么是 30 秒全局超时，要么是数据少了一半，要么是服务端报 AEAD 标签对不上。

最后查出来是**两件独立的事**，外加一条测试自己的毛病。

#### 一、`_channels` 登记在发包之后 —— 应答会被丢在地上

```csharp
// 老代码
lock (_stateLock) { …; _pendingOpens[localId] = completion; }

await SendAsync(CHANNEL_OPEN);          // ← 字节已经上线

lock (_stateLock) { _channels[localId] = channel; }   // ← 登记在这之后

return await completion.Task…;
```

收包循环是**另一个线程**。内存传输上服务端可以在那个 `await` 恢复之前
就把 `OPEN_CONFIRMATION` 送回来，于是：

```csharp
lock (_stateLock)
{
    _pendingOpens.Remove(recipient, out completion);   // 取走了
    _channels.TryGetValue(recipient, out channel);     // 还没有 → null
}
if (completion is null || channel is null) return;     // 应答被丢掉
```

`completion` 已经从表里摘走，却没人去完成它 —— `OpenChannelAsync`
就此等到天荒地老。挂哪条用例纯看那两个线程谁先跑，所以**每次都换一张脸**。

改法是把登记挪进创建通道的那把锁里，**在发包之前**：

```csharp
_pendingOpens[localId] = completion;
_channels[localId] = channel;          // ← 和 pending 同一把锁，同一时刻
```

顺带修掉一处因此暴露的老账：限额检查原来写
`_channels.Count + _pendingOpens.Count >= MaxChannels`。
正在打开的那条现在**两个表里都有**，加起来会把它数两遍 ——
等于把上限砍一半。改成只数 `_channels`。

回归用例 `发出通道打开请求之前通道就已经登记好` 不去赌时序，
而是拿一个包在客户端流外面的观察者钉住不变式：
**CHANNEL_OPEN 的字节落到流上的那一刻，通道必须已经在 `_channels` 里**。
把登记挪回 `await` 之后，它 126 毫秒就红，不是 30 秒超时。

#### 二、`DelayedStream` 收尾时 Dispose 了写锁

```csharp
await _inner.DisposeAsync();
_writeLock.Dispose();     // ← 正等在 WaitAsync 上的人，永远等不到了
```

`SemaphoreSlim.Dispose` **不唤醒等待者**，它们的 Task 再也不会完成。
而这条流正是「写要等时延」的地方，收尾时有人排在写锁后面是常态。
于是「关流」变成了「那个写入者挂死到超时」。
改法：不 Dispose —— 我们从不取 `AvailableWaitHandle`，
没有那个句柄时 SemaphoreSlim 不持有任何非托管资源。

#### 三、`带宽确实被限住了` 的断言压在边界上

限速器**先等额度、再记开销**，所以 N 次写只会等 N-1 次：
4 次 50 KB 在 1 MB/s 上的期望值正好是 150 ms，而断言写的是 `>= 150 ms`。
量到 149 ms 就红 —— 这不是运气差，是**算错了**。
改成按真实模型取 5 次（期望 200 ms），阈值留 25% 余量。

#### 怎么查出来的

前两轮猜测都错了，而且**错得很有说服力**：
先怀疑是 §11.2.8 那个水位 bug 没修干净（它的症状一模一样），
后来又怀疑 `SftpFileStream` 的同步包异步会把线程池饿死。
两条都查了，两条都不是。

真正管用的是三件事：

1. **把偶发变成可抓的现场。** 写一个看门狗：跑整套，超过 25 秒还没结束
   就 `dotnet-dump collect` 抓全量 dump，然后 `dumpasync` 看谁在等谁。
   一眼就看到 `OpenChannelAsync` 停在等 TCS，而两侧的收包循环都闲着。
2. **加诊断不能改变时序。** 第一版把每个报文号写进文件，
   跑四十轮一次都没复现 —— I/O 把竞态窗口盖住了。
   改成往一个 `byte[64]` 里记，零分配零 I/O，第十二轮就抓到了。
   dump 里读出客户端收到的报文号是 `20 31 21 6 51 52 91`，
   最后一个 `91` 正是 `OPEN_CONFIRMATION`：**应答收到了，人还在等**。
3. **别想当然地读 `Dictionary` 的字段。** `_count` 是**用过的槽位数**，
   `Remove` 不会减它，减的是 `_freeCount`。
   我盯着 `_count == 1` 推了半天「这项还在」，其实
   `_freeCount == 1` —— 它早就被取走了。
   这一个误读让我在错误的方向上多绕了好几圈。

> **偶发失败要么查到根因，要么就是还没查。** 它在测试里是随机红灯，
> 在真实链路上就是「偶尔开不了通道」和「偶尔传丢一半」。
> 三十轮全绿之前，不能说它修好了。

### 11.2.16 对着上游 #519 自查了一遍像素尺寸（2026-09-22）

[tmds/Tmds.Ssh#519](https://github.com/tmds/Tmds.Ssh/pull/519)（作者 `buchmiet`）
说的是：`pty-req` 与 `window-change` 各带四个 `uint32`（列、行、宽像素、高像素），
而上游只暴露了字符维度，像素那两个**恒为 0**。

不是我们的 PR，所以只看了它的问题描述，没看实现 —— 结论按 RFC 4254 §6.2 / §6.7
自查我们自己的代码得出。

**主问题不存在。** 像素尺寸在本库里从一开始就是一等公民（§5.3 的决策，
也正是当初决定自己实现的两个理由之一）：`TerminalSize` 四个字段贯通
`OpenShellAsync` → `pty-req` 与 `ResizeAsync` → `window-change`，
两条路径都有用例，其中 `伪终端能开起来` 还跑在真实 OpenSSH 上。

**但它顺带提到的第二件事，我们真的有，而且范围更大。**
那个 PR 会拒绝负的**像素**尺寸；我们四个字段全都是
`writer.WriteUInt32((uint)size.PixelWidth)` 这样的无检查强转，
一处校验都没有 —— 列数和行数同样漏。

`-1` 于是变成 `4294967295` 发出去。远端照单全收，然后按四十亿列排版：
**那不是「尺寸不对」，是乱码**，而且报错出现在远端程序里，指不回调用点。

改法是把校验放进类型本身，而不是放在两个序列化点：

- 四个属性都换成自己写的 `init` 访问器 —— 自动属性插不进校验，
  而写在 `init` 里 `with` 也绕不过去；
- 因此不能再用位置式记录（那样四个属性只能是自动属性），
  `Deconstruct` 手写补回来，用例钉住它还在；
- `0` 仍然合法。它的意思是「不知道」，拒掉它只会逼调用方瞎编一个数。

三条用例：构造时拒负数（四个字段各一次）、`with` 改成负数一样被拒、
`0` 合法且 `Deconstruct` 还在。

> 这一条的价值不在那几行校验，而在**「主问题没有」不等于「这条 PR 与我们无关」**。
> 顺着别人的 bug 报告往自己代码里看，比照着标题判断有没有中招靠谱得多。

### 11.2.17 CI 收敛到单平台，顺带发现仓库根本不是干净构建（2026-09-22）

#### CI：三平台矩阵砍成一个

这是一个类库，单 TFM（net11.0）—— 同一份源码在三个平台上编出来的 IL 一样，
「分平台构建」买不到任何东西，只是把 CI 时间乘以三。

矩阵原本想买的是**跑**测试的平台覆盖。而真正分平台的代码只有两处：
`SshAgentClient`（Windows 命名管道 vs `SSH_AUTH_SOCK`）与
`X11Display`（Windows 只有 TCP；类 Unix 多一个 `/tmp/.X11-unix/X{n}`，
Linux 再多一个抽象命名空间的）。

这两处的 Windows 分支**在日常开发里每次跑测试都会走到**（开发机是 Windows），
所以 CI 跑 Linux 恰好是开发环境的补集。macOS 与 Linux 的差别只有
「跳过抽象命名空间」那一个分支，由 `OperatingSystem.IsLinux()` 守着。

> 这个取舍的前提写进了 workflow 的注释：**它依赖「开发机是 Windows」**。
> 哪天主力环境变了，或者分平台的代码多起来，把矩阵加回去。

#### 顺带查出来的大事：`dotnet build` 一直在骗人

改完 CI 顺手验一下 `dotnet pack` 能不能过 —— 它**炸了**，报一堆 IDE 分析器错误。
而同一时刻 `dotnet build` 是绿的。

原因是**增量构建不会对没改过的文件重跑分析器**。
这个仓库从来没有在一次干净构建里绿过，只是没人跑过 `--no-incremental`：

```
110 error CA1707     (55 条，测试方法名里的下划线)
 10 error IDE0004    (多余的强制转换)
  6 error IDE0230    (该用 UTF-8 字符串文本)
  2 error IDE0062    (本地函数可以是 static)
```

**CI 第一次跑就会全红** —— 而 CI 从来没在真 runner 上跑过（§11.2.9 的表里就写着
「已写，未在真实 runner 上跑过」），所以这笔账一直挂着。

处理方式：

- `CA1707` 在测试项目里 `NoWarn` 掉，并写清理由：
  中文用例名里的下划线**不是命名风格，是协议标识符本身**
  （`原子覆盖式重命名走posix_rename`、`终端尺寸变化发出window_change`）。
  改成驼峰只会让用例名跟规范里的名字对不上，而用例名的唯一作用
  就是红了的时候一眼看出是哪条规矩破了。CA1707 保护的是公开 API，测试方法不是。
- 其余 9 条是真问题，逐条改掉（`private static readonly` 按 `.editorconfig`
  该是 PascalCase；局部 `const` 该是 camelCase；多余的转换删掉；
  `new byte[] { x }` 提成局部数组）。

> **别拿增量构建当门禁。** 绿了不代表干净 ——
> 它只代表「你改过的那几个文件是干净的」。
> 提交前至少跑一次 `--no-incremental`，那才是 CI 看到的东西。

#### 许可证改成 MIT

从 Apache-2.0 改为 MIT（136 个 `.cs` 的 SPDX 头、`LICENSE`、
`PackageLicenseExpression`、README / AGENTS / NOTICE / CI 里的说明）。

**放弃的是专利授权条款** —— Apache-2.0 §3 有，MIT 没有。
原先 README 把它当卖点写着「含明确的专利授权条款，商用无虞」，
这句话在 MIT 下是假的，已经改掉，并明确写出这一项差异。

`NOTICE.md` 保留：MIT 不要求它，但它记录的是第三方归属
（BouncyCastle 等），那些许可自己有要求。CI 里那条检查的措辞相应改了 ——
原先引的是「Apache-2.0 第 4(d) 条」。

### 11.2.18 补掉最后两个缺口：加密的 OpenSSH 私钥与证书认证（2026-09-22）

VelaShell 要切过来之前，先把它现在从 `Tmds.Ssh` 那里拿得到、而本库还没有的
两样东西补齐。这是**用户拍板的**：两条缺口各自都会被真实用户撞到，
而带着它们切换等于让人从「本来能用」退到「本来能用的现在不能用了」。

#### 一、加密的 OpenSSH 私钥 —— 给「不自己写密码学原语」开了一个有记录的例外

§11.2.12 里写着这条关不掉，理由是 `bcrypt_pbkdf` 要 Blowfish 的**密钥编排内部**，
BCL 没有、BouncyCastle 只给 `Init` + `ProcessBlock`。**那个判断到今天仍然成立**
（2.7.0 里 `BCrypt.DeriveRawKey` 依然是 `internal`，公开的 `Generate` 做的是
2^cost 轮的标准 bcrypt）。变的不是事实，是取舍：

> `ssh-keygen` 带口令时的默认产物就是这种格式。也就是说「绝大多数人手里的那把钥」
> 本库读不了。为了一条纪律的字面完整，让产品在最常见的路径上不可用，不划算。

所以 `src/VelaShell.Ssh/AGENTS.md` §3.3 的「**没有例外**」改成了「**一个例外，写明在此**」，
并把例外的范围钉死：

- **只有 `Keys/BcryptPbkdf.cs` 一个文件**，只做这一个 KDF，不导出任何分组加密能力 ——
  Blowfish 在那里面是纯内部细节，外面拿不到。
- **初始表不手抄。** P 数组与 4 个 S 盒共 **1042 个 32 位字**由 π 的十六进制位
  现算（Machin 公式，33344 位 + 128 位保护位）。抄错一个字，编译过、用例过
  （只要测试桩也用同一张错表），只有对着真实的私钥才露馅 ——
  而那时的症状是「口令明明对却解不开」。算出来的东西可以对着定义验，
  `BlowfishTableTests` 另外钉住首尾几个公认值（`P[0]=0x243F6A88`、
  `S3[255]=0x3AC372E6`）。
- **验证走真实产物。** 样本全部由真 `ssh-keygen`（OpenSSH 10.5p1）生成并提交在
  `tests/.../Keys/Fixtures/`，断言是「我们解出来的公钥与它写的 `.pub` 逐字节相同」。
  自己写一遍加密侧再自解自，两边会**一起错**，而那种错不报错。

支持的算法：`aes{128,192,256}-{ctr,cbc}`、`aes{128,256}-gcm@openssh.com`、
`chacha20-poly1305@openssh.com`。没有 `3des-cbc` —— 为了读一种没人用的格式
在安全库里带上 3DES 不划算，遇到时如实报错并给出 `ssh-keygen -p -Z aes256-ctr`。

**查这一条时踩的坑，值得单独记一句：**

> **AEAD 的认证标签在私钥区那个 `string` 的外面**，是容器末尾的裸字节，
> 不带长度前缀。一开始按「密文 ‖ 标签同在一个 string 里」写，症状极具迷惑性：
> 流密码解**前缀**照样是对的，于是两个 checkint 对得上、私钥看着解出来了，
> 只有标签永远验不过。ChaCha20-Poly1305 那一路因此卡了很久，
> 而 AES-GCM 那一路是当场失败 —— 同一个 bug，两种完全不同的表现。
>
> 定位它靠的不是继续推理，是**量**：把每个样本的 `section` 长度与它之后
> 还剩多少字节打出来，`section=176 / 之后还剩 16` 一行就说清了一切。

#### 二、OpenSSH 用户证书认证

证书认证**没有第三种认证方法** —— 它走的还是 `publickey`，只是请求里那个
「公钥 blob」字段装的是整张证书。真正的坑是一处不对称：

| 字段 | 内容 |
| --- | --- |
| 请求里的「公钥算法名」 | `ssh-ed25519-cert-v01@openssh.com`（**带**后缀） |
| 请求里的「公钥 blob」 | 整张证书 |
| 签名 blob 里的算法名 | `ssh-ed25519`（**不带**后缀） |

两处写反的症状都是一句 `Permission denied (publickey)`，
与「CA 不被信任」「主体不匹配」「证书过期」长得一模一样。

实现上把这处不对称**整个收进 `SshCertificateSigner`**，认证器一行没改 ——
它只管「拿 `Signer.PublicKey.Blob` 去出示、用挑出来的算法名去签」。
`SshPublicKey` 多了一个「换上证书身份」的构造：`KeyType` 与 `Blob` 是证书的，
验签仍由里面那把普通钥做，比对算法名前先去掉后缀。

`SshCertificateSigner.Create` **当场核对证书与私钥是不是一对**。
不核对的话配错了的表现同样是那句 `Permission denied`，
而这是一次本地字节比较，代价为零。

**本库不验证 CA 签名**，这是有意的：验证是服务端的事，客户端验了也不改变结果，
而客户端这边根本没有「哪些 CA 可信」那份名单。交出去的是
`SignatureKey` / `ValidBefore` / `CriticalOptions` 这些**事实**，
让使用者能显示、能在过期时说一句人话。

#### 三、`SshConnection.Disconnected`

VelaShell 的 `ISshClientWrapper.Disconnected` 是个 `CancellationToken`，
终端的读循环与自动重连都吊在它上面。原先本库只有 `IsAlive` 属性，
上层就只能轮询 —— 掉线到标签变色之间会有最多一个轮询周期的延迟。

它与内部的 `_lifetime` **分开**：`_lifetime` 用来把收发泵停下来，
而这一个是**对外**的信号，在 `Fault()` 时就放出去，那时收发泵可能还在收尾。
令牌在构造时取好并留住，不是每次从 CTS 上取 ——
那个属性在源被释放之后会抛，而「连接已经释放了」恰恰是最常去读它的时刻。

#### 验证

单元用例 517 条全绿（新增 22 条）。互操作对着真 OpenSSH 10.3 跑，
新增三条全过、无一跳过：加密私钥能登录、口令错时在本地就报出来、
证书认证能登录（服务端配了 `TrustedUserCAKeys`，
由 `scripts/ssh/interop/trust-ca.sh` 装进去）。

### 11.2.19 一次通读审查之后的修补（2026-09-22）

对整个库做了一遍通读审查，重点是「是不是真异步」「压缩与 X11 是否完整可靠」「代理设置是否生效」。
结论与修补一并记在这里。

#### 异步：没有 `Task.Run` 包同步的假异步，但有几处真问题

`Task.Run` 在库里只用来启动异步循环（它只把循环的同步开头挪到线程池，循环本身是真 `await` IO），
不是「把同步阻塞包成 Task」。真正的问题是这几处：

| 问题 | 修补 |
| --- | --- |
| 保活的每次探测**没有期限** —— 半开连接上第一次探测就永远等下去，判死逻辑形同虚设 | 每次探测以 `KeepAliveInterval` 为期限；超时的探测留在账本里，迟到的应答照样对齐 |
| 协议违规时先判死再发 `DISCONNECT`，而发送路径第一步就拒绝已故障的会话 —— `DISCONNECT` 从未上线 | 先发（最多等 2 秒）再判死；判死时真正停下收发泵、关掉所有通道 |
| 发送路径是「一把锁 + 每帧一次 Flush」，与 §5.4 描述不符；接收循环会被 TCP 背压挂住 | 落实 §5.4 的发送泵（见上）；接收循环只投递不等待 |
| 重协商期间暂存区满时，接收循环回一个 CLOSE 就会等闸门，而闸门要接收循环完成 KEX 才开 —— 自锁 | 背压挪到入队并区分发送方，接收循环与 KEX 不受背压 |
| `SftpFileStream` 的同步重载是 sync-over-async，违背原则 1 | 同步读写抛 `NotSupportedException`；同步 `Flush` 为空操作；同步 `Dispose` 不阻塞（`spec/06` §6.4） |
| 主机密钥裁决被算进连接超时（host 侧反馈） | 裁决期间连接计时器停表，持久化只认调用方令牌；跳板那一跳也让外层停表 |

另外几处会话层的正确性问题：接收缓冲在下一次读包时被复用，而「对端通道请求」事件与入站通道处理器
都拿着它 —— 改为复制；`OnData` 看状态与写管道之间的竞态会让一条通道的关闭打死整条会话 —— 放进同一把锁；
消费者提前 `Complete` 时剩下的字节从不回补窗口 —— 补上；认证后报文上限「说放宽但没放宽」—— 放宽并在开通道时校验；
`DISCONNECT` 的原因码与原话被丢掉 —— 交出去；`UNIMPLEMENTED` 的序号写死成 0 —— 改为被拒报文的序号。

#### 顺带：一个在 host 集成测试里才现形的挂死

host（VelaShell）换到本库之后，它的 Docker shell 集成测试有一半随机挂在 15 秒超时上：
exec 探针在同一条连接上开关一次之后，交互 shell 再也没有回显。诊断埋点（线程号 + 每个报文）
显示发出去的数据早已刷上线、服务端的回显也已经到了套接字 —— 是**接收循环自己停了 15 秒**：
它在处理探针通道的 `CHANNEL_CLOSE` 时调了 `CancellationTokenSource.Cancel()`，
而 `Cancel()` 在当前线程上同步执行回调，一串同步完成的续体（通道泵结束 → 探针的 DisposeAsync 恢复
→ RunAsync 返回 → 用例方法）就在接收循环的线程上跑了起来；用例接着 `Thread.Sleep` 轮询。

这个 bug 在本轮之前就在（同一个 `Cancel()`），只是 Tmds 那边没有同样的路径。
修法是一条规则：**接收循环与发送泵上绝不执行使用者的代码**（`spec/05` §8），
取消一律 `CancelAsync()`。回归用例 `调用方在await之后同步阻塞时接收循环不受影响` 修前 3/3 必挂，
修后 host 的集成套件 32/32、耗时与 Tmds 基线持平（19 秒 / 18 秒）。

#### 压缩

核心实现（逐方向持久流、sync flush、炸弹上限、重协商时重置、`zlib@openssh.com` 延迟启用）是对的。
一处语义漏洞：普通 `zlib` 也被推迟到认证之后才装，而它应当从首次 NEWKEYS 起生效 —— 已改为在 NEWKEYS 处当场装上。
`ssh_config` 的 `Compression yes` 现在经映射真正生效。

#### X11

安全核心（假 cookie、常数时间比较、十六进制 cookie 字段、两种字节序、分片、有效期）是对的。修补：

- 非受信模式的 `xauth generate` 没有 `-f`，会**覆盖使用者 `.Xauthority` 里的完全授权 cookie** —— 改为写临时文件；
- 交互 shell 根本开不了 X11（`ssh -X` 最常见的用法）—— `SshShellOptions.X11` / `AgentForwarding` / `BeforeStart`；
- 处理器是连接级单例，多个会话互相挤掉 —— 新增按 cookie 分派的 `X11ChannelRouter`；
- 本端拒绝通道时入站处理器的并发槽位泄漏（agent / 远程转发同样）—— `IIncomingChannelHandler.OnOpenAborted`；
  同类型多处理器（两个 `-R` 互相挤掉）—— `Add/RemoveIncomingChannelHandler`；
- `xauth` 超时不杀进程、显示名丢掉主机与套接字路径、单连接模式本端不强制 —— 都已补上。

#### 代理：之前**一个都没有**

`SshDialKind` 里的 `Socks5` / `HttpConnect` / `SshJump`、§5.1 与 §6.2 的示例、
`ProxyRefused` / `ProxyAuthRequired`、`SshConnectException.Hops` 全是空壳；
`ssh_config` 的 `ProxyJump` 只被解析、从未被消费，`ProxyCommand` 连解析都没有，
整个 `SshConfigFile` 没有到连接参数的映射。

按净室规程**先写规格**（`spec/09-dialing.md`，依据 RFC 1928 / 1929 / 9110 / 7617 / 4254 §7.2
与 `ssh_config(5)` 的行为描述），再照规格实现：`Socks5Dialer`、`HttpConnectDialer`、`SshJumpDialer`、
`ProxyCommandDialer`、`DialerChain`、通道即流 `SshChannelStream`（`SshChannel.AsStream()`）、
逐跳失败信息、`SshConfigFile.CreateConnectionOptionsAsync` 与 `SshHostConfig.ApplyToShell`。

〔如实说明〕`ProxyCommandDialer` 在 Windows 上走匿名管道，而匿名管道不支持重叠 IO ——
它的异步读写由运行时在线程池线程上阻塞完成。这是平台限制；需要纯异步链路时用另外三个拨号器。

#### 验证

单元用例 568 条（549 通过、19 条互操作在无 sshd 时跳过）；对着真实 OpenSSH 10.3 的互操作 19/19 全过（含新加的「经真实服务端做跳板」）。本轮新增的覆盖：
对端沉默时保活判死、迟到的保活应答不错位、违规时 DISCONNECT 真的上线、DISCONNECT 原因码与原话、
UNIMPLEMENTED 序号与不回声、载荷不被后续报文覆盖、提前收尾照样回补窗口、认证后收 128 KiB 报文、
同类型多处理器、`OnOpenAborted`、普通 zlib、多会话 X11 分派、单连接本端强制、shell 的 X11 时序、
xauth 一定带 `-f`、裁决期间停表、SFTP 同步 API、SOCKS5（含认证、拒绝、IDN）、HTTP CONNECT
（含 407、只放行 80/443 的提示、**应答与 SSH 标识串同一次写到达**）、代理嵌套的逐跳信息、跳板、
代理命令的 stderr、`ssh_config` 映射（跳板链、环检测、主机密钥策略、IdentityFile、会话项）。

### 11.2.20 并入宿主之后的一次全库审查与修补（2026-09-24）

按八个子系统分头通读（帧层与密码套件、KEX 与重协商、会话核心、通道、认证与私钥、SFTP、转发、拨号与配置），
审查过程同样守净室规程（依据只有 RFC、OpenSSH `PROTOCOL*` 与本仓库的规格）。先修直接伤到使用者的，
再修 DoS 上限与窗口预算；其余发现留作后续。每一条修补都有一条先在撤掉修复时确认失败、再确认通过的用例。

#### 直接伤到使用者的

| 问题 | 根因 | 修补 |
| --- | --- | --- |
| agent 里只要有一张证书、一把 FIDO 或 DSA 钥，agent 认证、agent 转发、自动加钥一起失败 | 列身份时接的是 `SshWireFormatException`，而 `SshPublicKey.Parse` 抛的是 `SshPublicKeyException` | 接对异常，不认识的身份跳过 |
| agent 拒签整条凭据链中断；断网反被记成「跳过」，下一条凭据读到上一条的应答 | 认证链按异常**类型**筛「凭据问题」 | 只接凭据回调与签名器自己抛的（`spec/04` §3.4） |
| 通道关闭时写 stdin 的调用方永远挂住 | stdin 泵从循环中间 `return`，跳过了完成 reader；完成 writer 放不出挂起的 `FlushAsync` | 泵的每条出口都完成 reader |
| 释放连接时正在开的通道永远挂住 | 只有 `Fault` 结算 `_pendingOpens`，释放不走 `Fault` | 释放时一并结算 |
| 取消开通道会在服务端泄漏通道（占 `MaxSessions`） | 取消后迟到的确认被丢在地上 | 已上线的开通道请求不摘账本，确认到了就关 |
| CLOSE 可能永远不发；通道号在对端 CLOSE 之前就回收；CLOSE 之后还会发请求 / EOF / 数据 | 先设「已发」后发送；本端收尾即还号；发送前不看状态 | 「已发」与入队同一刻在入队锁里设；号扣到双向 CLOSE；CLOSE 之后一律不发（`spec/05` §1 规则 2、6） |
| 通道流释放时丢掉尾部、不发 EOF；`FlushAsync` 空操作 | 直接关通道 | 先冲刷 stdin、发 EOF 再关；`FlushAsync` 等数据交给会话（`spec/09` §5.1） |
| SFTP 取消几次之后所有操作挂住 | 没发出去的请求留在账本里，在途额度永不归还；故障时排队者不被唤醒 | 没上线就当场摘账本还额度；字节整帧提交、取消只打断背压；排队者连上生命周期 |
| 严格 KEX 下重协商之后连接断开（chacha20 / HMAC 套件） | 严格 KEX 每次按对端 KEXINIT 重算，重协商时不再带标记就不归零序号；测试桩错在同一处 | 首次交换定下、整条连接沿用；规则 (a) 只管首次交换（`spec/03` §6） |
| SFTP 下载吞吐被钉死在「块大小 ÷ RTT」 | 顺序读一次只发一个 `READ` | 顺序读预读（`spec/06` §5.5，先写规格再实现） |

#### DoS 上限与窗口预算

| 问题 | 修补 |
| --- | --- |
| 会话窗口总预算形同虚设：被拒一次退两次、扩窗从不计、关闭按扩后的大小退 | 退款以实际计过的数为准；扩窗先申请、缩窗退还（`spec/05` §3.3） |
| 对端只发不收时，接收循环的应答无界排队 | `MaxQueuedReplyBytes`（默认 16 MiB）硬上限，超限判违规；应答计入背压 |
| 对端灌未知通道请求，事件流无界增长（每条最长 256 KiB） | 未读的未知请求最多留 64 条 |
| 拒绝开通道时原样回显 64 KiB 的类型名 | 描述截到 256 字符 |
| 重协商没有超时：对端不回 KEXINIT 或卡在半路时连接无声停住 | 默认 2 分钟，超时以 `Timeout / Rekeying` 断开（`spec/03` §8.2） |
| 交换进行中再发起重协商会发出第二个 KEXINIT（「报文数到阈值」用例满跑约 1/4 挂在这里，**修前就在**） | 「在谈」覆盖到开闸为止；关闸与 KEXINIT 在同一把锁里入队 |

顺带：`ReadAllBytesAsync` 的初始容量曾按服务端报的长度预分配（谎报 2 GiB 就先分配 2 GiB），封顶 1 MiB。

#### 验证

单元用例 626 条（606 通过、20 条互操作在无 sshd 时跳过），满跑连续 10 轮全绿；宿主 `VelaShell.Infrastructure.Tests` 536 通过。
本轮没有对真实 OpenSSH 跑互操作 —— 严格 KEX 那一条的真实表现依赖服务端在重协商 KEXINIT 里带不带标记，
值得在下次起靶机时补一条「开着 chacha20 跨过重协商」的互操作用例。

### 11.2.21 第二批：审查报告的其余安全项（2026-09-24）

| 问题 | 修补 |
| --- | --- |
| 重协商时把主机密钥策略整个再跑一遍（交互式策略在会话中途弹窗、接收循环停着等；宽松策略让换过的钥悄悄通过） | 钉住首次交换的主机密钥，不再问策略；`K_S` 变了就以 `HostKeyChanged / Rekeying` 断开；重协商只谈与它同类型的算法（`spec/03` §8.4 本来就这么写） |
| 释放 agent 转发器之后，已经打开的 agent 通道照样替远端签名 | 转发器有自己的生命周期，释放时连已打开的通道一起断（`spec/07` §7.2） |
| `DISPLAY=localhost:N` 先试 Linux 抽象套接字，同机别的用户抢先绑上就收到真 cookie | 「连哪里」与「用哪个 cookie」分开：`localhost:N` 只走 TCP，挑 cookie 仍按本机显示（`spec/07` §7.5.6 一并改正） |
| 私钥 KDF 参数不设上限（Argon2 4 GiB 内存、bcrypt 一百万轮）而且在验 MAC 之前就要算 | bcrypt 上限 4096 轮；Argon2 内存 256 MiB、256 遍、并行度 1–16 且限制两者乘积；顺带按 `Key-Derivation` 选对 Argon2 变体 |
| Windows 上连命名管道 agent 不看另一头是谁（别的用户抢注 `openssh-ssh-agent` 就收到签名请求与加钥时的明文私钥） | 校验管道属主：当前用户 / SYSTEM / Administrators；不降模拟级别 —— OpenSSH 的 agent 服务要以连进来的用户身份存钥（`spec/07` §7.2） |
| 对端文本原样进异常消息（终端转义注入） | `PeerText.Sanitize`：控制字符、`DEL`、C1、双向控制符换成 `?` 并截断；原话仍在 `PeerDescription` / `ServerMessage`（`spec/08` §一） |
| `KnownHostsPolicy` 换一种没记过的类型就判「没见过」；`!pattern` 被忽略；追加不补换行；`ProxyCommand` 可注入 | `OtherKeyTypesKnown` 按「变了」处理，`IHostKeyTypePreference` 让连接时优先已记下的类型（`spec/03` §5.4）；取反否决整行；追加前补换行；代入前检查字符集（`spec/09` §六） |

验证：单元用例 655 条（635 通过、20 条互操作跳过），满跑连续 25 轮全绿；新增 29 条中 19 条先撤掉修复确认失败。
本机验证不到的两处如实记下：X11 `localhost` 的差别只在 Linux/macOS 上测得出；命名管道属主检查只测了「同一用户的 agent 照常能连」与判定函数，
属主为 SYSTEM 的真实 OpenSSH agent 管道与「别的用户抢注」都没有实测。

### 11.2.22 第三批：正确性、性能与设计，外加主机证书（2026-09-25）

审查报告剩下的第三到第五节（正确性、性能、设计）在这一轮修完。同一天用户给了算法支持面的口径 ——
「兼容主要算法即可，太旧的、不安全的先不考虑」—— 据此补上了主机证书，并把传统加密 PEM 明确划到不支持的一边。
行为都已写进规格，下面各条括注的是对应章节。

#### 正确性：出错必须看得出是出错

| 问题 | 决定 |
| --- | --- |
| 转发的搬运循环把出错当成 EOF：一个方向读出错，照样给对面发 FIN / `CHANNEL_EOF`，截断的数据被当成完整的收下；另一个方向一直挂着 | 正常读完只半关闭那一个方向；任何一个方向出错（含取消），两端一起**中止** —— TCP 是 linger 0 的 RST，通道是不先发 EOF 的 `CLOSE`；上报最先出错的那一侧，不报被牵连的那个取消（`spec/07` §2.2、§6） |
| 连接中途断了，通道的读者读到的是一个像 EOF 的「读完」 | 读抛出连接的故障；本端释放连接时抛 `ObjectDisposedException`；只有对端的 EOF / `CLOSE` 才算读完（`spec/05` §4.4） |
| 通道整个关了（`CLOSE`），搬运循环往它写的那个方向还卡在本机 socket 的读上，socket 与转发名额一直占着 | `SshChannel.Closed` 令牌；搬运循环据此停下那个方向（`spec/05` §4.4） |
| 连接的故障原样漏给调用方：内部的解析异常类型、裸的 `IOException` / `SocketException`，会话期间与建连期间都有；密钥交换期间报文中途断开报成 `ProtocolError`；`SshPublicKeyException` 与 `SshKeyExchangeException` 不在 `SshException` 之下 | 交出之前归成公开类型，建连期间（拨通之后到认证结束）用同一套口径，`Phase` 记失败的那一步（`spec/08` §2.1）；两个异常改派生自 `SshException`（`spec/08` §2）。**已知局限，没改**：会话期间归一出来的故障 `Phase` 一律是 `Open`，即使发生在重协商期间 |

这一组共用一个判断：**「对端说完了」与「链路断了」在读的一方眼里必须不同。**当成一回事，
传到一半的文件、跑到一半的命令输出会被当成完整结果交出去，终端也分不清用户敲了 `exit` 还是链路断了（后者才该自动重连）。

#### 性能：控制报文不能排在数据后面

| 问题 | 决定 |
| --- | --- |
| 大量上传时，另一条通道的 `WINDOW_ADJUST` 排在出站积压后面（还要先在背压上等空位）：对端一直等窗口，下载被上传拖到几乎停住 | 本端的窗口回补走插队队列，发送泵每取一项先看它，不受背压；插队只让它提前，不越过 `CLOSE` 与重协商闸门（`spec/05` §3.2） |
| 保活探测在背压上等，或者上线之后才开始计时：死链的典型样子正是发送泵卡在一次写上，探测跟着卡住，恰恰在最需要判死的时候判不了 | 探测不受背压，期限从入队那一刻算起；每个周期至多一帧，不会无界（`spec/05` §6.3） |
| 自适应窗口只看「见底」：消费者一慢，窗口就一路翻到上限，几十 MiB 没读的数据堆在本端 | 见底**并且**读的一方最近读空过才扩；连续 3 轮没见底就缩（`spec/05` §3.3，本文 §5.5） |
| 对端开通道时，接收循环就地等使用者的处理器（它可能弹窗、查配置）：处理器一慢，整条连接上所有通道的收包都停住 | 解析与查处理器在接收循环上做完，「问处理器、建通道、回确认」交给后台；同时在决定的最多 64 条，超出的以 `RESOURCE_SHORTAGE` 拒绝（`spec/05` §8.1） |

#### 性能：热路径上的分配

- **通道数据的缓冲从池里租**：上传路径上每一块曾经新分配一个数组。发送方的 `await` 返回时发送泵已经不再引用它，可以立刻还回池里；
  **只有在重协商期间被闸门暂存时才复制一份** —— 暂存要引用到开闸，而那时发送方早已把缓冲还掉了。直接发走的那一条不必复制：加密时已经拷走了。
- **SFTP 请求帧直接写进通道的管道**，不经中转缓冲。曾经先拼进一个从 256 字节起翻倍长的缓冲，一个 256 KiB 的 `WRITE` 要重新分配十来次、把数据多搬一遍。
- **密码套件的对象跟着套件走，不按报文新建**：HMAC 上下文复用；ChaCha20-Poly1305 的两个引擎与 Poly1305 常驻，每个报文只换 nonce；
  CTR 的异或按向量宽度一批一批做，计数器一次加上块数而不是逐块加一。一个套件只服务一个方向、只在一个线程上用，复用是安全的。
  代价是一处新风险：换钥之后新的一组必须从新钥重新起步，残留一点旧状态第一个新钥报文就解不开 ——
  而自己加密自己解密时两边错得一模一样，只有对着另一份实现才看得出来。所以互操作用例让重协商落在大块输出的**中途**（见下面的验证）。

实测（Release，32 KiB 载荷）：

| 套件 | 吞吐 | 每报文分配 |
| --- | --- | --- |
| AES-CTR + HMAC | 约 800 → 约 1100–1200 MB/s | 288 B → 0 |
| ChaCha20-Poly1305 | 约 430–480 MB/s（上限在 BouncyCastle） | 约 1.4 KB → 272 B |
| AES-GCM | 未变：封装约 6.2–6.9 GB/s，解封约 5 GB/s | 0（本来就是 0） |

#### 链路模拟器自己成了瓶颈

`DelayedStream`（`Transport/LinkCharacteristics.cs`）曾经每一次写都在写锁里整段睡掉单向时延、再交给下层：
写与写被时延串了起来，链路上任一时刻最多一次写在飞，吞吐被压在「单次写的大小 ÷ 单向时延」——
**窗口与管线深度的用例量到的，有一部分是模拟器自己**。

现在它是流水线式的，与真实链路一致：写入按带宽计时之后立刻返回，数据在单向时延之后才对另一端可读；
在途的数据放在一个有上限的队列里，由后台任务按顺序、到点交给下层 —— 下层写不动时队列会满，写入方照样会等，背压不因为流水线而消失。
`DisposeAsync` 先让在途数据送达再关下层（FIN 排在已发出的字节后面；不这样，最后那条 `DISCONNECT` 另一端就收不到），
下层卡住时等单向时延再加 1 秒就当链路被切断；同步 `Dispose` 不等，相当于直接切断。

> 测试设施也是被测系统的一部分：它的模型错了，量出来的就是它自己。

#### 拨号与 `ssh_config`

| 问题 | 决定 |
| --- | --- |
| 多个地址逐个顺序试：通告了 IPv6 却不通的网络上，第一个地址要等约 21 秒的 SYN 超时才轮到 IPv4 | Happy Eyeballs（RFC 8305）：按地址族交替、每 250 ms 错开发起、先连上的胜出、其余关掉（`spec/09` §2.5） |
| TCP 这一步固定 30 秒，连接上配置的更长超时对它无效 | 拨号器默认不限时，由连接超时统管（`spec/09` §2.4） |
| 跳板上的认证计入外层连接超时；外层到点时报成「建立 TCP 连接超时」；分不清调用方取消与到点 | 跳板认证期间外层停表；到点报 `Timeout / Dialing` 并标出哪一跳；取消原样往外传（`spec/09` §2.4） |
| `Include` 的内容接在整个文件后面：主文件里更靠后的 `Host *` 压过被包含文件里为具体主机写的设置；同一个文件被两个块各包含一次，第二次被当成环跳过。改成就地展开之后，被包含文件一开新块外层条件就丢了，`Include` 之后的设置落进被包含文件的最后一个块 | 就地展开、带条件的包含：被包含文件自己的块套在外层条件之下（`SshConfigBlock.Enclosing`，每一层都满足才生效），读完回到 `Include` 所在的块；环检测只看当前这条包含链（`spec/09` §7.1） |
| 判不了的 `Match` 条件算成「不满足」，前面加个 `!` 就成了满足；`Match host` 拿输入的别名去比 | 三态：判不了就整块不生效，取反也一样；`Match host` 比 `HostName` 改写之后的名字（`spec/09` §7.1） |
| 一个 `IdentityFile` 读不出来，整个解析失败；跳板与目标各读一遍同一把钥（KDF 跑两遍、口令问两遍） | 只跳过那一把并通知调用方；一次解析内按路径共用，不跨调用缓存（`spec/09` §7） |
| 调用方为目标准备的口令也交给了跳板 | 口令与键盘交互只给最终目标，跳板只拿公钥凭据（`spec/09` §7） |
| `ask` / 缺省时只要配置写了 `UserKnownHostsFile`，调用方给的策略就被丢掉；`none` / `/dev/null` 被当成文件路径 | 调用方策略优先；`none` / `/dev/null` 即不用 `known_hosts`（`spec/09` §7） |

#### 主机证书，与不读的私钥格式

按「主要算法、不要太旧」的口径：

- **主机证书做了**（`spec/03` §5.5）。证书算法排在普通算法**之后**：没为这台主机配 CA 时，谈成证书得不到任何额外保证，排在后面就保证行为与以前完全一样。
  `known_hosts` 里有对上这台主机的 `@cert-authority` 时才验证证书；记下的是证书里那把**普通钥**（证书每次重签 blob 都会变，记证书等于下次必报「变了」）；
  由 CA 管的主机出示一把没有担保的钥时拒绝，**不退回 TOFU** —— 给主机配 CA，要的正是不再靠第一次盲信；CA 的 SHA-1 `ssh-rsa` 签名不认。
  CA 担保而证书不合格、或者协商出证书算法而 `K_S` 不是证书，都报 `HostKeyRejected`（`spec/08` §3）。
- **重协商只谈与钉住的钥同类型、且同为证书（或同为普通钥）的主机密钥算法**（`SshConnection.RestrictToPinnedHostKey`）。
  曾经判定时去掉了证书后缀：钉住的是证书时普通算法也留着，一旦谈成普通算法，服务端出示的是那把普通钥，与钉住的证书对不上，
  连接被当成「换了主机密钥」断开（`spec/03` §5.5）。
- **传统加密 PEM（`Proc-Type: 4,ENCRYPTED`）不读**，只支持现代格式（OpenSSH、PuTTY `.ppk`、PKCS#8，以及不加密的 PKCS#1 / SEC1）。
  口令只经一次 MD5 就成了密钥、常配 3DES，为它在安全库里带一套解密不划算，而转换只要一条命令。
  在要口令**之前**就拒绝并给出转换办法（`ssh-keygen -p` 改一次口令即转成 OpenSSH 格式）；曾经交给 BCL，BCL 不认，报出来的却是「口令多半不对」（`spec/04` §4.6）。

#### 验证

全套 761 条用例，其中 22 条是对真实 OpenSSH（`linuxserver/openssh-server`）的互操作用例，这一轮在起着靶机的情况下跑，没有跳过。
新增的两条互操作补上了此前只能在内存桩上验的东西：默认清单里的每一种加密算法（chacha20-poly1305、AES-GCM、AES-CTR）
在大块输出**中途**跨过一次重协商照常收发 —— §11.2.20 留下的那条待办，也是上面「套件对象复用」的真正验收；
以及按 `known_hosts` 的 `@cert-authority` 验过一张真实 `ssh-keygen` 签发的主机证书。

### 11.2.23 X11 转发的几处加固：一次净室对照（2026-09-25）

起因：宿主作者给 Tmds.Ssh 贡献的 X11 转发，被那边的维护者改过之后合入了。想看看那些改动对本库有没有启发，
同时不破坏 §2 的第 1、2 条（规范优先、两阶段隔离）。做法：

1. 由一个**不带本库任何上下文**的独立分析会话去读对方的改动，只交回行为层面的自然语言描述 ——
   零代码、零对方标识符，只许用协议层面的名字（SSH 报文名、RFC 章节、X11 协议术语）；
2. 本库这边只拿那份描述，对照 RFC 4254 §6.3、X11 核心协议、X.Org 的 SECURITY 扩展规格与 `xauth(1)`，
   先改 `spec/07`、`spec/09`，再照规格实现。对方的源码没有进入实现会话。

对照下来，大部分行为本库已经有，或者更稳妥：一条连接上多个转发各持假 cookie；尽力而为时只吞 X11 本身的失败、连接断了照常抛；
严格模式报具体原因；`localhost:N` 按 TCP 显示原样交给 `xauth`（改写成 `unix:N` 在只有 TCP 的显示上会连不上）。做了四处：

| 问题 | 决定 |
| --- | --- |
| 建立报文的两段授权字段各允许 64 KiB：对端只发一个声称很长的头，我们就在核对 cookie 之前一直等到 30 秒的时限 | 各不超过 256 字节，读到 12 字节的头就判（`spec/07` §7.5.5） |
| `xauth` 的 `timeout` 与我们的有效期相等：两边各自计时，临界时刻我们刚接下的通道会被 X server 拒；有效期为 0 时却给了 X server 20 分钟 | 有效期再加 60 秒；为 0 时传 0 —— SECURITY 扩展里 0 就是永不过期（`spec/07` §7.5.7） |
| 给 `xauth` 建临时目录失败时原样抛 `IOException`：尽力而为的会话整个起不来 | 报成 `SshForwardException`，算「X11 没开成」；连接断了与取消照常抛（`spec/07` §7.5.8） |
| `ssh_config` 的 `ForwardX11Timeout` 不认 | 按 ssh_config 的时间格式解析（`1h30m`、纯数字为秒），`0` 为整条连接期间有效，写不对就用默认（`spec/09` §7） |

保留的有意差异：有效期对受信模式同样生效（OpenSSH 的 `ForwardX11Timeout` 只管非受信）—— 受信模式危险得多，它反而没有期限说不通。

### 11.3 与 VelaShell 的切换策略

1. VelaShell 的 `ISshClientWrapper` / `ISftpClientWrapper` / `IShellStreamWrapper`
   **已经是库中立的抽象** —— 这是当初从 SSH.NET 迁过来时留下的最有价值的资产。
   切换的爆炸半径就是 `Infrastructure/Ssh/` 一个目录。
2. 先让 `VelaShell.SshClientWrapper` 与 `TmdsSshClientWrapper` **并存**，用一个 feature flag 切。
   两套实现跑同一批测试，逐项对齐。
3. 对齐后删 Tmds 实现与 §9 那张表里的补丁，flag 一并撤掉。

---

## 十二 风险与「不做的事」

### 风险

| 风险 | 缓解 |
| --- | --- |
| **互操作长尾** —— SSH 的坑全在老设备上 | §10.3 矩阵从 M1 起就跑，不留到最后；老设备算法优先级单独一组 |
| **密码学实现出错** | 原则 6：只装配不造原语（ChaCha20 / Poly1305 也走 BouncyCastle）；唯一自写的是 `bcrypt_pbkdf`（§11.2.18），只做那一个 KDF，只认真 `ssh-keygen` 的产物做端到端比对；1.0 前外部审计 |
| **工期超预期** | 里程碑按能力切，每个 M 都是可用状态；M6 之前 VelaShell 继续用 Tmds，随时可以停 |
| **人手** —— 24 周的活 | 认清楚再开工。**中途放弃的代价是一个半成品的 SSH 栈**，那比现在的 1,200 行补丁糟得多 |
| 「还是像」 | §2.2 的净室规程从 M0 第一次提交就执行 —— 不是最后来补，那时候已经晚了 |

### 明确不做

- **SSH 服务端**（测试桩除外）。方向完全不同，不在范围里。
- **SSH1**。已死，且不安全。
- **自己实现 AES / SHA / RSA / ECDH**。BCL 有，且有硬件加速。
- **同步 API**。原则 1。
- **X11 转发的另一端**（X server）不在本库。转发通道本身**已经做了**（§11.2.14）；X server 是宿主仓库里另一个库 `VelaShell.XServer`，两者经 `X11ForwardOptions.LocalConnector` 对接（§11.2.14 补记）。
- **Mosh**。另一套协议栈，与 VelaShell 现有决策一致。

---

## 十三 待拍板的决策

| # | 决策 | 建议 |
| :-: | --- | --- |
| 1 | **做还是不做**（24 周） | **已定**：先做 M0（规格 + 骨架），M0 出口再决定是否续推 M1–M7 |
| 2 | 库名与命名空间 | **已定**：仓库 `velashell-ssh`，包与命名空间 `VelaShell.Ssh`。M0 结束前改名仍为零成本 |
| 3 | 许可证 | **已定**：MIT 单许可开源（2026-09-22 由 Apache-2.0 改定）。MIT **不含**专利授权条款 —— 这是相对 Apache-2.0 放弃的那一项；VelaShell 照常闭源使用 |
| 4 | 兼容层做不做 | 做，只做 VelaShell 用到的那一面，第一天就标 `[Obsolete]`（§6.3） |
| 5 | TFM 与语言版本 | **已定**：`net11.0` 单目标 + `LangVersion=preview`。放弃 net10 的潜在用户面，换来零 polyfill 与最新 BCL |
| 6 | 要不要开源 | **已定**：开源 |
| 7 | M0 的规格写到多细 | 建议写到「每条报文的字段表 + 时序图」这一级。**§2.2 第 2 条的隔离靠它成立** —— 规格不细，实现阶段就会忍不住回去翻源码 |

---

*§13 的 1 / 2 / 3 / 5 / 6 已于 2026-09-21 拍板（见表内标注）；4 与 7 在 M0 出口前定。*
