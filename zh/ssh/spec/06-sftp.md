# 06 · SFTP

> 规范依据：draft-ietf-secsh-filexfer-02（**SFTP v3，事实标准**）；
> OpenSSH `PROTOCOL` 的 SFTP 扩展章节（`posix-rename`、`statvfs`、`fsync`、
> `hardlink`、`limits`、`copy-data`、`home-directory`、`expand-path`）。
>
> 对应实现：`Sftp/`，分三层：`SftpWire`（纯编解码）、
> `SftpRequestPipeline`（流水线）、`SftpFileSystem`（面向使用者）。

> **为什么是 v3 而不是更高版本**：v3 是 draft-02，OpenSSH 只实现它，
> 而 OpenSSH 是绝大多数 SFTP 服务端的实现或行为基准。v4–v6 在真实世界里
> 几乎见不到；为它们写代码是为不存在的对端付成本。
> 〔决策〕**只实现 v3**，协商到更高版本时降级到 3。

---

## 一 承载与握手

SFTP 跑在一条 `session` 通道的 `subsystem` 请求之上：

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务端
    C->>S: CHANNEL_OPEN "session"
    S->>C: CHANNEL_OPEN_CONFIRMATION
    C->>S: CHANNEL_REQUEST "subsystem" = "sftp" (want_reply=true)
    S->>C: CHANNEL_SUCCESS
    C->>S: SSH_FXP_INIT (version=3)
    S->>C: SSH_FXP_VERSION (version ‖ 扩展对)
    Note over C: 解析扩展 → SftpCapabilities
    opt 服务端宣告了 limits@openssh.com
        C->>S: SSH_FXP_EXTENDED "limits@openssh.com"
        S->>C: SSH_FXP_EXTENDED_REPLY
        Note over C: 据此定管线深度与块大小（§5.2）
    end
```

〔互操作〕若 `subsystem` 被拒（部分服务端只允许 `exec`），
〔决策〕**不自动回退到 `exec sftp-server`**。
理由：回退等于在服务端管理员明确禁用 subsystem 的情况下绕过它的配置。
把失败如实报出来（`SftpUnavailable`，并在消息里说明可能是服务端禁用了 sftp 子系统）。

---

## 二 SFTP 报文的外层结构

**注意：这一层与 SSH 的二进制报文无关**，SFTP 有自己的分帧，
跑在 SSH 通道的字节流上。

```
uint32   length      —— 不含自身这 4 字节
byte     type        —— SSH_FXP_*
uint32   request-id  —— 除 INIT / VERSION 外都有
byte[]   type 相关
```

| 约束 | 值 |
| --- | --- |
| `length` 上限 | 〔决策〕**256 KiB + 1024**（数据块上限加协议头余量）。超限断开通道 |
| `request-id` | `uint32`，我们从 1 递增、**回绕**。0 保留不用 |

**一条通道上可以有大量在途请求**，应答顺序**不保证**与发送顺序一致 ——
这正是 SFTP 能跑出吞吐的原因，也是必须用 id 而不是队列来对齐应答的原因
（与通道请求的 FIFO 语义不同，见 `05-connection.md` §5.1）。

---

## 三 报文类型

### 3.1 请求（客户端 → 服务端）

| # | 类型 | 用途 |
| :-: | --- | --- |
| 1 | `SSH_FXP_INIT` | 握手（**无 request-id**） |
| 3 | `SSH_FXP_OPEN` | 打开文件 → handle |
| 4 | `SSH_FXP_CLOSE` | 关闭 handle |
| 5 | `SSH_FXP_READ` | 按偏移读 |
| 6 | `SSH_FXP_WRITE` | 按偏移写 |
| 7 | `SSH_FXP_LSTAT` | stat，**不跟随**符号链接 |
| 8 | `SSH_FXP_FSTAT` | 按 handle stat |
| 9 | `SSH_FXP_SETSTAT` | 设属性 |
| 10 | `SSH_FXP_FSETSTAT` | 按 handle 设属性 |
| 11 | `SSH_FXP_OPENDIR` | 打开目录 → handle |
| 12 | `SSH_FXP_READDIR` | 读一批目录项 |
| 13 | `SSH_FXP_REMOVE` | 删文件 |
| 14 | `SSH_FXP_MKDIR` | 建目录 |
| 15 | `SSH_FXP_RMDIR` | 删空目录 |
| 16 | `SSH_FXP_REALPATH` | 规范化路径 |
| 17 | `SSH_FXP_STAT` | stat，**跟随**符号链接 |
| 18 | `SSH_FXP_RENAME` | 重命名 |
| 19 | `SSH_FXP_READLINK` | 读链接目标 |
| 20 | `SSH_FXP_SYMLINK` | 建符号链接（**参数顺序有坑，见 §4.5**） |
| 200 | `SSH_FXP_EXTENDED` | 厂商扩展 |

### 3.2 应答（服务端 → 客户端）

| # | 类型 | 内容 |
| :-: | --- | --- |
| 2 | `SSH_FXP_VERSION` | `uint32 version` ‖ 重复的 `string name ‖ string data` |
| 101 | `SSH_FXP_STATUS` | `uint32 code` ‖ `string message` ‖ `string lang` |
| 102 | `SSH_FXP_HANDLE` | `string handle`（**≤ 256 字节**） |
| 103 | `SSH_FXP_DATA` | `string data` |
| 104 | `SSH_FXP_NAME` | `uint32 count` ‖ 重复的 `string filename ‖ string longname ‖ ATTRS` |
| 105 | `SSH_FXP_ATTRS` | ATTRS |
| 201 | `SSH_FXP_EXTENDED_REPLY` | 扩展相关 |

### 3.3 状态码

| 码 | 名称 | 我们的映射 |
| :-: | --- | --- |
| 0 | `OK` | 成功 |
| 1 | `EOF` | **不是错误**：`READ` 时表示读到末尾，`READDIR` 时表示目录读完 |
| 2 | `NO_SUCH_FILE` | `SftpErrorCode.NotFound` |
| 3 | `PERMISSION_DENIED` | `AccessDenied` |
| 4 | `FAILURE` | `Failure` —— **服务端的万能错误码**，见下 |
| 5 | `BAD_MESSAGE` | `ProtocolError` |
| 6 | `NO_CONNECTION` | `NotConnected` |
| 7 | `CONNECTION_LOST` | `ConnectionLost` |
| 8 | `OP_UNSUPPORTED` | `Unsupported` |

〔重要〕**码 4（`FAILURE`）承载了 v3 里的绝大多数真实错误** ——
「目录非空」「文件已存在」「磁盘满」「配额超限」在 v3 里全是 4。
因此 `SftpException.Message` **必须**带上服务端给的 `message` 文本，
那是唯一能区分它们的信息。〔决策〕同时提供
`SftpException.ServerMessage` 原文字段，让上层能做自己的模式匹配
而不必去解析我们拼过的消息。

---

## 四 关键报文的细节

### 4.1 `SSH_FXP_OPEN`

| # | 类型 | 字段 |
| :-: | --- | --- |
| 4 | `string` | 路径（UTF-8） |
| 5 | `uint32` | pflags |
| 6 | ATTRS | 创建时的属性（通常只给权限） |

pflags：

| 位 | 名称 | 含义 |
| :-: | --- | --- |
| 0x01 | `READ` | |
| 0x02 | `WRITE` | |
| 0x04 | `APPEND` | 每次写都追加到末尾，**忽略偏移** |
| 0x08 | `CREAT` | 不存在则创建 |
| 0x10 | `TRUNC` | 截断已有内容（须配 `CREAT`） |
| 0x20 | `EXCL` | 已存在则失败（须配 `CREAT`） |

〔决策〕**创建文件时的默认权限 0644**，目录 0755，都可配。
不传 ATTRS 会让服务端用它自己的默认值（通常受 umask 影响），
结果不可预测 —— 传明确的值。

### 4.2 ATTRS 结构

```
uint32   flags
uint64   size           —— flags & 0x01
uint32   uid            —— flags & 0x02
uint32   gid            —— flags & 0x02（uid/gid 是同一个标志位！）
uint32   permissions    —— flags & 0x04
uint32   atime          —— flags & 0x08
uint32   mtime          —— flags & 0x08（atime/mtime 也是同一个标志位）
uint32   extended_count —— flags & 0x80000000
重复：string type ‖ string data
```

**三个坑**：

1. **`uid` 与 `gid` 共用一个标志位，`atime` 与 `mtime` 共用一个。**
   要设 mtime 就必须同时给 atime。〔决策〕只想改 mtime 时，
   先 `STAT` 取回当前 atime 一并写回。
2. **时间是 32 位 Unix 秒**，2038 年会溢出。v3 没有解法，照实现。
   〔决策〕读到的值按无符号解释可以撑到 2106 年 ——
   但服务端通常按有符号发，所以**按有符号读**，与 OpenSSH 一致。
3. `permissions` 的高位是文件类型（`S_IFMT`）：
   `0o100000` 普通文件、`0o040000` 目录、`0o120000` 符号链接。
   **v3 没有单独的类型字段，类型只能从这里取。**

### 4.3 `SSH_FXP_READ` / `SSH_FXP_WRITE`

| READ | | | WRITE | | |
| :-: | --- | --- | :-: | --- | --- |
| 4 | `string` | handle | 4 | `string` | handle |
| 5 | `uint64` | offset | 5 | `uint64` | offset |
| 6 | `uint32` | length | 6 | `string` | data |

**都是绝对偏移，服务端不维护文件位置。**
这意味着读写天然可以乱序并发 —— 这正是 §5 的流水线基础，
也是 §6 的水位线问题的来源。

〔重要〕**`READ` 返回的数据可能少于请求的长度**，这不是错误，
必须循环读直到拿够或遇到 `EOF`。

### 4.4 `SSH_FXP_READDIR`

- 每次返回**一批**目录项，不是全部。
- 返回 `STATUS = EOF` 表示读完。
- `longname` 是 `ls -l` 风格的一行文本，**格式未标准化**。
  〔决策〕**不解析 `longname`**，一切信息取自 ATTRS。
  解析它是各家 SFTP 客户端 bug 的经典来源（时间格式、locale、列对齐全都因服务端而异）。
  但**保留原文**供使用者需要时用。
- `.` 与 `..` **会**出现在结果里。〔决策〕`SftpFileSystem` 层默认过滤掉，
  提供开关。

### 4.5 `SSH_FXP_SYMLINK` 的参数顺序

> **这是 SFTP 里最有名的一个坑。**

draft-02 规定的顺序是 `linkpath` 然后 `targetpath`。
**但 OpenSSH 的实现把两者写反了**（OpenSSH bugzilla #861），
而 OpenSSH 是事实标准，所有客户端都跟着它错。

〔决策〕**按 OpenSSH 的顺序发**：先 `targetpath`，后 `linkpath`。

```
uint32   request-id
string   targetpath     ← 链接指向哪里（OpenSSH 顺序）
string   linkpath       ← 在哪里创建链接
```

〔决策〕**不提供「按 draft 顺序」的开关。** 没有已知的服务端按 draft 实现；
加一个永远不该被打开的开关，只会让人在排查别的问题时误开它。
若将来真遇到，用 `posix-rename` 那一路的扩展机制处理。

这条必须在代码注释里也写清楚，否则将来一定会有人"顺手修正"它。

### 4.6 `SSH_FXP_REALPATH`

用于把相对路径、`~`、`.`、`..` 规范化成绝对路径。
返回 `SSH_FXP_NAME` 且 `count == 1`。

〔决策〕**连接建立后立刻对 `"."` 做一次 `REALPATH`**，
拿到工作目录作为 `SftpFileSystem.WorkingDirectory` 的初值。
这是唯一可靠的「用户家目录在哪」的答案 —— 比拼 `/home/{user}` 靠谱得多。

---

## 五 请求流水线

### 5.1 结构

```mermaid
flowchart LR
    A[SftpFileSystem<br/>面向使用者] --> B[SftpRequestPipeline<br/>id 分配 · 在途窗口 · 取消]
    B --> C[RequestLedger<br/>池化 IValueTaskSource]
    B --> D[SftpWire<br/>纯编解码]
    D --> E[SSH 通道 PipeWriter/PipeReader]
```

`SftpWire` 是**纯函数**：`bytes ↔ SftpMessage`，无状态、无 I/O。
这让它可以用报文样本逐字节断言，不需要任何对端 —— 这是整个 SFTP 层
最容易也最值得测透的一部分。

### 5.2 深度与块大小的自适应

〔决策〕**按服务端宣告的 `limits@openssh.com` 定，而不是写死。**

`limits@openssh.com` 的应答给出四个值：

| 字段 | 用途 |
| --- | --- |
| `max-packet-length` | 单个 SFTP 报文上限 |
| `max-read-length` | 单次 `READ` 的 length 上限 |
| `max-write-length` | 单次 `WRITE` 的 data 上限 |
| `max-open-handles` | 同时打开的 handle 数上限（0 = 未知） |

没有这个扩展时的保守默认（〔决策〕）：

| 项 | 默认 |
| --- | --- |
| 块大小 | 32 KiB（SSH 通道 max packet 的典型值） |
| 在途请求数 | 由 BDP 估计：`clamp(RTT × 目标带宽 / 块大小, 8, 256)` |

**为什么深度要自适应**：在途请求数 × 块大小就是 SFTP 层的「窗口」。
和通道窗口一样（`05-connection.md` §3.3），固定值在高 RTT 链路上直接封死吞吐。
写死 64 × 32 KB = 2 MiB，200 ms RTT 下同样是 10 MB/s 封顶。

### 5.3 取消

`RequestLedger` 的取消语义**必须**明确：

- 取消一个在途请求 **不会**让服务端停下来 —— SFTP 没有取消报文。
  我们只是不再等它的应答。
- 因此取消后**必须**继续消费并丢弃那个 id 的应答，
  且**必须处理「应答里带着一个需要关闭的 handle」的情况** ——
  `OPEN` 被取消但服务端已经打开了文件，那个 handle 不关就泄漏在服务端。
  〔决策〕账本在取消时保留一个「善后回调」，收到迟到应答时执行它（发 `CLOSE`）。

### 5.4 通道关闭时的收尾

通道断开 → 账本里所有在途请求**一次性**以同一个 `SshChannelClosedException` 收尾。
这是 L6 统一账本的直接收益：这段逻辑只写一次、只证明一次。

---

## 六 写入水位线 —— 消灭「盲退 2 MB」

### 6.1 问题

流水线满载时有 N 个在途 `WRITE`，**应答顺序不保证**。
传输中途断开时：

```
偏移 0 ....... 1 MiB ....... 2 MiB ....... 3 MiB
已确认：  [==========]        [====]        [====]
                      ↑ 空洞          ↑ 空洞
文件长度 = 3 MiB  ← 但 1–2 MiB 那段其实没写进去
```

服务端报告的**文件长度**只代表「已确认的**最高**偏移」，
不代表它之前的每个字节都落盘了。中间可能留有读作 0 的空洞。

现有做法是**从文件长度盲退一个完整的在途窗口**（例如 64 × 32 KB = 2 MiB），
使续传起点之前的数据可信。代价是每次续传都要重传 2 MiB，
而且这个数字是「猜」出来的 —— 换个实现或换个配置就不对了。

### 6.2 解法：连续确认偏移

管线为每个写 handle 维护：

| 字段 | 含义 |
| --- | --- |
| `_ackedRanges` | 已确认的区间集合（有序、合并相邻） |
| `DurableLength` | **从 0 开始连续已确认**的最高偏移 |

```
每收到一个 WRITE 的 OK 应答：
    把 [offset, offset+len) 并入 _ackedRanges（合并相邻区间）
    DurableLength = 从 0 起第一个空洞的位置（即第一个区间的右端，若它从 0 开始）
```

`SftpStream.DurableLength` 暴露它。断点续传从这个数续，**精确，不用猜**。

〔实现要点〕区间集合要用**有序数组 + 二分插入**而不是字典 ——
顺序写入时新区间几乎总是接在最后一个的尾部，
合并后集合大小恒为 1，代价 O(1)。乱序写入时集合才会增长，
且上限就是在途请求数（N ≤ 256），完全可控。

〔决策〕**通道断开时把 `DurableLength` 写进异常**
（`SftpTransferInterruptedException.DurableLength`），
让上层不必再去 stat 一次、更不必盲退。

### 6.3 顺序保证的另一条路

〔决策〕同时提供 `SftpWriteMode.Sequential`：
在途请求数固定为 1，牺牲吞吐换「文件长度就是可信长度」。
用于那些必须保证任何时刻文件都是前缀完整的场景（例如写配置文件）。

---

### 6.4 文件流只有异步形态

〔决策〕（`architecture.md` 原则 1）文件流是 `Stream` 的派生类，而 `Stream` 自带同步的
`Read` / `Write` / `SetLength` / `Flush` / `Dispose`。同步版本只能靠「阻塞一个线程等网络往返」实现 ——
UI 线程上是界面卡住一个 RTT，线程池上并发一多就是饿死。所以：

| 同步成员 | 行为 |
| --- | --- |
| `Read` / `Write` / `SetLength`（及单字节、`Span` 重载） | 抛 `NotSupportedException`，消息里指明该用的异步版本 |
| `Flush` | **不阻塞的空操作**（本端没有缓冲）；已知的写入失败照样抛出。要确认落盘用 `FlushAsync` |
| `Dispose` | **不阻塞**：收尾（等在途写入确认、关句柄）交给后台，立刻返回；看不到收尾的错误，要看就用 `await using` |

`Flush` 保留为不抛的空操作，是因为包装流（`StreamWriter` 之类）在自己的收尾里会同步调用它；
让它抛会把一个无害的调用变成失败。

## 七 扩展

### 7.1 我们使用的扩展

| 扩展 | 用途 | 没有时的行为 |
| --- | --- | --- |
| `posix-rename@openssh.com` | **原子**重命名（覆盖目标） | 退化到 `SSH_FXP_RENAME`，**并把这个事实报出来** |
| `hardlink@openssh.com` | 建硬链接 | 抛 `Unsupported` |
| `fsync@openssh.com` | 强制落盘 | 抛 `Unsupported` |
| `statvfs@openssh.com` | 文件系统用量 | 抛 `Unsupported` |
| `limits@openssh.com` | §5.2 | 用保守默认 |
| `copy-data` | **服务端内**复制，不经过网络 | 退化到「下载再上传」 |
| `home-directory` | 取指定用户的家目录 | 用 `REALPATH "."` |
| `expand-path@openssh.com` | 展开 `~` | 用 `REALPATH` |

### 7.2 能力查询是公开 API

```
SftpCapabilities Capabilities { get; }
  bool HasPosixRename / HasHardlink / HasFsync / HasStatVfs / HasCopyData ...
  IReadOnlyDictionary<string,string> RawExtensions { get; }
```

〔决策〕**能力必须可查，而不只是内部降级。**

理由很具体：`posix-rename` 与普通 `rename` 的语义**不一样** ——
前者原子覆盖，后者在目标存在时失败（且某些服务端连跨目录移动都拒）。
库内部静默降级，上层就无从知道自己拿到的是哪一种语义，
也无法在 UI 上提示「这台服务器不支持原子覆盖，移动操作可能不是原子的」。

### 7.3 `SSH_FXP_EXTENDED` 的形状

| # | 类型 | 字段 |
| :-: | --- | --- |
| 4 | `string` | 扩展名 |
| 5+ | 扩展相关 | |

`ISftpExtension` 是公开的扩展点（架构 §8 第 10 项），
让使用者能加厂商私有扩展而不必改库。

---

## 八 符号链接的口径

〔决策〕**列目录与 stat 都用 `LSTAT`（不跟随），链接项再补一次跟随的 `STAT` 与 `READLINK`。**

得到的条目：

| 字段 | 含义 |
| --- | --- |
| `IsSymbolicLink` | **链接本身**是不是链接 |
| `LinkTarget` | `READLINK` 的原文（可能是相对路径） |
| 其余字段（`Length` / `IsDirectory` / 时间） | 描述**链接指向的对象** |
| 断链（跟随 `STAT` 失败） | 保留链接自身的属性，`IsDirectory = false`，**不是 `null`** |

理由：
- 直接用跟随的 `STAT` 会让「这是个链接」这个事实彻底消失，
  于是删除一个指向目录的链接会变成递归删除目标目录里的东西 —— 这是数据事故。
- 断链返回 `null` 也不对：链接本身是存在的，删除它不能先报「找不到」。

〔性能〕链接项的补充请求**必须并发发出**（它们在同一条通道上流水线）。
`/usr/lib` 那种几百个 `.so` 链接的目录，串行补就是几百轮往返，并发补只是一轮。

---

## 九 边界与错误速查

| 情况 | 处理 |
| --- | --- |
| `length` 超上限 | 断开通道，`ProtocolError` |
| handle 超过 256 字节 | `ProtocolError`（draft-02 规定的上限） |
| 收到未知 `request-id` 的应答 | 〔决策〕忽略 + debug 日志（可能是已取消请求的迟到应答，§5.3） |
| 收到未知报文类型 | `ProtocolError`（SFTP 层没有 `UNIMPLEMENTED` 机制） |
| 服务端 version > 3 | 降到 3 继续 |
| 服务端 version < 3 | 〔决策〕**拒绝**，抛 `SftpUnsupportedVersion`。v0–v2 差异过大，不值得支持 |
| `READ` 返回的数据多于请求的 length | `ProtocolError` |
| `READDIR` 返回的 `count` 与实际项数不符 | `ProtocolError` |
| 路径含 `\0` | 本地拒绝（`ArgumentException`），不发给服务端 |
| 在途请求数达上限 | 等待（背压），不报错 |
