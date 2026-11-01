# Tunnel Feature Planning (Plan)

> Updated 2026-10-24. The original items 1 through 4 under "To Implement", plus the three follow-up iterations (automatic reconnect recovery / traffic statistics / port-conflict precheck), have all been delivered. On 2026-10-24 four things the SSH library already had were wired up (live rates and current connections, per-tunnel rate limits, remote dynamic SOCKS with an allow list, Unix domain sockets), and the notice for a failed start-up auto-start gained "Retry". Remaining ideas are listed at the end.

## Implemented

- Four types are available: local forwarding (`-L`), remote forwarding (`-R`), **dynamic forwarding (`-D` / SOCKS5)** and **remote dynamic SOCKS (`-R` with no target, 2026-10-24)**. Choose the type from a dropdown at creation time. Dynamic forwarding automatically hides the target-host/port inputs.
- Select the remote host from saved SSH sessions in the explorer. Enter the service port manually.
- Inline tunnel actions: stop / start / delete. Deleting an active tunnel stops it automatically first.
- The form's "Cancel" action clears and resets it. Creation errors are shown inline.
- **Tunnel configuration persistence**: stored as a document per server in the SonnetDB `tunnels` collection (`docId = profileId`, through `IAppDataStore`). After an application restart, tunnels return as "stopped" and can be started with one click. Each server is restored only once per application run, and a restoration failure does not affect the panel.
- **Dedicated host session**: a tunnel's lifecycle is independent of terminal tabs. Creating or starting a tunnel while disconnected automatically establishes a background connection dedicated to tunnels. The background connection is automatically disconnected when the last tunnel for that server is removed.
- **Multi-device synchronization**: tunnel configuration is included in Gist cloud synchronization under the "connection configuration" scope (`plan.md` §13.C).

### Metered forwarding (host-owned data path, 2026-08-30)

The underlying library, Tmds.Ssh, performs the whole of port-forward pumping internally and
**exposes no connection or byte counters at all**, which is why `TunnelInfo.BytesTransferred` sat at
zero for so long. Producing statistics means owning the data path, so
`Infrastructure/Ssh/MeteredPortForwardHandle` replaced the old approach of returning the library's
object directly:

| Type | Listening side | Outbound | Notes |
| --- | --- | --- | --- |
| Local forward | The host's own `TcpListener` | `SshClient.OpenTcpConnectionAsync` (direct-tcpip) | Same shape as the library's internals; no extra hop |
| Dynamic forward | The host's own `TcpListener` | Same, after a SOCKS5 handshake negotiates the target | Server-side handshake lives in `Infrastructure/Ssh/Socks5Negotiation` (RFC 1928, CONNECT + no-auth only) |
| Remote forward | Server side, still opened by the library | The library forwards to a local ephemeral metering listener, and the host relays on to the real local target | One extra loopback copy buys the same statistics |

All three paths converge on one "listen → open outbound → pump both ways" loop that accumulates bytes
and connections along the way. The pump preserves **half-close semantics** (EOF in one direction sends
EOF onward: `SshDataStream.WriteEof` on the SSH side, `Shutdown(Send)` on the socket side); without
that, protocols that "send the request, shut down the write side, then wait for the response" would
read nothing back.

> 〔Follow-up〕After the switch to the SSH library VelaShell.Ssh, the host-owned data path above **has
> been deleted**: metering is built into the library, the three kinds of forwarding are handled by the
> library's `LocalPortForwarder` (`Start` / `StartDynamic`, including the SOCKS5 server) and
> `RemotePortForwarder` (`StartAsync`), and remote forwarding no longer relays through a local
> ephemeral listener. All that is left in the host is the thin adapter
> `Infrastructure/Ssh/LibraryPortForwardHandle`, which wires the library's connection counts, byte
> counts (sent plus received) and error events onto `IPortForwardHandle`.
> For the current state see "Port-forwarding tunnels" in [`architecture.md`](architecture.md) and
> [`ssh/spec/07-forwarding.md`](../ssh/spec/07-forwarding.md).

### Traffic statistics (2026-08-30)

- Each tunnel row shows `3 conns · 1.4 MB` inline; when connections are actively transferring, the
  live count is called out as well (`3 conns (2 live) · 1.4 MB`).
- The byte count is **upstream + downstream** combined, and counts only payload that actually crossed
  the forward — no SSH protocol overhead, no SOCKS handshake bytes.
- Readings are copied from the underlying handle onto `TunnelInfo` by `ITunnelService.RefreshStatistics()`,
  called from the panel's 5-second status timer. Stopping a tunnel takes a final reading before releasing
  the handle, so a "stopped" tunnel still shows the total it moved.
- This is the description row the fuXS7 design draft reserved.

### Automatic reconnect recovery (2026-08-30)

- The create/edit form carries a "reconnect automatically after a dropout" switch backed by
  `TunnelConfig.AutoReconnect` (persisted and cloud-synced with the rest of the configuration; older
  configurations missing the field deserialize as `false`). Tunnels with it on carry an `AUTO` badge.
- **Off by default**: an automatic redial can trigger a credential prompt, and can also retry endlessly
  against a server that really is down, so it is the user's call per tunnel.
- Once the panel's status timer detects that a background session dropped, it redials the host session
  for the opted-in tunnels and rebuilds their forwards from the original configuration. Dropout
  detection covers **every** server the panel holds, not just the selected one.
- Failures back off along 10s → 30s → 1min → 2min → 5min, so a genuinely offline server is not redialed
  every five seconds.
- **A tunnel the user stopped is never brought back up**: `TunnelItemViewModel.StoppedByUser` records
  "the user pressed stop" and automatic recovery skips it. A rebuilt tunnel is a new entry, so it
  naturally returns to a recoverable state.

### Port-conflict precheck (2026-08-30)

- Before creating a local or dynamic forward, the local port is checked against the system's TCP
  listener table. A hit throws `TunnelPortInUseException` and reports "local port 27017 is already in
  use by another program" instead of the low-level socket error only network programmers can read.
- Rule: same port number, and either side bound to "all interfaces", or both listening on the same
  address (the same port on different NICs can coexist).
- Remote forwarding listens on the server, so no local precheck applies.
- The race between the precheck and the actual bind is known and acceptable: the precheck explains the
  common case, and the underlying exception is still raised if the bind genuinely fails.
- The probe is injectable (the `isLocalPortInUse` parameter of the `TunnelService` constructor); without
  it, unit tests would depend on whether the machine running them happens to have a database listening
  on 5432.

### Live rates and current connections (2026-10-24)

- `IPortForwardHandle` exposes the rate in each direction and a snapshot of the connections, taken straight from the SSH library
  forwarder's `Throughput` (the average over the last three whole seconds) and `Connections`; `TunnelService.RefreshStatistics` copies
  them into `TunnelInfo` along with the counters, and stopping a tunnel zeroes the rates and empties the connection list (the cumulative
  bytes stay).
- A rate line sits under the statistics line: while running, `↑ 12.5 KB/s  ↓ 468.8 KB/s` (↑ sent into the tunnel from this machine, ↓
  received back from it).
- "› Open connections: N" is collapsed by default; expanded, it lists "source → target" and "open for … · ↑ bytes ↓ bytes" per
  connection, at most 20, with "N more not shown" for the rest; clients that connected over a Unix socket have no name, so the source
  reads "unknown source".
- **Refreshed every second only while the panel is open** (`TunnelPanelViewModel.IsOpen` is set by the host as the panel shows and hides,
  and the one-second traffic clock runs only then); liveness checks and automatic recovery stay on the 5-second tick.

### Per-tunnel rate limits (2026-10-24)

- `TunnelConfig.MaxBytesPerSecond` (null = unlimited); the form takes "Rate limit, KB/s per direction (0 = none)", 0–1048576 in
  steps of 64, stored as bytes / second; values that do not divide evenly round up when editing, so a limit is never quietly widened to 0 =
  unlimited.
- Each direction is limited separately, and all connections on the tunnel share the allowance; with a limit set, the rate line ends with
  "· Limit 512.0 KB/s", and only that half remains while the tunnel is stopped.
- Creating, starting, rebuilding after an edit, automatic reconnect and start-up auto-start all carry it through the same
  `CreateTunnelAsync` into both kinds of library forward options. The library fixes the allowance when the forwarder is built, so a change
  takes effect when the tunnel is restarted — running tunnels cannot be edited anyway.
- Bursts never exceed one second's allowance (since 2026-10-25): the library splits a large read into segments of at most 32 KiB and
  releases them one by one ([spec/07 §5.1](../ssh/spec/07-forwarding.md)), so the rate and per-connection byte counts on the panel climb
  steadily instead of stalling for seconds and then jumping.

### Remote dynamic SOCKS and the allow list (2026-10-24)

- A new type, `TunnelType.RemoteDynamicForward` (appended at the end of the enum: the local store keeps the numeric value, so the first
  three values stay put): a port on the server acts as a SOCKS5 proxy, through which remote programs have this machine connect to targets
  for them. The listen address / port on the server reuse the remote forward's `RemoteHost` / `RemotePort`, and `LocalPort` is stored as 0.
- The allow list `TunnelConfig.PermitRemoteOpen` is written like ssh_config's `PermitRemoteOpen` / the library's `RemoteOpenPolicy`:
  `host:port`, with `*` `?` wildcards in the host, `*` allowed as the port, IPv6 in brackets, separated by whitespace or commas; a lone `any`
  allows everything, and `any` mixed with other entries is an error. Names are compared as the remote side gives them, never resolved;
  a rule written as an IP matches only IP literals and a rule written as a name only DNS names (details in the SSH library's
  [spec/07 §4.6](../ssh/spec/07-forwarding.md)).
- **No default, never empty**: this turns the local machine into a SOCKS proxy for the remote side, so what may go out must be stated by the
  user. The form's "Create" stays disabled while it is empty and names a bad entry in red; the service layer (`Core/Tunnels/RemoteOpenRules`)
  likewise refuses an empty list or bad rules from hand-edited or synced profiles, instead of treating them as allow-all or deny-all.
- A request outside the list gets the SOCKS "not allowed by ruleset" reply from the library (`0x02`), and the tunnel row records a
  localized note that the target the remote side asked for is not on the allow list and was refused.
- In the form the local port field folds away, the host field is called "Listen address (on the server)", and the checkbox is "Listen only
  on the server itself (127.0.0.1)" (ticked by default); the list row gains an "Allowed: …" line.

### Unix domain socket forwarding (2026-10-24)

- Following the combinations the library supports: in a **local forward** each side may be a socket (local TCP / local socket → server TCP /
  server socket, all four work); in a **remote forward** each side may be one as well (server TCP / server socket → local TCP / local socket,
  all four work; since 2026-10-31 — before that the library only exposed "sockets on both sides", see
  [SSH spec 07 §4.4](../ssh/spec/07-forwarding.md)); dynamic and remote dynamic have no socket form.
- No new type: `TunnelConfig` gains two optional fields, `LocalSocketPath` / `RemoteSocketPath` (null = use host and port), so the shape of
  the existing types does not change. 〔History〕From 2026-10-24 to 10-31 a remote forward with only one side given was not built
  ("either ports on both ends or Unix sockets on both ends").
- The form: local / remote forwards gain a "Use Unix domain sockets" checkbox that shows two path fields (each may be left blank, and a blank
  side keeps using its port); the side with a path greys out its port, host and "lock to 127.0.0.1". Local paths must be absolute and at most 107 bytes in
  UTF-8 (103 on macOS); server paths must start with `/`.
- Before listening locally the path is prechecked: if a file is already there the tunnel is not built and **the file is not deleted for the
  user** (most likely a leftover from last time, but it may be another program's socket), matching the library's default
  `AllowSocketReplacement = false`; stopping the tunnel deletes the socket file it created. Windows supports AF_UNIX from Windows 10 1803,
  and the socket file inherits the directory's ACL.
- The list row writes paths into the route, such as "This computer listens on 127.0.0.1:2375 → server-side target /var/run/docker.sock".

### "Retry" on a failed start-up auto-start (2026-10-24)

- **Could not connect to the server** → Retry reconnects that server and builds all of its auto-start tunnels that are not up yet; **one
  tunnel failed** (for example, the port is taken) → only that one is retried; another failure is reported again, again with "Retry".
- A retry is a user action: the host-key prompt may appear as usual, and a server whose automatic recovery was paused because
  authentication needed a person authenticates again; if that tunnel was started, edited or deleted by the user in the meantime, Retry
  does nothing.
- A successful retry announces itself bottom right ("Tunnel “x” on srv is up."); notices from automatic reconnects (rebuilding after a
  dropout) that failed on the host key / Tailscale authentication carry no Retry — what they need is the user starting the tunnel from the
  panel and checking the prompt.

## Remaining ideas

1. **Split cumulative traffic by direction**: the cumulative `BytesTransferred` is still a combined figure; rates are already split by
   direction (see above).
2. ~~**Rate**: only totals exist today, no instantaneous rate (KB/s); per-second sampling would give it.~~ — implemented on 2026-10-24,
   see "Live rates and current connections".
3. **SOCKS5 authentication for dynamic forwarding**: the server-side handshake accepts no-auth only
   (it listens on loopback, so any local process can use it). If binding to `0.0.0.0` is ever supported,
   username/password authentication should be required alongside it.
