# 行为规格

> **这套文档是 VelaShell.Ssh 实现的唯一依据。**
>
> 写实现的时候，桌面上应当只有两样东西：本目录下的规格，以及它引用的 RFC。
> **不要打开任何其它 SSH 实现的源码** —— [`src/VelaShell.Ssh/AGENTS.md`](https://github.com/joesdu/VelaShell/blob/main/src/VelaShell.Ssh/AGENTS.md) §2 纪律 2。

| 文件 | 内容 | 实现层 |
| --- | --- | --- |
| [`00-overview.md`](00-overview.md) | 术语、记法、数据类型、通用安全下限、算法总表与优先级 | — |
| [`01-transport-framing.md`](01-transport-framing.md) | 二进制报文协议、填充、序号、**密码套件的三种形状**、收发时序与合并 | `Transport/` `Crypto/` |
| [`02-version-exchange.md`](02-version-exchange.md) | 标识串交换、前导行、互操作细节 | `Session/` |
| [`03-key-exchange.md`](03-key-exchange.md) | 算法协商、七种 KEX、**交换哈希逐字段**、密钥派生、严格 KEX、重协商与发送闸门 | `Crypto/` `Session/` |
| [`04-authentication.md`](04-authentication.md) | 方法调度、**部分成功（2FA）**、publickey 签名输入、**keyboard-interactive**、扩展协商 | `Auth/` |
| [`05-connection.md`](05-connection.md) | 通道生命周期、**流控与自适应窗口**、通道请求、pty 与像素尺寸 | `Channels/` |
| [`06-sftp.md`](06-sftp.md) | SFTP v3 wire、请求流水线、**写入水位线**、扩展与能力查询、符号链接口径 | `Sftp/` |
| [`07-forwarding.md`](07-forwarding.md) | 四种转发、SOCKS5 子集、半关闭、**库内计量**、agent 转发 | `Forwarding/` |
| [`08-failures.md`](08-failures.md) | 失败分类学、三个带结构化上下文的异常、度量、追踪、报文旁路 | `Diagnostics/` |
| [`09-dialing.md`](09-dialing.md) | 拨号层：SOCKS5、HTTP CONNECT、跳板、代理命令、嵌套、逐跳失败信息、`ssh_config` 映射 | `Transport/` `Config/` |

## 先读哪几节

按「最容易出错、也最贵的先读」排：

1. **[03 §4 交换哈希](03-key-exchange.md)** —— 每一个字节顺序错误都表现为同一句
   「签名验证失败」，而且常常是**概率性**的（`mpint` 前导零）。
   这一节的表应当直接变成单测的数据源。
2. **[04 §3.3 部分成功](04-authentication.md)** —— 把 `partial_success = true`
   当成失败，2FA 就永远连不上。
3. **[01 §1.2 填充与对齐](01-transport-framing.md)** —— AEAD 下长度字段不参与对齐，
   写错了在某些包长上才会暴露。
4. **[06 §4.5 SYMLINK 参数顺序](06-sftp.md)** —— 规范写反了，全世界跟着 OpenSSH 错。
   代码里必须写清楚，否则将来一定有人"顺手修正"它。
5. **[05 §1 EOF 不是 CLOSE](05-connection.md)** —— 半关闭处理错会静默截断数据。

## 怎么改这套规格

- 规格里写不清楚的地方，**回去查 RFC 并把规格补上**，而不是去别处找现成答案。
- 每条〔决策〕都要给理由 —— 将来的人会问「为什么是这样」，
  答案不该只存在于某个人的记忆里。
- 每条〔互操作〕都要写明**是哪个服务端、什么版本、怎么验证的**。
  没有具体对象的「据说有些服务器」不算。
- 改了行为就要同步改规格，两者在同一个 PR 里。
