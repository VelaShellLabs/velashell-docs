# 00 · 行为规格总则

> **这套文档是 VelaShell.Ssh 实现的唯一依据。**
>
> 写实现的时候，桌面上应当只有两样东西：本目录下的规格，以及它所引用的 RFC。
> **不要打开任何其它 SSH 实现的源码** —— 这是 [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) §2 纪律 2，
> 也是本项目「可证明的独立实现」这个立身之本的全部内容。
>
> 规格里写不清楚的地方，回去查 RFC 并**把规格补上**，而不是去别处找现成答案。

---

## 一 这套文档怎么读

| 文件 | 内容 | 对应实现层 |
| --- | --- | --- |
| `00-overview.md` | 本文：术语、记法、通用约定 | — |
| [`01-transport-framing.md`](01-transport-framing.md) | 二进制报文协议、分帧、填充、加密形状 | L2 帧层 / L3 密码套件 |
| [`02-version-exchange.md`](02-version-exchange.md) | 版本标识串交换 | L4 状态机 |
| [`03-key-exchange.md`](03-key-exchange.md) | 算法协商、密钥交换、交换哈希、密钥派生、重协商、严格 KEX | L3 / L4 |
| [`04-authentication.md`](04-authentication.md) | 服务请求、各认证方法、部分成功、扩展协商 | L7 认证 |
| [`05-connection.md`](05-connection.md) | 通道、流控窗口、通道请求、exec / shell / pty | L5 通道 |
| [`06-sftp.md`](06-sftp.md) | SFTP v3 wire 格式与扩展 | L7 SFTP |
| [`07-forwarding.md`](07-forwarding.md) | 本地 / 远程 / 动态转发、agent 转发 | L8 转发 |
| [`08-failures.md`](08-failures.md) | 失败分类学、可观测性契约 | L9 |

## 二 规格的三种语句

沿用 RFC 2119 / RFC 8174 的关键词，中文写作时一律用下列固定译法：

| 英文 | 本文写法 | 含义 |
| --- | --- | --- |
| MUST / REQUIRED / SHALL | **必须** | 不这样做就是协议违规 |
| MUST NOT / SHALL NOT | **禁止** | 同上 |
| SHOULD / RECOMMENDED | **应当** | 有充分理由才可偏离，偏离要写明理由 |
| SHOULD NOT | **不应** | 同上 |
| MAY / OPTIONAL | **可以** | 实现自由 |

另有两类本项目自己的标注：

- **〔决策〕** —— RFC 留白、由我们定的行为。**每一条都要给出理由**，
  因为将来的人会问「为什么是这样」，而答案不该只存在于某个人的记忆里。
- **〔互操作〕** —— 某些服务端的实际行为与 RFC 不符，我们必须迁就。
  必须写明**是哪个服务端、什么版本、怎么验证的**。没有具体对象的「据说有些服务器」不算。

## 三 数据类型（RFC 4251 §5）

SSH 的 wire 格式只有七种类型。全部**大端序**。

| 类型 | 编码 | 备注 |
| --- | --- | --- |
| `byte` | 1 字节 | |
| `boolean` | 1 字节 | 0 为假，**非 0 均为真**（发送时必须发 1） |
| `uint32` | 4 字节，大端 | |
| `uint64` | 8 字节，大端 | |
| `string` | `uint32` 长度 + 该长度的原始字节 | **是字节串，不是文本**；可含 `\0`；长度不含自身的 4 字节 |
| `mpint` | 按 `string` 编码的二进制补码大整数 | 见下 |
| `name-list` | 按 `string` 编码的、逗号分隔的 US-ASCII 名字 | 见下 |

### 3.1 `mpint` 的两个坑

1. **正数最高位为 1 时，必须在前面补一个 `0x00`。** 否则按二进制补码读会读成负数。
2. **零编码为长度 0 的 string**（即四个字节 `00 00 00 00`），不是一个 `0x00` 字节。

实现时这两条都要有单测钉住 —— 它们是 SSH 实现里最经典的两个 off-by-one。

### 3.2 `name-list` 的约束

- 每个名字：非空、只含可打印 US-ASCII、**不含逗号**。
- 名字区分大小写。
- 空列表编码为长度 0 的 string。
- 含 `@` 的名字是厂商扩展（`name@domain`），`domain` 是定义者拥有的域名。
  **不含 `@` 的名字必须是 IANA 注册过的** —— 我们不自造无域名后缀的算法名。

### 3.3 `string` 的文本化

协议层面 `string` 是字节串。需要当文本用时（用户名、错误消息、横幅），
按 RFC 4251 §6 的建议**按 UTF-8 解码**。

〔决策〕**解码失败时不抛异常，改用替换字符。**
理由：解码失败的字段通常是横幅、错误消息、文件名这类展示性内容；
因为一个乱码字节把整条连接打掉，对用户是纯粹的损失。
但**用户名与算法名例外** —— 它们参与协议判定与签名输入，解码失败必须视为协议错误。

## 四 本文的报文表怎么写

每个报文用统一的三段式描述：

**① 字段表** —— 按 wire 顺序，逐字段给出类型与含义：

| # | 类型 | 字段 | 说明 |
| :-: | --- | --- | --- |
| 1 | `byte` | `SSH_MSG_XXX` | 消息编号 |
| 2 | `uint32` | `recipient channel` | 对端分配的通道号 |

**② 时序** —— 用 mermaid 序列图画清楚谁先发、谁必须应答、哪些可以并发。

**③ 边界与错误** —— 字段的取值范围、上限、非法值怎么处理。
**这一段是规格里最容易被略过、也最容易在实现里出事的一段**，不许留空。

## 五 通用的安全下限

下面几条在所有报文处理中一律适用，各文件不再重复：

1. **所有长度字段必须先校验再使用。** `string` 的长度必须落在
   「剩余可读字节数」之内，且不超过该字段在本规格中规定的上限。
   规格没给上限的字段，实现必须自己定一个并写进规格 ——
   **不存在「这个字段不会太大」这种理由**，报文来自不可信的对端。
2. **禁止按长度字段预分配内存。** 先确认数据确实可读，再分配。
3. **解析失败一律断开连接**，不做「跳过这个字段继续读」的容错。
   SSH 的报文是定长拼接的，一处错位之后的所有内容都不可信。
4. **比较密钥、签名、MAC 必须用定时安全比较**（`CryptographicOperations.FixedTimeEquals`）。
5. **禁止把密钥材料写进日志**，包括 DEBUG 级别。
   报文旁路（`IPacketTap`）默认只给元信息，载荷要显式开启且文档中写明风险。

## 六 我们支持的算法（总表）

各算法的细节在 [`03-key-exchange.md`](03-key-exchange.md)，这里只给全景与取舍。

### 6.1 密钥交换

| 算法 | 依据 | 默认启用 | 备注 |
| --- | --- | :-: | --- |
| `mlkem768x25519-sha256` | RFC 10042（原 draft-kampanakis-curdle-ssh-pq-ke） | ✅ 最高优先 | 后量子混合；BCL 有 ML-KEM |
| `mlkem768nistp256-sha256` | RFC 10042 | ✅ | 后量子混合（ML-KEM-768 + P-256），面向 FIPS；也是 `FipsApprovedOnly` 的第一位（§6.6、03 §3.7） |
| `mlkem1024nistp384-sha384` | RFC 10042 | ✅ | 后量子混合（ML-KEM-1024 + P-384），面向 FIPS；也在 `FipsApprovedOnly` 里 |
| `sntrup761x25519-sha512` | OpenSSH `PROTOCOL` | ✅ | 后量子混合；OpenSSH 8.5+ 默认 |
| `sntrup761x25519-sha512@openssh.com` | 同上 | ✅ | 同一算法的旧名，兼容 OpenSSH < 9.9 |
| `curve25519-sha256` | RFC 8731 | ✅ | |
| `curve25519-sha256@libssh.org` | 同上 | ✅ | 同一算法的旧名 |
| `ecdh-sha2-nistp256/384/521` | RFC 5656 | ✅ | |
| `diffie-hellman-group14-sha256` | RFC 8268 | ✅ | |
| `diffie-hellman-group16-sha512` | RFC 8268 | ✅ | |
| `diffie-hellman-group-exchange-sha256` | RFC 4419 | ✅ | 〔互操作〕老设备、加固过的服务端常只给这个。排在椭圆曲线之后、DH 标准群之前；群由服务端现给，要查（03 §3.5） |
| `diffie-hellman-group14-sha1` | RFC 4253 | ❌ 默认关 | 〔互操作〕Cisco IOS / 老 VRP 只有它。**必须用户显式开启**（§6.6） |
| `ext-info-c` / `kex-strict-c-v00@openssh.com` | RFC 8308 / OpenSSH | ✅ | 不是真算法，是**指示符**，见 03 |

默认清单的顺序是：`mlkem768x25519-sha256`、`mlkem768nistp256-sha256`、`mlkem1024nistp384-sha384`、
`sntrup761x25519-sha512`、`sntrup761x25519-sha512@openssh.com`、`curve25519-sha256`、`curve25519-sha256@libssh.org`、
`ecdh-sha2-nistp256/384/521`、`diffie-hellman-group-exchange-sha256`、`diffie-hellman-group16-sha512`、`diffie-hellman-group14-sha256`。

〔决策〕**两种 NIST 曲线的 ML-KEM 混合进默认清单**，位置这样定：

- **排在 curve25519 与 ECDH 之前**（§7 第 1 条：后量子混合 > 椭圆曲线）。开了 FIPS 策略的服务端不给 X25519 与 sntrup761，
  清单里没有这两种的话只能退到不带后量子的 `ecdh-sha2-nistp256`，丢掉对「先截获、以后再解」的防护。
- **排在 `mlkem768x25519-sha256` 之后**：同为 ML-KEM，X25519 那一半更容易写成没有侧信道的、也更快（RFC 10042 §5），
  OpenSSH 与 RHEL 的默认都把它放第一。三种都给的服务端（RHEL 10.2 的 DEFAULT 策略）谈成的仍是它，与以前一样。
- **排在 sntrup761 之前**：ML-KEM 是 FIPS 203 标准化的 KEM、RFC 10042 注册的方法；sntrup761 留着只为兼容 OpenSSH 8.5–9.8（03 §3.6），
  而那些版本没有 nistp 混合，对它们结果不变。
- **768 / P-256 在 1024 / P-384 之前**：报文小（`C_INIT` 1249 对 1665 字节）、算得快；与 RHEL 10.2 的 DEFAULT 与 FIPS 策略的顺序一致；
  上游 OpenSSH 10.6 只有前一种。要 1024 / P-384 优先的（例如照 CNSA 2.0），自己组清单。
- 上游 OpenSSH 默认不开 `mlkem768nistp256-sha256`，这不影响我们放进客户端清单：服务端不给，协商就跳过它，代价只是 KEXINIT 里多两个名字。

### 6.2 主机密钥

| 算法 | 依据 | 默认 |
| --- | --- | :-: |
| `ssh-ed25519` | RFC 8709 | ✅ |
| `ecdsa-sha2-nistp256/384/521` | RFC 5656 | ✅ |
| `rsa-sha2-512` / `rsa-sha2-256` | RFC 8332 | ✅ |
| 上面三行的 `-cert-v01@openssh.com` 变体（主机证书） | OpenSSH `PROTOCOL.certkeys` | ✅ 排在全部普通算法之后 |
| `ssh-rsa`（SHA-1 签名） | RFC 4253 | ❌ 默认关（§6.6） |
| `ssh-rsa-cert-v01@openssh.com`（SHA-1 签名的证书） | OpenSSH `PROTOCOL.certkeys` | ❌ 不在默认清单，§6.6 的开关也不加它 |
| `ssh-dss` | RFC 4253 | ❌ **不实现**。1024 位定长，已不可接受 |

默认清单的顺序是：`ssh-ed25519`、`ecdsa-sha2-nistp256/384/521`、`rsa-sha2-512`、`rsa-sha2-256`，
然后是同样顺序的六个证书变体。证书变体为什么排在后面、什么时候会被提到前面，见 03 §5.5。

### 6.3 加密

| 算法 | 形状 | 默认 |
| --- | --- | :-: |
| `chacha20-poly1305@openssh.com` | AEAD，长度字段**单独加密** | ✅ 最高优先 |
| `aes256-gcm@openssh.com` / `aes128-gcm@openssh.com` | AEAD，长度字段为明文 AAD | ✅ |
| `aes256-ctr` / `aes192-ctr` / `aes128-ctr` | 流式 + 独立 MAC | ✅ |
| `aes256-cbc` / `aes128-cbc` | 块式 + 独立 MAC | ❌ **不实现**。§6.6 的开关也不含它 |
| `3des-cbc` / `arcfour*` | — | ❌ **不实现** |

〔决策〕**CBC 不实现，老算法开关也不把它报给对端。**
没实现的名字一旦出现在我们的 KEXINIT 里，只会在一台只剩 CBC 的设备上被「谈成」，
然后在派生密钥时才失败 —— 报出来的是一句「尚未实现」，比「没有共同的加密算法」难懂得多，
而且只在连某一台设备时出现。不报它，这类设备在协商当场失败，异常里带着双方的完整名单（03 §2.2）。
`aes256-cbc` / `aes128-cbc` 的名字常量仍然保留并标明未实现，只用于辨认对端清单里的名字，不能放进我们的清单。

### 6.4 MAC（只在非 AEAD 加密下使用）

| 算法 | 默认 |
| --- | :-: |
| `hmac-sha2-256-etm@openssh.com` / `hmac-sha2-512-etm@openssh.com` | ✅ 最高优先（Encrypt-then-MAC） |
| `hmac-sha2-256` / `hmac-sha2-512` | ✅ |
| `hmac-sha1-etm@openssh.com` / `hmac-sha1` | ❌ 默认关。〔互操作〕老设备 |
| `hmac-md5*` / `*-96` 截断变体 | ❌ **不实现** |

### 6.5 压缩

| 算法 | 默认 |
| --- | :-: |
| `none` | ✅ |
| `zlib@openssh.com`（认证后才开始压缩） | 可选，默认关 |
| `zlib`（RFC 4253，握手期即压缩） | ❌ 不实现 |

〔决策〕**只实现 `zlib@openssh.com`，不实现裸 `zlib`。** 裸 `zlib` 从首次 NEWKEYS 起就压，
认证报文（口令、公钥签名）也在压缩流里 —— 密文长度会泄漏明文的可压缩性，未认证的连接方就能做
CRIME 类的压缩旁路。OpenSSH 服务端早已只在认证后压缩，不提供裸 `zlib` 不影响实际互通。
使用者即使手动把 `zlib` 加进清单，连接前的清单校验就会拒绝它（§6.6），而不是悄悄按不压缩处理（见 01 §六）。

〔决策〕**默认不开压缩**，与 OpenSSH 一致。
理由：现代链路上压缩通常不划算（CPU 换带宽），且压缩 + 加密的组合有
CRIME 类侧信道的历史教训。需要的人（弱网、高延迟）用 `WithCompression()` 显式打开。

### 6.6 老算法开关与清单校验

默认清单之外的老算法由 `SshAlgorithmSet.WithLegacyInterop()` 一次放开：在各类清单**末尾追加**
`diffie-hellman-group14-sha1`、`ssh-rsa`（SHA-1 签名的主机密钥）、`hmac-sha1-etm@openssh.com` 与 `hmac-sha1`。
追加而不是前置 —— 对端只要还支持一个现代算法，协商结果就与不开时相同。
它**不含** CBC（§6.3）。

使用者也可以自己组清单，但每一类都不能为空，且密钥交换、加密、MAC、压缩这四类里的每个名字
都必须是本库实现了的。**连接开始时、拨号之前**就校验，不合格直接抛 `ArgumentException`，
不去拨号。规则与理由见 03 §2.2。

〔决策〕**算法目录公开**（`SshAlgorithmCatalog`）：每一类（`SshAlgorithmCategory`）实现了哪些名字（`Implemented`，
按偏好顺序：默认清单在前、老算法在后）、常见却没实现的有哪些（`KnownUnimplemented`，如 CBC、`umac-*`、`ssh-dss`）。
使用者让用户自己写清单时据此当场说清「没实现」还是「不认识」。它与上面的校验、各个工厂表同一个口径，用例逐个核对。
〔历史〕曾经没有这份目录，宿主用 `Default.WithLegacyInterop()` 推算实现了哪些，再手工维护一份「认得但没实现」的名单。

〔决策〕**只含 FIPS 认可算法的清单**：`SshAlgorithmSet.FipsApprovedOnly`，与 RHEL 的 FIPS 加密策略对 SSH 放行的一致 ——
密钥交换先是两种面向 FIPS 的后量子混合，再是 NIST 曲线的 ECDH 与 DH 标准群，主机密钥只用 ECDSA 与 SHA-2 的 RSA（含证书），加密只用 AES-GCM / AES-CTR，
MAC 只用 HMAC-SHA2；不含 X25519、Ed25519、ChaCha20-Poly1305，也不含带 X25519 或 sntrup761 的后量子混合。**它只限定了算法，不等于本库通过了 FIPS 140 验证**：
AES、SHA-2、ECDH、ECDSA、RSA 走 BCL（Windows 上是经过验证的 CNG，别的平台取决于系统的 OpenSSL），ML-KEM 在平台支持时（`MLKem.IsSupported`）走 BCL、
否则与 DH 标准群的模幂一样走 BouncyCastle。

它的密钥交换清单依次是：`mlkem768nistp256-sha256`、`mlkem1024nistp384-sha384`、`ecdh-sha2-nistp256/384/521`、
`diffie-hellman-group16-sha512`、`diffie-hellman-group14-sha256`。

〔决策〕**两种 NIST 曲线的 ML-KEM 混合（03 §3.7）排在这份清单的最前面。**它们只用 FIPS 认可的原语（ML-KEM、P-256 / P-384 上的 ECDH、SHA-2），
RFC 10042 附录 B 的看法是它的组合方式看起来属于 FIPS 认可的派生（NIST SP 800-227 没有点名）；RHEL 10.2 的 FIPS 策略给 sshd 的清单正是这两种在前、ECDH 在后。
合规环境要的正是「只谈认可的算法」与「先截获、以后再解」的防护两样都有 —— 不给 X25519 的服务端上，没有这两种就只剩不带后量子的 ECDH。
不支持它们的服务端照旧谈成 `ecdh-sha2-nistp256`，与以前一样。768 / P-256 在前，理由同 §6.1。

〔历史〕这个预设最初不含这两种混合：那时还没有能对照验证线上格式的服务端，写了也只能自己证明自己。
2026-10-06 找到了靶机（AlmaLinux 10.2 的 OpenSSH 9.9p1，RHEL 10.2 的下游补丁，两种都有），草案也已定稿为 RFC 10042，于是补上（03 §3.7）。

## 七 算法优先级的排法

〔决策〕**客户端列表的顺序就是我们的偏好顺序**，取「客户端列表中第一个双方都支持的」
（RFC 4253 §7.1 的规则）。排序原则：

1. 安全性优先于性能：后量子混合 > 椭圆曲线 > 有限域 DH。
2. AEAD 优先于「加密 + MAC」：少一次数据遍历，且没有 MAC 与加密顺序的历史坑。
3. 同等安全性下，选有硬件加速的：AES-GCM 在有 AES-NI 的机器上优于 ChaCha20；
   **但默认第一位仍是 ChaCha20-Poly1305** —— 它在所有平台上都快，
   而 AES 在无 AES-NI 的设备上会掉到很难看的数字。
   〔决策〕运行期按 `System.Runtime.Intrinsics.X86.Aes.IsSupported`（且 `Pclmulqdq.IsSupported`）/
   `System.Runtime.Intrinsics.Arm.Aes.IsSupported` 动态调整这两者的相对顺序：
   有 AES 硬件加速时 AES-GCM 排在 ChaCha20-Poly1305 之前。
4. 默认关闭的（老算法）不进默认列表，只在用户显式配置时加入。

## 八 术语表

| 术语 | 含义 |
| --- | --- |
| **帧 / frame** | 一个完整的 SSH 二进制报文（长度 + 填充长度 + 载荷 + 填充 + MAC） |
| **载荷 / payload** | 帧中去掉长度、填充长度与填充之后的部分，第一个字节是消息编号 |
| **序号 / sequence number** | 每个方向各一个 `uint32`，从 0 开始、逐帧递增、**溢出回绕**。参与 MAC 计算 |
| **密码套件 / cipher suite** | 「加密算法 + MAC 算法」的组合。AEAD 算法自带完整性，不配 MAC |
| **形状 / shape** | 密码套件在分帧上的差异：长度字段是否加密、AAD 多长、tag 多长、是否 EtM |
| **发送闸门 / send gate** | 重协商期间只放行传输层消息的那道闸。见 03 与 architecture.md §5.4 |
| **水位线 / watermark** | SFTP 写入中「已**连续**确认的最高偏移」。见 06 |
