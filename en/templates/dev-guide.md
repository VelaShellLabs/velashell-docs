# VelaShell Plugin Development Guide (v1)

> Status: **Implemented and available with the repository**. This guide describes the currently implemented plugin system (dual host modes + full Avalonia UI); the long-term design blueprint for the complete plugin platform (permission system, plugin store, and more) lives in the **main repository** as documents 01–15 under
> [`docs-en/plugins/`](../plugins/). Unimplemented parts follow the blueprint and will be delivered in phases as needed.

## 1. Architecture in One Minute

- **Dual host modes** (selected by manifest `hostMode`):
  - `inProcess` (default): the plugin runs in its own **collectible AssemblyLoadContext** inside the host process, with zero IPC overhead; panels can be docked into the main window's tab area;
  - `isolated`: the plugin runs in a separate VelaShell.PluginHost process (named-pipe RPC), so crashes and hangs do not affect the host; panels are separate card windows (use `inProcess` for true docked panels).
  The **plugin source code is identical** in both modes.
- **Dependency isolation**: the plugin resolves its bundled dependencies (any NuGet packages) from its plugin directory according to its `deps.json`, without interfering with the host; only two categories of assemblies are forcibly shared with the loader to guarantee type identity:
  `VelaShell.PluginSdk` and `Avalonia*` (therefore the Avalonia version must match the host, see §5.9).
- **Narrow waist contract**: plugins reference `VelaShell.PluginSdk` (BCL dependencies only); UI code uses full Avalonia directly (compile-only reference). **Never** copy the SDK or Avalonia assemblies into the plugin directory.
- **Fault isolation**: in-process mode uses full-path guards (a single plugin is marked Failed, but infinite loops and runaway memory cannot be prevented); isolated mode uses the process boundary (even a hard failure only loses that plugin).
- **Zero startup overhead**: discovery only reads `plugin.json` and does not touch assemblies, while discovery and activation run on a background thread after the main window is displayed; when there are no plugins, the startup path adds only two directory-existence checks.

Related source:

Since 2026-08-27 the toolchain is split across three repositories, each versioned on its own cadence:

| Repository / location | Contents |
| --- | --- |
| [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) `src/VelaShell.PluginSdk/` | Contract: entry interfaces, capability interfaces, DTOs, and manifest models |
| [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) `src/VelaShell.PluginSdk.Testing/` | Test doubles: `TestPluginContext` and in-memory implementations of each capability |
| [velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) `src/VelaShell.PluginSdk.Build/` | The single NuGet package a plugin project references: MSBuild props/targets plus the bundled packer (`VelaShell.PluginSdk.Packer`, same repository; moved over from velashell-plugin-cli on 2026-09-11) |
| [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli) `src/VelaShell.Plugin.Cli/` | The `vela-plugin` command line tool (validate/pack/sign/dev mount) |
| **this repository** `src/VelaShell.Plugin.Templates/` | `dotnet new` templates (`velaplugin` / `velaplugin-ui` / `velaplugin-protocol` / `velaplugin-workspace`) |

The host-side implementation lives in the **main repository** [joesdu/VelaShell](https://github.com/joesdu/VelaShell):
`src/VelaShell.Infrastructure/Plugins/` (discovery/loading/activation/deactivation and capability bridging),
`src/VelaShell.Presentation/Plugins/` (bridge from command capabilities to the command registry), and
`src/VelaShell.PluginHost/` (isolated plugin host process: RPC proxy context, built-in Avalonia, dock embedding).

## 2. Quick Start

### 2.1 Where the First-Party Plugins Live

The officially maintained plugins (Redis / S3 / Telnet / Serial / MongoDB / Docker panel / RDP) live in
[VelaShellLabs/velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins), not in the main repository.
The exception is the AI plugin: it lives under `plugins/` in the main repository joesdu/VelaShell and is built and
released with the app (it borrows the host's AvaloniaEdit and can only load in-process; see `plugins/README.md` there).

**They take exactly the same path your own plugin does** — they reference
`VelaShell.PluginSdk.Build` from nuget.org, with no in-repository privileges. So §2.2 below is the
whole story; follow it and you are done. For real-world examples (protocol capabilities, panels,
third-party dependencies shipped alongside the plugin), read any project under `plugins/` there.

That repository defines two extra things that only concern **shipping with the main application**.
You will not need them for a third-party plugin, but you will run into them when reading its csproj files:

- `<VelaPluginShip>` (default `true`): plugins set to `false` are left out of the release artifacts (the `.vpx`
  files attached to a Release and the `plugins/` layout `build/PluginBundle.proj` collects) and only laid out locally (every plugin is `true` today);
- after each build the output is mirrored to `artifacts/plugins/<directory name>/` and to the
  application directory named by `VELASHELL_DEV_APP_DIR` — effectively a built-in `vela-plugin dev init`.

> **Directory name = id with dots replaced by hyphens** (`velashell.ai` → `velashell-ai`). macOS `codesign`
> treats directories containing dots inside an `.app` as nested bundles. Using the id unchanged as the directory name causes signing to fail directly
> (`bundle format unrecognized, invalid, or unsuitable`). The directory name **does not participate in any logic**.
> The host enumerates subdirectories and reads the id from `plugin.json`, so this is only a packaging-side naming convention.
> Plugins the user installs from a `.vpx` into the data directory are still named by id (those live outside the `.app` and are not signed).

### 2.2 Plugins Outside the Repository (Third-Party): from `dotnet new` to installed in five minutes

The SDK ships as NuGet packages, and a plugin project needs **exactly one** of them.

```bash
# One-off: install the templates and the CLI
dotnet new install VelaShell.Plugin.Templates
dotnet tool install -g VelaShell.Plugin.Cli      # optional, see below

# Create the project (id = acme.snippets)
dotnet new velaplugin -n Snippets --publisher acme --authorName "Your Name"
#   velaplugin            basic: entry class + one command
#   velaplugin-ui         adds an Avalonia panel (AXAML)
#   velaplugin-protocol   a terminal protocol (§5.13.1), with a test project and a local echo server
#   velaplugin-workspace  a workspace connection type (§5.14), with a test project and a local echo server
#   --hostMode inProcess|isolated   first two only; protocol / workspace templates are fixed to inProcess

cd Snippets
dotnet build -c Release -t:PackVpx               # → bin/vpx/acme.snippets-0.1.0.vpx
```

The generated `.csproj` has a single dependency:

```xml
<PackageReference Include="VelaShell.PluginSdk.Build" Version="2.1.4" />
```

That one package brings everything a plugin project needs: the `VelaShell.PluginSdk` contract assembly, **Avalonia pinned to exactly the host's version** (including its AXAML compiler), `EnableDynamicLoading`, `plugin.json` copied to the output, shared assemblies kept out of the plugin directory, build-time manifest validation, and the `PackVpx` target. Do **not** reference `VelaShell.PluginSdk` or `Avalonia` separately: a version mismatch fails the build with `VELA1001` instead of surfacing at runtime as a cross-load-context cast failure on the user's machine.

| Package | Referenced by | Purpose |
| --- | --- | --- |
| `VelaShell.PluginSdk.Build` | **the plugin project** | All of the above. Reference this one alone |
| `VelaShell.PluginSdk` | (flows transitively) | Contract assembly, BCL-only |
| `VelaShell.PluginSdk.Testing` | the plugin's **test project** | `TestPluginContext` and in-memory capability doubles |
| `VelaShell.Plugin.Cli` (`vela-plugin`) | developer machine | dotnet tool: dev inner loop, doctor, validate, pack, sign |
| `VelaShell.Plugin.Templates` | developer machine | `dotnet new` templates |

> **The SDK is not a dotnet tool.** All three `VelaShell.PluginSdk*` packages are ordinary NuGet packages consumed via `PackageReference`; only `vela-plugin` (`VelaShell.Plugin.Cli`) is a dotnet tool. And **packing does not require installing it** — the packer ships inside `VelaShell.PluginSdk.Build`, so `dotnet build -t:PackVpx` works out of the box. Install the global tool to validate, sign, inspect packages, run `vela-plugin doctor`, or set up the development inner loop (`vela-plugin dev init`) outside a build.

**Method 1: from the plugin marketplace (least effort)**.

```bash
vela-plugin install velashell.redis      # since SDK 1.5.0
```

Fetches the package by id from the [marketplace](http://market.easilynet.top), checks its digests and signature, and extracts it into `~/.velashell/plugins/<id>/`; restart the host to load it. `search` / `list` / `update` / `uninstall` round it out — see the [CLI manual](../cli/cli.md#2-installing-from-the-marketplace).

**Method 2: `.vpx` package**. Sidebar plugin icon → Plugin Management page → select the file with "Install .vpx…" (validates the container and manifest, guards against zip slip and zip bombs, extracts into the user directory, replaces the old version for the same id, and activates according to the activation policy). The command-line equivalent is `vela-plugin install <package.vpx>`. Uninstallation is also a one-click operation on the management page (deletes the directory and clears database data).

> The management page and the CLI write to the **same directory**. The only difference is that the management page also records a protected installation receipt for post-install tamper detection; the CLI cannot produce that receipt (the host instead records the contents it finds the first time it sees such a directory, and loads it after a restart), but it performs every pre-install check. Do not mix the two paths for one plugin — installing over a manager-installed plugin reads as "files changed" to the host. The trade-off is spelled out in the CLI manual.

`.vpx` is VelaShell's **own container format**, not a renamed zip — see §12 for the layout and signing.

**Method 3: Place the directory directly**. Put the build output (entry DLL + deps.json + bundled dependencies + plugin.json) in:

```text
~/.velashell/plugins/<plugin id>/                    (Windows/Linux/macOS)
```

Restart VelaShell to load it. Again: do not put `VelaShell.PluginSdk.dll` or `Avalonia*.dll` there (the loader forcibly shares its own copies; adding them only increases the package size).

> Built-in application plugins (`<application directory>/plugins`) are read-only, so the management page does not offer uninstallation; user-installed plugins
> (`.vpx` files or directories placed in the user directory) can be uninstalled.

### 2.3 Inner Loop and Debugging

Third-party plugins do not need the "pack → install → look" cycle. Three steps:

```bash
dotnet tool install -g VelaShell.Plugin.Cli   # one-off
dotnet build
vela-plugin dev init                          # writes the IDE launch profile
```

Then press **F5** in your IDE (profile `VelaShell`). The full command surface is in the
[CLI manual](../cli/cli.md); this section explains what it does and why.

#### The host announces itself

`dev init` does not guess where VelaShell is installed — three platforms mean three sets of
install locations, plus portable copies and self-updates that moved the binary. Probing logic
is long and perpetually wrong. Instead, **the host writes itself into `~/.velashell/host.json`
on every launch**: executable path, version, apiLevel, bundled SDK version, Avalonia version,
data root, PluginHost path.

```bash
vela-plugin hosts     # list the installations registered on this machine
```

So there is exactly one prerequisite: **VelaShell must have been started at least once on this
machine**. If it has not, point the tool at the binary with `vela-plugin dev init --exe <path>`.
When several installations coexist (release plus preview), the most recently started one wins
by default and `--host 1.5` picks a specific one.

#### The generated launch profile

```jsonc
"VelaShell": {
  "commandName": "Executable",
  "executablePath": "…/VelaShell.exe",
  "commandLineArgs": "--dev-root …/Snippets/bin/Debug --wait-debugger acme.snippets --data-root …/.velashell-dev"
}
```

| Argument | The problem it solves |
| --- | --- |
| `--dev-root` | Mounts the project output into the host. It **travels with the project**, not with the machine — two plugin projects, or two branches, never contaminate each other |
| `--wait-debugger` | An isolated plugin's child process suspends **before loading the plugin assembly** and waits for you to attach |
| `--data-root` | The debug instance gets its own data root |

The third one is easy to underestimate: you almost certainly keep VelaShell open all day, and
**a second instance sharing the data root hits the single-instance guard** (SonnetDB holds an
exclusive lock on its WAL). It shows "already running" and exits cleanly — which looks exactly
like a broken launch profile. With a separate data root both instances coexist, and your
throw-away debug sessions never pollute your real configuration. To deliberately test inside
your everyday configuration, pass `--shared-data` (and quit the everyday instance first).

All three arguments have environment-variable equivalents (`VELA_PLUGIN_DEV_ROOT`,
`VELA_PLUGIN_WAIT_DEBUGGER`); **arguments win**. The third source is
`~/.velashell/plugins.dev.txt` (written by `vela-plugin dev link`), which suits "keep this
plugin mounted in my everyday instance forever". All three sources merge; development roots
are scanned **after** the regular ones and first id wins, so a plugin you are developing never
displaces an installed plugin with the same id.

#### After you change code: reload, do not restart

```bash
dotnet build
```

Go back to the plugin manager page and press **Reload** on that plugin's row: deactivate →
unload the ALC / reclaim the process → **re-read the manifest** → load again. The manifest is
re-read too, so a changed version, command set or protocol tab shows up as well.

On Windows this used to be impossible: the ALC loads through `LoadFromAssemblyPath` and holds
the entry DLL open for as long as the plugin lives, which degraded the inner loop into
"close the host → rebuild → start again". Development plugins now **load from a shadow copy**
(`~/.velashell/dev-shadow/<id>/gen-N`, a new generation per load, old ones removed when
possible), so the project's `bin` can be rebuilt at any time. The production path does not use
shadow copies and behaves exactly as before.

To skip even the button: `vela-plugin dev init --watch` (i.e. `--dev-watch`) makes the host
watch the development roots and reload whenever the entry assembly's timestamp changes
(debounced by 1.5 s so the build can finish). It is off by default — file watchers misbehave on
network and shared drives, and that is not a cost everyone should pay by default.

> A development plugin's **disabled state** lives in `~/.velashell/plugins.dev.disabled`, not in
> the build output — otherwise the `.disabled` marker would sit in `bin` and produce the classic
> "but I rebuilt it, why is it still disabled?".

#### Breakpoints

| Plugin shape | How to debug |
| --- | --- |
| `inProcess` | The host process started by F5 already has the debugger attached, and the plugin is loaded into it, so breakpoints hit immediately — including the first line of `ActivateAsync`. With a debugger attached the host stretches the activation timeout from 10 seconds to 10 minutes |
| `isolated` | The plugin runs in a `VelaShell.PluginHost` child process. For plugins matched by `--wait-debugger <id>` the child suspends **before loading the plugin assembly**; its pid is shown on the plugin manager page, printed to the log, and written to `~/.velashell/logs/plugin-host-<id>.pid` |

For plugins matched by `--wait-debugger` the host also **relaxes the activation timeout and
stops the heartbeat** — otherwise a breakpoint freezes every thread in the plugin process, two
missed pings in a row kill it, and the symptom is "the plugin disappears the moment I hit a
breakpoint".

#### When something is wrong, ask doctor

```bash
vela-plugin doctor
```

One pass over: whether a host is registered, the three compatibility gates
(`apiLevel` / `minSdkVersion` / `minHostVersion`), whether the output directory contains
`plugin.json` and a `.deps.json`, whether `VelaShell.PluginSdk.dll` or `Avalonia*.dll` were
accidentally copied into the output, and whether the launch profile still holds a placeholder.
It exits with code 1 when it finds a blocking problem, so it fits in CI.

When you do not want to start the host at all, plugin logic runs in ordinary unit tests against
`TestPluginContext` from `VelaShell.PluginSdk.Testing` (see §7).

### 2.4 Starting from the protocol / workspace templates

When what you are building is a new protocol (the Telnet or serial kind, §5.13.1) or a workspace connection type (the Redis or MySQL
kind, §5.14), start from `velaplugin-protocol` / `velaplugin-workspace`; what they save you is exactly the parts hardest to get right:

```bash
dotnet new velaplugin-protocol  -n EchoProto --publisher acme   # terminal protocol
dotnet new velaplugin-workspace -n EchoDesk  --publisher acme   # workspace connection type
```

- **What you get**: the plugin project sits at the root (`dotnet build -t:PackVpx` and `vela-plugin dev init` work as with the other
  templates); `<Name>.Tests/` is a test project (MSTest + `VelaShell.PluginSdk.Testing`, the same version as `.Build`); `tools/EchoServer.cs`
  is a local echo server. The last two are kept out of the plugin build by `DefaultItemExcludes` and never reach the `.vpx`. There is **no
  .slnx at the root**: with both a .csproj and a .slnx there, `dotnet build` fails with MSB1011. The parameters follow the other templates
  (`--publisher` / `--authorName` / `--sdkVersion` / `--pluginId`); **there is no `--hostMode`**, because the protocol and workspace
  capabilities only exist in-process, and the manifest already says `minSdkVersion: "2.0.4"` (it uses `PluginIcon`).
- **Trying it**: `dotnet test <Name>.Tests` runs without a host; to see it in the host, run `dotnet run --file tools/EchoServer.cs` in
  another terminal (`--file` cannot be left out — the file's comments say why; it listens on 127.0.0.1:7007 by default), press F5 to start
  the debug host (§2.3), and create a connection of the new kind pointing at `127.0.0.1:7007`; what you type (or the lines you send from the
  panel) comes back.
- **What to replace** (marked ✏️ in the code): in the protocol template, `BuildDescriptor`, `ConnectionSettings`, `ProtocolTerminal` and the
  transport inside `ProtocolSession`; in the workspace template, `LineConnection` (swap in your client library — third-party exceptions must
  not cross the plugin boundary), `WorkspaceView` and `WorkspaceState`. **What to leave alone**: registration, re-registering on a language
  change, unregistering on deactivation; the disciplines of the read pump / write gate / disconnect-as-EOF / a close that wakes a pending read
  stay as they are.
- **The test project has to turn on `EnablePreviewFeatures` itself** (the templates already do): the contract assembly carries
  `RequiresPreviewFeatures`, and without it you get CA2252. The test output has **no Avalonia runtime** (the SDK package makes Avalonia
  compile-only, and that flows down the ProjectReference), so tests must not JIT into panel code — the workspace template's
  `WorkspaceDocument` deliberately touches no Avalonia types so unit tests can drive it fully.

## 3. Manifest (`plugin.json`) Reference

| Field | Required | Description |
| --- | --- | --- |
| `id` | ✓ | Globally unique, `[a-z0-9.-]`, must start and end with a letter or digit, ≤64 characters. All command ids use it as a prefix |
| `version` | ✓ | semver (`1.2.0` / `1.2.0-beta.1`) |
| `displayName` | ✓ | Display name |
| `entry` | ✓ | Relative path to the entry assembly, must end in `.dll`; absolute paths and `..` segments are rejected |
| `description` | | One-sentence description |
| `publisher` | | Publisher identity (will be bound to the signing key and take part in trust decisions) |
| `author` | | Author, shown on the plugin manager page (e.g. `"Joe <joe@example.com>"`, ≤128 characters, no control characters). When omitted the page falls back to `publisher` |
| `apiLevel` | | Defaults to 1; a generation higher than the host supports is marked Incompatible and not loaded |
| `minHostVersion` | | Minimum required host version; if unmet, the plugin is Incompatible |
| `activationEvents` | | Omitted or containing `"onStartup"` = activate at startup; containing only `"onCommand:<command id>"` = **lazy activation** (load/start the process only when the placeholder command is invoked; the corresponding placeholder must be declared in `contributes.commands`) |
| `contributes.commands` | | Declarative command placeholders `[{id,title,category}]`: available in the command palette during discovery; ids must be prefixed by the plugin id; on activation, the plugin should `Register` a real handler with the same id to replace the placeholder |
| `idlePolicy` | | `"keepAlive"` (default) / `"recyclable"`: in isolated mode, a process is reclaimed after continuous idleness (no RPC and no open panels, 15 minutes by default), while placeholder commands remain available for a later trigger |
| `homepage` / `license` | | Metadata |

JSON comments and trailing commas are allowed. Plugins that fail validation produce a readable reason in the log
(`[PluginManager] Rejected plugin at ...`).

## 4. Lifecycle

```text
Discovered ──activate (background batch after startup)──▶ Active ──host exits──▶ Deactivated
    │                                   │
    ├─ .disabled marker → Disabled      └─ loading/activation error, activation timeout (10s) → Failed (unload ALC)
    ├─ invalid manifest / id conflict → Invalid
    └─ apiLevel / minHostVersion mismatch → Incompatible
```

Contract essentials:

- `ActivateAsync` **must return quickly** (10-second limit): start long-running work in your own background task and use the `context.Shutdown` token to respond to shutdown.
- `DeactivateAsync` has an approximately 2-second limit (application exit path) and is abandoned on timeout. Commands and event subscriptions registered through the SDK are cleaned up automatically by the host; only your own resources need to be finalized.
- Entry type: exactly one public, non-abstract class with `[VelaPlugin]` that implements `IVelaPlugin`, with a public parameterless constructor.
- After deactivation or failure, the ALC is unloaded with `Unload()`: do not put your own types into host static fields or long-lived events, or the assembly cannot be reclaimed.

## 5. Capability API Reference (`IPluginContext`)

### 5.1 Log: Logging

```csharp
context.Log.Info("hello");
context.Log.Error("failed", ex);
```

Writes to the host Trace pipeline, automatically adds the `[Plugin:<id>]` prefix, and is thread-safe. It also goes into this plugin's own
log file `~/.velashell/logs/plugins/<id>.log` (rotated once at 1 MB), whose tail users can see under Plugin management → "View log". An
isolated plugin's `Console.WriteLine` and the stack trace an unhandled exception writes to stderr land in that file too; `Trace.WriteLine`
inside the plugin process is not collected, so anything users should see goes through `context.Log`.

### 5.2 Storage: Private Key-Value Storage

```csharp
int n = await context.Storage.GetAsync<int>("count", ct);
await context.Storage.SetAsync("count", n + 1, ct);
```

Data is stored in the host's **SonnetDB** (`plugin_data` collection, primary key `<plugin id>|kv|<key>`):

- **Strong isolation by plugin**: the capability instance carries only its own id prefix, so a plugin cannot read another plugin's data (the plugin id character set excludes the separator `|`, so the namespace cannot escape); in isolated processes, RPC routes to the same implementation and the plugin process never connects directly to the database;
- **Automatic cleanup on uninstall**: after the plugin directory is removed from `plugins/`, the host clears its database namespace and data directory as a whole on the next startup (`.disabled` means disabled, not uninstalled, so data is retained);
- A single value should generally be ≤256KB; write large data directly to files under `context.DataDirectory` (also swept on uninstall). A headless host without a database automatically falls back to JSON files in the data directory.

### 5.2b TimeSeries: Private Time-Series Store

Use it for data that is appended over time and looked up by tag (conversation records, metric samples, event streams); small configuration still belongs in Storage.

```csharp
ITimeSeries series = await context.TimeSeries.OpenAsync(new("chat_messages",
[
    TimeSeriesColumn.Tag("conv"),                                   // tag = index dimension
    TimeSeriesColumn.Field("role", TimeSeriesValueKind.Text),
    TimeSeriesColumn.Field("seq",  TimeSeriesValueKind.Integer),
    TimeSeriesColumn.Field("text", TimeSeriesValueKind.Text)
]), ct);

var clock = new TimeSeriesClock();                                  // strictly increasing timestamps
await series.WriteAsync(new(clock.Next(),
    new Dictionary<string, string> { ["conv"] = id },
    new Dictionary<string, TimeSeriesValue>
    {
        ["role"] = TimeSeriesValue.FromText("user"),
        ["seq"]  = TimeSeriesValue.FromInteger(0),
        ["text"] = TimeSeriesValue.FromText(message)
    }), ct);

IReadOnlyList<TimeSeriesPoint> latest = await series.QueryAsync(new()
{
    Tags = new Dictionary<string, string> { ["conv"] = id },
    Descending = false,
    Limit = 500
}, ct);
await series.DeleteAsync(new Dictionary<string, string> { ["conv"] = id }, ct);   // delete a whole series
```

The data goes into a time-series measurement in the host's **SonnetDB**, with the physical name `pts_<plugin namespace>_<short name>`:

- **Strict isolation by plugin**: the namespace is derived from the plugin id (with a hash fallback, so `a.b` and `a-b` do not collide),
  and a plugin can only see its own measurements; uninstalling drops everything under the prefix;
- **Same series, same millisecond = overwrite**: a series is the measurement plus the full set of tag values. For high-frequency writes always
  take timestamps from `TimeSeriesClock.Next()`; conversely, the rule can be used **on purpose** — writing "one summary per conversation" at a
  fixed timestamp naturally keeps only the latest one (that is how the AI plugin's conversation list works);
- **Query semantics**: filter by tags / time first, then the host sorts and takes `Limit`. With tens of thousands of matching points the scan
  is truncated, so narrow it with `Since` / `Until` or finer tags;
- **Quotas** (`TimeSeriesLimits`): ≤8 measurements per plugin, ≤32 columns per measurement, tag values ≤200 characters,
  text fields ≤1 MB, ≤1000 points per batch, ≤5000 rows per query. Measurement and column names are limited to `[a-z][a-z0-9_]*`, at most 40 characters;
- On a headless host without a database `OpenAsync` throws `InvalidOperationException` (it **never** drops data silently),
  and the plugin should degrade accordingly. Unit tests use `InMemoryTimeSeries` (the same validation and overwrite semantics).

### 5.3 Sessions: Session Enumeration (Redacted)

```csharp
IReadOnlyList<SessionInfo> sessions = await context.Sessions.ListAsync(ct);
SessionInfo? one = await context.Sessions.GetAsync(sessionId, ct);
```

`SessionInfo` contains only connection metadata (host/port/username/status/time), **never credentials**.
`SessionId` is the first parameter of other remote capabilities. v1 plugins cannot initiate connections.

### 5.4 RemoteFs: Remote Files (SFTP)

```csharp
var entries = await context.RemoteFs.ListDirectoryAsync(sid, "/var/www", ct);
RemoteFileEntry? stat = await context.RemoteFs.StatAsync(sid, "/etc/nginx/nginx.conf", ct); // missing → null
byte[] conf = await context.RemoteFs.ReadAllBytesAsync(sid, "/etc/nginx/nginx.conf", ct: ct);
await context.RemoteFs.DownloadFileAsync(sid, "/var/log/app.log", localPath, progress, ct);
```

- Reuses the user's established session channel and does not authenticate again; an invalid session throws `PluginSessionNotFoundException`.
- `StatAsync` returns `null` for a missing path (consistent with host semantics; do not use exceptions to test for existence).
- `ReadAllBytesAsync` has a 16MB limit by default; use `OpenReadAsync` (**read-only sequential stream**, process while reading without a temporary file) or `DownloadFileAsync` for large files. In isolated mode, `OpenReadAsync` fetches chunks over RPC and does not support Seek.
- The host throttles progress callbacks (≥100ms), so you can update state directly.

### 5.5 RemoteExec: Remote Commands (One-Off / Streaming)

A separate exec channel: it does not enter the user's terminal or pollute shell history and environment. Two forms:

**One-off** (probe commands) —

```csharp
ExecResult r = await context.RemoteExec.RunAsync(sid, "docker ps --format json",
    new ExecOptions { Timeout = TimeSpan.FromSeconds(10) }, ct);

if (!r.IsSuccess)                     // a non-zero exit code does **not** throw — a failed command is a normal result
{
    context.Log.Warn(r.FailureText);  // prefers the first line of stderr, falling back to stdout / the exit code
}
Parse(r.Output);                      // stdout and stderr are two separate streams
```

- The default timeout is 30s, capped at 10min; a timeout throws `TimeoutException`.
- **`Output` contains standard output only**; `Error` is standard error and `ExitCode` the exit code — all three are on
  `ExecResult`. Stop building `2>&1; echo $?` sentinels yourself: merging the two streams lets a single `WARNING:` line choke the
  code that parses `--format json`, and the sentinel breaks as soon as the user's login shell is fish/csh.

**Streaming** (long-running commands: `docker logs -f`, `docker events`, `tail -F`) —

```csharp
sealed class LineSink(Action<string> onLine) : IProgress<ExecOutput>
{
    public void Report(ExecOutput o) => onLine(o.Line);   // forward synchronously; order is arrival order
}

ExecStreamResult done = await context.RemoteExec.StreamAsync(sid,
    "docker logs -f --tail 200 web",
    new ExecStreamOptions { Timeout = null },             // null = no deadline; finish through the cancellation token
    new LineSink(AppendToView), panelClosedToken);
```

- Output is called back **line by line**, and the host does not throttle it (the value of logs is that they are immediate); batch it yourself if you need to.
- **Do not pass `System.Progress<T>`**: it `Post`s every callback to the synchronization context or the thread pool, so neither order
  nor thread is guaranteed any more — harmless for a progress percentage, a disaster for a screen of logs. The host calls `Report`
  sequentially on the thread that reads the lines, so as long as your implementation is synchronous, line order is guaranteed.
- When the cancellation token fires, the host sends `TERM` to the remote process and closes the channel, and the method throws
  `OperationCanceledException`; when `Timeout` expires it throws `TimeoutException`. **The plugin must hold that token and fire it.**
- At most `IRemoteExecApi.MaxConcurrentStreams` (4) streams per plugin can be in flight at once; beyond that the call throws
  `InvalidOperationException`. Streams have no deadline and each holds an SSH channel; without a cap, a plugin that forgot to cancel
  could exhaust the server's `MaxSessions` — and then what breaks is the user's connection, not just this plugin.
- **Isolated mode**: when the host reports `SupportsExecutionOperations` in the handshake (true for hosts since SDK 2.1.0), the
  cancellation token reaches the host through `exec/cancel` and a stream without `Timeout` has no deadline there either; against older
  hosts the isolated process cannot deliver the cancellation, so a stream without `Timeout` gets a two-hour deadline — use `inProcess`
  if you need immediate cancellation on old hosts. See [IPC §7.1](../plugins/05-ipc-protocol.md#71-execution-operations-sdk-210).

> Neither form suits interactive commands (those that need a pseudo-terminal or keyboard input) — that is the terminal tab's job;
> type the command in with `context.Terminal.WriteAsync` (requires user authorization).

### 5.5b RemoteTunnel: Raw Byte Streams to a Remote Socket / TCP Endpoint (SDK 1.2)

```csharp
await using Stream stream = await context.RemoteTunnel
    .OpenUnixSocketAsync(sessionId, "/var/run/docker.sock", cancellationToken: token);
// A remote TCP endpoint works too (the address is resolved from the **remote** side):
// await context.RemoteTunnel.OpenTcpAsync(sessionId, "127.0.0.1", 6379, cancellationToken: token);
```

**When you need it.** Both forms of remote execution are **text**: `RunAsync` decodes the whole output as UTF-8 into
one string, and `StreamAsync` calls back line by line split on `\n`. For a binary protocol that is not "a bit slower", it is
**silent data corruption** — UTF-8 decoding replaces invalid bytes with U+FFFD (irreversibly), and line splitting cuts a frame
in two at `0x0A`. The Docker Engine API's chunked transfer, the tar stream of `/archive` and the 8-byte multiplexed frames of
`/exec` all fall into this category. To speak such a protocol, use a tunnel.

- The returned `Stream` is readable and writable; `Dispose` closes the SSH channel and returns the quota — **the caller must dispose it**.
- The cancellation token applies only to **establishing** the tunnel; once the channel is up, reads and writes are governed by the
  caller's own tokens and the remote end. A long-lived connection that just sits there, like `docker events`, is the normal shape of a
  tunnel; giving it an overall deadline would only make the UI drop the stream mysteriously at minute N.
- At most `IRemoteTunnelApi.MaxConcurrentTunnels` (16) per plugin, for the same reason as streaming execution.
- **`inProcess` only**: what it hands out is a live stream, and proxying it across processes gains nothing beyond copying every byte
  once more. Calling it in an isolated process throws `NotSupportedException`.

> Unlike the host's local port forwarding, a tunnel **does not open a listening port on the local machine**; the stream goes only to the
> plugin that asked for it. When the other end is a root-equivalent socket, that difference is not an optimization but a precondition.

### 5.5c TerminalView: Borrowing the Host's Terminal Emulator (SDK 1.3)

A plugin that wants a **real terminal** (one that can run `top` / `vim` / `less`) does not have to write its own ANSI parser:

```csharp
if (!context.TerminalView.IsAvailable)
{
    // Older host or isolated process: fall back to line-based output rather than letting the button blow up.
    return;
}
// Must be called on the UI thread (views are constructed there anyway).
IPluginTerminalView view = context.TerminalView.Create(new() { ScrollbackLines = 5000 });
var host = new ContentControl { Content = (Control)view.Control };

// The remote size must follow the control, or vim keeps drawing at the old size and the screen comes out misaligned.
view.Resized += (cols, rows) => _ = session.ResizeAsync(rows, cols);

// One line wires up the duplex stream: reading in the background, rendering back on the UI thread, user keystrokes
// written back in order — the host does all of it.
await view.AttachAsync(session.Stream, token);
```

- The `Control` handed out is an Avalonia control (surfaced as `object` — the SDK layer deliberately knows nothing about UI frameworks,
  the same convention as the content factory of `IUiApi.ShowPanelAsync`).
- Its appearance **follows the host's terminal settings** by default (font, size, line height, colours, cursor, gutter).
  A user who has tuned the terminal font once should not have to do it again because the terminal is in a plugin panel.
- `AttachAsync` attaches one stream at a time; attaching another first disconnects the previous one. It does **not** dispose the stream
  passed in.
- `Dispose` destroys the control and disconnects the current stream.
- **`inProcess` only**: what it hands out is a live native control, which cannot be embedded across processes.
  Calling it in an isolated process throws `NotSupportedException` — ask `IsAvailable` first.

> How it divides the work with `ITerminalApi`: that one is a side channel onto **the host's existing sessions** (read the buffer, search
> the output, write back with authorization); this one is the plugin's **own** terminal. The former operates someone else's terminal, the
> latter owns one of its own.

### 5.6 Commands: Command Palette

```csharp
context.Commands.Register(new(
    $"{context.PluginId}.refresh", "Demo: Refresh", "Demo",
    async ct => { /* Runs on a background thread; exceptions are logged automatically */ }));
```

- The id must be prefixed with `<pluginId>.` (enforced by the host to prevent plugins from impersonating one another).
- After registration, the command appears in the command palette (Ctrl+P); the plugin is responsible for title localization (it can retrieve translations through `context.Host.Locale` and re-register on `LocaleChanged`).
- The command body runs on a **background thread**. Do not touch the UI; slow operations will not freeze the interface.
- All commands are automatically unregistered when the plugin is deactivated; the handle returned by `Register` can be used for early unregistration.

### 5.7 Events: Host Events

```csharp
context.Events.SessionConnected += s => context.Log.Info($"{s.Host} connected");
context.Events.ThemeChanged     += theme => ...;
context.Events.LocaleChanged    += locale => ...;
```

Handlers are invoked off the UI thread and **must return quickly without throwing** (exceptions are caught and logged); hand off expensive work to your own background task. Subscriptions are removed automatically on deactivation.

### 5.8 Host: Host Information

`context.Host.AppVersion / ApiLevel / Locale / Theme` (the latter two are live).

### 5.9 Ui: Panels (Full Avalonia)

Plugins use **full Avalonia** to design their own UI: choose compile-time AXAML or pure code, bring your own styles, resources, and localization, and add any third-party component packages (distributed in the plugin directory and isolated from other plugins by the ALC).

**The only hard constraint: the Avalonia version must match the host.** `VelaShell.PluginSdk.Build` already takes care of it: it hands Avalonia to the plugin project at an exact version and compile-only (the reference in the package carries `ExcludeAssets="runtime"`) — Avalonia DLLs are never copied into the plugin directory, and the loader shares the same set at runtime (in-process = the host's; isolated process = bundled with PluginHost). Which version is pinned is decided by `VelaAvaloniaVersion` in the SDK repository's `Directory.Build.props` (SDK 2.1.5 pins 12.1.4). **Do not add your own `PackageReference` to Avalonia in the plugin project**: if the resolved version does not match, the build fails with `VELA1001`.

Third-party packages based on Avalonia, such as control libraries, are referenced normally (their DLLs ship with the plugin). The exception is packages whose **assembly name starts with `Avalonia`** (such as `AvaloniaEdit`): the loader decides sharing by name prefix, so they fall back to the loading side's copy too — only ones the host itself ships can be used, and they must likewise be compile-only and match the host's version (the SDK exports `$(VelaSdkPinnedAvaloniaEditVersion)` for checking); see [SDK reference §7](../sdk/sdk-reference.md).

Opening a panel: the host invokes the content factory on the **UI thread**, and you only need to return your control:

```csharp
IPluginPanel panel = await context.Ui.ShowPanelAsync(
    new() { Title = "My Panel", DisplayMode = PanelDisplayMode.Document },
    () => new MyPanelView(context));   // A compile-time AXAML UserControl, or any Control
panel.Closed += () => ...;             // Triggered when the user closes it or the plugin is deactivated
await panel.CloseAsync();              // Programmatic close
```

The plugin chooses the display mode:

- `PanelDisplayMode.Document` — docked tab: enters the main window's tab area, and users can **drag it to any split position** or split it from the context menu, making it equal to terminal/SFTP tabs. **Only inProcess mode** provides true docking; isolated mode always uses a separate card window (cross-process dock embedding conflicts with tab switching and has been deprecated);
- `PanelDisplayMode.Window` — separate window: in-process uses a host-style custom-drawn card window; an isolated process uses the plugin process's own window (automatically follows the host's light or dark theme).

A docked tab can also pick its **placement** (`PanelOptions.Placement`; ignored in window mode):

```csharp
new() { Title = "AI Assistant", Placement = PanelPlacement.Right, PlacementRatio = 0.3 }
```

`Tabs` (the default) joins the current tab group; `Right`/`Left`/`Bottom`/`Top` split off a column at
the matching outer edge of the tab area, sized by `PlacementRatio` (its share of the tab area,
0.15–0.85, default 0.3). Placement goes through the same drag-and-drop docking path, so the result is
identical to the user dragging it there — dragging it back, splitting again, and closing all take the
ordinary route. `PlacementRatio` only sets the width *on open*: a width the user drags is not written
back, so a plugin that wants to remember the preference stores it itself and passes it in next time
(the AI plugin does exactly that — see "Side pane width (%)" on its settings page).

Window mode can also put **action buttons on the title bar** (`PanelOptions.TitleActions`; ignored
when docked): they sit immediately left of the minimize button, in the given order, in the same style
as the tool buttons on the main window's title bar. Use them for entry points that belong to the window
but do not deserve content-area space (the AI plugin's model-settings window opens its global settings this way):

```csharp
new()
{
    Title = "Models", DisplayMode = PanelDisplayMode.Window,
    TitleActions = [new PanelTitleAction(GearPathData, "Global settings", OpenGlobalSettings)]
}
```

The icon is lucide-style 24×24 **SVG path data**, not a resource key — an isolated process has no
access to the host's `Icon.*` resource dictionary; the host scales the stroke to the title-bar size.
The callback runs on the UI thread.

#### Tab icons (`PanelOptions.Icon`, SDK 2.0.4)

A docked tab carries a **generic plug** by default — the host does not know what your panel is for,
and it should not: a host-side "plugin id → icon" lookup table is a table third-party plugins can
never get into. Hand over your own glyph instead:

```csharp
new()
{
    Title = "AI Assistant",
    // lucide's bot, six strokes
    Icon = PluginIcon.Stroked(
        "M12 8V4H8 M6 8h12a2 2 0 0 1 2 2v8a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2v-8a2 2 0 0 1 2-2Z "
        + "M2 14h2 M20 14h2 M15 13v2 M9 13v2"),
    Placement = PanelPlacement.Right
}
```

`PluginIcon` is the **single entry point** for tab icons; protocols (`ProtocolDescriptor.Icon`) and
workspaces (`WorkspaceDescriptor.Icon`) take the same type. See
[SDK reference §3.6](../sdk/sdk-reference.md). The two factories map onto the two real cases:

- `PluginIcon.Stroked(path)` — a lucide stroke glyph; the view box is always 24, just copy the path;
- `PluginIcon.Filled(path, viewBoxSize: 1024)` — a brand logo: filled, and its view box is usually
  not 24. **Leave the view box out and it defaults to 24, so scaling a 1024 logo against 24 blows it
  up more than fortyfold** — that is usually what is behind an "icon does not show up" report.

**Window mode takes this icon too**: it is drawn at the far left of the custom title bar, both
in-process and in an isolated process. With several plugins' settings windows open side by side,
the icon in each title bar is what tells them apart — before this, in-process windows all carried
the same generic plug and isolated windows had no icon at all.

A malformed path will not take the UI down: if the host fails to parse it, it treats the icon as
absent and falls back to the generic plug.

⚠️ Do not confuse this with `TitleActions` above: that draws the **action buttons on the right of
the title bar**, always a 24×24 stroke glyph with no notion of fill or view box. This one draws the
tab, and the glyph at the **far left** of the title bar.

**Theme tokens: write `{DynamicResource VelaXxx}` to follow the host theme.** All of the host's `Vela*` design tokens (semantic brushes, font-size scale, and font families) are available to plugins and follow light/dark switching immediately:

```xml
<Style Selector="TextBlock.title">
  <Setter Property="Foreground" Value="{DynamicResource VelaTextPrimary}" />
</Style>
<Border Background="{DynamicResource VelaBgInput}"
        BorderBrush="{DynamicResource VelaBorderPrimary}" />
<TextBlock Foreground="{DynamicResource VelaAccent}" />
```

- In-process: controls are in the host visual tree, so tokens are naturally available;
- Isolated process: after the handshake, the host sends a token snapshot (resolved for the current light/dark variant) over RPC, and PluginHost injects it into its application resources; it sends the snapshot again when the theme changes, so DynamicResource refreshes automatically. Embedded font sections (`fonts:...#`) do not cross processes; font tokens carry only the system fallback chain.
- Common tokens: `VelaTextPrimary/Secondary/Muted/Tertiary`, `VelaBgSurface/Page/Input/Hover`, `VelaBorderPrimary/Secondary`, `VelaAccent`, `VelaWarning/Error/Info`, `VelaFontSize9..16`, `VelaUiFont/VelaUiMonoFont`. See `src/VelaShell.Controls/Themes/VelaTokens.axaml` and `src/VelaShell/Themes/{Dark,Light}Theme.axaml` in the main repository for the complete list.
- An unresolved token (due to a spelling error or an older host version) leaves the property at its default value and does not report an error. Provide semantic fallbacks for critical colors.

Discipline:

- The plugin directly handles control events and updates (standard Avalonia style; after `await`, execution returns to the UI thread automatically); update controls from background threads through `Dispatcher.UIThread`.
- Attach each control instance to only one panel; a panel is a live control and has no concept of refreshing the entire tree.
- An AXAML view calls the `InitializeComponent()` **generated by the Avalonia source generator**; do not hand-write
  `private void InitializeComponent() => AvaloniaXamlLoader.Load(this);` any more — overload resolution picks the hand-written one, the XAML
  still loads, but every `x:Name` field is null and the constructor throws an NRE on first use. (The `velaplugin-ui` template ≤ 2.0.1 does
  exactly that; the next release fixes it.)
- Localization is the plugin's responsibility: retrieve translations through `context.Host.Locale` and hot-update on `LocaleChanged`.
- The host automatically closes all of a plugin's panels when the plugin is deactivated.
- The smallest starting point is `dotnet new velaplugin-ui` (a compile-time AXAML panel + a command, see §2.2); a complete example at real scale (compile-time AXAML, bilingual copy, tunnelling over a saved SSH session): [`plugins/VelaShell.Plugin.Mongo`](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Mongo).

#### Posting to the message center (`PostNotificationAsync`, SDK 2.1.3)

Besides panels, a plugin can post a message **worth keeping and reading again** to the host's message center (the
bell in the sidebar) — the nightly backup failed, a certificate expires in 7 days, a background job finished and
needs you to confirm the result. Do **not** send progress, runtime state such as connected / disconnected, operation
receipts such as "copied", or debugging information there: the first belong in your own panel, the last in `Log`.

```csharp
if (context.Ui.SupportsNotifications)   // false on older hosts, where PostNotificationAsync throws NotSupportedException
{
    PluginNotificationOutcome outcome = await context.Ui.PostNotificationAsync(new PluginNotification
    {
        Id = "backup-failed:web-01",          // a stable de-duplication key within this plugin; do not randomize it
        Title = "Backup failed",
        Body = "The nightly backup of web-01 did not complete.",
        Level = PluginNotificationLevel.Warning,
        ActionCommandId = "acme.backup.open", // must be one of this plugin's commands
        ActionLabel = "View",
    });
}
```

- De-duplication comes before rate limiting: posting the same `Id` again returns `AlreadyPosted` (nothing added,
  nothing changed); each plugin gets at most 6 **new** messages per minute, beyond which the call returns
  `RateLimited` without throwing; when the user mutes the plugin on the plugin manager page the call returns `Muted`
  — do not retry under another id, and do not fall back to some other channel.
- Put the action command in the manifest's `contributes.commands`: messages survive restarts, so the plugin may not
  be active yet when the user clicks. There is deliberately no URL field — open web pages from the command.
- No new permission is needed: in-app notifications are already part of the normal permission `ui.contributions`.
  Using it means declaring `minSdkVersion: 2.1.3`.
- The full rules (length limits, retention and cleanup, `ui/notify` for isolated processes, `FakeUi`'s test members)
  are in [SDK reference §3.7](../sdk/sdk-reference.md#37-posting-to-the-message-center-sdk-213).

### 5.10 Secrets: Encrypted Secret Storage

```csharp
await context.Secrets.SetAsync("api-token", token);
string? saved = await context.Secrets.GetAsync("api-token");
await context.Secrets.DeleteAsync("api-token");
```

Difference from Storage: values are **encrypted by the host's secret protector before entering SonnetDB** (on Windows, a local key wrapped with DPAPI; primary key `<plugin id>|secret|<name>`, isolated by plugin and cleared on uninstall just like KV), and in isolated mode secrets are stored only on the host side. Always put API tokens and passwords here, never in Storage.
Suitable for small, short strings; if the protector is unavailable, the capability reports an error directly and never falls back to plaintext.

### 5.11 Terminal: Read/Search/Authorized Writes

```csharp
string tail = await context.Terminal.GetOutputAsync(sid, maxLines: 500);
var hits = await context.Terminal.SearchOutputAsync(sid, "error");         // Substring, case-insensitive
var rx   = await context.Terminal.SearchOutputAsync(sid, @"\d{3} error", isRegex: true);
await context.Terminal.WriteAsync(sid, "docker ps\n");                     // Triggers an authorization dialog
```

- Reading and searching use a **buffer snapshot** (scrollback plus the current screen, plain text without colors); regular expressions have a 1s timeout.
- **Writes require user authorization**: the host dialog offers four choices: Only this time / This run / Always / Deny; "Always allow" is persisted for the plugin in SonnetDB and can be revoked on the plugin management page. A denial throws `PluginPermissionDeniedException` (the plugin should degrade gracefully and not repeatedly bother the user).
- Writes go through the host's existing input serialization queue ("as if typed by the user"), with a limit of ≤4KB per call; a command executes only when a newline is sent.

### 5.12 Clipboard: System Clipboard

```csharp
await context.Clipboard.SetTextAsync(text);
string? current = await context.Clipboard.GetTextAsync();
```

Runs through the host main window (routed over RPC in isolated mode, with identical semantics). The clipboard often contains user passwords:
never log or transmit content that you read.

### 5.13 Protocols: Plugin-Provided Protocols (File / Terminal)

Lets a plugin provide a **new protocol** that sits on the connection page as an equal of SSH/SFTP/FTP. A protocol comes in two shapes that
share the same `ProtocolDescriptor` (tab, default port, connection form) and differ only in the implementation passed at registration:

| Shape | Registration | A session opens | Examples |
| --- | --- | --- | --- |
| File protocol | `Register(descriptor, IProtocolFileSystem)` | The dual-pane file browser | S3, WebDAV |
| Terminal protocol | `Register(descriptor, IProtocolTerminal)` | A terminal tab | Telnet, serial, raw TCP |

File protocols first:

```csharp
context.Protocols.Register(
    new ProtocolDescriptor
    {
        Id = context.PluginId,               // or $"{context.PluginId}.<sub-protocol>"
        DisplayName = "S3",
        Icon = PluginIcon.Stroked(CloudPathData),   // the session tab's icon; a generic plug when omitted
        DefaultPort = 443,
        HostLabel = "Endpoint",              // relabels the host / user name / password fields
        Features = ProtocolFeatures.ServerSideCopy | ProtocolFeatures.AnonymousAccess,
        Fields = [ new() { Key = "region", Label = "Region", DefaultValue = "us-east-1" } ],
        Actions = [ new("share", "Copy share link", ProtocolActionScope.File) ],
    },
    new MyFileSystem(context));
```

- **Implementing `IProtocolFileSystem` is enough**: its methods map one to one onto the host's internal remote-file contract, so the
  dual-pane browser, the transfer queue, throttling, drag and drop and conflict policies all work for you **with no changes**.
- **The connection form is declarative**: `Fields` describes keys, labels, shapes (text / password / boolean / integer / choice) and
  defaults; the host renders the controls and hands what the user entered back unchanged in `ProtocolConnectRequest.Settings`. **The plugin
  writes no connection-dialog UI code at all.** Fields marked `IsSecret` are encrypted at rest together with the password. Too many fields
  push the dialog off the screen: mark tuning parameters (chunk size, concurrency and the like — with sensible defaults, rarely changed) as
  `IsAdvanced`, and the host puts them on the connection dialog's "Advanced" page (the "General" page draws only the common fields) with the
  field count on the tab; when editing an existing profile, a dot appears on the tab as soon as any advanced field differs from its declared
  default.
- **Declare mutually exclusive fields with `VisibleWhen`, not with a line of small print**:

  ```csharp
  new() { Key = "masterName", Label = "Master name", VisibleWhen = new("mode", "sentinel") },
  new() { Key = "database",   Label = "Default database", VisibleWhen = new("mode", ["standalone", "sentinel"]) },
  ```

  The condition's shape is **deliberately closed**: the only test is "a key ∈ a set of values" — no expressions, no and / or / not. Once
  expressions come in, validation, localization and explaining "why did this field disappear" all get out of hand together — that is
  general declarative UI, which blueprint 08 explicitly does not do; a plugin that truly needs several conditions should split into two
  connection types. When the condition does not hold the field disappears from the form, but **its stored value is kept and passed back**
  (the same storage semantics as `IsHidden`): a user who filled in the master name in sentinel mode, switched to standalone for a look and
  switched back should not find that the UI wiped what they typed. `VisibleWhen` and `IsAdvanced` combine with **and**, and "condition does
  not hold" wins — a field that does not apply right now must not appear on the "Advanced" page either (nor count on its tab).
- **The context menu is declarative too**: `Actions` can be drawn in the very frame of the right click; on click the host calls
  `InvokeActionAsync`, and what happens is up to you (usually opening a `Ui` panel).
- **No need to throttle progress yourself**: the host already coalesces to ≥100 ms and keeps it monotonic under concurrent out-of-order
  reports, so report after every chunk.
- **Throttling comes from the host**: `await context.Protocols.GetTransferOptionsAsync()` returns the global bandwidth limits and the
  timestamp policy (they are user preferences, not something each protocol should configure); read it **at the start of each transfer**.
- **Activation runs on the thread pool**: the host pushes the lazy `onProtocol` activation off the UI thread with `Task.Run` (assembly
  loading and `ActivateAsync` are synchronous stretches), so a bit of blocking initialization in `ActivateAsync` will not freeze the UI —
  but the 10-second activation limit still applies.
- **Exceptions have conventions**: `ProtocolAuthenticationException` (the host shows the login dialog again),
  `ProtocolCertificateTrustException` (the host shows a trust prompt, records the fingerprint and reconnects), `ProtocolConnectionException`,
  `ProtocolUnsupportedException` (shown as "not applicable" rather than as a failure).
- **Optional interfaces to implement alongside**: the protocol implementation class may also implement the interfaces
  below. The host checks the type, uses what it finds and treats the protocol as not supporting what it does not,
  with everything else unaffected: `IProtocolStreamUpload` (2.0.6, writes a sequentially read stream as a remote file,
  used when a "remote + remote" dual-pane copy relays through memory), `IProtocolLastWriteTime` (2.1.3, sets the remote
  modification time, used by directory sync to align times after an upload) and `IProtocolChecksum` (2.1.3,
  server-side digests, `RemoteChecksumAlgorithm`; directory sync compares digests first and transfers verify with it).
  The rules for the last two: **a digest must come from the server itself** — never read a whole file back just to
  produce a value; if you implement it but this session cannot do it (no MFMT / HASH on the server, read-only object
  timestamps), throw `ProtocolUnsupportedException` — **do not succeed silently**; a single file that cannot be
  digested gets null. Entries can also report links: `RemoteFileEntry.IsSymbolicLink` / `LinkTarget` (2.1.3;
  `IsDirectory` / `Size` / `LastModified` describe the link target); **creating** links still has no contract.
  Declare `minSdkVersion` up to whichever surface you use. Details in
  [SDK reference §3.3d](../sdk/sdk-reference.md#33d-file-capabilities-for-plugin-protocols-sdk-213); the S3 plugin
  implements digests and deliberately not modification times, for reasons given in the README in the complete-example
  directory below.
- **To appear on the connection page before the plugin is loaded**, declare both in `plugin.json`:

```jsonc
{
  "contributes": { "protocols": [{ "id": "acme.storage", "displayName": "Acme", "defaultPort": 443 }] },
  "activationEvents": [ "onProtocol:acme.storage" ]   // loaded only when the user clicks this tab
}
```

> **Hard constraint: the protocol capability is `inProcess` only.** Protocols are a high-frequency channel on which the host calls back
> into the plugin (streaming reads included), while the isolated process's RPC only carries the plugin → host direction. A manifest that
> declares `contributes.protocols` and asks for `isolated` is rejected at discovery with the reason.
>
> **A protocol id cannot change once published** — it lands in users' session profiles, and renaming it means old profiles no longer
> recognize their protocol. The id must be **all lowercase** `[a-z0-9.-]`, ≤128 characters, and either equal the plugin id or start with
> `<plugin id>.`; manifest validation and the runtime `Register` share the same check (`PluginManifestReader.IsValidProtocolId`). Forcing
> lowercase removes case ambiguity — the id is compared in three places (the registry, the UI, the stored profile), and as soon as uppercase
> is allowed, `Foo.Bar` and `foo.bar` get judged "the same" in one place and "different" in another.

Complete example: [`plugins/VelaShell.Plugin.S3`](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.S3) (protocol + two management panels + 22 bucket settings).
File protocols have no template (S3 is the only first-party one, too few to distil a common skeleton from); the quickest start for a
terminal protocol is `dotnet new velaplugin-protocol` (see §2.4).

#### 5.13.1 Terminal protocols (`IProtocolTerminal`)

Protocols such as Telnet, serial and raw TCP have no file system; what they have is a **duplex byte channel**. Register it:

```csharp
context.Protocols.Register(
    new ProtocolDescriptor
    {
        Id = context.PluginId,
        DisplayName = "Telnet",
        DefaultPort = 23,
        // Login happens in-band (the peer prints login: itself): let the host fold away the user name / password fields.
        Features = ProtocolFeatures.AnonymousAccess | ProtocolFeatures.NoCredentials,
        Fields = [ new() { Key = "enterMode", Label = "Enter sends", Kind = ProtocolSettingKind.Choice, /* … */ } ],
    },
    new MyTerminal(context));      // IProtocolTerminal
```

Two interfaces are all you implement:

```csharp
Task<IProtocolTerminalSession> ConnectAsync(ProtocolConnectRequest request, ProtocolTerminalOptions options, CancellationToken ct);

// IProtocolTerminalSession
ValueTask<int>  ReadAsync(Memory<byte> buffer, CancellationToken ct);        // 0 = session over
ValueTask       WriteAsync(ReadOnlyMemory<byte> data, CancellationToken ct);
ValueTask       ResizeAsync(int columns, int rows, CancellationToken ct);
```

- **Move bytes only**: VT parsing, scrollback, search, session logs, session recording and ZMODEM all live on the host side; one more layer
  of interpretation in the plugin would only fight the host's terminal engine.
- **Normalize a dropped connection to EOF** (`ReadAsync` returns 0) — **do not throw**. Returning 0 is what leads to "the tab shows
  disconnected, press Enter to reconnect". The host has a safety net too: if the plugin throws anyway it is treated as EOF.
- **Serialize the write side yourself**: key presses, size reports (Telnet's NAWS) and negotiation replies are all writes, and interleaving
  them tears a frame in half. The read side belongs to the host's bridge alone and is never concurrent.
- **`ResizeAsync` is fire-and-forget**: the host does not wait for it (a window resize can send one every frame); if the protocol has no
  such mechanism, implement it as a no-op instead of throwing.
- **`DisposeAsync` must be able to wake a pending read**: when closing a tab the host cancels the read token first and then calls it in the
  background, but the plugin itself must not block forever on the close path either (a serial port's `Close()` can hang for good when
  hardware flow control is stuck).
- A terminal protocol has **no** SessionId: the SFTP panel, task manager, resource monitor and tunnels grey out for it automatically, and
  "run a command after connecting" is not sent (on Telnet the first thing you see is `login:`, and an injected command would be typed into
  the login prompt).
- **Re-register with the same implementation instance on a language change**: the host treats "the same id with a different implementation"
  as the old implementation going away and closes the open tabs under that protocol; re-registering the same id with the same instance counts
  as a replacement. **Drop the old registration handle — do not dispose it**: after the replacement it no longer stands for the entry in the
  registry.

Quickest start: `dotnet new velaplugin-protocol` (see §2.4).

Complete example: [`plugins/VelaShell.Plugin.Telnet`](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Telnet) (RFC 854 negotiation + NAWS + 8-bit transparency, no third-party dependencies).

### 5.14 Workspaces: Plugin-Provided Non-File Connections (Redis / MySQL / Kafka …)

The protocol capability (§5.13) assumes "this protocol looks like a file system". For the ones that **do not** — key-value stores, message
queues, databases — use the workspace capability: the connection dialog, encrypted credentials at rest, the login prompt, cloud sync, the
session tree and recent connections all reuse what the host already has, and **the plugin only hands over an Avalonia control**.

```csharp
context.Workspaces.Register(
    new WorkspaceDescriptor
    {
        Id = context.PluginId,               // or $"{context.PluginId}.<sub-type>"
        DisplayName = "Redis",
        // A brand logo is usually filled and its view box is not 24 — say both; see SDK reference §3.6
        Icon = PluginIcon.Filled(RedisLogoPathData, viewBoxSize: 1030),
        DefaultPort = 6379,
        HostLabel = "Server address",        // relabels the host / user name / password fields
        Fields = [ new() { Key = "mode", Label = "Deployment", Kind = ProtocolSettingKind.Choice, … } ],
        Features = WorkspaceFeatures.AnonymousAccess | WorkspaceFeatures.CertificateTrust,
        TrustedThumbprintSettingKey = "trustedThumbprint"   // must be an IsHidden field in Fields
    },
    new MyWorkspaceProvider(context));
```

The provider is called when the host opens a session and hands back a document:

```csharp
internal sealed class MyWorkspaceProvider(IPluginContext context) : IWorkspaceProvider
{
    public async Task<IWorkspaceDocument> OpenAsync(WorkspaceConnectRequest request, CancellationToken ct)
    {
        // request.Host/Port/Username/Password are **one-time** credentials; Settings already has defaults filled in.
        MyConnection connection = await MyConnection.ConnectAsync(request, ct);   // must be connected before returning
        return new MyDocument(connection);      // the host calls CreateView() once on the UI thread
    }
}
```

- **The form shares one declaration with the protocol capability** (`ProtocolSettingField`): text / password / boolean / integer / choice /
  **saved SSH profile**, six shapes; tuning fields are marked `IsAdvanced` for the "Advanced" page, and fields that only make sense in some
  mode are declared with `VisibleWhen` (see §5.13). **The plugin has no connection-dialog UI code at all.**
- **The "Test" button takes the plugin's own path**: the plugin's handshake is not SSH, and the host will not throw SSH at your port. It
  branches by declaration — the workspace shape really opens once with `OpenAsync` and immediately calls `DisposeAsync` (building and
  tearing down the tunnel with it), the file-system shape opens a session and closes it. So `OpenAsync` **must be callable repeatedly,
  disposable without a view and free of local resource leaks**. A real handshake may change remote state; do not promise no remote side effects.
  The message of an exception appears verbatim in the dialog, so write
  it so the user can fix the profile from it.
- **Declarative SSH tunnels**: declare `WorkspaceFeatures.SshTunnel` on the connection type and give it a field of shape
  `ProtocolSettingKind.SshSession`, and the host builds the SSH session and the local port forward **before** opening the session, handing
  the rewritten local endpoint to `OpenAsync` (the real target is in `WorkspaceConnectRequest.Tunnel`, for showing where it comes from); the
  tunnel is torn down automatically when the document closes. **So the plugin writes no SSH code and never sees a credential** — building
  the session has to go through the host's existing two-step authentication, fingerprint checks and ProxyJump chain, and "credentials never
  leave the host" is a hard rule.
- **Proposing a connection**: `context.Workspaces.ProposeConnectionAsync(...)` makes the host open its own "New connection" dialog
  pre-filled (for "the plugin discovered some service"). **A plugin cannot write the host's session store itself** — that is user data with
  credentials inside; it can only propose, and the user looks it over and presses Save. The proposed connection type id must belong to the
  plugin.
- **Exception conventions are exactly those of the protocol capability**: `ProtocolAuthenticationException` (the host shows the login
  dialog again), `ProtocolCertificateTrustException` (trust prompt, record the fingerprint, reconnect), `ProtocolConnectionException`,
  `ProtocolUnsupportedException`. **Always tell authentication failures apart** — only then does the host show the login dialog again, and
  "could not connect" is the least useful feedback for "you mistyped the password".
- **Document lifecycle**: the host calls `CreateView()` once on the UI thread (it returns a `Control`; throwing or returning a non-control
  only makes that one tab show a line of explanation and never takes the host down); `StatusChanged` drives the status dots on the tab and in
  the session tree; `ReconnectAsync` backs the tab's "Reconnect"; `DisposeAsync` runs when the user closes the tab or the plugin is
  deactivated.
- **Saving / restoring state**: the SDK has no document state contract (`IWorkspaceDocument` has only `CreateView` / `Status` /
  `ReconnectAsync` / `DisposeAsync`), and the host does not reopen these tabs after a restart. UI state to remember across sessions (drafts,
  recent sends …) goes into `context.Storage` keyed by **endpoint** (through a tunnel, the real target in `request.Tunnel`).
  `WorkspaceConnectRequest.SessionId` is freshly assigned on every open and cannot be the key. `OpenAsync` and the "Test" path (Open →
  Dispose) write no state; only a document that has read state back writes it.
- **Re-register on a language change, but reuse the same provider instance**: the registry treats "the same id with a different
  implementation" as the old implementation going away and tells the host to close every open document of that type — the user only switched
  the language, and the tabs should not all disappear.
- **To appear on the connection page before the plugin is loaded**, declare both in `plugin.json`:

```jsonc
{
  "contributes": { "workspaces": [{ "id": "acme.cache", "displayName": "Acme", "defaultPort": 6379 }] },
  "activationEvents": [ "onWorkspace:acme.cache" ]   // loaded only when the user clicks this tab
}
```

> **Hard constraint: the workspace capability is `inProcess` only.** The host asks the plugin for an Avalonia control to put into the dock
> area, and native controls cannot be embedded across processes (blueprint 08 dropped HWND adoption). A manifest that declares
> `contributes.workspaces` and asks for `isolated` is rejected at discovery with the reason.
>
> **The id cannot change once published**, with exactly the same rules as protocol ids (all lowercase, prefixed with the plugin id, ≤128
> characters); the two **share one check and one strip of tabs**, so within one manifest a workspace id must not collide with a protocol id
> either.

Complete example: [`plugins/VelaShell.Plugin.Redis`](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Redis) (keyspace browser + type details + declarative connection form).
Quickest start: `dotnet new velaplugin-workspace` (see §2.4; it comes with a field for going through an SSH tunnel — delete it by following
the comments if you do not need it).

#### 5.14.1 Workspace variants and custom pages (`VariantKey` / `Variants` / `Pages`)

A workspace often stands for a "family" of connection types (MySQL / PostgreSQL / SQLite …). Instead of registering one tab per
type, register one workspace and use **variants** to describe what the connection dialog looks like once a type is chosen.

**Variants (`WorkspaceDescriptor.VariantKey` / `Variants`)**

- `VariantKey` names **one field key** in `Fields`; that field's value is the "current type". `Variants` is a list of
  `WorkspaceVariant`; the one whose `Value` equals the field's value applies. When the profile has no value for the key, the field's
  `DefaultValue` is used (the same fallback chain as when the launch settings are assembled).
- A variant overrides only what it declares; **anything left `null` is inherited from the descriptor**:
  - Port (`DefaultPort`): the host follows the variant only while the port **is still a default value**; a port the user typed is left alone.
  - The host-field label.
  - `Features`: **replaced as a whole, not merged.** Once a variant sets `Features`, the entire capability set comes from the variant, so
    leaving a bit out turns it off. Network variants should be supersets of the descriptor's `Features`; a file-based variant (SQLite)
    uses `NoCredentials | NoEndpoint` and omits `SshTunnel` and `CertificateTrust`.
  - Whether credentials, the SSH tunnel and certificate trust are available follows the variant's `Features`; the connection dialog and the
    pre-open check read the same result.
- The variant field itself is **no longer shown as a row** in the dialog (see the type-picker step in
  [Interaction and UI Specs §13.1](../host/interaction-and-ui-specs.md)); it remains the carrier of the value and is persisted in
  `PluginSettings` as usual.

**Icon on the type-picker card (`WorkspaceVariant.Icon`)**

- Type `PluginIcon?`, the same shape as the descriptor `Icon` and the session tab; `null` falls back to `WorkspaceDescriptor.Icon`.
- The **name** on the card is not declared again on the variant: the host takes the `Label` of the entry in the `VariantKey` field's
  `Choices` whose `Value` matches.
- Whether the cards appear is not an SDK switch; the host decides: the `VariantKey` field is a `Choice`, not hidden, not free-text
  (`AllowsCustomValue` is `false`), with at least two candidates. If not, the dialog is exactly as before.
- Available since: 2.1.5.

**Custom pages (`WorkspaceDescriptor.Pages` / `ProtocolSettingPage` / `ProtocolSettingField.Page`)**

```csharp
new WorkspaceDescriptor {
    Pages  = [ new ProtocolSettingPage("database", "Database"), new ProtocolSettingPage("ssl", "SSL") ],
    Fields = [ new ProtocolSettingField { Key = "schema", Page = "database", /* … */ } ],
}
```

- `ProtocolSettingPage(Key, Label)`: `Key` is a stable identifier compared **ordinally** and independent of language; `Label` is localized
  by the plugin (the same way as a field's `Label` and `Choices`: re-register the descriptor when the locale changes).
- The order of `Pages` is the tab order, placed between "General" and "Advanced"; the labels of "General" and "Advanced" come from the
  host and are not in this list.
- Why key + label rather than an enum: "Database" and "SSL" are domain pages; the host cannot prepare a closed set of localized labels for
  arbitrary plugins, and every new page would need an SDK release.
- `ProtocolSettingField.Page`:
  - **Takes precedence over `IsAdvanced`**: a field with `Page` goes to that page and `IsAdvanced` is no longer consulted; `IsAdvanced`
    only decides whether a field **without** `Page` goes to "Advanced" or "General". "Advanced" is not made a page because it has
    host-specific behavior (count badge, modified-value dot, auto-expand when editing an existing profile), which custom pages do not have.
  - A key not in `Pages`: the host treats it as `null` and **does not report an error** (the same attitude as a `VisibleWhen` that
    names a missing key). Guard the spelling with a contract test in the plugin.
  - **ANDed** with `VisibleWhen`: the field is shown or hidden by its condition inside its page; a page whose fields are all invisible
    is hidden, and if the user is on it they fall back to "General".
  - Only effective on descriptors that declare `Pages`: `ProtocolDescriptor` (file / terminal protocols) has no `Pages` yet, so `Page` is ignored there.
- Available since: 2.1.5.

## 6. Isolated Process Mode (`isolated`)

Declare `"hostMode": "isolated"` in the manifest and the plugin runs in a separate
**VelaShell.PluginHost** process (one process per plugin, implementing the process design in 02/04/05):

```jsonc
{ "id": "acme.my-plugin", ..., "hostMode": "isolated" }
```

- **Zero source changes for the plugin**: the capability interfaces of `IPluginContext` are replaced with RPC proxies on the PluginHost side. This fulfills the SDK's transport-independent design. Change `hostMode` to switch modes.
- **Transport**: named pipe (random name + one-time token + current-user-only access; on macOS/Linux, .NET named pipes use Unix Domain Sockets underneath and are naturally cross-platform). The protocol is a lightweight bidirectional RPC using length-prefixed JSON (see the implementation notes in 05; the dependency tree deliberately does not introduce StreamJsonRpc or MessagePack).
- **Isolation benefits**: a plugin crash or hang affects only itself. Unexpected exits trigger **automatic restart** with backoff (1s → 5s → 30s by default; more than 3 failures within a 5-minute window is marked Failed and automatic recovery is abandoned); two consecutive missed heartbeat responses (30s by default) mark the process hung and force-kill/restart it; when the host exits, the plugin process exits automatically under parent-process supervision and never remains as an orphan.
- **Credentials do not leave the main process**: the plugin process receives only the pipe name and token; SSH keys and passwords never cross the process boundary.
- **Capability differences** (compared with in-process mode):

| Capability | Isolated mode |
| --- | --- |
| Sessions / RemoteExec / RemoteFs | ✅ RPC proxies, identical semantics (transport uses same-machine file paths, and progress returns through notifications) |
| Ui (full Avalonia) | ✅ Avalonia is built into the plugin process and windows are fully functional (software rendering saves memory by default; set `VELA_PLUGIN_GPU=1` to enable it); `Vela*` theme tokens are sent over RPC and are equally available |
| Commands / Events | ✅ Registry remains in the host; triggers and events return through notifications |
| Storage / Log | ✅ KV is routed over RPC into the host's SonnetDB (the plugin process does not write local files); logs are forwarded to the host with local Trace as a fallback |
| Secrets / Clipboard | ✅ Routed over RPC to execute on the host (secrets are encrypted and persisted only on the host side) |
| Dock into the main window's tab area | ❌ **Always a separate card window** (stable and consistent with the main program). Cross-process dock embedding (HWND adoption) has fundamental tension with dock tab reparenting (stuttering and windows floating out), and is **deprecated**; use inProcess for true docked tabs. The stable cross-platform solution is the shared-memory surface in blueprint 08 (future) |
| `CancellationToken` propagation across processes | ⚠️ Generally not propagated; both sides use timeouts as a fallback (`exec` follows `ExecOptions.Timeout`). **Streaming / sudo execution is the exception**: when the host reports `SupportsExecutionOperations` (since SDK 2.1.0) cancellation reaches it through `exec/cancel` and streams have no deadline; on older hosts a stream without `Timeout` gets a two-hour deadline (see §5.5) |

- **Selection guidance**: use the default `inProcess` for first-party or trusted plugins (zero IPC overhead and draggable docked panels); use `isolated` for third-party or experimental plugins (one extra process provides crash isolation, with panels displayed in separate card windows).

## 7. Testing Plugins

Reference `VelaShell.PluginSdk.Testing` to write pure in-memory unit tests without the host:

```csharp
using VelaShell.PluginSdk.Testing;

[TestMethod]
public async Task Refresh_ListsContainers()
{
    using var ctx = new TestPluginContext();
    SessionInfo session = ctx.FakeSessions.AddConnected(host: "prod-1");
    ctx.FakeRemoteExec.Handler = (_, cmd) => cmd.StartsWith("docker ps") ? "abc123 nginx" : "";

    var plugin = new DemoPlugin();
    await plugin.ActivateAsync(ctx, CancellationToken.None);
    await ctx.RecordingCommands.RunAsync("velashell.demo.refresh");

    Assert.IsTrue(ctx.CollectingLog.Entries.Any(e => e.Message.Contains("nginx")));
}
```

Test doubles: `CollectingLogger`, `InMemoryStorage`, `FakeSessions`, `FakeRemoteFs`
(an in-memory path tree with semantics aligned to the real implementation), `FakeRemoteExec` (scripted responses),
`RecordingCommands` (use `RunAsync` to drive command bodies), `TestHostEvents` (the `Raise` method),
`TestHostInfo`, `FakeUi` (records panels and lazy content factories), `FakeSecrets`, and `FakeClipboard`.

The `<Name>.Tests` that `velaplugin-protocol` / `velaplugin-workspace` generate are complete examples of this section (descriptor checks at
activation, form keys matching parse keys, same instance across a language change, unregistering on deactivation, sending and receiving
against the local echo server, disconnect as EOF / going Faulted and recovering on reconnect).

## 8. Deployment, Disabling, and Troubleshooting

| Item | Location/Method |
| --- | --- |
| Built-in application plugins | `<application directory>/plugins/<id with dots replaced by hyphens>/` (for example, `plugins/velashell-ai/`; see §2.1) |
| Manually installed plugins | `~/.velashell/plugins/<id>/` (this is outside the `.app` and does not participate in signing, so the directory is still named by id) |
| Plugin data | KV/secrets in the host's SonnetDB (`plugin_data` collection); files in `<data root>/plugin-data/<id>/` |
| Uninstallation cleanup | Remove the plugin directory from `plugins/` → the next startup automatically clears its database data and data directory as a whole (`.disabled` only disables it and retains data) |
| Disable one plugin | Place an empty `.disabled` file in the plugin directory |
| Disable all plugins (troubleshooting) | Environment variable `VELASHELL_DISABLE_PLUGINS=1` |
| Id conflict | First match wins according to root-directory order (application directory first); the later one is marked Invalid |
| Logs | Debug output/Trace, with `[PluginManager]` and `[Plugin:<id>]` prefixes; each plugin also has its own `~/.velashell/logs/plugins/<id>.log` (`context.Log`, the isolated process's stdout / stderr, the activation / failure / crash entries the host records), whose tail is shown under Plugin management → "View log"; deleted on uninstall |

## 9. Performance and Behavior Discipline

The host is a terminal application that is extremely sensitive to memory and latency. Plugins must follow these rules:

1. **Do not block**: `ActivateAsync` must return in seconds; event handlers and command bodies must not wait on synchronous I/O.
2. **Do not poll**: use events instead of timers; when a timer is truly necessary, use an interval of ≥5s and stop it immediately when `Shutdown` is triggered.
3. **Use only the `Ui` capability for UI**: the host invokes panel content factories on the UI thread; command bodies and event handlers run on background threads, and controls must be updated through `Dispatcher.UIThread`. Reflection into the host's internal UI is not covered by compatibility guarantees.
4. **Manage memory**: download large files to disk instead of reading them into memory; cap caches; release everything on deactivation (otherwise the ALC cannot be reclaimed and memory grows steadily).
5. **Be friendly to remote systems**: combine probe commands into one execution (one exec with multiple output sections) instead of sending a command to the remote host every second.

## 10. Versioning and Compatibility Guarantees

- **apiLevel (currently = 1)**: within a generation, the SDK only adds and never changes or removes items (interface methods, DTO fields, and manifest schema). Breaking changes raise apiLevel; the host rejects plugins from a higher generation and gives a clear message.
- **minHostVersion**: declare this when the plugin depends on newer host capabilities. An older host marks the plugin Incompatible instead of allowing a runtime failure.
- **SDK package version vs. assembly version**: the five SDK packages are versioned **independently of the host** and released on their own cadence. On the assembly side:
  - `AssemblyVersion` is `<major>.0.0.0` and **moves only with the major** — plugins bind to that identity at compile time, so moving it per patch would force every compiled plugin to rebind for nothing;
  - `FileVersion` and `InformationalVersion` carry the full version (the latter including the prerelease suffix), so the Explorer property page and `vela-plugin` report the real version rather than being stuck at 1.0.0;
  - the rule is **SDK major == apiLevel**. A major bump means the contract broke, so `apiLevel` goes up with it and an older host rejects the plugin at **discovery** time with a readable reason, instead of throwing an assembly binding error at load time.
- **Host-mode independence**: capability interfaces are transport-independent. Switching `hostMode` between inProcess and isolated requires no plugin source changes (this is implemented); the only exception is behavior that differs in isolated mode, as shown in the capability differences table in §6.

## 11. Deliberate Gaps from the Long-Term Blueprint

| Blueprint capability | v1 status |
| --- | --- |
| One process + IPC per plugin (02/04/05) | **Implemented** (`hostMode: "isolated"`, see §6): named pipes + lightweight RPC + heartbeat + automatic crash-restart backoff |
| Permission system + Broker (06) | Not implemented: v1 targets first-party and self-installed plugins; installation implies trust |
| UI contribution points / VelaUI (08) | Available: command-palette commands + full Avalonia panels (dockable tabs in inProcess; always separate card windows in isolated processes) + plugin management page. The VelaUI declarative tree is **not being pursued** by user decision; cross-process dock embedding is deprecated (see the notes in 08); sidebar/status-bar mounting points are deferred |
| `.vpx` packaging / signing / store (03/10) | **Packaging and signing are implemented** (own container + ECDSA signatures, see §12); **the marketplace has a client**: `vela-plugin install/search/update/list/uninstall` against [market.easilynet.top](http://market.easilynet.top) or a self-hosted source (see the [CLI manual](../cli/cli.md#2-installing-from-the-marketplace)). **A marketplace UI inside the host is still future work** |
| Activation events / lazy activation (03) | **Implemented**: `onStartup` / `onCommand:<id>` + `contributes.commands` placeholders; other event types (onSessionConnect/onFileOpen, etc.) are deferred |
| Idle reclamation (04) | **Implemented** (isolated mode + `idlePolicy: "recyclable"`) |
| secrets / clipboard capability domains (07) | **Implemented** (§5.10/§5.11; without a permission system, installation implies trust) |
| terminal / localFs / audio / ai and other capability domains (07) | Not opened; any new capability domain must be added back to the blueprint and only add to the API without changing it |

Discipline for adding capabilities: first record them in this file and the corresponding blueprint document, then add the interface to `VelaShell.PluginSdk`; within the same apiLevel, only add and never change.

## 12. The `.vpx` Package Format and Signing

`.vpx` is VelaShell's **own container format**, not a renamed zip: general-purpose archivers
cannot open it, and the host refuses to install a plain zip. The implementation lives in
[velashell-plugin-sdk](https://github.com/VelaShellLabs/velashell-plugin-sdk) `src/VelaShell.PluginSdk/Packaging/VpxContainer.cs`; reading (host install) and writing
(`vela-plugin pack`) share that one file, so "the tool produced a package the host will not take"
cannot happen.

### 12.1 Layout

Little-endian; the header is a fixed 64 bytes:

```text
Offset  Size  Contents
0       4     Magic 56 50 58 1A ("VPX" + 0x1A)
4       2     Container format version (currently 1)
6       2     Flags (bit0 = payload masked, bit1 = signature block present)
8       8     Payload length in bytes
16      32    Payload SHA-256
48      8     Mask nonce
56      4     Header CRC32 (over the first 56 bytes)
60      4     Reserved
64      N     Payload: zip bytes (transformed when masking is on)
64+N    4+M   Optional signature block: int32 length + UTF-8 JSON
```

- The trailing `0x1A` is the DOS end-of-file marker, borrowed from PNG: `type` / `cat` on the
  package stops there instead of spraying a screenful of noise.
- **Masking** XORs 32-byte blocks with `SHA-256(nonce ‖ block index)`. It is self-inverse and
  randomly addressable, which is why the payload stream stays seekable (a hard requirement of
  `ZipArchive` in read mode) while `PK\x03\x04` is nowhere to be found in the file.

> To be explicit about the boundary: **the magic bytes and the mask are format identification and
> a guard against mistakes, not a security boundary.** A plugin is native executable code, so
> anything needed to "decrypt" it is necessarily on the client; a determined person can always
> peel the payload out. Real integrity and provenance come from SHA-256 (corruption and truncation)
> and the signature (tampering and impersonation).

### 12.2 Signing

The algorithm is **ECDSA P-256 + SHA-256** over the 64-byte header — and since the header carries
the payload length and digest, that is equivalent to signing the whole package. Ed25519 was
deliberately not used: it is not in the BCL, and pulling in a third-party library would break the
"the contract assembly has no heavyweight dependencies" rule (blueprint 10 §1 originally specified
Ed25519 and has been amended).

```bash
vela-plugin keygen -o acme.pem                    # create a key pair, print the public key (Base64 SPKI)
vela-plugin pack bin/Release/net11.0 -k acme.pem  # pack and sign
# or during a build: dotnet build -c Release -t:PackVpx -p:VelaSigningKey=acme.pem
vela-plugin verify pkg.vpx -k <public key base64> # check the payload digest and the signature
vela-plugin info   pkg.vpx                        # header, signature state and manifest
```

The four verdicts and how the host acts on them:

| Verdict | Meaning | Host behaviour |
| --- | --- | --- |
| `Unsigned` | No signature block | The plugin manager asks first; allowed once only after the user clicks “Install anyway”; rejected when `RequireTrustedPackageSignature` is on |
| `Trusted` | Valid signature whose key is in the trusted set (`TrustedPackageKeys` plus publishers the user has trusted in the plugin manager) | Allowed, with a log line |
| `Untrusted` | Valid signature, key not in the trusted set (the set starts empty, so a signature seen for the first time lands here) | The plugin manager shows the public-key fingerprint; allowed only after the user clicks “Trust publisher and install”, and the key enters the local trust store once the install has succeeded; rejected when `RequireTrustedPackageSignature` is on |
| `Invalid` | Signature block corrupt or verification failed | **Always rejected**, regardless of policy — that is tampering, which is far worse than "unsigned" |

The trusted set and the strict switch live on `PluginManagerOptions` (`TrustedPackageKeys` /
`RequireTrustedPackageSignature`); out of the box the set is empty and the switch is off.

On the command line, `vela-plugin install` asks y/N about an unsigned package in an interactive
terminal and refuses it when **non-interactive** (pass `--allow-unsigned` to install one anyway), and it takes
`--trust <fingerprint>` so "it must be signed by this publisher" can be written into CI.

**Publisher continuity (only for plugins installed from the plugin manager).** A valid signature only
proves the package has not been changed since it was signed, not that the signer is the same person as last
time. So when the plugin manager installs a plugin, its install receipt pins the signer's public key. Any
later package with the same id (a manually installed `.vpx`, or the row's “Update to x.y.z”) whose signer
does not match — or which is not signed at all — is neither rejected outright nor let through silently: the
host shows “Plugin publisher changed” with the pinned and the incoming fingerprints side by side,
and replaces the plugin only after the user clicks “Install anyway”, pinning the new key as the new baseline.
When refused, the installed version is left untouched (the check runs before the old version is removed).
This is trust on first use plus continuity, not marketplace certification — the host has no root of trust.

**Side-loaded plugins cannot pin a publisher.** A plugin installed with `vela-plugin install` / `update`,
or dropped into the directory directly, has no such receipt; the host only records what it first saw as a
baseline, with no public key in it — so this check stays silent for them (otherwise a plugin installed from
the command line could never be updated by any route). The CLI does not compare against the old fingerprint
in `.vela-install.json` either. The result: **when a package with the same id but a different signer is
installed over it, the host has no way to stop that impersonating overwrite**; after a restart it just
records a new baseline. Have users install plugins whose publisher matters from the plugin manager; on the
command line, pass `--trust <fingerprint>` every time so a package with a mismatching fingerprint is refused.
Installing over a side-loaded plugin from the plugin manager does not ask “publisher changed” either (there is
no earlier identity to compare); pinning starts once that install has succeeded.

### 12.3 Other Install-Time Gates

- **Zip slip**: any entry that would land outside the target directory is rejected.
- **Zip bombs**: at most 10,000 entries and 512 MB unpacked, accounted by **bytes actually written** —
  the length in the central directory is written by the package itself, so a bomb can happily
  claim 1 KB and then emit 10 GB.
- **Payload cap**: 512 MB per package, which also bounds a garbage length in a corrupt header.

### 12.4 There Is No Plain-Zip Compatibility Mode

The host accepts **containers only**: a renamed zip is always rejected, with the remedy in the
error message itself (`this is a plain zip archive - repack it with vela-plugin pack`).

No compatibility switch exists on purpose. No `.vpx` package was ever shipped before the container
format was defined, so there is no installed base to protect — and keeping a "looks like a plugin
package, install it" side door would give away the format's main value while forcing two extraction
paths to be maintained forever.
