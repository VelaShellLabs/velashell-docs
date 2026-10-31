# Release process (contract SDK)

> This page covers how **this repository** (velashell-plugin-sdk) is released. For `vela-plugin` see
> [velashell-plugin-cli](../cli/release-process.md);
> for the `dotnet new` templates see
> [velashell-plugin-templates](../templates/release-process.md).

One release of this repository produces three packages, which share the version number in the Release tag:

| Package | Contents |
| --- | --- |
| `VelaShell.PluginSdk` | The contract assembly |
| `VelaShell.PluginSdk.Testing` | Test doubles |
| `VelaShell.PluginSdk.Build` | The **one package a plugin project references**: targets + the bundled packer + the dependency pins |

`.Build` moved here from velashell-plugin-cli on 2026-09-11; the `.vpx` packer (`src/VelaShell.PluginSdk.Packer`,
`IsPackable=false`) lives in this repository too and is not a package of its own — it ships only as build output
under the `.Build` package's `tools/`. `.Build` reaches both the contract and the packer through `ProjectReference`,
so the contract and packer handed to plugin authors are always the ones from that same release.

**Downstream does not have to release along with it.** The cli and templates repositories each have their own
version number: the cli raises the `VelaShell.PluginSdk` version it references only when it wants the new contract,
and the templates repository raises `VelaBuildPackageVersion` (which `.Build` version generated projects reference)
only when it wants new projects to point at the new release. The SDK shipping 1.6.0 does not mean `vela-plugin` has
to become 1.6.0 — that is exactly what the split was for.

---

## 1. How to release

Three steps:

1. Only for a **breaking change**: bump `VelaPluginApi.Level` in `src/VelaShell.PluginSdk/VelaPluginApi.cs`
   by one, by hand. The script checks it but **does not change it for you** — whether the contract broke is a
   human judgement, not something inferred from a version number.

2. Stamp the version locally and merge it into `main` together with the feature changes:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.6.0
   ```

3. **Publish a Release** on GitHub with the tag `v1.6.0` (with or without the `v` — the pipeline does a
   `TrimStart` — but using it consistently is recommended). For a pre-release, tick prerelease and use a tag
   such as `v1.6.0-preview.1`.

Once the pipeline has parsed the tag, the **first thing** it does is run `Set-Version.ps1` as well. That changes
only the working tree on the runner and **is never written back to the repository** — so the artifact version
always equals the Release tag, whatever the repository happened to have committed at the time.

### If you forgot to stamp the version before releasing

This release is not affected (the Stamp step already covered it), but `main` is now behind: the CI "Version
consistency check" on `main` goes red once. Run `Set-Version.ps1` locally with the command it prints and merge
a follow-up PR.

### Re-running by hand

The `release` event occasionally does not fire. Actions page → pick the Release workflow → Run workflow →
enter the tag. Pushes use `--skip-duplicate`, so re-running for the same tag is idempotent. To validate without
pushing, tick `dryRun`.

---

## 2. Setting up NuGet Trusted Publishing

No API key is stored for pushing: the workflow takes this run's GitHub OIDC token to nuget.org and exchanges it
for a temporary key valid for one hour. On the nuget.org side a **policy** decides "which workflow in which
repository may push packages on my behalf".

⚠️ **After the split you need three policies, one per repository** — a policy matches on (owner, repository,
workflow file), and all three of those differ between the three repositories. A policy's owner covers **every**
package under that account, so you do not need one per package.

This repository's policy:

| Policy field | Value |
| --- | --- |
| Policy name | `velashell-plugin-sdk` (anything you will recognise) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-sdk` |
| **Workflow File** | `release.yml` — **the file name only**, without the `.github/workflows/` prefix |
| Environment | Leave empty (the workflow does not use GitHub Environments) |

To create it: sign in to nuget.org → your user name at the top right → **Trusted Publishing** → **Add**.

While you are there, delete the pre-split policy that points at `velashell-plugin-toolchain` — that repository
no longer pushes packages, and keeping the policy only leaves an extra piece of trust surface.

### ⚠️ A new policy has a 7-day window

A policy newly created on a private repository is "**temporarily active**": it must be used for a successful
publish within 7 days or it lapses automatically (the window can be reopened at any time). The reason is that
nuget.org records the GitHub repository ID and owner ID into the policy on the first successful publish, pinning
it to that repository (a defence against resurrection attacks of the "delete the repository and recreate one with
the same name" kind) — and without a real publish it cannot get those two IDs. So **publish once as soon as the
policy is created**, even if it is only a preview.

Policies on public repositories are usually permanently active straight away, but publishing once to check is
still the low-effort option.

### When no key comes back, check these three first

When the `NuGet login` step fails, nine times out of ten the policy does not match:

* Repository still says `velashell-plugin-toolchain`;
* Workflow File was entered as `.github/workflows/release.yml`;
* `NUGET_USER` was set to an email address — it must be the nuget.org **user name** (profile name).
  The workflow reads `vars.NUGET_USER` by default and falls back to `joes_du` when it is not set.

Also, the job must keep `permissions: id-token: write`; without it GitHub does not issue an OIDC token at all.

---

## 3. Version discipline

`VelaSdkVersion` in `Directory.Build.props` is this repository's default version; at release time the tag
overrides it.

### Three version numbers, three jobs — do not mix them up

| Property | Value | Role |
| --- | --- | --- |
| `AssemblyVersion` | `<major>.0.0.0` | The **binding identity**; it moves only with the major version. Plugins bind to this identity at compile time |
| `FileVersion` | The full numeric version (`1.6.0`) | The one shown on the Explorer properties page |
| `InformationalVersion` | The full version including any pre-release suffix | The version `vela-plugin` reports |

`AssemblyVersion` is pinned to the major version because letting it follow the patch number would make every
compiled plugin rebind on every patch release, for no benefit. A new major version, on the other hand, means the
contract broke — and at that moment **apiLevel must be bumped by one as well**, so an older host cleanly refuses
the plugin by apiLevel at **discovery time** instead of throwing an incomprehensible binding exception at load
time.

### "SDK major version == apiLevel"

`Set-Version.ps1` enforces this rule: to release `2.x.x` you must first change `VelaPluginApi.Level` to `2`, and
vice versa. apiLevel is a **property of the contract**, so this check only has a home in this repository —
neither the CLI nor the templates are allowed to touch it.

### Where the version number lives (maintained by the script)

After the split only four places are left, two of them in velashell-docs:

| Location | What goes wrong if it is missed |
| --- | --- |
| `VelaSdkVersion` in `Directory.Build.props` | The package version is wrong |
| `SdkVersion` in `src/VelaShell.PluginSdk/VelaPluginApi.cs` | **Nothing reports an error**; `vela-plugin doctor` simply reports the wrong host SDK version from then on, and plugins' `minSdkVersion` gate is judged wrongly with it |
| The version banner in velashell-docs' `zh/sdk/sdk-reference.md` | A stale number for people to copy |
| The version banner in velashell-docs' `en/sdk/sdk-reference.md` | Same as above |

The last two live in another repository and are **optional** locations: the script updates them only when it finds that repository through
`-DocsRoot` / `$env:VELASHELL_DOCS` / a sibling `../velashell-docs`, and skips them with a warning otherwise.

`pwsh scripts/Set-Version.ps1 <version>` changes them all in one go. CI runs it with `-Check` as a health check.

---

## 4. The Avalonia version pin: this repository is the authority

The Avalonia a plugin project compiles against and the host's Avalonia it is forced to share at run time inside
the host process must be the same version — the loader makes every `Avalonia*` assembly fall back to the loading
side, so a version drift means control types that do not match across ALCs, and it only blows up once a user
installs the plugin.

In the single-repository days, `VelaShell.PluginSdk.Build` made this check by `XmlPeek`-ing the host's
`src/Directory.Packages.props` directly. After the split the host's files are out of reach, so it now runs the
other way round, with this repository supplying the authoritative value:

```
Directory.Build.props: <VelaAvaloniaVersion>x.y.z</VelaAvaloniaVersion>      ← the authority
  ├── written into the VelaShell.PluginSdk package's buildTransitive/VelaShell.PluginSdk.props at pack time
  │     → $(VelaSdkPinnedAvaloniaVersion), visible to every project that references VelaShell.PluginSdk
  │       ├── VerifyAvaloniaMatchesSdk in the host's src/Directory.Build.targets
  │       └── the check of the same name in velashell-plugins' test projects (VELAP1000)
  └── read directly by VelaShell.PluginSdk.Build in this same repository (VerifyAvaloniaVersionPin):
        the exact range [x.y.z] on its csproj must equal it (VELA1006),
        the default of the same name in build/VelaShell.PluginSdk.Build.props must equal that range (VELA1000),
        and a missing property is itself an error (VELA1005)
```

That props file in the package is the only channel that carries this fact to the **host**, and leaving it out of
the package does not make `pack` fail — so both CI and Release have a "Verify the pin props shipped in the
package" step that unpacks the nupkg and checks it. The `.Build` side does not use that channel (same repository,
it reads the property directly).

**The full procedure for changing `VelaAvaloniaVersion`**:

1. Ship it in the same wave as the host (the host must reference the same version);
2. In **the same commit**, change both copies inside `.Build`: the exact Avalonia range on
   `VelaShell.PluginSdk.Build.csproj` and the `VelaAvaloniaVersion` default in `build/VelaShell.PluginSdk.Build.props`
   — otherwise VELA1006 / VELA1000 go red on the spot;
3. Release the SDK — all three packages go out together, so plugin projects get the new pin as soon as they move to
   that `.Build`;
4. Downstream follows: the host raises its SDK packages; velashell-plugins raises `.Build` / `.Testing` and, as
   VELAP1000 tells it, moves its test projects' `Avalonia*` to the new value; the templates repository raises
   `VelaBuildPackageVersion` when it wants new projects to point there.

---

## 5. Why the tests must run in Debug

`src/Directory.Build.props` turns on strong-name signing in Release (the host is signed, and a signed assembly
cannot reference an unsigned one, so the SDK has to be signed with the same key as the host). On a signed
assembly, `InternalsVisibleTo` must carry the friend's public key, and the friend itself must be signed with the
same key — the test assemblies meet neither condition.

These tests check the contract and do not depend on the optimisation level, so Debug is equivalent; and Debug's
`obj/bin` are kept apart from the Release directories `pack` uses, so unsigned output never gets mixed into the
package being published.

The key is not checked in (`*.snk` is in `.gitignore`); CI restores it to the repository root from the
`STRONG_NAME_KEY` secret. It is the **only** secret this repository needs.

> Neither the cli nor the templates repository signs anything, so they **do not need** this key —
> an unsigned assembly may reference a signed one, which is the right direction.

---

## 6. End-to-end smoke test

Before a release (both CI and Release run it), `scripts/Invoke-Smoke.ps1 -Feed ./artifacts/nuget -Version <version>`
runs: with the freshly packed packages, it copies the hand-written minimal plugin project under `tests/smoke/` (the
same shape as the `velaplugin-ui` template: a command + a panel + compile-time AXAML) into a temporary directory,
restores from the local source, runs `dotnet build -t:PackVpx`, confirms a `.vpx` really came out, reads the container
back with the packer from the `.Build` package's `tools/`, and checks whether any shared assembly leaked into the plugin
output directory. The local source holds only the three packages this repository has just packed, and nothing leaves
the repository — it is a routine CI step, not a way to hand local packages to other repositories.

**This step is not a formality.** A plugin project is an environment outside the repository and inherits none of
the repository's `Directory.Build.props` (the two empty `Directory.Build.props`/`.targets` in the fixture are
there to cut off the upward search). Historically it is what caught two pitfalls first:

* The SDK carries `RequiresPreviewFeatures`, so a plugin project that does not turn on the preview switch gets
  `CA2252` across the board;
* NuGet does not flow `build` assets transitively by default, so Avalonia's AXAML compiler never reaches the
  plugin project — the symptom is controls that cannot be found at run time, not a build error.

Both pitfalls only show up when nothing at all is inherited. It also guards "the packer really made it into the
package": the packer used for the read-back is deliberately taken from the package, so a missing packer turns this
step red instead of handing plugin authors a `VELA1003` at `-t:PackVpx` time. **Run it whenever you change `.Build`'s
targets or the packer's command surface.**

> This step moved here from the cli repository with `.Build` on 2026-09-11. Earlier still (before the split) it used
> `dotnet new velaplugin-ui`; with the templates moved out it can no longer do that: a problem in the templates
> repository would turn this repository's CI red for no reason, and releasing this repository's packages would first
> require a templates package. End-to-end verification of the templates themselves is the job of the templates
> repository's own CI.
