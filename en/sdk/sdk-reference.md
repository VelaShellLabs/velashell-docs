# Plugin SDK Reference

> Applies to **SDK 2.0.0 / apiLevel 2**
> See also (other repositories): [Development Guide](../templates/dev-guide.md) (tutorial) · [CLI Manual](../cli/cli.md) · [Packaging and Publishing](../templates/publishing.md)

This page is the **lookup-oriented** view of the SDK: package layout, contract surface, what each
capability can do and what constrains it, version history, and the test doubles. For a
step-by-step walkthrough, read the [Development Guide](../templates/dev-guide.md) first.

---

## 1. Packages

| Package                         | Repository                                                                                | Referenced by                          | Contents                                                                                                                                       |
| ------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **`VelaShell.PluginSdk.Build`** | [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli)             | **the plugin project (this one only)** | MSBuild props/targets, a version-pinned Avalonia, the bundled packer, manifest validation, the `PackVpx` target                                |
| `VelaShell.PluginSdk`           | this repository                                                                           | pulled in transitively                 | The contract assembly: entry interface, `IPluginContext` and every capability interface, DTOs, manifest model, `.vpx` container, host registry |
| `VelaShell.PluginSdk.Testing`   | this repository                                                                           | the plugin's **test** project          | `TestPluginContext` and in-memory doubles for every capability                                                                                 |
| `VelaShell.Plugin.Cli`          | [velashell-plugin-cli](https://github.com/VelaShellLabs/velashell-plugin-cli)             | developer machine (dotnet tool)        | `vela-plugin`: inner loop, validation, packing, signing, health check                                                                          |
| `VelaShell.Plugin.Templates`    | [velashell-plugin-templates](https://github.com/VelaShellLabs/velashell-plugin-templates) | developer machine                      | `dotnet new velaplugin` / `velaplugin-ui`                                                                                                      |

The `PackageReference` line a plugin project needs — **including the current version** — lives in the
[Development Guide](../templates/dev-guide.md). That version belongs to
`VelaShell.PluginSdk.Build`, which now versions independently of this SDK, so repeating it here
would only guarantee it goes stale.

Do **not** reference `VelaShell.PluginSdk` or `Avalonia` separately — a version mismatch is a
build error (`VELA1001`) instead of a runtime control-cast failure on a user's machine.

The contract assembly depends only on the BCL. That is a discipline: it is the **only type
source shared between host and plugin**, so a third-party dependency inside it would be imposed
on every plugin.

---

## 2. Entry contract

```csharp
using VelaShell.PluginSdk;

[VelaPlugin]                                  // exactly one public, non-abstract, parameterless class
public sealed class DemoPlugin : IVelaPlugin
{
    public Task ActivateAsync(IPluginContext context, CancellationToken ct)
    {
        context.Log.Info("activated");
        return Task.CompletedTask;            // must return quickly (10-second limit)
    }

    public Task DeactivateAsync(CancellationToken ct) => Task.CompletedTask;  // ~2-second limit
}
```

| Constraint         | Detail                                                                                                                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Activation limit   | 10 seconds; **stretched to 10 minutes while a debugger is attached**                                                                                                                          |
| Deactivation limit | About 2 seconds (application exit path), abandoned on timeout                                                                                                                                 |
| Long-running work  | Start your own background task and observe `context.Shutdown`                                                                                                                                 |
| Resource cleanup   | Commands and event subscriptions registered through the SDK are cleaned up by the host; never put your own types into host static fields or long-lived events, or the ALC cannot be reclaimed |

---

## 3. `IPluginContext` capabilities

`IPluginContext` is the single entry point into the host. **Every interface is transport
agnostic** (async methods, DTOs, opaque ids only), so the same plugin source runs in both
`inProcess` and `isolated` mode.

### 3.1 Identity and infrastructure

| Member                       | Description                                                                                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `PluginId` / `PluginVersion` | From `plugin.json`                                                                                                                                                       |
| `DataDirectory`              | Private directory (already created). **All local writes belong here**; it is deleted on uninstall                                                                        |
| `Host` (`IHostInfo`)         | Host version, apiLevel, current locale. `Host.Theme` only holds the coarse `dark`/`light`/`system` value — for the actual theme identity and palette see `Theme` in §3.5 |
| `Log` (`IPluginLogger`)      | `Debug/Info/Warn/Error` into the host log pipeline (prefixed with the plugin id)                                                                                         |
| `Shutdown`                   | Shutdown token: capability calls may start failing afterwards, so wind down                                                                                              |

### 3.2 Data

| Capability                      | Key methods                                                                                                                                | Notes                                                                                                 |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `Storage` (`IPluginStorage`)    | `GetAsync<T>` / `SetAsync<T>` / `RemoveAsync` / `GetKeysAsync`                                                                             | Per-plugin namespaced KV backed by SonnetDB (JSON files in headless setups)                           |
| `Secrets` (`ISecretsApi`)       | `GetAsync` / `SetAsync` / `DeleteAsync`                                                                                                    | Encrypted private key-value store. **No plaintext fallback** — unavailable is reported as unavailable |
| `TimeSeries` (`ITimeSeriesApi`) | `OpenAsync` / `ListAsync` / `DropAsync`; on a series `WriteAsync` / `QueryAsync` / `CountAsync` / `DistinctTagValuesAsync` / `DeleteAsync` | A private embedded time-series store (append by time, retrieve by tag)                                |

### 3.3 Sessions and remote access

| Capability                          | Key methods                                                        | Notes                                                                                                |
| ----------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| `Sessions` (`ISessionsApi`)         | `ListAsync` / `GetAsync`                                           | Enumerate SSH sessions, **redacted, never credentials**                                              |
| `RemoteFs` (`IRemoteFsApi`)         | directory / attributes / read / write / transfer / rename / delete | SFTP over an existing session                                                                        |
| `RemoteExec` (`IRemoteExecApi`)     | `RunAsync` (whole result) / `StreamAsync` (per line)               | A separate channel — **never the user's terminal**                                                   |
| `RemoteTunnel` (`IRemoteTunnelApi`) | `OpenUnixSocketAsync` / `OpenTcpAsync`                             | A **raw byte duplex stream** to a remote endpoint (Docker Engine API, tar streams). `inProcess` only |
| `Terminal` (`ITerminalApi`)         | `GetOutputAsync` / `SearchOutputAsync` / `WriteAsync`              | Read/search session output; **writing input requires user consent** (revocable on the manager page)  |

### 3.4 UI and extension points

| Capability                          | Key methods                                          | Notes                                                                                                                                                      |
| ----------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Commands` (`ICommandsApi`)         | `Register` / `TryExecute`                            | Command ids must be prefixed by the plugin id; manifest placeholders are replaced by real handlers on activation                                           |
| `Ui` (`IUiApi`)                     | `ShowPanelAsync(options, contentFactory)`            | Present your own Avalonia controls: dockable main-window tabs in `inProcess`, standalone card windows in `isolated`                                        |
| `TerminalView` (`ITerminalViewApi`) | `Create(...)`                                        | **Borrows the host's terminal emulator** (VT parsing, screen buffer, selection, IME, key encoding) as a control you embed in your own UI. `inProcess` only |
| `Protocols` (`IProtocolsApi`)       | register a protocol implementation                   | Your own remote **file** protocol, a first-class citizen of the connection page next to SSH/SFTP/FTP. `inProcess` only                                     |
| `Workspaces` (`IWorkspacesApi`)     | register a workspace provider                        | **Non-file** connection types (Redis, MySQL, …) whose session document you render entirely. `inProcess` only                                               |
| `Clipboard` (`IClipboardApi`)       | text read/write                                      | System clipboard                                                                                                                                           |
| `Events` (`IHostEvents`)            | session connect/disconnect, theme and locale changes | Subscriptions are cleaned up by the host on deactivation                                                                                                   |
| `Theme` (`IHostThemeApi`)           | `Current` / `Colors` / `GetColor` / `Changed`        | Theme identity, the fully resolved `Vela*` palette, and a signal that covers every kind of theme change. See §3.5                                          |

> **The four `inProcess`-only capabilities** (`RemoteTunnel`, `TerminalView`, `Protocols`,
> `Workspaces`) throw "capability unavailable" in isolated mode. They hand out live native
> objects or raw streams, which have no equivalent across a process boundary — if you need them,
> set `hostMode` to `inProcess`.

### 3.5 Theming (SDK 2.0)

**You do not need this interface to colour your UI.** In Avalonia, always write
`{DynamicResource VelaXxx}` — the host derives those tokens per theme and swaps the whole set
at once, and they follow automatically in both `inProcess` and `isolated` mode.

What this interface is for is the three places `DynamicResource` cannot reach:

1. you need a `Color` rather than a `Brush` (syntax-highlighting definitions, custom drawing, image export);
2. you resolve colours once in code (converters, computations outside a control template) — those need a "re-read now" signal;
3. you want to branch on **theme identity** (a different illustration or icon set for one theme).

```csharp
HostThemeInfo theme = ctx.Theme.Current;
// theme.Id            "tokyo-night"  — already resolved, never "system"
// theme.Name          "Tokyo Night"
// theme.IsDark        true
// theme.Variant       "dark"
// theme.FollowsSystem whether the user picked "follow system"
// theme.Accent        "#7AA2F7"      — includes the user's accent override

string? bg = ctx.Theme.GetColor("VelaBgTerminal");   // "#FF1A1B26"

ctx.Theme.Changed += info => { /* Current and Colors are already updated */ };
```

> **Do not use `Host.Theme` to detect a theme change.** `IHostInfo.Theme` only ever holds
> `dark`, `light` or `system`: the host ships a dozen named themes and they are all collapsed
> to their light/dark name there. So for a VelaDark → Tokyo Night switch, the value of
> `Host.Theme` **and** the argument of `IHostEvents.ThemeChanged` are unchanged — code that
> compares that argument misses the change entirely. And when it reads `system`, it does not
> tell you whether the UI is currently light or dark. `IHostThemeApi.Changed` is the union of
> all three cases: named-theme switch, system light/dark flip, and accent override.

---

## 4. Manifest

The full field table is in the [Development Guide §3](../templates/dev-guide.md); the three version gates are
in [Packaging and Publishing §1.2](../templates/publishing.md). Three things bite most often:

1. `id` is immutable after publication (command prefix + data namespace).
2. Using newer SDK surface requires `minSdkVersion`, otherwise old hosts fail at runtime with
   `MissingMethodException`.
3. Declarative contributions (`contributes.commands` / `protocols` / `workspaces`) take effect
   **during discovery**, without loading any assembly — that is what makes zero startup cost and
   lazy activation possible.

---

## 5. SDK version history

`apiLevel` only moves on **breaking** changes (2.0 raised it from `1` to `2`); additive surface is gated by
`minSdkVersion`.
`apiLevel` only moves on **breaking** changes; additive surface is gated by `minSdkVersion`.
The discipline is "SDK major == apiLevel" — the whole 1.x series is `apiLevel 1`,
and **2.0 onwards is `apiLevel 2`**.

| SDK     | Added                                                                                                                                                                                                                                                                                                                                                                                                                                | Does a plugin need `minSdkVersion`?                    |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ |
| 1.0     | First contract                                                                                                                                                                                                                                                                                                                                                                                                                       | —                                                      |
| 1.1     | `ExecResult` gains stderr and exit code; streaming remote exec                                                                                                                                                                                                                                                                                                                                                                       | Yes, if used                                           |
| 1.2     | `IRemoteTunnelApi` (raw byte duplex stream)                                                                                                                                                                                                                                                                                                                                                                                          | Yes, if used (`1.2.0`)                                 |
| 1.3     | `ITerminalViewApi` (borrow the host terminal control)                                                                                                                                                                                                                                                                                                                                                                                | Yes, if used (`1.3.0`)                                 |
| 1.3.1   | Workspace **variants**: `WorkspaceVariant`, `VariantKey`/`Variants`, `NoCredentials`/`NoEndpoint`                                                                                                                                                                                                                                                                                                                                    | Yes, if used (`1.3.1`)                                 |
| 1.4     | `HostRegistry` (host self-registration for `vela-plugin`)                                                                                                                                                                                                                                                                                                                                                                            | **No** — toolchain surface, plugin code never calls it |
| **1.5** | Connection-form additions for protocols: `ProtocolFeatures.NoEndpoint` (hide the port column), `ProtocolSettingKind.DynamicChoice` + `IProtocolChoiceSource` (choices fetched when the form opens), `AllowsCustomValue` / `HostKind` / `HostChoices` / `HostAllowsCustomValue` (editable combo boxes; the host column can be a combo too). Driven by the serial plugin — ports are hot-plugged, baud rates have non-standard values. | Yes, if used (`1.5.0`)                                 |
| **2.0** | **Generation bump (`apiLevel` 1 → 2)** — see "Migrating to 2.0" below. The surface it carries is `IHostThemeApi` (`ctx.Theme`, see §3.5): theme identity, the fully resolved palette, and a `Changed` signal covering every kind of theme change. Before it, a plugin could only see the three values of `IHostInfo.Theme` — no named theme, no accent, and no change at all when switching within the same light/dark variant.      | No — express it with `apiLevel: 2`                     |

### Migrating to 2.0

2.0 is the **first** time this series moves `apiLevel`. Bumping the major version takes the
contract assembly's `AssemblyVersion` from `1.0.0.0` to `2.0.0.0` (see `$(VelaSdkMajor).0.0.0`
in the SDK repo's `src/Directory.Build.props`), so **every already-compiled plugin must be
rebuilt** before a 2.x host will load it.

Two things to do:

1. Move `VelaShell.PluginSdk.Build` to the matching version, rebuild, and repack the `.vpx`.
2. Set `apiLevel` to `2` in `plugin.json`.

Step 2 is optional for running on a 2.x host (a host accepts any generation up to its own),
but it is what you want **when the plugin lands on a 1.x host**: you get the discovery-time
message "Plugin targets apiLevel 2, host supports up to 1. Update VelaShell." instead of an
opaque assembly-binding failure. That is precisely what `apiLevel` exists for.

The new surface itself is still purely additive: nothing removed, no signature changed. The one
source-level incompatibility is the new `Theme` member on `IPluginContext` — plugins **consume**
that interface and are unaffected; anyone who wrote their own `IPluginContext` implementation
(usually a test double) must add the member, or switch to `TestPluginContext`.

---

## 6. Testing without starting the host

```csharp
using VelaShell.PluginSdk.Testing;

[TestMethod]
public async Task Activate_RegistersCommand()
{
    using var context = new TestPluginContext();
    var plugin = new DemoPlugin();

    await plugin.ActivateAsync(context, CancellationToken.None);

    Assert.Contains("acme.demo.run", context.RecordingCommands.Registered);
    await plugin.DeactivateAsync(CancellationToken.None);
}
```

Available doubles: `CollectingLogger`, `InMemoryStorage`, `InMemoryTimeSeries`, `FakeSessions`,
`FakeRemoteFs`, `FakeRemoteExec`, `FakeRemoteTunnel`, `FakeTerminal`, `FakeTerminalViewApi`,
`FakeUi`, `FakeSecrets`, `FakeClipboard`, `RecordingCommands`, `RecordingProtocols`,
`RecordingWorkspaces`, `TestHostEvents`, `TestHostInfo`.

Besides `AddConnected`, `FakeSessions` offers `AddSaved` (build a saved configuration) and three hooks:
`DenyOpen` / `OpenFailure` simulate "the user said no" and "it would not connect", while `LastOpenReason`
lets you assert that the reason shown to the user actually made it through — an implementation that passes
"the plugin needs to connect" works perfectly and still turns the confirmation dialog into a blind button.

What unit tests cannot cover (real UI, real sessions, real protocol tabs) belongs to the inner
loop: `vela-plugin dev init` → F5, see the [CLI Manual](../cli/cli.md).

---

## 7. Loading model: three rules

1. **One collectible ALC per plugin**; the plugin's own dependencies resolve inside its
   directory through `deps.json`, so you may reference any NuGet package without colliding with
   the host.
2. **Two assembly families are always shared**: `VelaShell.PluginSdk` and `Avalonia*` fall back
   to the loading side. Your Avalonia version must match the host's (the SDK package pins it),
   and third-party packages whose name starts with `Avalonia` cannot be used.
3. **Development plugins load from a shadow copy** (`~/.velashell/dev-shadow/<id>/gen-N`), so
   the running host does not lock your `bin` and Reload on the manager page picks up a rebuild.
   The production path does not use shadow copies.

---

## 8. Host registry (`HostRegistry`, SDK 1.4)

`VelaShell.PluginSdk.Hosting.HostRegistry` reads and writes `~/.velashell/host.json`, where the
host registers its executable path, version, apiLevel, bundled SDK version, Avalonia version and
data root on every launch.

It targets the **toolchain** (`vela-plugin dev init` / `doctor` / `hosts`); plugin runtime code
does not use it. If you write your own build script or IDE integration:

```csharp
HostRegistryEntry? host = HostRegistry.Resolve();          // most recently started
HostRegistryEntry? preview = HostRegistry.Resolve("1.5");  // by version
IReadOnlyList<HostRegistryEntry> all = HostRegistry.List();
```

Every read path returns an empty list instead of throwing when the file is missing or corrupt —
it is a cache that speeds up tooling, and a broken cache should mean "you have to point at the
path yourself", never "something fails to start".
