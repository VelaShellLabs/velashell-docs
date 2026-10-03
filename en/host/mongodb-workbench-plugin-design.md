# MongoDB Workbench: Plugin Design

> Written: 2026-10-02 · Code: `plugins/VelaShell.Plugin.Mongo` in `VelaShellLabs/velashell-plugins`
>
> Background reading: [`../templates/dev-guide.md`](../templates/dev-guide.md) (capabilities and rules),
> the Docker panel's `README.md` in the plugins repository (the precedent for opening from the command palette and managing its own connection targets),
> and [`VelaShell/DESIGN.md`](https://github.com/joesdu/VelaShell/blob/main/DESIGN.md) (design tokens and components).
>
> Design file: `velashell-plugin-mongo.pen` at the root of the plugins repository (23 boards: 00 cover … 22 states, feedback and confirmations).
> This document records **why** things are built the way they are; what you need to know to change the code lives in the plugin's `README.md`.

---

## 1. Summary

- **Shape**: the same route as the Docker panel. The manifest contributes one command (`contributes.commands` + `onCommand:velashell.mongo.open`);
  the assembly and the driver load only when the user picks "MongoDB: Open MongoDB workbench" in the command palette (Ctrl+P), which opens a
  document panel. Loaded in process. **The plugin manages its own connections**: saved connections are listed at the root of the object tree and
  created / edited in the board-10 dialog; MongoDB does not appear in the host's New Connection window or session tree. Only SDK surface that already
  existed in 2.0.6 is used (`minSdkVersion: "2.0.2"`, for `ISessionsApi`).
- **Engine**: the official MongoDB.Driver 3.x (Apache-2.0), shipped in the plugin folder; driver exceptions are turned into one plain sentence at the
  boundary and classified (unreachable / authentication failed / untrusted certificate / bad configuration / SSH jump not established), and the
  failure card offers buttons by class.
- **UI skeleton**: Navicat's object tree + large-icon toolbar + object tabs (objects / collection / query / pipeline / GridFS / design / monitor /
  slow queries / users), plus Compass's aggregation pipeline, schema analysis, index advice and GridFS.
- **Editing model**: grid / tree / JSON share one copy of the data; every edit goes into a **staging area**, the commands to run can be previewed
  before commit, commit is one BulkWrite with **optimistic concurrency**, and it can be undone for 10 seconds afterwards.
- **Guards**: the environment tag (development / testing / production) derives the read-only default and write confirmation; read-only is a
  client-side guard, and greyed-out actions come from `connectionStatus`.

## 2. Connecting: managed by the plugin

### Why not a host connection type

Board 10 "New connection · MongoDB" is a whole dialog: a segmented server shape, one member per row, side-by-side authentication, two "secure
channel" cards, and a side panel with a live connection string, step-by-step test results and discovered members. The host's New Connection window
only understands declarative fields (blueprint [07](../plugins/07-capability-apis.md): "settings forms are a declarative schema, not a control tree").
Drawing this board there meant adding a field-layout and "connection inspector" contract to the SDK and having the host implement it. That route was
tried (SDK 2.0.7) and withdrawn in full: carrying a public contract in the host forever for one plugin's form is worse than letting the plugin keep
the whole thing inside its own panel, the way the Docker panel does. The earlier approach — registering as a workspace connection type with no
connection entry inside the workbench — is retired with it.

### Where things live

| Concern | Approach |
| --- | --- |
| Entry | "MongoDB: Open MongoDB workbench" in the command palette; if the panel is already open it is brought forward. The leftmost toolbar button "Connect" and the "+" in the object-tree header create a connection; double-clicking a connection row connects it, and its context menu has connect / disconnect / edit / duplicate / delete |
| Saved connections | Plugin storage (`IPluginStorage`, key `connections`); settings are stored under the string keys in `MongoSettings`, the same ones parsing uses. The tree root lists every connection (grey dot: not connected, green: connected, red: failed); grouped ones sit under a collapsible group row; recently used ones come first |
| Password | The host's encrypted key store (`ISecretsApi`, keyed by connection id); never in the connection string and never in plugin storage. Leaving the password box empty while editing keeps it; switching to anonymous deletes it |
| Several connections at once | Each connected connection gets a session object, and that object is what its tabs receive — the same collection in two connections is two tabs; the toolbar, the read-only switch, the server badge and the status strip follow the connection of the active tab |
| SSH jump host | Pick one of the SSH connections saved in the host (`ISessionsApi.ListSavedAsync`). On connect the plugin asks the host to open it (`OpenAsync`; the reason is shown to the user verbatim), starts a forwarding port on the local loopback, and every driver connection is paired with a stream opened on the jump host through `IRemoteTunnelApi.OpenTcpAsync`. Connections through the same jump host share one session, which closes when the last one leaves — and only if the plugin opened it (the host refuses to close the user's own terminal sessions). The plugin never sees the jump host's credentials |
| Certificate trust | When a self-signed certificate blocks the connection, the failure card lists the certificate (subject, issuer, expiry, SHA-1) and offers "Trust this certificate and reconnect", which pins that one thumbprint |

| Board 10 | Implementation |
| --- | --- |
| Name / group / environment | The "Basic" row; the group can be an existing one or a new one; the environment is three chips with tone dots, and picking production turns on read-only and write confirmation (unless the policy was changed by hand) |
| Host list / SRV record / connection string URI | A segmented control; the host list has one host per row (the first is the connection's host) and "+ Add host"; SRV is a single domain field and turns TLS on by default; the URI form is the whole string |
| Replica set, default database, readPreference | One row split into thirds (hidden for the URI form, where the string decides) |
| Mechanism / authSource / username / password | The mechanism defaults to negotiating SCRAM-SHA-256, with SCRAM-SHA-1, X.509 and PLAIN/LDAP as alternatives; the "Key store" tag next to the password box says where it is kept |
| "Secure channel": SSH jump / TLS cards | The jump card's switch is disabled, with the reason written underneath, when no SSH connection is saved or SRV is selected (one already on can still be turned off); TLS, once on, adds CA and client-certificate fields |
| Side panel: connection string | Built by string work only and redacted (the password is always `****`), coloured by segment role, copyable; a half-typed URI is shown as typed, redacted |
| Side panel: test results / discovered members | Step by step: SSH tunnel (open the session + start forwarding) / TCP (on the jump host when tunnelled) / authentication / hello / privilege check; a read-only account is a **warning** on the privilege step; members come from `replSetGetStatus` (falling back to `hello` without the privilege) with roles and replication lag, and each role badge is put back on the matching host row. Changing any parameter discards the result |
| Side panel: three security-policy boxes | Read-only, confirm writes, disable `dropDatabase`; the first two have **no default** — a missing key follows the environment (on for production, off otherwise) |
| Footer: Test connection / Cancel / Save only / Save and connect | As drawn; for a connection that is currently connected, "Save and connect" disconnects first |
| (off-board) tuning | A collapsed "Advanced" section at the end: appName, connect timeout, default maxTimeMS, rows per page, schema sample size, extended JSON mode, direct connection, show system databases |

**Connecting / connection failed** (board 22, "connection states") follows the host's "tab first, session second" habit: connecting opens a
placeholder tab ("Connecting to X" + which jump host + Cancel), which becomes the default database's object list once connected; on failure it turns
into "Can't connect to X" + where it was connecting to + the reason in monospace + Edit connection / Close tab / Reconnect, with no modal dialog. The
panel has no host status bar to use, so it draws its own 24px status strip: current connection · scope · the active tab's status.

Two easy mistakes:

1. **Through an SSH jump host the connection must be direct.** The driver connects to the local forwarding port, but it then tries the replica-set
   members at their **real addresses**, which this machine cannot reach, and server selection times out. So tunnelled connections force
   `directConnection` and drop the replica-set name.
2. **The real reason behind a server-selection timeout is in the heartbeats.** On a first-connect failure the driver only says "server selection timed
   out"; authentication failures, TLS rejections and closed ports hide in each member's heartbeat exception. The plugin collects them before disposing
   of the connection and classifies them (authentication failure → go fix the password; TLS → offer trust when possible; anything else → unreachable),
   cutting off the full cluster description the driver appends to the message.

## 3. Boards and their implementation

| Board | Implementation |
| --- | --- |
| 01 grid · 02 JSON view · 13 tree view | Collection workbench: typed column headers, BSON colouring, inline editing, inspector (fields / JSON / collection info, per-field type menu), staging and paging; tree view with its context menu; inline JSON-card editing with value-distribution suggestions |
| 03 query editor · 14 explain | Parsing and running a mongosh subset, context-aware completion (including the fields a preceding `$group` outputs), diagnostics with quick fixes, code lenses, several result tabs; explain output flattened into a stage flow with candidate plans, ESR advice and hint comparison |
| 04 aggregation pipeline | Stage cards (enable/disable, drag to reorder, per-stage output preview), two-way text mode, code export in five languages, create view, `$out` / `$merge` |
| 05 document editor | Form / JSON / diff modes, minimal `$set` / `$unset`, client-side `$jsonSchema` pre-check, "redo on the latest version" after an optimistic-concurrency conflict |
| 06 GridFS | Virtual directories, revisions of the same file name, streaming upload and download, drag and drop, orphan-chunk check, metadata editing |
| 07 indexes · 08 schema · 16 validation | `$indexStats` usage and build progress, ESR suggestions from slow queries in `system.profile`; `$sample`-based analysis and a generated `$jsonSchema`; server-side rule pre-check (`$nor`), trial documents, version history |
| 10 new connection | See section 2 (the plugin's own dialog) |
| 11 server monitor · 15 slow queries | Delta sampling of `serverStatus` / `replSetGetStatus`, stacked bar charts, member cards, storage top list, inferred events; `$currentOp` with killOp, profiler level, slow queries grouped by query shape |
| 12 object list · 17 new collection | Navicat-style object page (stats, DDL, privileges); five collection kinds with a live command preview |
| 18 users and roles | Built-in role matrix, custom roles, authentication restrictions, effective-privilege preview; saving issues incremental grant / revoke commands |
| 19 export · 20 import · 21 data transfer | JSON / CSV / XLSX / BSON dump (mongodump compatible) / shell script; import dry-run with validator-aware issues; copying collections, buckets, indexes and rules across sessions |
| 22 states, feedback and confirmations | Connecting / connection-failed placeholder cards (see section 2), empty / no results / long-running query (killOp), read-only banner, undo toast, delete confirmation by typed name, edit-conflict dialog |

Interactions the boards do not draw but daily use depends on (following Navicat's habits):

- **Drilling into objects / arrays**: in the collection grid and the query result grid, double-clicking (or Enter, or "Open as table" in the context menu) an object
  or array cell replaces the grid with that level as a table — one row per array element numbered by its index, the union of fields as columns when elements are
  documents plus a "value" column for scalar elements; an object is a single row. A breadcrumb (`Document #3 › items › [2] › attrs`) jumps back to any level,
  and Backspace / Alt+← goes up one level and lands on the cell it came from. In the collection grid the drilled table stays editable: cell paths are absolute
  (`items.2.qty`), so staging, commit, undo and conflict handling are the same path as editing a nested field in the tree view; the bottom bar's "+ / −" add and
  delete array elements. Filter-by-value drops array indexes (`items.sku`) — an indexed filter is almost never what was meant.
- **Column widths**: every table's header dividers can be dragged, and double-clicking one fits the column to its content (measuring the header and the realized
  rows; rows a virtualized list has not created are not measured).
- **Collapsing the right-hand panel**: the collection grid's document inspector and the JSON view's outline can be collapsed with a button at the
  bottom-right corner, giving the data the full width; newly opened collection tabs remember the last choice.
- **Panes**: the object tree, the inspector, every detail side panel, query editor vs results, the explain side panel, each pipeline stage's editor vs preview,
  and the document editor's preview vs command can all be resized. A splitter is drawn as the original 1px line with a grab area 3px wider on each side.

## 4. Editing and write guards

- The **staging area** records field-level changes per `_id` (nested paths included), inserts and deletes; a commit is one BulkWrite. Each `updateOne` filter
  includes the original values of the changed fields (or `updatedAt`); `matched = 0` is treated as a conflict and opens a field-by-field "mine vs server" dialog
  instead of overwriting someone else's change.
- **Undo**: pre-images are kept before committing; the toast's "Undo (10)" rolls back with them.
- **Read-only**: every write path calls `EnsureWritable(db)` first; in read-only mode it shows a notice with "Unlock writes" (which asks for confirmation on production).
  Real read-only belongs in server-side privileges; this layer stops the classic accident of bulk-editing in a tab you thought was a test database.
- **Destructive actions** always use a red outline button and state the consequence; on production (or when what is deleted holds data) the name must be typed;
  `dropDatabase` can be disabled altogether.

## 5. Colour: the board palette mapped to host tokens

Board 09's BSON palette equals the host's terminal colours under VelaDark, so it maps directly to `VelaShellBlue / Cyan / Green / Yellow / Magenta`, `VelaWarning`
and `VelaTextTertiary`, and follows theme switches. The boards' `warningDim / errorDim / successDim / infoDim` have no host token, so the plugin derives them with the
host's rule (seed colour at a fixed alpha) and re-tints them on theme change. The only private colours are the monitor chart's four categorical colours
(`MongoChart1–4`): the host's semantic colours each carry a meaning (Warning means warning), and reusing them as categories would be misread.

## 6. Credential boundary

The plugin hands the password of each MongoDB connection to the host's encrypted key store (`ISecretsApi`); plugin storage holds only settings
without the password. The plugin still cannot read the host's session store: the SSH jump host is opened by asking the host for a saved connection by
id, and not one byte of the jump host's credentials passes through the plugin. The "target connection" of a data transfer is **another connection this
plugin currently has open**, or a connection string the user types in (which the user hands over themselves).

## 7. Testing

- Tests against a real MongoDB follow the repository convention of **skipping by environment** (Inconclusive when there is no `127.0.0.1:27017`); tests that write
  only touch their own temporary databases.
- **Screenshot regression**: the test host loads the host's VelaDark tokens and renders with Skia; one screenshot per board is written to disk and compared with the
  board PNG. `tests/VelaShell.Plugin.Mongo.Tests/seed-shop.js` creates a `shop` database shaped like the boards' data.
