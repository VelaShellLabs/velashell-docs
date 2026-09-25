# 06 · SFTP

> Normative basis: draft-ietf-secsh-filexfer-02 (**SFTP v3, the de facto standard**);
> the SFTP extensions section of OpenSSH `PROTOCOL` (`posix-rename`, `statvfs`, `fsync`,
> `hardlink`, `limits`, `copy-data`, `home-directory`, `expand-path`).
>
> Implementation: `Sftp/`, in three layers: `SftpWire` (pure encoding/decoding),
> `SftpRequestPipeline` (pipelining), `SftpFileSystem` (caller-facing).
>
> 中文：[`../../../zh/ssh/spec/06-sftp.md`](../../../zh/ssh/spec/06-sftp.md)

> **Why v3 and not a higher version**: v3 is draft-02, and OpenSSH implements only it,
> while OpenSSH is the implementation or behavioral baseline for the vast majority of SFTP servers. v4–v6 are
> almost never seen in the real world; writing code for them is paying costs for peers that do not exist.
> 〔Decision〕**Implement v3 only**; when a higher version is negotiated, downgrade to 3.

---

## 1 Transport and handshake

SFTP runs on top of a `subsystem` request on a `session` channel:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: CHANNEL_OPEN "session"
    S->>C: CHANNEL_OPEN_CONFIRMATION
    C->>S: CHANNEL_REQUEST "subsystem" = "sftp" (want_reply=true)
    S->>C: CHANNEL_SUCCESS
    C->>S: SSH_FXP_INIT (version=3)
    S->>C: SSH_FXP_VERSION (version ‖ extension pairs)
    Note over C: Parse extensions → SftpCapabilities
    opt Server announced limits@openssh.com
        C->>S: SSH_FXP_EXTENDED "limits@openssh.com"
        S->>C: SSH_FXP_EXTENDED_REPLY
        Note over C: Use it to set pipeline depth and block size (§5.2)
    end
```

〔Interop〕If `subsystem` is refused (some servers allow only `exec`),
〔Decision〕**do not automatically fall back to `exec sftp-server`**.
Rationale: falling back amounts to bypassing the server administrator's configuration when they have explicitly disabled the subsystem.
Report the failure truthfully (`SftpUnavailable`, with the message explaining that the server may have disabled the sftp subsystem).

---

## 2 Outer structure of SFTP messages

**Note: this layer has nothing to do with SSH binary packets.** SFTP has its own framing,
running on the byte stream of an SSH channel.

```
uint32   length      — excludes these 4 bytes themselves
byte     type        — SSH_FXP_*
uint32   request-id  — present in all except INIT / VERSION
byte[]   type-specific
```

| Constraint | Value |
| --- | --- |
| `length` upper limit | 〔Decision〕**256 KiB + 1024** (data block limit plus protocol header headroom). Exceeding it closes the channel |
| `request-id` | `uint32`; we increment from 1 and **wrap around**. 0 is reserved and unused |

**A channel can have a large number of in-flight requests**, and the reply order is **not guaranteed** to match the sending order —
this is precisely why SFTP can achieve throughput, and why replies must be matched by id rather than by a queue
(unlike the FIFO semantics of channel requests, see `05-connection.md` §5.1).

---

## 3 Message types

### 3.1 Requests (client → server)

| # | Type | Purpose |
| :-: | --- | --- |
| 1 | `SSH_FXP_INIT` | Handshake (**no request-id**) |
| 3 | `SSH_FXP_OPEN` | Open file → handle |
| 4 | `SSH_FXP_CLOSE` | Close handle |
| 5 | `SSH_FXP_READ` | Read at offset |
| 6 | `SSH_FXP_WRITE` | Write at offset |
| 7 | `SSH_FXP_LSTAT` | stat, **not following** symbolic links |
| 8 | `SSH_FXP_FSTAT` | stat by handle |
| 9 | `SSH_FXP_SETSTAT` | Set attributes |
| 10 | `SSH_FXP_FSETSTAT` | Set attributes by handle |
| 11 | `SSH_FXP_OPENDIR` | Open directory → handle |
| 12 | `SSH_FXP_READDIR` | Read a batch of directory entries |
| 13 | `SSH_FXP_REMOVE` | Delete file |
| 14 | `SSH_FXP_MKDIR` | Create directory |
| 15 | `SSH_FXP_RMDIR` | Delete empty directory |
| 16 | `SSH_FXP_REALPATH` | Canonicalize path |
| 17 | `SSH_FXP_STAT` | stat, **following** symbolic links |
| 18 | `SSH_FXP_RENAME` | Rename |
| 19 | `SSH_FXP_READLINK` | Read link target |
| 20 | `SSH_FXP_SYMLINK` | Create symbolic link (**argument order is a trap, see §4.5**) |
| 200 | `SSH_FXP_EXTENDED` | Vendor extension |

### 3.2 Replies (server → client)

| # | Type | Content |
| :-: | --- | --- |
| 2 | `SSH_FXP_VERSION` | `uint32 version` ‖ repeated `string name ‖ string data` |
| 101 | `SSH_FXP_STATUS` | `uint32 code` ‖ `string message` ‖ `string lang` |
| 102 | `SSH_FXP_HANDLE` | `string handle` (**≤ 256 bytes**) |
| 103 | `SSH_FXP_DATA` | `string data` |
| 104 | `SSH_FXP_NAME` | `uint32 count` ‖ repeated `string filename ‖ string longname ‖ ATTRS` |
| 105 | `SSH_FXP_ATTRS` | ATTRS |
| 201 | `SSH_FXP_EXTENDED_REPLY` | Extension-specific |

### 3.3 Status codes

| Code | Name | Our mapping (`SftpException.StatusCode`, of type `SftpStatusCode`) |
| :-: | --- | --- |
| 0 | `OK` | Success |
| 1 | `EOF` | **Not an error**: for `READ` it means end of file reached; for `READDIR` it means the directory has been fully read |
| 2 | `NO_SUCH_FILE` | `NoSuchFile` |
| 3 | `PERMISSION_DENIED` | `PermissionDenied` |
| 4 | `FAILURE` | `Failure` — **the server's catch-all error code**, see below |
| 5 | `BAD_MESSAGE` | `BadMessage` |
| 6 | `NO_CONNECTION` | `NoConnection` |
| 7 | `CONNECTION_LOST` | `ConnectionLost` |
| 8 | `OP_UNSUPPORTED` | `OperationUnsupported` |

〔Important〕**Code 4 (`FAILURE`) carries the vast majority of real errors in v3** —
"directory not empty", "file already exists", "disk full", "quota exceeded" are all 4 in v3.
Therefore `SftpException.Message` **must** include the `message` text given by the server,
which is the only information that can distinguish them. 〔Decision〕Also provide
the `SftpException.ServerMessage` raw-text field, so upper layers can do their own pattern matching
without having to parse the message we composed.

---

## 4 Details of key messages

### 4.1 `SSH_FXP_OPEN`

| # | Type | Field |
| :-: | --- | --- |
| 4 | `string` | Path (UTF-8) |
| 5 | `uint32` | pflags |
| 6 | ATTRS | Attributes at creation (usually only permissions) |

pflags:

| Bit | Name | Meaning |
| :-: | --- | --- |
| 0x01 | `READ` | |
| 0x02 | `WRITE` | |
| 0x04 | `APPEND` | Every write appends to the end, **ignoring the offset** |
| 0x08 | `CREAT` | Create if it does not exist |
| 0x10 | `TRUNC` | Truncate existing content (must be combined with `CREAT`) |
| 0x20 | `EXCL` | Fail if it already exists (must be combined with `CREAT`) |

〔Decision〕**Default permissions when creating files are 0644**, directories 0755, both configurable.
Not passing ATTRS makes the server use its own default (usually affected by umask),
with unpredictable results — pass explicit values.

**Ask for the length right after opening.** A non-truncating open (read, resume, write without `TRUNC`) sends one `FSTAT` on the handle as soon as it has it;
the size becomes the starting value of the stream's `Length` — `Seek(SeekOrigin.End)`, the read-ahead bound of §5.5 and the resume prefix of §6.6 all depend on it.
An open with `TRUNC` need not ask: the length is 0.

- The server refusing `FSTAT` (an error status) **does not fail the open**: the length starts at 0 and is marked "unknown"; the stream reads and writes as usual,
  only `Length` and `Seek(SeekOrigin.End)` are inaccurate, and sequential reads cannot build up read-ahead (§5.5 only sends ahead within the known length).
- 〔Decision〕**Any other failure after `OPEN` succeeded and before the stream reaches the caller (cancellation, a broken pipeline, a malformed `FSTAT` reply)
  first sends `CLOSE` for the handle, then propagates the error.** At that moment the handle exists only on the server and nothing on our side holds it yet;
  without the close, every such failure leaks one, until the server's `max-open-handles` runs out. This is the other half of the problem in §5.3 ("a late reply to a cancelled `OPEN` must be closed").
- For the same reason, arguments that can be checked without the server (e.g. a resume offset must not be negative, §6.6) are checked **before** `OPEN` is sent.

### 4.2 ATTRS structure

```
uint32   flags
uint64   size           — flags & 0x01
uint32   uid            — flags & 0x02
uint32   gid            — flags & 0x02 (uid/gid share the same flag bit!)
uint32   permissions    — flags & 0x04
uint32   atime          — flags & 0x08
uint32   mtime          — flags & 0x08 (atime/mtime also share the same flag bit)
uint32   extended_count — flags & 0x80000000
repeated: string type ‖ string data
```

**Three traps**:

1. **`uid` and `gid` share one flag bit; `atime` and `mtime` share one.**
   To set mtime you must supply atime as well. 〔Decision〕When only mtime should change,
   first `STAT` to retrieve the current atime and write it back together.
2. **Times are 32-bit Unix seconds** and will overflow in 2038. v3 has no solution; implement as specified.
   〔Decision〕Interpreting the value read as unsigned would last until 2106 —
   but servers usually send it as signed, so **read it as signed**, consistent with OpenSSH.
3. The high bits of `permissions` are the file type (`S_IFMT`):
   `0o100000` regular file, `0o040000` directory, `0o120000` symbolic link.
   **v3 has no separate type field; the type can only be taken from here.**

### 4.3 `SSH_FXP_READ` / `SSH_FXP_WRITE`

| READ | | | WRITE | | |
| :-: | --- | --- | :-: | --- | --- |
| 4 | `string` | handle | 4 | `string` | handle |
| 5 | `uint64` | offset | 5 | `uint64` | offset |
| 6 | `uint32` | length | 6 | `string` | data |

**Both use absolute offsets; the server does not maintain a file position.**
This means reads and writes can naturally be concurrent and out of order — this is the basis of the pipeline in §5,
and also the source of the watermark problem in §6.

〔Important〕**`READ` may return less data than the requested length**; this is not an error,
and reading must loop until enough is obtained or `EOF` is encountered.

### 4.4 `SSH_FXP_READDIR`

- Each call returns **a batch** of directory entries, not all of them.
- A return of `STATUS = EOF` means reading is complete.
- `longname` is a line of text in `ls -l` style, **with a non-standardized format**.
  〔Decision〕**Do not parse `longname`**; all information is taken from ATTRS.
  Parsing it is a classic source of bugs in SFTP clients (time format, locale, column alignment all vary by server).
  But **keep the raw text** for callers who need it.
- `.` and `..` **do** appear in the results. 〔Decision〕The `SftpFileSystem` layer filters them out by default,
  with a switch provided.

### 4.5 Argument order of `SSH_FXP_SYMLINK`

> **This is the most famous trap in SFTP.**

draft-02 specifies the order as `linkpath` then `targetpath`.
**But OpenSSH's implementation swaps the two** (OpenSSH bugzilla #861),
and since OpenSSH is the de facto standard, all clients follow it in the same mistake.

〔Decision〕**Send in OpenSSH's order**: `targetpath` first, then `linkpath`.

```
uint32   request-id
string   targetpath     ← where the link points to (OpenSSH order)
string   linkpath       ← where the link is created
```

〔Decision〕**No "draft order" switch is provided.** There is no known server that implements the draft order;
adding a switch that should never be turned on only invites someone to turn it on by mistake while investigating some other problem.
If one is ever actually encountered, handle it via an extension mechanism along the lines of `posix-rename`.

This must also be stated clearly in a code comment, otherwise someone will certainly "fix it in passing" in the future.

### 4.6 `SSH_FXP_REALPATH`

Used to canonicalize relative paths, `~`, `.`, `..` into absolute paths.
Returns `SSH_FXP_NAME` with `count == 1`.

〔Decision〕**Immediately after the connection is established, do a `REALPATH` on `"."`**,
taking the result as the initial value of `SftpFileSystem.WorkingDirectory`.
This is the only reliable answer to "where is the user's home directory" — far more reliable than assembling `/home/{user}`.

---

## 5 Request pipeline

### 5.1 Structure

```mermaid
flowchart LR
    A[SftpFileSystem<br/>caller-facing] --> B[SftpRequestPipeline<br/>id allocation · in-flight window · cancellation]
    B --> C[RequestLedger<br/>pooled IValueTaskSource]
    B --> D[SftpWire<br/>pure encoding/decoding]
    D --> E[SSH channel PipeWriter/PipeReader]
```

`SftpWire` is a **pure function**: `bytes ↔ SftpMessage`, stateless, no I/O.
This lets it be asserted byte by byte against message samples without any peer — it is the part of the whole SFTP layer
that is easiest and most worthwhile to test thoroughly.

### 5.2 Adaptive depth and block size

〔Decision〕**Determined by the server's announced `limits@openssh.com`, not hard-coded.**

The reply to `limits@openssh.com` gives four values:

| Field | Purpose |
| --- | --- |
| `max-packet-length` | Upper limit of a single SFTP message |
| `max-read-length` | Upper limit of the length of a single `READ` |
| `max-write-length` | Upper limit of the data of a single `WRITE` |
| `max-open-handles` | Upper limit of simultaneously open handles (0 = unknown) |

Conservative defaults when this extension is absent (〔Decision〕):

| Item | Default |
| --- | --- |
| Block size | 32 KiB (the typical value of an SSH channel's max packet) |

The number of in-flight requests (the depth) does not depend on this extension; it adapts by itself per the criterion below.

**Why depth must be adaptive**: in-flight requests × block size is the SFTP layer's "window".
Just like the channel window (`05-connection.md` §3.3), a fixed value caps throughput outright on high-RTT links.
Hard-coding 64 × 32 KB = 2 MiB likewise caps at 10 MB/s with a 200 ms RTT.

**How the depth adapts**: 〔Decision〕**watch whether requests hit the limit; do not estimate the bandwidth-delay product from RTT.**

| Item | Rule |
| --- | --- |
| Start | `SftpOptions.MaxInFlight` (default 64), which is also the floor when shrinking |
| Evaluation | Once every 32 requests, count how many requests in that window had to **wait** for an in-flight slot |
| Grow | More than half waited → the depth is the bottleneck; double it, up to `SftpOptions.MaxPipelineDepth` (default 256) |
| Shrink | None waited → shrink to three quarters (not below the starting value). Slots are reclaimed **without blocking**: if they are in use, try again next window rather than stalling callers here |
| Off | `SftpOptions.AdaptivePipelineDepth = false`: the depth stays at the starting value and memory use is deterministic (in-flight count × block size), at the cost of throughput being capped at "depth × block size ÷ RTT" on high-RTT links |

Rationale: computing the depth from the bandwidth-delay product requires estimating bandwidth first, and that estimate is very unstable on a link that also carries other traffic;
"did requests have to wait for a slot" is a more direct signal that is harder to get wrong — the same idea as the channel window growing when it "runs dry" (`05-connection.md` §3.3).

**Choosing the block size**: 〔Decision〕start from the caller's `SftpOptions.BlockSize` (0 = not specified),
take the minimum of it and the caps below, then clamp to [1, 256 KiB]:

| Cap | Why it counts |
| --- | --- |
| `max-write-length` | An over-long `WRITE` is rejected — OpenSSH drops the SFTP session outright on an over-long message, along with every other in-flight request |
| `max-read-length` | An over-long `READ` is truncated, and sequential reads treat a short read as "a hole in between" (§5.5): every block discards the queue and read-ahead never builds up |
| `max-packet-length` minus 1 KiB | Besides the data, the request header (length, type, id, a handle of up to 256 bytes, offset) must fit; 1 KiB is ample |
| 256 KiB | The largest message we accept (256 KiB + 1024 in §2): one `DATA` reply plus protocol header must fit |

- A **0 from the server means "not stated", not "one byte"**, and takes no part in the minimum. Taken literally, the block size would become 1 —
  a 1 MB file would need a million requests.
- When the caller does not specify: if the server gave caps, take them (e.g. an announcement of 256 KiB packets and 255 KiB reads/writes yields a 255 KiB block);
  if all three are 0, use 32 KiB.
- A block size the caller specifies **obeys the same caps**. When the extension is not announced, or announced but the query fails, the read, write and packet caps
  are the conservative defaults in the table above (32 KiB) — so in that case a larger requested block still yields 32 KiB.

**At most one block per request.** Writes: `WriteAsync` and `WriteAtAsync` both split data longer than a block into several block-sized `WRITE`s.
Reads: each `READ` asks for at most one block; when the buffer given to `ReadAtAsync` is larger than a block, it reads one block and returns fewer bytes than the buffer holds —
a short read that §4.3 allows; callers loop anyway, so the semantics are unchanged.

### 5.3 Cancellation

The cancellation semantics of `RequestLedger` **must** be explicit:

- Cancelling an in-flight request does **not** make the server stop — SFTP has no cancel message.
  We simply stop waiting for its reply.
- Therefore, after cancellation, the reply for that id **must** still be consumed and discarded,
  and **the case of "the reply carries a handle that needs closing" must be handled** —
  if `OPEN` is cancelled but the server has already opened the file, that handle leaks on the server unless closed.
  〔Decision〕On cancellation, the ledger retains a "cleanup callback" and executes it (sending `CLOSE`) when the late reply arrives.
- **A request that never went out is withdrawn on the spot.** A request cancelled while queued on the send lock, or whose message failed to encode, will never get a reply for its id;
  yet the in-flight slot it holds is returned only by a reply — left in the ledger, it is never returned, and after a few cancellations every SFTP operation hangs.
  〔Decision〕Such a request is removed from the ledger and its slot returned immediately; only requests that already went out take the "keep it for the late reply" path above.
- **The line for "went out" must be clear.** The message is written straight into the channel's pipe (no intermediate buffer), and it counts as sent once the whole frame is written;
  after that, the caller's cancellation only interrupts the wait on backpressure and never leaves half a frame. 〔Decision〕The pipe write does not carry the caller's token —
  with it, the same cancellation exception could mean either "not a single byte written" or "already committed and will go out anyway",
  and the two cases require opposite handling of the in-flight slot.

### 5.4 Wrap-up when the channel closes

Once the pipeline stops, all in-flight requests in the ledger are completed **at once** with the same exception — the first fault cause;
requests sent afterwards throw that exception immediately too. This is a direct benefit of the unified L6 ledger: this logic is written only once and proven only once.

| Why it stopped | Exception the in-flight requests receive |
| --- | --- |
| The channel ended normally (the peer sent EOF / CLOSE, e.g. sftp-server exited; or the channel was closed locally) | `SftpTransferInterruptedException`. The pipeline layer does not know how far writes got, so its `DurableLength` is 0; a file stream replaces it in `FlushAsync` / on close with one carrying that stream's exact `DurableLength` (§6.2), keeping the cause as the inner exception |
| The connection dropped | The connection fault handed out by the channel's reader, as is (`ObjectDisposedException` when the connection was disposed locally) |
| A malformed frame arrived (length over the limit, too short for a `request-id`, §9) | `SshProtocolException` (`ProtocolError`) |
| `SftpFileSystem` (and with it the pipeline) disposed locally | `SftpUnavailableException` |

Callers that have no id yet — those waiting on the in-flight limit or the send lock — must be **released too**: they receive the fault from the table above,
or, in a race during disposal, `ObjectDisposedException`. In-flight slots are returned only by replies, and no reply comes after the pipeline stops; without the release,
waiters such as the batch of links resolved concurrently while listing a directory (§8) would wait forever.

### 5.5 Read-ahead for sequential reads

**Problem**: if a file stream's sequential read (`ReadAsync`) sends one `READ` at a time and waits for it before sending the next,
throughput is pinned at "block size ÷ RTT" — on a 100 ms link a 32 KiB block gives only about 320 KB/s,
and none of the depth the pipelined write side uses comes into play. Downloads take exactly this path.

〔Decision〕**Sequential reads keep a read-ahead window: in-flight `READ`s are queued in offset order and handed to the reader one by one.**

- **Sequential reads only.** `ReadAtAsync` (random read at an offset) still sends one request at a time.
  When the read position changes (`Seek` / `Position`), or the same handle is written to or resized, the whole queue is discarded and restarts from the new position.
- **Slow start**: the window starts at 1 request and doubles each time a full block is handed out, up to the in-flight limit (`SftpOptions.MaxInFlight`).
  Callers that read only the first few bytes of a file don't trigger a burst of needless requests.
- **No read-ahead past the known length**: requests are sent ahead only for offsets within the known length; beyond it, at most one request —
  to reach `EOF`, or to notice that the file grew after it was opened. The known length comes from `FSTAT` at open time and is pushed forward by the data read.
- **Short read** (the server returns fewer bytes than requested, and it is not `EOF`): hand out those bytes and discard the rest of the queue —
  there is a hole between those requests' offsets and the read position, and restarting from the read position is the simplest and least error-prone option (the read-in-a-loop semantics of §4.3 are unchanged).
- **Discarded requests must not be abandoned**: their replies still arrive, and are released when they do (the payload is rented from a pool).
- **Cancellation cancels only that wait**: read-ahead requests belong to the stream, not to a single `ReadAsync`; when the caller cancels one read, the queue stays as it is and the next read continues from it.
- On `EOF` or an error status: the whole queue is discarded; `EOF` returns 0, errors are thrown as usual.

---

## 6 Write watermark — eliminating the "blind 2 MB rollback"

### 6.1 The problem

When the pipeline is fully loaded there are N in-flight `WRITE`s, and **reply order is not guaranteed**.
If the transfer is interrupted midway:

```
offset 0 ....... 1 MiB ....... 2 MiB ....... 3 MiB
acked:    [==========]        [====]        [====]
                      ↑ hole          ↑ hole
file length = 3 MiB  ← but the 1–2 MiB range was never actually written
```

The **file length** reported by the server only represents "the **highest** acknowledged offset";
it does not mean every byte before it has been persisted. There may be holes in between that read as 0.

The existing approach is to **blindly roll back a full in-flight window from the file length** (e.g. 64 × 32 KB = 2 MiB),
so that the data before the resume point can be trusted. The cost is retransmitting 2 MiB on every resume,
and the number is a "guess" — change the implementation or the configuration and it is wrong.

### 6.2 Solution: contiguous acknowledged offset

The pipeline maintains, for each write handle:

| Field | Meaning |
| --- | --- |
| `_ackedRanges` | Set of acknowledged ranges (ordered, adjacent ranges merged) |
| `DurableLength` | The highest offset **contiguously acknowledged from 0** |

```
On each OK reply to a WRITE:
    merge [offset, offset+len) into _ackedRanges (merging adjacent ranges)
    DurableLength = position of the first hole from 0 (i.e. the right end of the first range, if it starts at 0)
```

`SftpFileStream.DurableLength` exposes it. Resumable transfer continues from this number — **exact, no guessing** (see §6.6 for how to open a resume).

〔Implementation note〕The range set must use an **ordered array + binary insertion** rather than a dictionary —
with sequential writes a new range almost always attaches to the tail of the last one,
so after merging the set size is constantly 1, at O(1) cost. The set only grows with out-of-order writes,
and its upper bound is the number of in-flight requests (N ≤ 256) — fully under control.

〔Decision〕**When the channel disconnects, put `DurableLength` into the exception**
(`SftpTransferInterruptedException.DurableLength`),
so upper layers need not stat again, much less roll back blindly.

### 6.3 Another route to ordering guarantees

〔Decision〕Also provide `SftpWriteMode.Sequential`:
the number of in-flight requests is fixed at 1, sacrificing throughput in exchange for "the file length is the trustworthy length".
Used for scenarios that must guarantee the file is a complete prefix at any moment (e.g. writing configuration files).

---

### 6.4 File streams are async-only

〔Decision〕(`architecture.md` principle 1) The file stream derives from `Stream`, and `Stream` comes with synchronous
`Read` / `Write` / `SetLength` / `Flush` / `Dispose`. Synchronous versions can only be implemented by "blocking a thread waiting for a network round trip" —
on the UI thread that freezes the interface for one RTT, and on the thread pool, with enough concurrency, it means starvation. Therefore:

| Synchronous member | Behavior |
| --- | --- |
| `Read` / `Write` / `SetLength` (and single-byte, `Span` overloads) | Throw `NotSupportedException`, with the message naming the async version to use |
| `Flush` | **Non-blocking no-op** (there is no local buffer); known write failures are still thrown. Use `FlushAsync` to confirm persistence |
| `Dispose` | **Non-blocking**: wrap-up (waiting for in-flight write acknowledgements, closing the handle) is handed to the background and it returns immediately; wrap-up errors (write interruption, `CLOSE` failure, §6.5) are not visible — to see them, use `await using` |

`Flush` is kept as a non-throwing no-op because wrapper streams (such as `StreamWriter`) call it synchronously in their own wrap-up;
making it throw would turn a harmless call into a failure.

The async **array overloads** (the `byte[], int, int, CancellationToken` versions of `ReadAsync` / `WriteAsync`) and the old-style
`BeginRead` / `EndRead` / `BeginWrite` / `EndWrite` **work as usual**, all forwarding to the async implementation.
〔Decision〕They must be overridden explicitly: `Stream`'s default implementation routes them to the synchronous `Read` / `Write`,
so without the override a caller who clearly used an asynchronous form would get the "async only" `NotSupportedException`.

### 6.5 Closing: the `CLOSE` reply matters

The wrap-up order of `DisposeAsync` (`await using`):

1. Discard read-ahead still in flight (§5.5); its replies are released when they arrive;
2. Wait for all in-flight writes to be acknowledged (same as `FlushAsync`);
3. **Send `CLOSE` whether or not the previous step succeeded** — a handle must be closed even after a failed write, or it leaks on the server;
4. Decide what to report per the table below.

| Situation | Behavior |
| --- | --- |
| An in-flight write failed | Throw `SftpTransferInterruptedException` (carrying `DurableLength`). `CLOSE` is still sent, but its status is not examined — what happened first is the root cause |
| All writes acknowledged, and `CLOSE` on a **writable stream** returns an error status | Throw `SftpException` (code and the server's text as in §3.3) |
| `CLOSE` on a read-only stream returns an error status | Not reported |
| `CLOSE` cannot be sent or gets no reply (channel gone, pipeline broken) | Not reported |
| Closing an already closed stream (including after a synchronous `Dispose`) | No-op, does not throw |

〔Decision〕**A writable stream examines the status of `CLOSE`.** Some servers (NFS deferred writes, quotas) report a write failure only at close;
swallowing it makes the caller believe the file was written completely. "Writable" is by how the stream was opened, whether or not this session wrote anything
(and not by `CanWrite` — that is already false after close, see below).
Failing to close a read-only stream does not matter; if the channel is already gone the server reclaims the handle itself, and the real cause has already been reported by another path —
throwing again on the dispose path would only hide it. A synchronous `Dispose` hands all of this to the background (§6.4) and **cannot see a `CLOSE` failure** — write files with `await using`.

How the stream's members behave after close:

| Member | After close |
| --- | --- |
| Those that need the handle: reads, writes (including the offset-based `ReadAtAsync` / `WriteAtAsync`, the array overloads and `BeginRead` / `BeginWrite`), `Seek`, `SetLengthAsync`, `GetAttributesAsync`, `FsyncAsync` | Throw `ObjectDisposedException` and send nothing |
| `CanRead` / `CanWrite` | Become false (the `Stream` contract: a disposed stream can be neither read nor written) |
| `Position` / `Length` / `DurableLength` | **Stay readable** |

〔Decision〕**Everything that needs the handle is refused.** The reason is concrete: an OpenSSH handle is an index into a server-side table,
and once closed it is given to the next file opened — writing again with the old handle writes into someone else's file.
〔Decision〕**Position, length and `DurableLength` stay readable**: they do not touch the handle, and after a failed close
the caller needs exactly them to decide where to resume (§6.6) — making them throw too would lock the information a resume needs inside the disposed stream.

### 6.6 Resuming: `OpenAppendAsync`

`OpenAppendAsync(path, offset)` opens the file (creating it if missing) **without truncating**, puts the write position at `offset`, and returns a write-only stream.
Combined with `DurableLength` (the stream's, or the one in `SftpTransferInterruptedException`) this is exact resumable transfer.
The stream's `Length` is the real length from `FSTAT` at open time (§4.1), so `Seek(0, SeekOrigin.End)` lands at the end of the file.

- 〔Decision〕**Use `WRITE | CREAT`, not the `APPEND` pflag.** The resume point is `DurableLength`, which may be **smaller** than the file length —
  the part after it contains holes (§6.1) and must be **overwritten** from the resume point. `APPEND` makes the server ignore offsets (§4.1),
  so the data would land at the end of the file with the holes left in place; nor would we know where each block landed, so the range bookkeeping of §6.2 would be impossible.
- 〔Decision〕**The returned stream counts `[0, offset)` as acknowledged**, so `DurableLength` starts there rather than at 0.
  Otherwise, if the resumed transfer is interrupted again, `DurableLength` reports 0 and the next resume starts over from scratch.
  - The file length is known at open time and `offset` exceeds it: count only up to the end of the file — `[file length, offset)` is a hole, not acknowledged data.
  - The length is unknown (`FSTAT` refused, §4.1): count up to `offset`.
- **The caller vouches for this premise**: `offset` should be the `DurableLength` reported by the previous transfer. Passing the server-reported file length as `offset`
  counts the holes of §6.1 as "acknowledged" too — exactly the guess the watermark exists to eliminate.
- A negative `offset` is rejected before `OPEN` is sent (`ArgumentOutOfRangeException`), leaving no handle that nobody closes (§4.1).

## 7 Extensions

### 7.1 Extensions we use

| Extension | Purpose | Behavior when absent |
| --- | --- | --- |
| `posix-rename@openssh.com` | **Atomic** rename (overwriting the target) | Degrade to `SSH_FXP_RENAME`, **and report this fact** |
| `hardlink@openssh.com` | Create hard link | Throw `Unsupported` |
| `fsync@openssh.com` | Force persistence to disk | Throw `Unsupported` |
| `statvfs@openssh.com` | File system usage | Throw `Unsupported` |
| `limits@openssh.com` | §5.2 | Use conservative defaults |
| `copy-data` | Copy **within the server**, without going over the network | Degrade to "download then upload" |
| `home-directory` | Get a given user's home directory | Use `REALPATH "."` |
| `expand-path@openssh.com` | Expand `~` | Use `REALPATH` |

### 7.2 Capability query is a public API

```
SftpCapabilities Capabilities { get; }
  bool HasPosixRename / HasHardlink / HasFsync / HasStatVfs / HasCopyData ...
  IReadOnlyDictionary<string,string> RawExtensions { get; }
```

〔Decision〕**Capabilities must be queryable, not merely degraded internally.**

The reason is concrete: the semantics of `posix-rename` and plain `rename` **differ** —
the former overwrites atomically, the latter fails when the target exists (and some servers even reject cross-directory moves).
If the library degrades silently internally, upper layers have no way of knowing which semantics they got,
nor can they warn in the UI that "this server does not support atomic overwrite; the move operation may not be atomic".

### 7.3 Shape of `SSH_FXP_EXTENDED`

| # | Type | Field |
| :-: | --- | --- |
| 4 | `string` | Extension name |
| 5+ | Extension-specific | |

`ISftpExtension` is a public extension point (architecture §8 item 10),
letting callers add vendor-private extensions without modifying the library.

---

## 8 Symbolic link conventions

〔Decision〕**Both directory listing and stat use `LSTAT` (not following); link entries additionally get a following `STAT` and a `READLINK`.**

Resulting entry:

| Field | Meaning |
| --- | --- |
| `IsSymbolicLink` | Whether **the link itself** is a link |
| `LinkTarget` | Raw text of `READLINK` (may be a relative path) |
| Other fields (`Length` / `IsDirectory` / times) | Describe **the object the link points to** |
| Broken link (following `STAT` fails) | Keep the link's own attributes, `IsDirectory = false`, **not `null`** |

Rationale:
- Using the following `STAT` directly would make the fact "this is a link" disappear entirely,
  so deleting a link that points to a directory would turn into recursively deleting the contents of the target directory — that is a data incident.
- Returning `null` for a broken link is also wrong: the link itself exists, and deleting it must not first report "not found".

〔Performance〕The supplementary requests for link entries **must be issued concurrently** (they are pipelined on the same channel).
For a directory like `/usr/lib` with hundreds of `.so` links, filling them in serially takes hundreds of round trips; concurrently it is just one round.

〔Decision〕**A failing supplementary request affects only its own entry.** If the server refuses, or that one reply is malformed (e.g. `READLINK` returns two entries):
`LinkTarget` is `null`, and a failed following `STAT` is treated as a broken link — one odd link must not make the whole directory unlistable.
**A broken pipeline** (channel disconnected, malformed frame received) is not covered by this and is thrown as usual: that is not this entry's problem.

---

## 9 Edge cases and errors quick reference

| Situation | Handling |
| --- | --- |
| `length` exceeds the upper limit | Close the channel, `ProtocolError` |
| handle exceeds 256 bytes | `ProtocolError` (the limit specified by draft-02) |
| Reply received for an unknown `request-id` | 〔Decision〕Ignore + debug log (may be a late reply to a cancelled request, §5.3) |
| Unknown message type received | `ProtocolError` (the SFTP layer has no `UNIMPLEMENTED` mechanism) |
| Server version > 3 | Downgrade to 3 and continue |
| Server version < 3 | 〔Decision〕**Refuse**, throw `SftpUnsupportedVersion`. v0–v2 differ too much to be worth supporting |
| `READ` returns more data than the requested length | `ProtocolError` |
| `count` returned by `READDIR` does not match the actual number of entries | `ProtocolError` |
| Malformed reply payload (truncated field, length out of bounds, etc.) | 〔Decision〕Throw the public `SshProtocolException` (`ProtocolError`); **the internal parsing exception must not leak** — it is an internal type that callers cannot catch by type. Affects only that one call: parsing happens on the caller's path, and the pipeline remains usable |
| A reply too short to hold even the `request-id` | Pipeline fault; all in-flight requests fail together (§5.4) |
| Path contains `\0` | Rejected locally (`ArgumentException`), not sent to the server |
| In-flight request count reaches the limit | Wait (backpressure), no error |
| The file size seen by `ReadAllBytesAsync` | 〔Decision〕Used only to estimate the initial capacity, **capped at 1 MiB** — it is a number from the peer, and a small file that claims 2 GiB must not make us allocate 2 GiB up front. With more actual data the buffer grows as reads arrive; there is no separate total cap, but the whole file must fit in one array, so large files should use the stream. Reading goes through sequential reads and gets §5.5's read-ahead |
