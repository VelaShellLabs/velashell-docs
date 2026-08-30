# Tunnel Feature Planning (Plan)

> Updated 2026-08-30. The original items 1 through 4 under "To Implement", plus the three follow-up iterations (automatic reconnect recovery / traffic statistics / port-conflict precheck), have all been delivered. Remaining ideas are listed at the end.

## Implemented

- Three types are available: local forwarding (`-L`), remote forwarding (`-R`), and **dynamic forwarding (`-D` / SOCKS5)**. Choose the type from a dropdown at creation time. Dynamic forwarding automatically hides the target-host/port inputs.
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

## Remaining ideas

1. **Split traffic by direction**: `BytesTransferred` is currently a combined figure; showing the two
   directions separately would make it quicker to tell uploads from downloads.
2. **Rate**: only totals exist today, no instantaneous rate (KB/s); per-second sampling would give it.
3. **SOCKS5 authentication for dynamic forwarding**: the server-side handshake accepts no-auth only
   (it listens on loopback, so any local process can use it). If binding to `0.0.0.0` is ever supported,
   username/password authentication should be required alongside it.
