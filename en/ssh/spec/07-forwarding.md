# 07 · Forwarding and Tunnels

> Normative basis: RFC 4254 §7 (TCP/IP Port Forwarding);
> OpenSSH `PROTOCOL`'s `*-streamlocal@openssh.com` and `auth-agent-req@openssh.com`;
> RFC 1928 (SOCKS5, used for dynamic forwarding).
>
> Implementation: `Forwarding/` (L8).
>
> 中文：[`../../../zh/ssh/spec/07-forwarding.md`](../../../zh/ssh/spec/07-forwarding.md)

---

## 1. Four kinds of forwarding

| Form | OpenSSH flag | Who listens | What goes outbound |
| --- | :-: | --- | --- |
| **Local forwarding** | `-L` | Us (local machine) | A `direct-tcpip` channel |
| **Dynamic forwarding** | `-D` | Us (local machine, running SOCKS5) | A `direct-tcpip` channel, target given by the SOCKS handshake |
| **Remote forwarding** | `-R` | The server | The server opens a `forwarded-tcpip` channel; we connect to the local target |
| **Remote dynamic forwarding** | `-R [bind:]port` (no target) | The server | The server opens a `forwarded-tcpip` channel carrying SOCKS5; we connect to the target it asks for, subject to an allowlist (§4.6) |
| **Direct tunnel** | None (`-W` is close) | Nobody listens | `direct-tcpip` / `direct-streamlocal`; the stream is handed straight to the caller |

**The fourth is the easiest to overlook, yet it is often the one you should be using.**
When connecting to `/var/run/docker.sock` or some internal HTTP API, the local machine does **not need** a listening port —
opening one actually means any process on the same machine can connect to it. For a root-equivalent endpoint, this is not an optimization but a prerequisite.

---

## 2. Local forwarding `-L`

```mermaid
sequenceDiagram
    participant App as Local application
    participant F as PortForwarder
    participant S as SSH server
    participant T as Remote target

    Note over F: Listening on 127.0.0.1:8080
    App->>F: TCP connect
    F->>S: CHANNEL_OPEN "direct-tcpip"<br/>host=10.0.0.9 port=80<br/>orig=127.0.0.1:54321
    alt Server allows
        S->>T: connect 10.0.0.9:80
        S->>F: CHANNEL_OPEN_CONFIRMATION
        loop Bidirectional copy, half-close per direction
            App->>F: bytes
            F->>S: CHANNEL_DATA
            S->>F: CHANNEL_DATA
            F->>App: bytes
        end
    else Rejected (AllowTcpForwarding no / target unreachable)
        S->>F: CHANNEL_OPEN_FAILURE(reason)
        F->>App: TCP reset
    end
```

〔Decision〕When the server refuses to open the tunnel (or has not answered within the channel-open time limit of §3.2), that local connection is **reset** (RST) and `Error` (`ChannelOpen`) is raised:
a refusal is an error too, the same rule as §2.2's "an error is not an EOF". 〔History〕The connection used to be closed **normally** (FIN), so the local application read an end with no data at all
and could not tell "the server refused" from "the target sent nothing". Local forwarding to a remote Unix socket is handled the same way; when listening on a local Unix socket, which may not support linger 0, the socket is simply closed.
(Dynamic forwarding also sends a SOCKS failure reply that tells the client why, then closes normally, §3.2.)

### 2.1 Extra fields of `direct-tcpip`

| # | Type | Field | Notes |
| :-: | --- | --- | --- |
| 6 | `string` | `host to connect` | **Resolved from the server's point of view** — `localhost` means the server's loopback |
| 7 | `uint32` | `port to connect` | |
| 8 | `string` | `originator IP` | Local originator address |
| 9 | `uint32` | `originator port` | |

〔Decision〕**The originator is filled in truthfully** (the real local address and port).
The server writes it to its logs; a fake value would leave the server administrator unable to trace anything, with no privacy benefit whatsoever
(the server already knows where our connection comes from).

### 2.2 Half-close and error teardown

Both TCP and SSH channels support half-close, and they **must be mapped per direction**; and **a clean end must be kept apart from an error**.
The two sides of a forwarded connection are the local socket and the SSH channel; what happens on one side reaches the other as follows:

| What happens on one side | What the other side sees |
| --- | --- |
| Local application does `shutdown(SEND)` (we read a FIN) | `CHANNEL_EOF`; the other direction keeps flowing |
| `CHANNEL_EOF` received | `shutdown(SEND)` on the local socket (FIN); the other direction keeps flowing |
| Both directions end cleanly | Channel `CHANNEL_CLOSE`, local socket closed |
| The peer's `CHANNEL_CLOSE` received | Data already received is still delivered, then the local socket gets a FIN; the direction writing into the channel stops there, and the local socket is closed afterwards (the connection layer replies `CLOSE`) |
| Read or write error on the local socket (reset, etc.) | The channel gets **no EOF**, just `CHANNEL_CLOSE` |
| Read or write error on the channel (including the SSH connection dropping) | The local socket is **reset** (RST) |
| Cancellation from disposing the forwarder or the SSH connection dropping | Treated as an error: both sides are aborted together |

**Treating EOF as "connection over" truncates data.** Typical symptom:
`curl` POSTs a request body through the tunnel and waits for the response, but we close the whole channel
when it shuts down its write side, so the response never arrives.

〔Decision〕**An error is not an EOF.** When either direction fails (cancellation included), both sides are **aborted** together: the TCP side is closed with linger 0, so the far end receives an RST
(Unix sockets may not support linger 0; then it is simply closed); the channel side gets `CHANNEL_CLOSE` without a preceding `EOF`.
Closing as if it had ended cleanly (FIN / `EOF`) is wrong — the far end would take truncated data for complete data, and a half-downloaded file would look finished;
stopping only the failed direction is wrong too — the other direction would hang until the far end happens to close.
The reported cause is taken from **the side that failed first**, not from the side that was aborted as a consequence (that is usually just a cancellation).

〔Decision〕**Once the peer's `CLOSE` arrives, the direction writing into the channel stops, and that is not an error.** That direction is most likely blocked reading the local socket at that point —
the local program is waiting for a response and will not close first; if it is not stopped, the socket and a concurrency slot stay occupied.
The direction reading from the channel is unaffected; data already received is drained as usual. Whatever the local end sends afterwards, the OS answers with an RST once the socket is closed.

### 2.3 Bind address

| `BindAddress` | Behavior |
| --- | --- |
| Not given (〔Decision〕**default**) / `localhost` | **Both loopbacks**: `127.0.0.1` and `::1`, on the same port; only the local machine can connect |
| `127.0.0.1` | IPv4 loopback only |
| `0.0.0.0` / `::` | Reachable from the LAN. **Must be specified explicitly by the user** |
| A specific interface address | Listens only on that interface |

〔Decision〕**Bind to loopback by default.** The other end of a tunnel is often an internal database or management interface;
binding to `0.0.0.0` by default would expose it to everyone on the same network segment.
OpenSSH defaults to this as well (`GatewayPorts no`).

〔Decision〕**Listen on both loopbacks** (Q5): many runtimes resolve `localhost` to `::1` first (Node 17 onward does), so with only `127.0.0.1` they fail to connect or try `::1` first;
and with `[::1]:same port` left empty, any process on the machine can grab it, and a client trying `::1` first hands it things like the database password —— exactly the same-machine exposure that "bind to loopback by default" is meant to prevent.
Without an IPv6 loopback only `127.0.0.1` is used; when that port on `[::1]` is already taken by another process **the forward is not started** (`ForwardBindFailed`, with the reason in the message),
and with port 0 another port is tried. 〔History〕Only `127.0.0.1` used to be the default.

〔Decision〕**Port 0 means the OS assigns one**; the assigned endpoint is reported back via `LocalPortForwarder.BoundEndPoint` (the IPv4 one) and `BoundEndPoints` (all of them).

### 2.4 Listener resilience and teardown

Local and dynamic forwarding share the same listener (`PortForwarder`); the following holds for both.

〔Decision〕**Back off when accepting fails.** When accepting an inbound connection fails, raise one `Error` (`ForwardErrorReason.Accept`), then wait before accepting again:
50 ms the first time, doubling each time after that, capped at 1 second; one successful accept resets it.
Failures such as file-descriptor exhaustion (EMFILE) come back immediately and repeatedly — without a wait this is a loop that pins a core and raises tens of thousands of error events per second,
and it happens exactly when the machine is already under pressure.

〔Decision〕**When the SSH connection drops, close the listener and release the port.** Otherwise the port stays taken: re-creating the same forward after reconnecting only yields "port already in use",
and meanwhile every accepted connection just earns another "tunnel could not be opened". The listener is closed first and `IsActive` becomes false after that —
whoever sees it false can be sure the port has been released. In-flight connections are aborted as errors (§2.2); local applications receive an RST.
The forwarder object must still be disposed by the caller.

〔Decision〕**Disposal returns only after every connection has finished its teardown.** The forwarder tracks each connection in progress; disposal stops the listener,
cancels all connections (aborted as errors), waits for them to finish their teardown, and only then releases the concurrency slots and other resources.
It used to release them right away: a connection still tearing down would then return a slot to an already-disposed limiter, with the exception landing in a task nobody observed; and the connections were still open when disposal returned.

### 2.5 Unix domain sockets

> Basis: `direct-streamlocal@openssh.com` in OpenSSH `PROTOCOL`; ssh_config(5) `StreamLocalBindMask` / `StreamLocalBindUnlink`.

Two directions, which can be combined:

| Form | API | Local listener | Outbound |
| --- | --- | --- | --- |
| `-L 8080:/var/run/docker.sock` | `LocalPortForwarder.StartToUnixSocket(connection, remote path)` | TCP | `direct-streamlocal@openssh.com` (only `socket_path ‖ reserved`) |
| `-L /path/local.sock:host:port` | any start method + `LocalPortForwardOptions.ListenSocketPath` | a local socket file | unchanged (a TCP target, a remote socket, or SOCKS) |

Use: local Docker clients and database tools work directly against the remote `docker.sock` / database socket, and the remote side opens no TCP port at all;
when the local end listens on a socket file, **file permissions isolate it** —— on a multi-user machine other users cannot borrow your tunnel (a loopback port is open to every user).

〔Decision〕**Permissions of the local socket file**: on non-Windows systems it is set to `0600` after the listener starts (the same as OpenSSH's default `StreamLocalBindMask 0177`).
There is a short window between creation and the permission change; putting the socket in a directory only you can enter (such as `$XDG_RUNTIME_DIR`) removes it ——
the process-wide umask cannot be changed safely in a multithreaded process, so it is not used to close the gap. On Windows the socket file inherits the directory's ACL (keep it under the user profile).

〔Decision〕**If a file already exists at the path, fail by default and leave it alone** (`ForwardBindFailed`): it is most likely left over from last time, but it may be someone else's socket.
Only `AllowSocketReplacement = true` (like `StreamLocalBindUnlink yes`) deletes it before listening.

〔Decision〕**A path that is too long fails immediately** (108 bytes on Linux / Windows, 104 on macOS, both including the terminating 0).

〔Decision〕**On teardown, delete the socket file we created**: on disposal and when the SSH connection drops; not when binding itself failed —— that file is not ours.

Whether a tunnel to a remote socket can be opened (the server needs `AllowStreamLocalForwarding`) is only known when the first connection arrives, just as with TCP targets;
failure raises `Error` (`ChannelOpen`). 〔Verified〕Against a real OpenSSH: a socket file on Windows → the server's `ssh-agent` socket; a key added through it shows up in `ssh-add -l`.

---

## 3. Dynamic forwarding `-D` (SOCKS5)

The only difference from local forwarding: the target address is not fixed in configuration but is **given by the client in the SOCKS handshake**.

### 3.1 The SOCKS subset we implement

> Basis: RFC 1928.

| Item | Supported |
| --- | --- |
| Version | **SOCKS5 only** (0x05). SOCKS4/4a 〔Decision〕 not implemented |
| Authentication methods | Only `0x00` (no authentication) accepted. 〔Decision〕 see below |
| Commands | Only `CONNECT` (0x01). `BIND` / `UDP ASSOCIATE` get `0x07` (command not supported) |
| Address types | IPv4 (0x01), domain name (0x03), IPv6 (0x04) all supported |

〔Decision〕**SOCKS authentication is not implemented.**
Reason: this listener is on loopback by default (§2.3), and processes on the same machine can connect anyway;
adding a username/password layer gives the illusion that it is a security boundary, which it is not.
If you really need to restrict access, use OS mechanisms (firewall, namespaces).

〔Decision〕**Domain names are not resolved locally; they are passed to the server as-is.**
This is the single most important semantic of dynamic forwarding — `curl --socks5-hostname` depends on it.
Local resolution would cause a "DNS goes local, connection goes through the tunnel" split,
which simply fails for internal domain names and also leaks the destination.
〔Decision〕The domain name is decoded as UTF-8, and a non-ASCII name is converted to Punycode before it is passed on (as the SOCKS5 dialer does); invalid UTF-8, or a name that cannot be converted, gets `0x08`.
〔History〕Early versions decoded it as ASCII, turning every non-ASCII character into `?` and connecting to a host that does not exist.

〔Decision〕**A zero-length domain name gets `0x08` (address type not supported), and that connection is closed.** An empty name is not a target:
let through, opening the tunnel fails on an invalid argument and the client gets no SOCKS reply at all, with no way to tell what went wrong.
〔Decision〕**A target port of 0 gets `0x01` (general failure), and that connection is likewise closed during the handshake**, recorded as a SOCKS handshake error.
〔History〕Early versions let it through: opening the tunnel threw on the invalid argument, the client got no reply at all, and the error was recorded as a relay error.

### 3.2 Reply code mapping

| SSH `CHANNEL_OPEN_FAILURE` reason | SOCKS5 REP |
| :-: | :-: |
| 1 `ADMINISTRATIVELY_PROHIBITED` | 0x02 connection not allowed |
| 2 `CONNECT_FAILED` | 0x05 connection refused |
| 3 `UNKNOWN_CHANNEL_TYPE` | 0x01 general failure |
| 4 `RESOURCE_SHORTAGE` | 0x01 general failure |
| Channel open timed out (`ChannelOpenTimeout`) | 0x06 TTL expired |

Getting the mapping right has real consequences: `curl` and browsers decide from the REP code whether to retry
and which message to show the user. Replying `0x01` for everything throws that information away.

〔Decision〕**Opening a channel has a time limit of its own** (`LocalPortForwardOptions.ChannelOpenTimeout`, 30 seconds by default, `Timeout.InfiniteTimeSpan` for none),
shared by local and dynamic forwarding: the server confirms only after it has connected to the target, and for an unreachable target it answers only when its own TCP connect times out (commonly around two minutes) —
browsers and curl cannot wait that long, and would not see why. When the time is up this connection is given up: dynamic forwarding replies `0x06`, local forwarding resets the local connection (§2), and both raise `Error` (`ChannelOpen`).
A late confirmation is cleaned up by the connection and closed at once (the `CHANNEL_OPEN` has already gone out, so its answer runs its course instead of being dropped from the ledger), so it does not hold one of the server's session slots.
〔History〕There used to be no such limit, so the last row never applied: the forwarder waited for the server's answer.
If the forwarder is disposed or the SSH connection drops while one is still waiting, that connection is closed without any reply.
A failure without a reason code (for example our own channel count or window budget being exhausted) gets `0x01`.

### 3.3 Handshake time limit

〔Decision〕**The SOCKS handshake has a time limit** (`LocalPortForwardOptions.SocksHandshakeTimeout`, 30 seconds by default):
from accepting the connection until the `CONNECT` request has been read. On timeout that connection is closed and `Error` (`ForwardErrorReason.SocksHandshake`) is raised; the forwarder keeps running.
Every client that connects and says nothing holds a concurrency slot for nothing; without a limit, once they fill the cap (§8) no legitimate connection gets in.
Browsers and `curl` send the handshake as soon as they connect; 30 seconds is plenty.

---

## 4. Remote forwarding `-R`

### 4.1 Setup

```mermaid
sequenceDiagram
    participant F as RemotePortForwarder
    participant S as SSH server
    participant R as Remote client

    Note over F: Register the forwarded-tcpip handler first
    F->>S: GLOBAL_REQUEST "tcpip-forward"<br/>bind_addr ‖ bind_port (want_reply=true)
    alt Allowed
        S->>F: REQUEST_SUCCESS [‖ uint32 actual port]
        Note over F: Record the actual port right there on the receive loop
    else Rejected
        S->>F: REQUEST_FAILURE
        Note over F: Remove the handler and throw, **leave no half-open listener**
    end

    R->>S: Connects to the server's bind_port
    S->>F: CHANNEL_OPEN "forwarded-tcpip"<br/>bind_addr ‖ bind_port ‖ orig_addr ‖ orig_port
    F->>F: Find the matching forwarder by bind address + port
    alt Not found
        F->>S: CHANNEL_OPEN_FAILURE(1)
    else Found: connect to the local target first
        alt Connected
            F->>S: CHANNEL_OPEN_CONFIRMATION
            Note over F: Copy both ways
        else Cannot connect
            F->>S: CHANNEL_OPEN_FAILURE(2) "cannot connect to the forwarded local target"
            Note over F: Record a TargetConnect error locally
        end
    end
```

〔Decision〕**The local target is connected before the channel is confirmed**, the same order as agent forwarding (§7.1): if it cannot be reached, reply `CHANNEL_OPEN_FAILURE(2)` (connect failed),
with a description that only says it cannot connect to the forwarded local target — local addresses are not sent out. 〔History〕Early versions confirmed first and connected afterwards: when the target was unreachable the remote side saw "accepted, then closed at once",
and the server log had no connect failed.

〔Decision〕**A handler's refusal is answered with the matching reason code**: failing to reach what must be connected (the local target, the local agent) gets 2, a full connection limit gets 4 (resource shortage),
anything else gets 1. 〔History〕Early versions always replied 1.

〔Decision〕**The `Source` of the connection events is the originator in `forwarded-tcpip`** (who connected to the port exposed on the server): an `IPEndPoint` when it parses as an IP address,
otherwise a `DnsEndPoint` with the host name (not resolved); Unix socket forwarding has no such field, so it is `null`. 〔History〕Early versions discarded it, so `Source` was always `null`.

### 4.2 Three musts

1. **When `bind_port = 0`, the actual port is in the payload of `REQUEST_SUCCESS`**
   (a `uint32`). `want_reply` must be true, otherwise the port number cannot be obtained.
2. **Server-initiated `forwarded-tcpip` channels must be routed to the matching forwarder by "bind address + port".**
   A single SSH session can carry multiple remote forwards, and they all share this one channel type.
   〔Decision〕The routing table key is `(bind_addr verbatim string, bind_port)` —
   **do not** normalize the address (`""`, `"*"`, `"0.0.0.0"`, `"localhost"` have different semantics on the server,
   and what it sends back to us is the exact string we used in the request).
3. **The handler is registered before the request is sent, and the actual port is recorded the moment the reply arrives.**
   Right after replying `REQUEST_SUCCESS` the server may open a forwarded channel (someone is already waiting to connect to that port), and that `CHANNEL_OPEN`
   is processed by the receive loop immediately afterwards. Registering the handler only after the reply, or recording the port in the caller's continuation (which runs on the thread pool),
   means that channel has already been rejected as "unclaimed".
   〔Decision〕So the reply ledger lets a request be registered with a callback: when the reply arrives, the callback is invoked synchronously on the receive loop first (recording the port),
   and only then is the waiting task completed. The callback must be short and non-blocking; if the connection drops, it receives a failed reply.
   The handler is removed when the server refuses or sending the request fails.

### 4.3 Cancellation and the grace period

The `cancel-tcpip-forward` global request, with the same fields as `tcpip-forward` (the port is the one actually bound).
〔Decision〕After cancellation, **continue accepting in-flight `forwarded-tcpip` channels** for a few seconds (`DrainGrace`, 2 seconds);
otherwise connections that are being established get rejected for no apparent reason. Disposal proceeds in this order:

1. Send `cancel-tcpip-forward` (`cancel-streamlocal-forward@openssh.com` for the Unix socket variant). From this moment `IsActive` is false.
2. During the grace period the handler stays in place: matching forwarded channels are still accepted and relayed, and existing connections keep relaying.
3. When the grace period ends, the handler is removed — forwarded channels arriving after that are rejected.
4. All of this forwarder's connections are ended, aborted as errors (§2.2): the local target receives an RST, and the server's channel receives a `CLOSE` without `EOF`.

If the SSH connection is already gone (the cancel request cannot be sent, or the connection drops during the grace period), there is no waiting — no more forwarded channels will arrive.

〔Decision〕**Waiting for the cancel request's reply has a time limit (5 seconds).** On a half-dead link (keepalive off, or a long interval) the reply may never come;
without a limit, disposal would hang until TCP retransmission gives up (about 15 minutes by default on Linux), and the caller's "stop tunnel" would hang with it.
When the time is up the link is treated as unusable: the handler is removed and connections are ended as usual, without waiting out the grace period — the same idea as the time limit on channel disposal (`05-connection.md`).

〔History〕The early implementation marked itself "disposed" as soon as disposal began, and the handler rejected everything once that flag was set —
the grace period did nothing, and in-flight forwarded channels were rejected anyway.

〔Decision〕**If setup is abandoned, a listener the server grants afterwards is withdrawn.** When `tcpip-forward` (or the streamlocal variant) is already on the wire,
the caller cancels while waiting for the reply (or waiting fails), and the server then replies `REQUEST_SUCCESS`,
a cancel request with `want_reply = false` is sent (no reply is registered, so nothing hangs on a reply nobody waits for).
Which arrives first — the reply or "the caller gave up" — is settled by an atomic state, and whichever comes second sends the cancel, so even simultaneous arrival is covered.
It used to only remove the local handler: the server's listener stayed open until the connection dropped and every incoming connection was refused; an immediate retry on a fixed port always failed with "port already in use".

### 4.4 Unix socket variant

`streamlocal-forward@openssh.com` / `cancel-streamlocal-forward@openssh.com`
(global requests) + `forwarded-streamlocal@openssh.com` (channel type);
the fields replace `addr ‖ port` with a single `string socket_path`.

### 4.5 State and lifetime

| Property | Meaning |
| --- | --- |
| `BoundPort` | The port the server **actually** bound: taken from the `REQUEST_SUCCESS` payload when port 0 was requested, otherwise the requested port. 0 for the Unix socket variant |
| `IsActive` | False once disposal has begun, or once the SSH connection has dropped. It reads the connection's state directly and does not wait for any callback |

〔Decision〕If port 0 was requested and the server replies `REQUEST_SUCCESS` without a port: remove the handler and throw.
Routing forwarded channels by `(bind_addr, 0)` would match none of them, and the symptom is "the forward looks established, but every incoming connection is rejected".

〔Decision〕**Each forwarded connection is tied to the lifetimes of both the SSH connection and the forwarder**; whichever ends first ends it too (aborted as an error, §2.2).
Tied only to the connection, a forwarder's connections would keep relaying after the forwarder was disposed, until the whole SSH connection dropped.

Once the SSH connection drops, the server's listener disappears with it, and there is no local port to release; the forwarder object must still be disposed (disposal sees that the connection is gone and skips the grace period).

### 4.6 Remote dynamic forwarding and the allowlist (`PermitRemoteOpen`)

> Basis: RFC 1928 (SOCKS5); the target-less form of ssh(1) `-R` (OpenSSH 7.6 and later) and ssh_config(5) `PermitRemoteOpen`.

Use: the remote server needs to reach places only this machine can reach (an internal package mirror, an internal API) and there is no VPN —— programs on the remote side use the server's port as a SOCKS5 proxy,
and this machine connects on their behalf. On the server side it is an ordinary `tcpip-forward` (§4.1); nothing extra is required of the server.

`RemotePortForwarder.StartDynamicAsync(connection, permitRemoteOpen, options)`:

1. Ask the server to listen, as in §4.1 (with port 0 the actual port comes in the reply). The forwarder's kind is `ForwardKind.RemoteDynamic` (the `kind` tag of metrics and events follows it).
2. A forwarded channel is **confirmed first** (the target is only known after the handshake, so it cannot be connected beforehand as in §4.1); it takes a concurrency slot as usual.
3. Run the server side of SOCKS5 on the channel: the same subset as §3.1 (`CONNECT` only, no authentication), with the same handshake time limit
   (`RemotePortForwardOptions.SocksHandshakeTimeout`, 30 seconds by default).
4. A target not on the allowlist: reply `0x02` (not allowed) and raise `Error` (`ForwardErrorReason.TargetNotPermitted`).
5. On the allowlist: resolve and connect **on this machine** (names resolve locally —— reaching what this machine can reach is the whole point). A failed connect replies by cause: refused `0x05`,
   network unreachable `0x03`, host unreachable or not resolvable `0x04`, timed out `0x06`, anything else `0x01`, and raises `Error` (`TargetConnect`).
6. Once connected, reply `0x00`, then the same relay loop as every other forward (§6).

〔Decision〕**The allowlist must be given; there is no default.** This turns this machine into the remote side's SOCKS proxy: whatever internal network this machine reaches, the remote side reaches too.
What to let out must be stated by the caller; to allow everything, pass `RemoteOpenPolicy.Any` explicitly (OpenSSH's `PermitRemoteOpen any`); to allow nothing, `None`.

〔Decision〕**How rules are written and matched**: each rule is `host:port`; the host may use `*` / `?` wildcards and is case-insensitive, IPv6 goes in brackets (`[::1]:22`), the port is a number or `*`.
**Matching uses the name the remote side gave in the handshake, without resolving it first**: a rule with an IP does not match a request with a name, and vice versa —— better a wrong refusal
than letting a name that resolves to an internal address bypass the list.

〔Decision〕**A failure reply must actually go out**: after the reply, drain stdin and send `EOF` before closing the channel; **record the event before replying** ——
the remote side may come back with another request as soon as it sees the reply, and the event must not trail it.

〔Verified〕Against a real OpenSSH: OpenBSD `nc -X 5 -x 127.0.0.1:port` on the remote side reaches a service on this machine through it, in both directions; a target not on the list is refused.

---

## 5. Metering — in the library, not in the caller

> This is the most direct benefit of this library over the existing implementation.

`PortForwarder` (the common base class of `LocalPortForwarder` and `RemotePortForwarder`) exposes:

```
ForwardKind Kind { get; }
EndPoint?   BoundEndPoint { get; }      // local forwarding: with port 0, this holds the actual port (remote forwarding uses BoundPort)
bool        IsActive { get; }

int  ActiveConnections { get; }
long TotalConnections  { get; }
long BytesSent     { get; }             // local → remote
long BytesReceived { get; }             // remote → local

event EventHandler<ForwardConnectionEventArgs> ConnectionOpened;
event EventHandler<ForwardConnectionEventArgs> ConnectionClosed;   // includes that connection's byte counts and duration
event EventHandler<ForwardErrorEventArgs>      Error;              // a single connection failed; the forwarder keeps running
```

`ForwardErrorEventArgs.Reason` is the enum `ForwardErrorReason` (`Accept` / `ConnectionLimit` / `SocksHandshake` / `ChannelOpen` /
`TargetConnect` / `Relay` / `SetupSkipped` / `TargetNotPermitted`; the zero value is `Unknown`), not a string —— callers branch on it instead of having to recognize an agreed-upon piece of text.

Also published through `System.Diagnostics.Metrics`:

| Instrument | Type | Tags |
| --- | --- | --- |
| `velashell.ssh.forward.connections.active` | UpDownCounter | `kind` |
| `velashell.ssh.forward.connections.total` | Counter | `kind` |
| `velashell.ssh.forward.bytes` | Counter | `kind`, `direction` |
| `velashell.ssh.forward.errors` | Counter | `kind`, `reason` |

〔Decision〕**The tags are only the low-cardinality `kind`, `direction` and `reason`; the listen address is not a tag.** The listen address is high-cardinality: one value per forward, and a random one when port 0 is given;
as a tag, the number of series in a time-series database would grow with every forward ever opened and never be reclaimed. For the traffic and connections of one particular forward, use the forwarder's own
`Throughput` / `Connections` (§5.1). 〔History〕This table used to list a `bind` tag that the code never emitted.

〔Decision〕**A forwarder reports why it stopped** (`PortForwarder.Completion`): when the connection ended, the connection's end reason (the same one as `SshConnection.Completion`);
on local disposal, `Aborted` — whichever comes first; it completes successfully with the reason as its result. A forwarder stops only for these two things — a failure of a single connection raises `Error` and the forwarder keeps running.
〔History〕There used to be none, so the host's tunnel panel had to watch the connection's end itself to turn "running" into a status with a reason.

〔Decision〕**Provide both paths**: events for the desktop UI (which refreshes a panel in real time),
Metrics for server scenarios (feeding OpenTelemetry). Picking only one would force users to rewrite the data plane themselves —
which is exactly the 376 lines we set out to eliminate.

〔Decision〕**Exceptions from event subscribers do not affect forwarding.** Subscribers are invoked one by one; whatever one throws is swallowed, without affecting later subscribers, let alone the connection.
Events are there to refresh a panel, and a bug in a subscriber should not turn into a forwarding failure —
a throwing `ConnectionOpened` subscriber used to stop that connection from relaying, and the active-connection count only ever went up.

〔Decision〕Cancellation caused by the forwarder itself shutting down (disposal, the SSH connection dropping) **is not an error of the individual connection** and does not raise `Error`.

〔Implementation note〕Counters use `Interlocked`; reads use `Volatile.Read`.
Byte counts are accumulated **in the copy loop**, not at the channel layer — channel-layer byte counts include protocol overhead,
whereas the panel should show application data volume.

### 5.1 Live throughput, connection snapshots and rate limits

| Member | Content |
| --- | --- |
| `PortForwarder.Throughput` | `ForwardThroughput(SentPerSecond, ReceivedPerSecond)`: the average over **the last three whole seconds** (application bytes / second), excluding the second in progress |
| `PortForwarder.Connections` | A snapshot of the connections being relayed (ordered by id): `Id` (as in the connection events), `Source`, `Target`, `StartedAt`, and `BytesSent` / `BytesReceived` so far; removed when the connection ends |
| `LocalPortForwardOptions.MaxBytesPerSecond` / `RemotePortForwardOptions.MaxBytesPerSecond` | At most this many application bytes per second in each direction; `null` (default) means unlimited, and a non-positive value throws when set |

〔Decision〕**Throughput is an average over whole-second windows, not an instantaneous value.** An instantaneous value jumps with every chunk and is unreadable on a panel;
excluding the second in progress costs one to two seconds of lag.

〔Decision〕**Rate limits are per forwarder and per direction**: all connections of a forwarder share one token bucket (one per direction) ——
"how much bandwidth this tunnel may take" belongs to the forwarder, not to a single connection; otherwise opening ten connections means ten times the rate.
**A burst of one second's allowance** is allowed; after that the deficit is carried and waited off; after a pause the allowance refills, but never beyond one second's worth.

〔Decision〕**Wait in the copy loop, before writing to the other side**: while waiting nothing more is read, the data stays in the read buffer, and backpressure flows back to the sender through the TCP window / channel window ——
no extra buffer and no dropped data. Use: with several tunnels open over a slow link, one download cannot crowd out the interactive terminal.

---

## 6. The copy loop

All three forwarding kinds converge on the same loop, **deliberately**:

```
listen/accept → establish outbound (factory delegate) → bidirectional copy (with metering and half-close) → teardown
```

Only "how the outbound side is established" differs:

| Form | Outbound factory |
| --- | --- |
| Local | `(_, ct) => conn.OpenTunnelAsync(host, port, ct)` |
| Dynamic | `(inbound, ct) => { first run the SOCKS5 handshake on inbound to get the target; then OpenTunnelAsync }` |
| Remote | Inbound is the channel given by the server, outbound is local TCP — direction reversed, loop unchanged |

〔Decision〕**The copy layer is independent of SSH and testable on its own.**
It only sees two endpoints (`IRelayEndpoint`), each offering five things: read; write; "I will send no more"
(half-close — `shutdown(SEND)` for TCP, `CHANNEL_EOF` for a channel); "abort on error" (an RST for TCP, a `CLOSE` without `EOF` for a channel);
and a signal that "this end is entirely finished" (the channel received `CLOSE`, or the session is gone). So the main path of half-close, metering
and error teardown (§2.2) can be verified without standing up a real server.

An endpoint that cannot provide an abort action (a plain stream, for instance) degrades abort to closing it: the far end learns the connection is gone but cannot tell an error from a clean end.
One that cannot provide a half-close action does nothing on half-close: the far end only learns we are done sending when the whole connection closes.

**Buffers**: 〔Decision〕32 KiB per direction, rented from `ArrayPool`.
Same order of magnitude as an SSH channel's max packet, without making every connection hold a large chunk of memory
(1000 concurrent connections × 2 directions × 32 KiB = 64 MiB, acceptable).

---

## 7. Agent forwarding

> Basis: OpenSSH `PROTOCOL`'s `auth-agent-req@openssh.com`, and `PROTOCOL.agent` (including `session-bind@openssh.com`).

### 7.1 Mechanism

```mermaid
sequenceDiagram
    participant A as Local agent
    participant C as Client
    participant S as SSH server

    C->>A: Probe connection (closed as soon as it connects)
    alt Cannot connect
        Note over C: Do not send auth-agent-req; handle per FailureMode (§7.5.8)
    else Connected
        C->>S: CHANNEL_REQUEST "auth-agent-req@openssh.com" (want_reply = true)
        S->>C: CHANNEL_SUCCESS
        S->>C: CHANNEL_OPEN "auth-agent@openssh.com"
        C->>A: Connect to the local agent ‖ session binding (§7.4)
        alt Connected
            C->>S: CHANNEL_OPEN_CONFIRMATION
            Note over C,A: Parse each message, filter, then forward (§7.2)
        else Cannot connect
            C->>S: CHANNEL_OPEN_FAILURE(2) "local ssh-agent unavailable"
        end
    end
```

1. **Probe the local agent once first** (closing the connection as soon as it succeeds). If it cannot be reached, **do not send** the request; handle it per `FailureMode` (§7.5.8 — the same rule as X11).
2. Send `CHANNEL_REQUEST "auth-agent-req@openssh.com"` (`want_reply = true`) on the **session channel**.
3. The server may then open `CHANNEL_OPEN "auth-agent@openssh.com"` channels.
4. **Before confirming the channel**, connect to the local agent (Unix socket / Windows named pipe) and send it the session binding (§7.4).
5. Confirm the channel and bridge it to that local agent connection.

Handled by `IIncomingChannelHandler` (architecture §8, item 8).

〔Decision〕**If the local agent cannot be reached, forwarding is not announced.** Once announced, the remote `SSH_AUTH_SOCK` points at an agent that can never be reached:
every remote program that wants it makes a wasted trip, and the user cannot tell that the actual problem is that no agent is running locally.

〔Decision〕**The local agent is connected before the channel is confirmed; if it cannot be reached, reply `CHANNEL_OPEN_FAILURE`** (reason code 2, with the description saying only "local ssh-agent unavailable" —
local paths and pipe names are not sent out). The remote program learns on the spot that the agent is unusable. It used to confirm first and then connect in the background.
This step is not on the receive loop (the connection asks handlers in the background, 05 §8.1), so waiting on one local IPC call does no harm.
The local connection made before confirmation is taken over by the step that subsequently takes over the channel; if the channel ultimately does not open (the local channel count or window budget is exhausted), it is closed on the spot.

〔Decision〕**On Windows, the time limit for waiting for the agent's named pipe to appear is 3 seconds**; when it runs out, the failure is reported as `AgentNotRunning` (08 §3).
When the pipe does not exist, a connection attempt without a time limit **keeps retrying** until the pipe appears: the agent forwarding path used to carry only a cancellation token,
so when the agent service was not running, the remote `ssh` / `git` hung until the shell was closed. A file system existence check cannot judge a named pipe
(it reports "does not exist" even for a pipe that does exist), so a time limit is the only option. Authentication, adding keys and forwarding all share this one time limit in the library.

〔Decision〕**Not a byte-level pass-through**: each agent message (4-byte length + contents) is collected in full and decoded before it is handled —
only that way can §7.2's "forward only the specified keys" and "confirm each signature" be done.

〔Decision〕**The agent channel's receive window holds at least one whole agent message of the maximum size** (4-byte length + 256 KiB).
Nothing is consumed until a message is complete, and the window is replenished only as data is consumed — a window smaller than a message means the peer waits for window while we wait for the message, and neither can move.
It used to be 32 KiB, and signing slightly longer data (`ssh-keygen -Y sign`, certificates) stalled right there.
The window is only a credit, not memory allocated up front; ordinary messages are a few hundred bytes. A message with length 0 or over 256 KiB is treated as malformed and the channel is closed.

〔Decision〕**When an exchange of the local agent client is interrupted halfway (cancellation, a read/write error, an unreasonable length), that agent connection is retired**:
the agent protocol has no request ids, so the request may be only half written and the reply may still be on its way; every later call fails with `AgentUnavailable` and the caller reconnects
(a client connected by `ConnectAsync` reopens on its own along the session-declaration path, §7.4). 〔History〕Early versions kept using it, and the next request read the previous request's late reply.

### 7.2 Security requirements

> **Agent forwarding is a loaded gun.** Root on the remote host can, while forwarding is active,
> sign anything with your private key.

〔Decision〕Three hard constraints:

1. **Off by default**; must be explicitly enabled per connection.
2. **Must support "forward only the specified keys"** (`AgentForwardOptions.AllowedKeys`)
   instead of exposing the entire agent.
3. **An optional signature confirmation callback** (`AgentForwardOptions.ApproveSignature`):
   ask the user each time the remote requests a signature. For jump-host scenarios this is the only approach that lets people rest easy.

〔Decision〕**The allow-list compares the key inside a certificate.** A certificate and its key use the same private key: allowing the key allows its certificate,
and allowing the certificate allows the key. Listing identities and sign requests use the same comparison — a key that is not listed is refused even if the remote
has learned its public key elsewhere (for example `authorized_keys`) and asks for a signature directly; the request never reaches the local agent.

〔Decision〕**Keys in the agent that this library cannot recognize are skipped, rather than failing the whole list** (FIDO, DSA, certificate types this library does not support):
the remote does not see them in the list, and sign requests for them are refused — even when `AllowedKeys` is empty. Certificates it does recognize are listed, and their sign requests are passed to the agent as usual.
For RSA, the digest is chosen by the flags in the remote's request (SHA-256 / SHA-512; SHA-1 only when neither is set), **judged by the type of the key inside the certificate** —
an RSA certificate gets the SHA-2 the remote asked for, just like an RSA key; judged by the certificate's own type string, the flags would be ignored and the certificate signed with SHA-1.
It used to be that a single certificate in the agent made listing identities fail entirely, and agent forwarding with it.

〔Decision〕**Forwarding lasts exactly as long as the forwarder.** Disposing the forwarder not only stops accepting new `auth-agent@openssh.com` channels,
**it also closes the ones already open**. Removing the handler alone is not enough: the connection may still be open (SFTP, other sessions),
and an agent channel the remote opened earlier would keep signing for it until the whole connection drops.
After each request is read and before it is handled, the forwarder checks once more whether it has been disposed: the cancellation issued by disposal reaches each channel in the background,
and a request that arrives in between must not get one more signature after disposal has returned.

〔Decision〕**On Windows, before connecting to an agent exposed as a named pipe, confirm that the pipe's owner is trusted**: the current user, SYSTEM or Administrators;
otherwise refuse the connection. The OpenSSH agent's pipe name is fixed (`openssh-ssh-agent`); while the service isn't running, any local user can create that name first
and then receive our signing requests — and, when adding keys to the agent (§7.3), **plaintext private keys**. Whoever creates the pipe first can only make themselves its owner.
This is not defended by lowering the impersonation level to Identification: the OpenSSH agent service stores keys as the connecting user, so lowering it would break the legitimate agent too.

〔Decision〕**The local agent's default endpoint** (`SshAgentClient.DefaultEndpoint`, used both for connecting to the agent and for agent forwarding when no endpoint is given): on other platforms it follows
`SSH_AUTH_SOCK`; on Windows, `SSH_AUTH_SOCK` is adopted when it is a named pipe (`\\.\pipe\…`) — agents such as 1Password and KeePassXC are configured that way —
and otherwise the OpenSSH agent service's pipe is used. It more often points at a Git Bash / WSL Unix socket, which is a different agent that .NET cannot reach, so anything that is not a pipe is ignored.
〔History〕Windows used to ignore `SSH_AUTH_SOCK` altogether, and the host had to make the same check itself.
When the OpenSSH agent service is not running (its pipe is absent) and the current user has **Pageant** open, Pageant is used: since PuTTY 0.75 it speaks the same agent protocol on `\\.\pipe\pageant.<user name>.…`,
and the tail of the pipe name varies per machine, so the pipe list is searched for the prefix "`pageant.` + current user name + `.`". When both are present the OpenSSH one still wins (as before);
another user's Pageant is never picked. The owner check applies as usual.

〔Decision〕**We only forward; we do not implement an agent server.**
The local agent is provided by the OS (OpenSSH agent / Pageant / 1Password, etc.).
#### 7.2.1 Signature confirmation must say what the signature is for

> Basis: RFC 4252 §7 (the `publickey` signature input); OpenSSH `PROTOCOL` (`publickey-hostbound-v00@openssh.com`),
> `PROTOCOL.agent` (`session-bind@openssh.com`), `PROTOCOL.sshsig`.

If the confirmation callback only gets the key and its comment, the user cannot tell whether this is the `git pull` they just ran on the remote,
or someone on that machine using the key to sign in somewhere else — and per-signature confirmation is meaningless.
So when the data to be signed is recognizable, `AgentSignatureRequest` carries a few more fields:

| Property | When it is set | Where it comes from, and why it can be trusted |
| --- | --- | --- |
| `UserName` / `Service` | The data is a public-key sign-in | The user name and service in the signature input. The server checks them against the request, so the signature can only be used to sign in as that user |
| `DestinationHostKey` | A sign-in whose destination can be verified | `publickey-hostbound-v00@openssh.com`: the host key at the end of the signature input, which the server checks is its own. Plain `publickey`: a session binding (§7.4) that the remote hop sent **on the same agent channel**, **whose signature we verify ourselves**, with a session identifier equal to the `session_id` in the signature input |
| `SignatureNamespace` | The data is an SSHSIG (`ssh-keygen -Y sign`, git's SSH commit signing) | The namespace in the signature input, such as `git` or `file` |

Shapes of the signature input:

| Kind | Fields (in order) |
| --- | --- |
| Public-key sign-in (RFC 4252 §7) | string `session_id`; byte `50`; string user name; string service; string `publickey`; boolean TRUE; string signature algorithm; string public key |
| Host-bound sign-in | As above with method `publickey-hostbound-v00@openssh.com`, followed by string server host key |
| SSHSIG | 6 bytes `SSHSIG`; string namespace; string reserved; string hash algorithm; string message digest |

〔Decision〕**We verify the remote's session bindings ourselves.** The local agent cannot be relied on to do it: agents that do not know the extension
(Pageant, older Windows agents) answer `FAILURE` regardless, and from a `SUCCESS` we cannot tell whether it was verified. Without verifying, the remote could
declare any public key of "a host you trust" and the dialog would say "signing in to github.com". A binding that fails verification is still relayed to the agent (§7.4);
it is just not used as the destination.

〔Decision〕**The key presented in the sign-in request must be the key being asked to sign**, otherwise the data is not treated as a sign-in: that signature could not
sign in anywhere, and showing the user name inside it would only mislead.

〔Decision〕**This is for display only.** Anything unrecognized is left empty; it never causes a refusal or changes the request passed to the agent. Text that reaches
the UI has control and bidirectional-control characters replaced and is cut to 128 characters. Parsing and verification happen only when per-signature confirmation is on;
each agent channel keeps at most 16 verified bindings, and the oldest is dropped when more arrive.

〔Decision〕**Say when it cannot be verified.** When `DestinationHostKey` is `null` (the remote's ssh is too old to send session bindings, or deliberately does not),
the UI should say "cannot be verified" rather than nothing — saying nothing lets the user assume it is going where they think.

### 7.3 Adding keys to the local agent (`ssh-add`)

> Basis: the "adding keys", "private key formats" and "key constraints" sections of draft-miller-ssh-agent; OpenSSH `PROTOCOL.agent`.

This is a request made by an agent **client** (what `ssh-add` does), not an agent server — it does not conflict with the decision in 7.2.
Purpose: decrypt an encrypted private key once and hand it to the agent; later authentication and forwarding are signed through the agent, with no passphrase prompt.

**Request message**:

| Field | Type | Notes |
| --- | --- | --- |
| Message number | byte | `17` `SSH_AGENTC_ADD_IDENTITY`; `25` `SSH_AGENTC_ADD_ID_CONSTRAINED` when constraints are present |
| Key type | string | `ssh-ed25519` / `ssh-rsa` / `ecdsa-sha2-nistp256` / `-nistp384` / `-nistp521` |
| Private key contents | per type, see below | |
| Comment | string | UTF-8; the column `ssh-add -l` shows, usually the private key file path |
| Constraints | byte + arguments, repeatable | **only in `25`**, see below |

**Private key contents** (immediately after the key type):

| Key type | Fields (in order) |
| --- | --- |
| `ssh-ed25519` | string public key (32 bytes); string seed ‖ public key (64 bytes, seed first) |
| `ssh-rsa` | mpint n; mpint e; mpint d; mpint iqmp (q⁻¹ mod p); mpint p; mpint q |
| `ecdsa-sha2-*` | string curve name (`nistp256` / `nistp384` / `nistp521`); string public point (uncompressed, `0x04 ‖ X ‖ Y`); mpint private scalar d |

⚠️ For RSA it is **n first, then e** — the reverse of the public key blob (e first). Get it backwards and the agent still answers SUCCESS;
it only shows up on the first signature.

**Adding "certificate + private key" together** (what `ssh-add` does when it finds a matching `-cert.pub`; `AddIdentityAsync(key, certificate, …)`): the key type becomes the certificate's
(`ssh-ed25519-cert-v01@openssh.com` and so on), followed by string the whole certificate; after that come **only the private parts the certificate does not already carry**:

| Certificate type | Fields after the certificate (in order) |
| --- | --- |
| `ssh-ed25519-cert-v01@openssh.com` | string public key (32 bytes); string seed ‖ public key (64 bytes) — same as plain ed25519 |
| `ssh-rsa-cert-v01@openssh.com` | mpint d; mpint iqmp; mpint p; mpint q (n and e are in the certificate) |
| `ecdsa-sha2-*-cert-v01@openssh.com` | mpint private scalar d (the curve name and public point are in the certificate) |

Once added, the agent's identity list shows the certificate, and sign requests carry the certificate blob. 〔Decision〕If the certificate certifies a different key (its public key does not match the private key's),
or what was given is not a certificate at all, it is an immediate `ArgumentException` and nothing is sent to the agent. 〔Verified〕All three certificate types against a real OpenSSH 10.3 `ssh-agent`:
`ssh-add -l` lists them as `*-CERT`, and signatures made through it verify with the original public key.

**Constraints**:

| Number | Name | Argument | Meaning |
| :-: | --- | --- | --- |
| `1` | `SSH_AGENT_CONSTRAIN_LIFETIME` | uint32 seconds | The agent deletes the key itself when it expires. A fraction of a second is rounded up; 〔Decision〕a lifetime of 0, negative, or beyond uint32 throws `ArgumentOutOfRangeException` when set (it used to be silently clamped to 1 second, so the added key vanished a second later) |
| `2` | `SSH_AGENT_CONSTRAIN_CONFIRM` | none | The agent asks the user to confirm every signature (`ssh-add -c`) |
| `255` | `SSH_AGENT_CONSTRAIN_EXTENSION` | string extension name + the extension's own content | This library sends only one: the destination constraint `restrict-destination-v00@openssh.com` (`ssh-add -h`), see §7.3.2. Placed after `1` and `2` |

**Response**: `6` `SSH_AGENT_SUCCESS` means success; `5` `SSH_AGENT_FAILURE` throws `SshAgentException`.
The agent gives no reason, so the exception message names the three common ones: the agent does not support constraints (some agents reject `25` outright),
the agent is locked (`ssh-add -x`), or the agent does not support this key type. With destination constraints there is one more (§7.3.2).

〔Decision〕

1. **Only in-process private keys are accepted** (`InMemorySshSigner`). When a signer is backed by an agent / PKCS#11 / HSM the private key is not in hand at all.
   Certificates go through a separate overload that takes the certificate and the private key separately (see above).
2. **No constraints means `17`**; never send a `25` with an empty constraint list — some agents accept `17` but not `25`.
   Conversely, **constraints present always means `25`**: the agent does not read bytes left over at the end of a `17`, so constraints appended to a `17` are silently dropped (the ⚠️ in §7.3.2).
3. **The request buffer is zeroed right after use.** The buffer is reserved at its upper bound up front, so growth never leaves unzeroed copies on the heap.
4. **No duplicate check.** What happens when the same key is added twice is the agent's business (OpenSSH updates the comment and constraints);
   whether to look first with `REQUEST_IDENTITIES` is up to the caller.
5. **The library never adds keys on its own.** When to put something into the user's agent is the user's decision —
   the same principle as "never connect to the agent implicitly" in 04 §2.2. How long an added key lives is up to the agent
   (the Windows OpenSSH agent stores it in the registry, so it survives a reboot).

### 7.3.1 Managing the keys in the agent (remove, remove all, lock)

> Basis: the "Removing keys" and "Locking and unlocking" sections of draft-miller-ssh-agent.

| Request | Message number | Content | Method | Response |
| --- | :-: | --- | --- | --- |
| Remove one (`ssh-add -d`) | `18` `SSH_AGENTC_REMOVE_IDENTITY` | string public key blob (the certificate blob for a certificate) | `RemoveIdentityAsync` | SUCCESS → `true`; FAILURE (no such key, agent locked) → `false` |
| Remove all (`ssh-add -D`) | `19` `SSH_AGENTC_REMOVE_ALL_IDENTITIES` | none | `RemoveAllIdentitiesAsync` | FAILURE throws `SshAgentException` (`AgentRefused`, the message names "locked") |
| Lock (`ssh-add -x`) | `22` `SSH_AGENTC_LOCK` | string passphrase | `LockAsync` | SUCCESS → `true`; FAILURE (already locked) → `false` |
| Unlock (`ssh-add -X`) | `23` `SSH_AGENTC_UNLOCK` | string passphrase | `UnlockAsync` | SUCCESS → `true`; FAILURE (wrong passphrase, or not locked — the agent does not distinguish) → `false` |

〔Decision〕Removing one key and lock / unlock return a `bool` rather than throwing: "the key was not in the agent" and "already locked" are normal outcomes the caller just reflects in the UI.
Remove-all can only fail because the agent is locked or does not support it, so it throws. The passphrase in the request is zeroed after use.

A locked agent (measured against a real OpenSSH 10.3) answers the identity list with an empty list and refuses signing and remove-all until it is unlocked with the same passphrase —
a forwarded agent cannot sign during that time either; use it when stepping away from the machine.

### 7.3.2 Destination constraints (`restrict-destination-v00@openssh.com`, `ssh-add -h`)

> Basis: OpenSSH `PROTOCOL.agent` §2; RFC 9987 (the published form of draft-miller-ssh-agent) §5.2.7 "Key Constraints" and §5.2.7.3 "Constraint Extensions";
> OpenSSH's "SSH agent restriction" design note (openssh.com/agent-restrict.html); `-h` / `-H` in `ssh-add(1)`.
> `PROTOCOL.agent` only lists the fields and does not say how the layers nest. The nesting below and the agent's behavior were checked black-box against OpenSSH 10.3's `ssh-add` / `ssh-agent` / `ssh` / `sshd`
> (capturing the bytes on the agent socket, hand-built requests, real connections and real forwarding), marked 〔Verified〕.

〔History〕§7.4 has long said "the agent relies on session binding to enforce `ssh-add -h` constraints", and this library does send the binding — but that only **cooperates with** constraints someone else added:
when this library adds a key to the agent itself (§7.3) it cannot express such a constraint; `SshAgentKeyConstraints` only has lifetime and per-use confirmation.
Anyone who wanted "this key may only log in to these hosts" had to fall back to the command line.

**What it governs**: restricting a key to "only through where, only to which host, only as which user". The agent enforces the restriction, not this library —
this library only writes the restriction into the add request (the constraint table in §7.3) and makes session bindings during authentication and forwarding (§7.4), so the agent knows which host on which path each signature is for.

#### What a hop is

A destination constraint consists of a number of "hops"; each hop permits one leg of a path:

| Part of a hop | Meaning |
| --- | --- |
| Start | Empty = starting directly from the local machine (the one running the agent); otherwise a **forwarding host**: the key reached it through agent forwarding and is used onward from there |
| End | The host this leg arrives at |
| User name | Which user to log in as on the end host; empty = any |

Hosts are identified by **host key**, not by name: what session binding (§7.4) hands to the agent is the server's host public key, and that is what the agent compares.

| Goal | Hops needed |
| --- | --- |
| Log in to bastion directly from the local machine, any user | (local → bastion) |
| Log in to prod directly from the local machine as `deploy` | (local → prod, `deploy`) |
| Log in to bastion, forward the agent there, then log in from bastion to db as `deploy` | (local → bastion) **and** (bastion → db, `deploy`) |

How the agent decides (the 〔Verified〕 points were measured against OpenSSH 10.3):

1. **Every leg of the path a signature travels must be permitted by some hop.** Direct authentication from the local machine has a single leg; through forwarding, the first leg is "local → first forwarding host",
   each forwarding host adds one more leg, and the last leg reaches the destination host. 〔Verified〕With only (local → A) and (local → B) permitted, logging in to B after forwarding to A is refused — (A → B) is missing.
2. **For the key to be visible on a forwarding host, some hop must start at that host.** 〔Verified〕With only (local → A), `ssh-add -l` on A after forwarding says the agent has no identities;
   adding (A → B) makes it list the key.
3. **How a host is recognized**: the host public key in the session binding is among this hop host's "host keys"; or it is a host certificate whose signing CA is among this hop host's "CA keys",
   and the certificate's principals accept this hop host's **name**.
   - When recognized by host key alone, the name takes no part in the comparison. 〔Verified〕The same host key recorded in `known_hosts` under a different name, with the constraint added under that name, is still permitted; connecting to the same host through an alias is permitted too.
   - When recognized by CA, the name is compared as-is against the certificate's principals. 〔Verified〕Name `other` against principal `host-c` is refused; the name itself is **not a wildcard pattern**
     (`host-*` against `host-c` is refused), and it is **case-sensitive** (`HOST-C` against `host-c` is refused); when the certificate's principal itself has a wildcard (`*.example.org`),
     both a concrete host name (`web.example.org`) and a copy of that principal are permitted.
4. **The user name is only checked on the leg where this key logs in**, and may contain `*` / `?` wildcards. 〔Verified〕`alice` against login user `probe` is refused, `pro*` is permitted;
   when (local → A) is restricted to `alice` but the key is only forwarded through A and used to log in to B as `probe`, that `alice` has no effect.
5. **A connection without session binding**: the agent still lists the key but refuses to sign with it. 〔Verified〕`ssh-add -l` directly on the local machine lists it, `ssh-add -T` is refused.
   Authentication is refused just the same (the 9.9p2 measurement in §7.4) — which is why this library must bind.
6. **User authentication only.** The design note says so: the agent has to parse the signed data as a public key login to get at the session identifier and user name it checks; other signatures (SSHSIG and the like) cannot be checked and are always refused.
   〔Verified〕A local `ssh-keygen -Y sign` (which is what git's SSH commit signing uses) with this key is refused. Destination-constrained keys cannot sign commits; the host's UI should make that clear to the user.
7. **Constraints cannot be changed once added; to change them, add the key again.** 〔Verified〕Adding the same key again replaces the old constraints entirely with the new ones (consistent with §7.3 decision 4, "no duplicate check").

The path the agent sees when the key is used through forwarding:

```mermaid
sequenceDiagram
    participant G as Local agent
    participant C as This library (local)
    participant A as ssh on forwarding host A
    participant B as Destination host B

    C->>G: Session binding (A's host key, is_forwarding = true)
    Note over C,A: Agent forwarding channel (§7.1)
    A->>G: Forwarded: session binding (B's host key, is_forwarding = false)
    A->>G: Forwarded: sign request (log in to B, user deploy)
    Note over G: Path local → A → B:<br/>needs (local → A) and (A → B, deploy or any)
    G-->>A: Signature, or FAILURE
    A->>B: USERAUTH_REQUEST
```

#### Message

The destination constraint is one extension constraint (`255`) in the constraint table of §7.3, placed after lifetime and confirmation — 〔Verified〕that is the order `ssh-add -t … -c -h …` uses.
**All hops go into a single extension constraint** (〔Verified〕several `-h` options still send just one).
In the `PROTOCOL.agent` pseudo-structure every variable-length layer is **wrapped in a string of its own** (uint32 length + content), four layers in all:

| Field | Type | Notes |
| --- | --- | --- |
| Constraint number | byte | `255` `SSH_AGENT_CONSTRAIN_EXTENSION` (RFC 9987 §8.2) |
| Extension name | string | `restrict-destination-v00@openssh.com` |
| Hop list | string | Its content is the hops back to back, each one a string; **there is no count field**, read until this string ends |

One hop (the content of each string in the hop list):

| Field | Type | Notes |
| --- | --- | --- |
| Start | string | Its content is a "host description" (next table). Starting from the local machine it is an **empty host description**: three empty strings and no keys, 12 bytes in all |
| End | string | Host description |
| Reserved | string | Empty |

Host description (the content of the start and end strings):

| Field | Type | Notes |
| --- | --- | --- |
| User name | string | Start: **must be empty**; end: `UserName`, empty for any |
| Host name | string | UTF-8. Start: the start host's `Name` (empty when starting from the local machine); end: the end host's `Name` |
| Reserved | string | Empty |
| Host keys | zero or more groups of "string public key blob ‖ boolean is CA" | One group after another, **no count field**, until this string ends. `HostKeys` first (false), then `CertificateAuthorities` (true), each in list order. The boolean is only ever written as `0` / `1` |

Example: (local → host-c, one ed25519 host key, any user). 〔Verified〕Identical to the bytes `ssh-add -h host-c` sends:
start 12 bytes; end 74 bytes (4 + 0, 4 + 6, 4 + 0, then 4 + 51 for the public key blob and 1 byte of false);
hop 98 bytes (4 + 12, 4 + 74, 4 + 0); hop list 102 bytes (4 + 98); the whole constraint 147 bytes (1 + 4 + 36 + 4 + 102).

〔Verified〕What the agent checks in the content (hand-built requests):

| Request | Agent's reply |
| --- | --- |
| Non-empty start user name | FAILURE |
| Empty end host name; end with no keys | FAILURE |
| Start with a name but no keys, or keys but no name | FAILURE |
| Any reserved field non-empty | FAILURE |
| Hop list not wrapped in a string (hops directly after the extension name) | FAILURE |
| Unknown extension name | FAILURE (`5`, not `28`) |
| Empty hop list; "is CA" written as `2`; a certificate as a host key; a `,` in the host name | All answered SUCCESS — this library blocks every one of these at construction / set time (see below), so they are never sent |

⚠️ **Constraints require `25`.** 〔Verified〕Append the same constraint bytes to a `17` (`SSH_AGENTC_ADD_IDENTITY`) and the agent answers SUCCESS,
but the key it adds carries **no constraint at all** — it does not read the bytes left over at the end of a `17`. §7.3 decision 2, "no constraints means `17`", read backwards is "constraints present always means `25`";
for destination constraints this is a security matter: the user believes the key can only log in to bastion, when in fact it can log in anywhere. A unit test must pin it down.

#### Public API

〔Decision〕One property on `SshAgentKeyConstraints` and two new small records; the three `AddIdentityAsync` overloads stay as they are.

| Member | Shape | Notes |
| --- | --- | --- |
| `SshAgentKeyConstraints.AllowedHops` | `IReadOnlyList<SshAgentHop>?`, `init` | `null` (the default) = no destination constraint; when given, "only these hops are allowed" |
| `SshAgentHop` | `sealed record`, `Keys/SshAgentHop.cs` | Constructor parameters in order: end, user name (default `null`), start (default `null`); properties `Destination`, `UserName`, `Via`, read-only |
| `SshAgentHopHost` | `sealed record`, `Keys/SshAgentHopHost.cs` | Constructor parameters in order: name, host key list, CA key list (default `null`, treated as empty); properties `Name`, `HostKeys`, `CertificateAuthorities` (both lists are `IReadOnlyList<SshPublicKey>`), read-only |
| `SshAgentHopHost.FromKnownHosts` | static method returning `SshAgentHopHost?` | Builds a host from `known_hosts` (next subsection) |

- The start is called `Via` (`null` is the local machine), not "source": the design note's reminder is right — a hop means "used **through** this host", not "coming from this host".
- The two new records have read-only properties without `init`: all validation happens once in the constructor, and `with` cannot produce an invalid combination (`src/VelaShell.Ssh/AGENTS.md` §4.3, "invalid values throw at construction").
- Lists are copied into a read-only copy at construction / set time (like `AgentForwardOptions.AllowedKeys`); equality of `SshAgentHopHost`, `SshAgentHop` and `SshAgentKeyConstraints`
  compares **content**, order included (like `SshAlgorithmSet`).
- `SshAgentKeyConstraints` counts `AllowedHops` when deciding "are there any constraints": `AllowedHops` alone also sends `25`.
- Constraint number `255` and the extension name become named constants in `SshAgentMessage` (next to `SessionBindExtension`).

〔Decision〕Validation happens at construction / set time and throws on violation:

| Where | Rule | Throws |
| --- | --- | --- |
| Name of `SshAgentHopHost` | Not `null` | `ArgumentNullException` |
| Same | Non-empty; at most 255 bytes of UTF-8; no whitespace, control characters or `,` | `ArgumentException` |
| Host key list | Not `null` (the CA key list may be `null`) | `ArgumentNullException` |
| Elements of both lists | Not `null`; **not a certificate** (a host key is given as the key inside the certificate, a CA as the CA public key itself) | `ArgumentException` |
| Both lists together | After removing duplicates within each list by blob (keeping the first occurrence's position), 1–32 keys in total | `ArgumentException` |
| End of `SshAgentHop` | Not `null` | `ArgumentNullException` |
| User name of `SshAgentHop` | `null` = any; when given, non-empty, at most 255 bytes of UTF-8, no whitespace, control characters, `,` or `!` | `ArgumentException` |
| `AllowedHops` | `null`, or 1–64 entries with no `null` element; entries kept as given, not deduplicated | `ArgumentException` |
| The whole add message | Within the agent message limit (256 KiB, §7.1) | Throw `SshAgentException` (`LimitExceeded`) before sending; not a byte is sent |

Why these rules:

- **An empty list must not mean "any".** `AllowedHops = []` is either "allowed nowhere" — the added key would be useless — or the user unticked every box;
  treating it as "any" is exactly the trap `AgentForwardOptions.AllowedKeys` once fell into (§7.2). For "any", pass `null`. (The agent itself does accept an empty hop list, 〔Verified〕.)
- **No `,`, whitespace or control characters in host names.** In a host name they can only be a mistake (certificate principals are separated by `,` and never contain one),
  yet the agent accepts them all (〔Verified〕), and the key quietly becomes usable nowhere. `*` / `?` are accepted: a principal can itself be a wildcard pattern, and a name that copies it is permitted —
  but the name is **never** expanded as a pattern (point 3 above).
- **No `,` or `!` in user names.** `*` / `?` wildcards were verified; pattern lists and negation were not, and callers should not depend on semantics nobody can vouch for.
- **No certificates.** The agent does accept a certificate blob as a host key (〔Verified〕), but a host certificate's blob changes every time it is re-signed, so pinning one certificate means the key stops working a while later.
  What belongs there is the key inside the certificate, or the CA that signs it (`@cert-authority` in `known_hosts`, 03 §5.5).
- The limits (64 hops, 32 keys per host, 255-byte names) are sized so that normal use never comes near them and a mistake cannot drag out a giant request; the real hard limit is the 256 KiB of the whole message.

#### Building a host from `known_hosts` (`SshAgentHopHost.FromKnownHosts`)

〔Decision〕**Provided.** This is what `ssh-add -h` does: the user gives host names, and the host keys are looked up in `known_hosts`.
Collecting host keys by hand easily misses a type (the server presents ECDSA this time, while the list only has Ed25519), and the consequence of a miss is that the key is refused on that host.

Parameters: parsed entries (the result of `KnownHostsFile.LoadAsync` / `Parse`), host name, port (default 22). Which lines count:

| Line in `known_hosts` | How it counts |
| --- | --- |
| Plain line | If the host matches, its key goes into `HostKeys`. "Matches" uses **exactly the same rules** as `KnownHostsFile.Lookup` (03 §5.4): plain, hashed lines (`HashKnownHosts`), wildcards and negation, case-insensitive; for a port other than 22 only the `[host]:port` form counts |
| `@cert-authority` | If the host matches, the CA public key goes into `CertificateAuthorities` |
| `@revoked` | If the host matches, its key is **removed** from both lists, whether it comes earlier or later in the file |
| A key this library cannot decode; a certificate on a plain line | Skipped |
| An unrecognized `@` marker | The whole line is skipped (`KnownHostsFile.Parse` never accepts it anyway) |

- Each list is deduplicated by blob, keeping the order of first appearance in the file.
- **Name** = the given host name lower-cased (invariant), **without the port**. The name is only used against certificate principals, and principals carry no port; the port only decides which lines count.
  Lower-casing because the agent compares principals case-sensitively (point 3 above) and principals are lower-case by convention — consistent with `known_hosts`'s own rules (03 §5.4).
- Both lists empty: return `null`, and let the caller tell the user "this host is not in `known_hosts`; connect once and trust it first".
  〔Verified〕`ssh-add -h` exits with an error in this case (`No host keys found for destination`) and sends nothing to the agent.
- A host name that breaks the rules above, or more than 32 keys in total after deduplication: the same `ArgumentException` as the constructor; a port outside 1–65535: `ArgumentOutOfRangeException`.
- Only the entries passed in are looked at. Without `-H`, `ssh-add` searches four files (`~/.ssh/known_hosts`, `~/.ssh/known_hosts2`, `/etc/ssh/ssh_known_hosts`, `/etc/ssh/ssh_known_hosts2`);
  which files to search is the caller's decision — concatenate the entries of several files and pass them in.

〔Decision〕**Revoked keys are removed — this differs from `ssh-add`.** 〔Verified〕OpenSSH 10.3's `ssh-add -h` sends a host key that is also `@revoked` to the agent all the same.
This library's `KnownHostsFile.Lookup` always rules a revoked key `Revoked` and refuses it (03 §5.4); keeping it in the permit list would open a door in the agent for a key we ourselves do not accept.

#### Responses and errors

| Case | Result |
| --- | --- |
| `6` SUCCESS | Added |
| `5` FAILURE, or `28` EXTENSION_FAILURE | `SshAgentException` (`AgentRefused`). With `AllowedHops`, the message names, in addition to the three common reasons in §7.3, "the agent does not support destination constraints (`restrict-destination-v00@openssh.com`, available since OpenSSH 8.9; non-OpenSSH agents most likely lack it)" |
| Message larger than 256 KiB | Throw `SshAgentException` (`LimitExceeded`) before sending; the message says "the key or the destination constraints are too large" |

〔Decision〕

1. **Refused means refused; never fall back to adding the key without the constraint.** RFC 9987 §5.2.7 requires an agent to reject the whole request on an unrecognized constraint precisely so that "failure is safe";
   a client that quietly drops the constraint and retries tears that safeguard down.
2. **No probing whether the agent supports it.** 〔Verified〕OpenSSH 10.3 answers the `query` extension (`ssh-add -Q`) with only `session-bind@openssh.com` and does not list constraint extensions — asking tells nothing. Just send it and handle a refusal.
3. **`28` counts as a refusal too.** RFC 9987 says adding a key answers only SUCCESS / FAILURE (〔Verified〕OpenSSH answers `5` to every malformed constraint);
   if some other agent answers extension failure, that is still a refusal, not a protocol error.
4. **No new `SshFailureReason`.** The agent gives no reason, so `AgentRefused` is the truth; the caller knows it passed `AllowedHops` and the UI can add a hint on that basis
   without parsing the message. The host's `SshInterop.Localize` needs no change.
5. **One more hint when a signature is refused.** When a destination-constrained key is used on a host, user or path it does not permit, the agent refuses to sign, and the authenticator records `SkippedNoMaterial` for that credential
   and moves on to the next (04 §3.4). The message of a refused `SignAsync` adds, next to "the key is no longer in the agent" and "per-use confirmation was declined", "this key carries destination constraints (`ssh-add -h`)
   that do not permit this host, this user or this forwarding path".

#### How it works with session binding (§7.4) and agent forwarding (§7)

- **Authentication**: this library binds before a key in the agent signs for the first time (`is_forwarding = false`, §7.4) — exactly what the agent needs to judge the "local → destination" hop.
  This library lists keys first and binds afterwards: on an unbound connection the agent still lists constrained keys (point 5 above), so the key is also tried against a host that is not permitted —
  the server accepts the public key, the agent refuses to sign, it is recorded, and the next credential is tried. OpenSSH's `ssh` binds as soon as it connects to the agent (the design note);
  〔Verified〕against a host that is not permitted its output has no signing refusal for this key (it does when the user name is wrong) — on a bound connection the agent does not list the key to it. The only difference is one extra probe.
- **Forwarding**: each agent channel connects to the local agent on its own and binds with `is_forwarding = true`, and the remote hop's binding is passed through (§7.4) — the agent judges by points 1 and 2 above; the forwarding path needs no change.
  `AgentForwardOptions.AllowedKeys` still filters once more at this library's layer: one is the user's choice for this particular forwarding, the other a restriction the key carries with it and that holds for everyone; the two stack.

〔Decision〕**One agent connection authenticates for one session only.** The agent does not accept binding another session on a connection already bound for authentication (`PROTOCOL.agent` §1; 〔Verified〕see item 6 below),
yet the same `SshAgentClient` may authenticate several SSH connections in turn — when connecting with `ProxyJump` from `ssh_config`, the library hands the same agent credential to every hop, jump hosts and target alike.
The later hop's binding is then refused, the agent connection still records the earlier hop, and a destination-constrained key refuses to sign on the later hop even though its constraints permit it. Therefore:

- When authentication needs to bind a new session, and the last **accepted** authentication binding on this connection belongs to a different session, a client connected with `ConnectAsync` first reopens an agent connection to the same endpoint,
  binds on the new connection and closes the old one; listing and signing afterwards go over the new connection. When the previous binding was not accepted (the agent does not support binding), nothing is reopened — reopening would change nothing.
- Closing the old connection does no harm: the earlier hop's authentication finished before the later hop started (the later hop is dialed through the earlier one).
- A stream handed over through `FromStream` cannot be reopened, so the binding is still sent on the original connection and yields `false`; with such clients, constrained keys need one client per SSH connection.
- When one client is used by several connections to authenticate **at the same time**, constrained keys are not guaranteed to work (the sessions swap the connection out from under each other).

#### 〔Verified〕Interop cases against a real agent

Against OpenSSH 10.3's `ssh-agent` in Docker (the agent runs on the server and is reached through a tunnel), the results:

1. **Bytes**: unit tests pin the four levels of nesting and the 12 / 74 / 98 / 102 / 147 of the example in "Messages", plus one case each with a user name, with a start host, with two keys on one host, and with a CA;
   the byte-for-byte comparison with `ssh-add -h` was done when this section was written (see "Messages"). The real agent accepts the constraint and enforces it (the items below).
2. After adding, `ssh-add -l` on a connection without a binding lists the key.
3. **Authentication**: a permitted host connects, and a user name with wildcards is permitted too; when the host key or the user name does not match, the agent refuses to sign, and the credential is skipped and the next one is tried.
4. **CA**: when the server presents a host certificate and `known_hosts` has only the `@cert-authority` line, a hop built by `FromKnownHosts` connects; when the name is not among the certificate's principals the agent refuses to sign.
5. **Forwarding**: with the key carrying (local → A) and (A → B), `ssh-add -l` on A lists it and `ssh` from A to B succeeds; with only (local → A), A lists nothing and logging in to B is refused.
6. **Jump hosts**: one client opened by `ConnectAsync` authenticates the jump host and then the target, and both hops sign. A supplied stream (which cannot be reopened) is refused a signature on the target hop ——
   the agent really does refuse a second authentication binding on the same connection.
7. **Unconstrained keys are not dragged in**: after a supplied stream was bound for the jump host, an unconstrained key still signs for the target.
8. **Message number**: with `AllowedHops`, always `25`, pinned by a unit test; with that rule reverted, the real-agent case lets through what should have been refused.
9. **Refusal**: a fake agent answering `5` / `28` to the add request yields `AgentRefused` in both cases, with the destination-constraint hint, and no second add request follows.
10. Not verified: whether the Windows OpenSSH agent and Pageant accept this constraint —— that would mean adding keys to the agent the user actually uses.

〔Verified〕A pitfall when writing the negative cases: if authentication fails outright, OpenSSH 9.8+'s `PerSourcePenalties` blocks that source address for a while,
dragging down later cases that connect from the same address. The cases give a password fallback after the agent credentials and look at which method was finally used —— an agent refusal only skips that credential, and the server records no failure.

### 7.4 Session binding (`session-bind@openssh.com`)

> Basis: OpenSSH `PROTOCOL.agent` §1; `SSH_AGENTC_EXTENSION` in draft-miller-ssh-agent; the session identifier in RFC 4253 §7.2;
> the semantics of the constraints in OpenSSH's "SSH agent restriction" design note (openssh.com/agent-restrict.html).

`ssh-add -h` adds **destination constraints** to a key (usable only for certain hosts, forwardable only along certain paths; this library can add them too when adding keys, §7.3.2). To enforce them, the agent has to know
"which SSH session this agent connection is serving" — and that is what session binding tells it.

**Request message**:

| Field | Type | Notes |
| --- | --- | --- |
| Message number | byte | `27` `SSH_AGENTC_EXTENSION` |
| Extension name | string | `session-bind@openssh.com` |
| Host public key | string | Wire encoding of the server's host public key (`K_S` from the first exchange) |
| Session identifier | string | `H` from the first exchange (03 §4.3) |
| Signature | string | The signature blob over `H` that the server sent in its reply to the first exchange |
| `is_forwarding` | boolean | false for a connection used for authentication, true for one used for forwarding |

**Response**: `6` `SSH_AGENT_SUCCESS` means accepted; `5` `SSH_AGENT_FAILURE` or `28` `SSH_AGENT_EXTENSION_FAILURE` means not supported or refused.

〔Decision〕**All three come from the first key exchange.** The session identifier never changes afterwards, and the agent verifies this signature with the host public key —
a signature from rekeying signs that round's own `H`, which does not match the session identifier.

〔Decision〕**When to send it**:

| Purpose | Timing | `is_forwarding` |
| --- | --- | :-: |
| Authentication (`publickey`, key held in the agent) | Before a key on this agent connection is asked to sign for the first time; sent only once per connection and session. One connection makes an authentication binding for one session only; a different session reopens the connection (§7.3.2) | false |
| Forwarding | After each `auth-agent@openssh.com` channel has connected to the local agent, before the channel is confirmed (§7.1) | true |

〔Decision〕**The remote hop's own session binding is passed through**, and it is the only extension let through. Each hop along a forwarding chain appends its own binding after the previous hop's,
and that is how the agent recognizes the whole path; a binding can only make the agent **stricter** with this connection, so letting it through does not widen the remote's powers.
For any other extension inside `SSH_AGENTC_EXTENSION` we cannot be sure what it might do, so it still gets `FAILURE` and never reaches the local agent.
The binding must travel over the **same** local agent connection — each agent channel gets a connection of its own, which satisfies this exactly.

〔Decision〕**A failed binding is not an error.** When the agent replies `FAILURE` / `EXTENSION_FAILURE` (old agents, non-OpenSSH agents), listing keys and signing go on as usual;
the constraints simply do not take effect — consistent with the OpenSSH client. A few agents **drop the connection** when they receive a message they do not recognize:
in that case reconnect once without binding, and the subsequent listing, signing and forwarding carry on as usual. The signer on the authentication path holds this very client connection, so it must not be broken along with it.

〔Decision〕**`publickey-hostbound-v00@openssh.com` is not implemented** (04 §7.2). According to OpenSSH's design note, the first hop does not need it —
the destination is handed to the agent by the session binding. This library is always the first hop: when going through jump hosts, every hop is also authenticated directly from the local machine.

Why it is required (not just "nice to have"):

- **Authentication**: the real OpenSSH agent (tested with 9.9p2) refuses to sign with keys constrained by `ssh-add -h` on a connection that has **not been bound** —
  such keys used to be completely unusable with this library.
- **Forwarding**: a forwarder that does not take part in binding is exactly the case OpenSSH's design note describes as not degrading gracefully —
  the agent cannot tell that this connection has been forwarded and treats the remote as the origin machine itself, so the constraints are as good as absent.

---

## 7.5 X11 forwarding

> Basis: RFC 4254 §6.3 (`x11-req` and the `x11` channel), the connection setup message of the X11 core protocol,
> the `.Xauthority` file format, and OpenSSH's `ssh -X` / `-Y` behavior.
>
> 〔Scope〕**We implement only the forwarding side, not an X server** (architecture §12 "explicitly out of scope").
> We connect `x11` channels opened back by the server to an **existing local** X display.

### 7.5.1 Why the default for this must be "off"

X11 has no client isolation: **any client connected to the same display can read others' keystrokes,
capture others' windows, and inject events into others' windows**. So handing the local display to the remote
means handing the input and output of every local graphical session to the remote.

〔Decision〕By default X11 forwarding is **not requested**; enabling it must be explicit.
〔Decision〕By default use **untrusted** mode, consistent with `ssh -X`;
trusted mode (`-Y`) must be enabled explicitly on top of that.

### 7.5.2 Fake cookie — the security core of the whole mechanism

**Never send the real local X authorization cookie to the server.**

Approach (consistent with OpenSSH):

1. We generate a **random fake cookie** locally and send that in `x11-req`;
2. The server writes the fake cookie into the remote `.Xauthority`, and remote X clients use it to connect;
3. The server opens an `x11` channel back; we read its **first message** (the X11 connection setup message)
   and verify that the cookie inside is the fake cookie we sent;
4. Verification passes → **replace the fake cookie with the real local cookie**, then forward the message to the local X server;
5. Verification fails → **reject the channel**.

〔Decision〕Cookie comparison must be **constant-time**. A byte-by-byte short-circuit comparison leaks
"how many of the leading bytes were right", and an attacker can open many channels and try slowly.

〔Decision〕Each display has its own fake cookie — forwarding multiple displays over one connection does not cross wires.

### 7.5.3 `x11-req` (RFC 4254 §6.3.1)

```
byte      SSH_MSG_CHANNEL_REQUEST
uint32    recipient channel
string    "x11-req"
boolean   want_reply
boolean   single_connection
string    x11_authentication_protocol   // "MIT-MAGIC-COOKIE-1"
string    x11_authentication_cookie     // the fake cookie as **hexadecimal** text
uint32    x11_screen_number
```

〔Caution〕The cookie field is a **hexadecimal string**, not raw bytes. The symptom of writing raw bytes is that
the cookie stored by the remote `xauth` does not match what we verify, and the error message only says
"connection refused".

〔Decision〕Ordering: `pty-req` → **`x11-req`** → `auth-agent-req@openssh.com` → `env` → `shell`/`exec`
(`05-connection.md` §5.2). **Both interactive shells and one-shot commands are supported** — the most common use of `ssh -X` is a shell.
〔Decision〕`single_connection` is sent as `false` by default: one remote session often opens multiple X clients.
When the user asks for single-connection, besides sending `true`, **we also enforce it locally**: after the first X11 connection, all subsequent connections for the same forward are rejected —
we do not entrust a security constraint to the peer.
〔Decision〕The SFTP subsystem does **not inherit** the connection-level X11 switch — file transfer needs no display.

### 7.5.4 The `x11` channel (RFC 4254 §6.3.2)

Channel type `x11`, type-specific fields:

```
string    originator address
uint32    originator port
```

〔Decision〕**Accept only if X11 forwarding has been requested on this connection**; if not, always reject —
otherwise any server could proactively open channels onto our local display.

〔Decision〕**A single connection can have multiple X11 forwards** (one per session, each with its own fake cookie, display and lifetime).
The `x11` channel itself carries no field that points back to "which session requested it"; the only thing that distinguishes them is
**the cookie in the setup message** — so dispatch happens only after the setup message has been read: compare in constant time against each forward in turn,
and hand it to whichever one matches; if none matches, reject and record a rejection on every live forward.

- When all forwards have expired, reject at channel-open time;
- The concurrency limit is counted against the limit of the forward **after ownership is identified** — so there is no leak path of "occupied a slot but never reached handling";
- Releasing a forward removes only itself; when the last one is released, the connection stops accepting `x11` channels.

〔History〕The early implementation had only a single handler slot: a later request would evict an earlier one (the earlier session's X programs,
holding the correct cookie, were rejected as "wrong cookie"), and releasing any one of them removed the handler entirely.

### 7.5.5 X11 connection setup message

The first message sent by a connecting X11 client:

```
byte    byte_order          // 'B' = big-endian, 'l' = little-endian
byte    (unused)
uint16  protocol_major
uint16  protocol_minor
uint16  auth_protocol_name_length    n
uint16  auth_protocol_data_length    d
uint16  (unused)
byte[n] auth_protocol_name           // padded to a multiple of 4
byte[d] auth_protocol_data           // padded to a multiple of 4
```

〔Caution〕**Both byte orders must be recognized.** The byte order of the two length fields is determined by the first byte;
the symptom of parsing only one is "some clients can connect, others can't".

〔Decision〕Only `MIT-MAGIC-COOKIE-1` is supported; other authorization protocols (`XDM-AUTHORIZATION-1`)
are always rejected — consistent with OpenSSH.

〔Decision〕The message may **arrive in several pieces**: first accumulate the 12-byte fixed header,
then accumulate both data sections according to the lengths in the header. Assuming "the first read is the complete message"
fails randomly on small MTUs or slow links.

〔Decision〕**Each authorization field is limited to 256 bytes, and this is checked as soon as the 12-byte header is in.** Both lengths come from the remote side,
and these bytes must be buffered before the cookie is checked — waiting for whatever length the header claims (the protocol allows up to 64 KiB each)
would let a peer that has not proven anything decide how much we buffer and how long we wait. The only thing that can pass the check is the 18-byte
`MIT-MAGIC-COOKIE-1` plus the 16-byte fake cookie; every real X11 authorization protocol name and data is far below this limit. Anything larger rejects the channel on the spot, without connecting to the local X server.

### 7.5.6 Locating the local display

Forms of `DISPLAY`: `:0`, `:10.2`, `unix:0`, `host:0`, `[::1]:0`,
and macOS launchd socket paths.

〔Decision〕After parsing into the three parts `host` / `display_number` / `screen_number`:

| Case | Where to connect |
| --- | --- |
| Local socket (empty host, `unix`) | Try the Linux abstract socket first, then `/tmp/.X11-unix/X<N>`, then loopback TCP |
| Local socket + Windows | TCP `127.0.0.1:(6000+N)` (VcXsrv and the like) |
| `localhost` | TCP `127.0.0.1:(6000+N)` **only**; no local socket is tried |
| Remote host | TCP `host:(6000+N)` |

〔Decision〕**`localhost:N` is TCP, not a local socket.** By X convention it means `6000+N`, and that is exactly the form sshd sets for a nested `ssh -X`.
Treating it as a local socket and trying the Linux abstract socket is dangerous: the abstract namespace has no permission checks,
so another user on the same machine can bind `@/tmp/.X11-unix/X<N>` first and receive the **real** cookie we substitute in.
For cookie selection it still counts as a local display (the local host name's FamilyLocal entry, §7.5.7) — "where to connect" and "which cookie to use" are separate questions.

### 7.5.7 Where the real cookie comes from

| Mode | Source |
| --- | --- |
| Trusted (`-Y`) | Read `XAUTHORITY` or `~/.Xauthority`, **without running any external program** |
| Untrusted (`-X`, default) | Run `xauth -f <temporary file> generate <display> MIT-MAGIC-COOKIE-1 untrusted timeout <n>` and read the generated cookie from the temporary file |

〔Decision〕In trusted mode **do not invoke `xauth`** — reading the file is enough, and one fewer external program
means one less attack surface.
〔Decision〕When no matching entry is found in `.Xauthority`, use random data (consistent with OpenSSH):
letting the X server reject it is better than us guessing a cookie that is "probably right".
〔Decision〕**Entries with an empty cookie are skipped and the search continues** (the same goes for the temporary file generated by `xauth`):
an empty cookie opens no doors, and substituting it into the setup message as the real cookie would shut out the valid entry that follows it.
〔Decision〕Invoking `xauth` must have a **timeout** (30 seconds) — it can hang on an unresponsive X server.
On timeout or cancellation, **terminate its process tree** and leave no hanging child process; read its standard output and standard error **concurrently** —
waiting for exit without reading will fill the pipe once output grows, and the process will never exit.

〔Decision〕⚠️ **The generated restricted cookie is written to a temporary file accessible only to the current user (`-f`), deleted after use,
and never written into the user's `.Xauthority`.** The latter holds the **fully authorized** cookie for the local display;
`xauth generate` without `-f` would overwrite it with the restricted cookie — and once that expires,
the user's own local X programs can no longer connect to their own display.
(The authorization `xauth` uses to connect to the local display is still the user's: when an `.Xauthority` path is specified, it is passed via the `XAUTHORITY` environment variable.)

〔Decision〕The display name passed to `xauth` preserves the host and socket path: a local Unix socket is written as `:N`,
a remote display as `host:N`, and a macOS launchd one as the full socket path — if only `:N` were left,
`xauth` would look up (or generate) the entry for a different display.
〔Decision〕Forwarding has a **validity period** (default 20 minutes, corresponding to `ForwardX11Timeout` in `ssh_config`); after it expires, new `x11` channels are refused,
while existing ones are unaffected; no expiry (`Timeout.InfiniteTimeSpan`, written as 0 in `ssh_config`) means valid for the whole connection, and 0 or a negative value throws when set.
〔Intentional difference from OpenSSH〕In OpenSSH `ForwardX11Timeout` only governs untrusted mode; we apply it to **both modes** —
trusted mode is by far the more dangerous one, and it makes no sense for it alone to have no time limit.

〔Decision〕**The `timeout` given to `xauth` is our validity period plus 60 seconds; with no expiry, pass 0.**
The X SECURITY extension specifies that a restricted authorization is purged by the X server once it has spent `timeout` seconds in the state of
"no connection is using it", and that 0 means it never expires (the default when omitted is 60 seconds). The two sides keep separate clocks: the X server counts from
the moment of generation, we count from the forwarding request — with equal values there is an edge case where we have just accepted an `x11` channel and the X server
has just purged the authorization, so that connection is refused by the X server. The margin guarantees the X server side always ends later than ours.
With no expiry, any concrete number of seconds would make the X server purge the authorization after being idle that long while we still accept new connections —
so neither side has a time limit.

### 7.5.8 What to do on failure

This section applies **equally to X11 and agent forwarding (§7)**.

〔Decision〕**Two cases, because the user intent differs**, expressed by `FailureMode` (`ForwardFailureMode`) on the options:

| Who enabled it | `FailureMode` | On failure |
| --- | --- | --- |
| The caller **explicitly** requested it on this execution | `Fail` (default) | **Throw `SshForwardException`**, and the channel is closed with it — they explicitly want this forwarding; silently degrading would be lying to them |
| Only a connection-level switch (`ForwardX11 yes` / `ForwardAgent` in `ssh_config`, the host's connection settings) | `Continue` | **Start normally without this forwarding**; the reason is put on the result object's `X11SetupFailure` / `AgentSetupFailure` and counted in the forwarding error count (`reason = SetupSkipped`) — otherwise an existing configuration would make every command fail to run |

〔Decision〕**The failure policy is an enum, not a boolean.** It used to be `X11ForwardOptions.BestEffort`: a boolean's meaning cannot be read at the call site, and there is no room to add a third behavior.
The enum has **no "off" value**: whether to request at all is expressed by whether the option itself is given (`X11Forwarding` / `AgentForwarding` being null means no request);
adding an "off" as well would give two contradictory ways of saying the same thing. The zero value is `Fail`.
This library also **has no connection-level forwarding switch** (§7.5.1): probe commands and SFTP running on the same connection should not inherit some shell's forwarding, so a value like "follow the connection default" is not needed.

〔Decision〕**"Could not be set up" covers any preparation failure on the local side, as well as a refusal from the server**: for X11, no display, `xauth` missing or not runnable,
the temporary directory for `xauth` cannot be created, the server refusing `x11-req`; for the agent, the local agent cannot be reached (§7.1), the server refusing `auth-agent-req`.
**The connection itself breaking and the caller cancelling are still thrown as-is** — they are not swallowed as a forwarding failure: once swallowed, the next request fails on the same dead channel anyway and the real cause is lost.
The two are each handled by their own `FailureMode`; one failing to be set up does not drag down the other.

### 7.5.9 Reaching the local display through a connector

The local X server may live inside the caller's own process (for example an X server embedded in the host application).
Connecting to a local port is then only a detour, and it keeps a port open just for this. The forwarding options can carry a
**connector**: it is called once per incoming `x11` channel and returns a duplex stream that goes straight into the X server.

〔Decision〕The fake-cookie check stays **as is** (§7.5.2) — that layer guards against the remote side and has nothing to do with
how the local end is reached. Once the check passes, the cookie in the setup message is replaced with the caller-supplied
"local cookie" (empty if none was given) and the message is written into the stream.
Access control is the connector side's responsibility: the stream it hands out counts as an already-trusted local connection.

〔Decision〕**Trusted mode only.** Untrusted mode needs `xauth` to connect to the local display with full authorization and sign a
restricted cookie (§7.5.7), and there may be no display behind the connector for `xauth` to reach. When both are set, the X11
forwarding request fails (handled per the two cases of §7.5.8) instead of silently falling back to trusted mode — that would
quietly loosen the isolation the caller asked for.

〔Decision〕The screen number and diagnostics still come from the display address (`DISPLAY` or the one the caller gave), so a
display address is still required in connector mode.

〔Decision〕When the connector side is unavailable (the X server has stopped or been disposed), that `x11` channel is treated as
"local display unreachable": it is not counted as accepted, the channel is closed afterwards, and other channels on the same
forwarding are unaffected.

〔Decision〕**When the remote side sends `CHANNEL_EOF`, the connector's stream must also learn that the peer is done sending.**
With a local socket this is `shutdown(SEND)` (§2.2); a connector only hands over a stream, so another way is needed: if the
stream itself can close its write side alone (the in-process duplex stream), use that; otherwise **close the whole stream** —
in the X protocol a client that has finished sending has ended the connection; there is no "done sending, still waiting for a
reply" pattern, so falling back to a full close loses nothing. Skip this step and the X server never reads EOF: the remote
program exited long ago, yet its window stays up until the whole SSH session ends.

### 7.5.10 Error teardown

〔Decision〕Relaying an `x11` channel uses the same copy loop, so **a clean end and an error are kept apart exactly as in TCP forwarding** (§2.2):
a clean end is a per-direction half-close; when either direction fails, both sides are aborted together — with a local socket the display side gets an RST,
and the channel side gets a `CLOSE` without `EOF`.
A stream from a connector (§7.5.9) has no "reset" to offer, so abort degrades to closing the whole stream: the X server learns the connection is gone but cannot tell an error from a clean end.

## 8. Edge cases and errors at a glance

| Situation | Handling |
| --- | --- |
| Local port already in use | Throw `SshForwardException`, **leave no half-open listener** |
| `tcpip-forward` rejected | Throw; the message points out "the server may have disabled AllowTcpForwarding / GatewayPorts" |
| Channel open fails for a single connection | Raise the `Error` event, close that one inbound connection, **the forwarder keeps running** |
| A single connection fails while relaying | Both sides aborted together (local RST, channel `CLOSE` without `EOF`), `Error` raised, the forwarder keeps running (§2.2) |
| Accepting inbound connections fails repeatedly (e.g. EMFILE) | `Error` (`ForwardErrorReason.Accept`) each time; back off starting at 50 ms, doubling, capped at 1 second, then accept again (§2.4) |
| SSH session disconnected | Local/dynamic forwards close the listener and release the port; every forwarder's `IsActive` becomes false; in-flight connections are aborted as errors. The forwarder raises no separate `Error` for the disconnect (§2.4, §4.5) |
| Invalid SOCKS handshake | Close that one, count it in `errors`, the forwarder keeps running |
| Zero-length domain name in a SOCKS request | Reply `0x08`, close that one (§3.1) |
| Port 0 in a SOCKS request | Reply `0x01`, close that one (§3.1) |
| SOCKS handshake times out (30 seconds by default) | Close that one, raise `Error` (`ForwardErrorReason.SocksHandshake`), the forwarder keeps running (§3.3) |
| No matching forwarder for `forwarded-tcpip` | Reply `CHANNEL_OPEN_FAILURE(1)`; `(3)` when the connection has no remote forward at all |
| A forwarded channel arrives during a remote forward's disposal grace period | Accepted as usual if it matches (§4.3) |
| Invalid forwarding options (`MaxConnections` below 1, a listening port outside 0–65535, a non-positive SOCKS handshake timeout, a null bind address) | 〔Decision〕Throw `ArgumentOutOfRangeException` / `ArgumentNullException` when set; if anything fails after the listener is up, the listener is closed on the spot. 〔History〕They used to go unchecked: with `MaxConnections = 0` the listener was already up when constructing the forwarder threw, and the port stayed taken until GC |
| Starting a forward on a connection that is already disconnected (or disposed) | Fail outright: `ObjectDisposedException` once disposed, the recorded fault once declared dead — the same for local, dynamic and remote, and no listener is opened. 〔History〕Local / dynamic forwarding used to open the listener anyway and "succeed", returning a forwarder with `IsActive = false` |
| Concurrent connections exceed the limit (〔Decision〕default 1024 per forwarder) | Reject new inbound connections and raise `Error`; existing connections are unaffected. Local/dynamic forwarding: close that inbound connection, `Error` (`ForwardErrorReason.ConnectionLimit`) — 〔Decision〕the event fires **at most once per second**, carrying how many were rejected in the meantime; the error count in the metrics still records every rejection (〔History〕it used to fire for every rejection, so a burst of connections at the limit made the host push each one to the UI). A connection the peer resets before accept (`ConnectionReset` / `ConnectionAborted`) causes no backoff and no report — it is that connection's own business and the listener is fine (〔History〕it used to be reported as an accept failure, backing the whole listener off from 50 ms). Remote forwarding: reply `CHANNEL_OPEN_FAILURE(4)` (resource shortage, §4.1), and likewise count it in `errors` and raise `Error` (`ConnectionLimit`, also at most once per second); 〔History〕it used to refuse that channel only, with no event and no count |
| An event subscriber throws | Swallowed; other subscribers and the connection are unaffected (§5) |
| The local agent cannot be reached when agent forwarding is requested | Do not send `auth-agent-req`; throw or start normally per `FailureMode` (§7.1, §7.5.8); the reason code is carried over from the agent side (`AgentNotRunning` / `AgentUnavailable`) |
| The local agent cannot be reached when an `auth-agent@openssh.com` channel arrives | Reply `CHANNEL_OPEN_FAILURE(2)`, with the description saying only "local ssh-agent unavailable"; the session and the forwarder are unaffected (§7.1) |
| The local agent does not support session binding / drops the connection because of it | Forward as usual; on a drop, reconnect once without binding (§7.4) |
| The agent refuses to add a key with destination constraints (`5` / `28`) | Throw `SshAgentException` (`AgentRefused`), the message names "the agent may not support destination constraints"; **never** retry by adding it without the constraint (§7.3.2) |
| A destination-constrained key is used on a host, user or path it does not permit | The agent refuses to sign; authentication records `SkippedNoMaterial` for that credential and moves on to the next, a forwarding remote gets `FAILURE` (§7.3.2) |
| The remote sends an agent extension other than `session-bind@openssh.com` | Reply `FAILURE`; it never reaches the local agent (§7.4) |

〔Decision〕**A single connection's failure must never affect the forwarder itself.**
A tunnel has to be able to run for days, during which unreachable targets and reset connections are inevitable.
Treat those as fatal errors and the tunnel becomes unusable.
