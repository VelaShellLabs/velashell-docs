# Release process (CLI)

> This page covers how **this repository** (velashell-plugin-cli) is released. For the contract SDK and the
> build-support package `VelaShell.PluginSdk.Build` see [velashell-plugin-sdk](../sdk/release-process.md);
> for the `dotnet new` templates see
> [velashell-plugin-templates](../templates/release-process.md).

One release of this repository produces a single package, versioned by the Release tag:

| Package | Contents |
| --- | --- |
| `VelaShell.Plugin.Cli` | The dotnet tool `vela-plugin` |

`VelaShell.PluginSdk.Build` moved to velashell-plugin-sdk on 2026-09-11 and is released there at the same version
as the contract packages, in one release. The packer plugin projects use for `dotnet build -t:PackVpx` lives there
too (`VelaShell.PluginSdk.Packer`, shipped under the `.Build` package's `tools/`), and the end-to-end smoke test
(`scripts/Invoke-Smoke.ps1` + `tests/smoke/`) and the build-time Avalonia pin checks went with it — see §4 and §6
of the [SDK release process](../sdk/release-process.md).

**This repository has no downstream.** `vela-plugin` is a tool for people (marketplace, inner development loop,
signing, health checks); whether it ships has nothing to do with whether plugin authors can produce a package —
both sides go through the same `VpxContainer` in `VelaShell.PluginSdk`, so the package format stays consistent by
type, not by a version-number convention. Upstream does not have to release along with it either: the contract
SDK and the templates package each have their own version number, so this repository shipping 1.5.1 does not mean
the contract changed, and it does not require the templates to ship too.

---

## 1. How to release

Two steps:

1. Stamp the version locally and merge it into `main` together with the feature changes:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.5.1
   ```

2. **Publish a Release** on GitHub with the tag `v1.5.1` (with or without the `v` — the pipeline does a
   `TrimStart` — but using it consistently is recommended). For a pre-release, tick prerelease and use a tag
   such as `v1.6.0-preview.1`.

Once the pipeline has parsed the tag, the **first thing** it does is run `Set-Version.ps1` as well. That changes
only the working tree on the runner and **is never written back to the repository** — so the artifact version
always equals the Release tag.

### If you forgot to stamp the version before releasing

This release is not affected (the Stamp step already covered it), but `main` is now behind: the CI "Version
consistency check" on `main` goes red once. Run it locally with the command it prints and merge a follow-up PR.

### Re-running by hand

Actions page → pick the Release workflow → Run workflow → enter the tag. Pushes use `--skip-duplicate`, so
re-running for the same tag is idempotent. To validate without pushing, tick `dryRun`.

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
| Policy name | `velashell-plugin-cli` (anything you will recognise) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-cli` |
| **Workflow File** | `release.yml` — **the file name only**, without the `.github/workflows/` prefix |
| Environment | Leave empty (the workflow does not use GitHub Environments) |

To create it: sign in to nuget.org → your user name at the top right → **Trusted Publishing** → **Add**.

### ⚠️ A new policy has a 7-day window

A policy newly created on a private repository is "**temporarily active**": it must be used for a successful
publish within 7 days or it lapses automatically (the window can be reopened at any time). The reason is that
nuget.org records the GitHub repository ID and owner ID into the policy on the first successful publish, pinning
it to that repository (a defence against resurrection attacks of the "delete the repository and recreate one with
the same name" kind) — and without a real publish it cannot get those two IDs. So **publish once as soon as the
policy is created**, even if it is only a preview.

### When no key comes back, check these three first

When the `NuGet login` step fails, nine times out of ten the policy does not match:

* Repository still says `velashell-plugin-toolchain`;
* Workflow File was entered as `.github/workflows/release.yml`;
* `NUGET_USER` was set to an email address — it must be the nuget.org **user name** (profile name).
  The workflow reads `vars.NUGET_USER` by default and falls back to `joes_du` when it is not set.

Also, the job must keep `permissions: id-token: write`; without it GitHub does not issue an OIDC token at all.

---

## 3. This repository needs no secrets

The assemblies are **not strong-name signed**. That is the right direction: an unsigned assembly may reference a
signed one (`vela-plugin` → `VelaShell.PluginSdk`), just not the other way round. The contract SDK has to be
signed because the **host** (which is signed) references it — that is the sdk repository's business.

Pushing goes through OIDC. So fork PRs and the main branch run exactly the same CI path, and there is no
"degrade when the key is unavailable" branch to maintain.

---

## 4. The one cross-repository knob that `Set-Version.ps1` deliberately leaves alone

It is not a "version number"; it is a **dependency decision**: the `PackageReference` to `VelaShell.PluginSdk` in
the `src/VelaShell.Plugin.Cli` csproj — which contract release `vela-plugin` **itself** uses to read manifests and
`.vpx` containers.

It is deliberately written as a literal rather than pulled out into an MSBuild property in `Directory.Build.props`:
`Version="$(...)"` stops Dependabot and `dotnet add package` from recognising the dependency, and they are exactly
what this package is meant to be updated with — merging Dependabot's PR is the usual way to raise it.

Raising it = teaching `vela-plugin` the newer contract (new manifest fields, container format). Plugin authors'
**compile-target** contract has nothing to do with this repository; velashell-plugin-sdk's `.Build` decides it. A
patch release that only changes `vela-plugin`'s output format should not swap this out on the side — so it is a
separate decision that needs to be thought through. After raising it, build and test once, then release this
repository's package so users get it; not raising it is perfectly normal too — the old contract still packs and
validates fine.

> Before 2026-09-11 this repository also released `.Build` (the two packages had to share a version — `.Build`
> then bundled `vela-plugin` in `tools/` as its packer), and this section had a second knob, `VelaAvaloniaVersion`,
> with its `VELA1000` / `VELA1005` / `VELA1006` checks, plus the question of whether the templates repository should
> follow a release. All of that went to the sdk repository with `.Build`; releasing this repository requires no
> change in any other repository.
