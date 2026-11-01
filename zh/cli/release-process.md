# 发版流程(CLI)

> 本篇只讲**本仓库**(velashell-plugin-cli)怎么发版。契约 SDK 与构建支持包 `VelaShell.PluginSdk.Build` 见
> [velashell-plugin-sdk](../sdk/release-process.md),
> `dotnet new` 模板见
> [velashell-plugin-templates](../templates/release-process.md)。

本仓库一次发布只产出一个包,版本号取自 Release 标签:

| 包 | 内容 |
| --- | --- |
| `VelaShell.Plugin.Cli` | dotnet tool `vela-plugin` |

`VelaShell.PluginSdk.Build` 于 2026-09-11 搬去 velashell-plugin-sdk,在那边与契约包同版本、同一次发布。
插件工程 `dotnet build -t:PackVpx` 用的打包器也在那边(`VelaShell.PluginSdk.Packer`,随 `.Build` 包的 `tools/` 分发),
端到端冒烟(`scripts/Invoke-Smoke.ps1` + `tests/smoke/`)与 Avalonia 版本锁的构建期核对随它一起搬走了 ——
见 [SDK 发版流程](../sdk/release-process.md) 的 §四、§六。

**本仓库没有下游。** `vela-plugin` 是面向人的工具(商店、开发内环、签名、体检),发不发版与插件作者能不能出包
毫无关系:两边都走 `VelaShell.PluginSdk` 里的同一个 `VpxContainer`,包格式的一致由类型保证,不靠版本号约定。
上游也不必跟着发:契约 SDK 与模板包各有各的版本号,本仓库发 1.5.1 不代表契约动了,也不要求模板跟着发。

---

## 一、怎么发

两步:

1. 本地落版本号,连同功能改动一起合进 `main`:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.5.1
   ```

2. 在 GitHub 上**发 Release**,标签填 `v1.5.1`(带不带 `v` 都行,流水线会 `TrimStart`,
   但建议统一带)。预发布勾 prerelease,标签用 `v1.6.0-preview.1`。

流水线在解析出标签之后**第一件事**也会跑一遍 `Set-Version.ps1`,只改 runner 上的工作区、
**不回写仓库** —— 于是产物版本号永远等于 Release 标签。

### 忘了在发版前落版本号怎么办

不影响这一次发布(Stamp 步骤已经兜住了),但 `main` 落后了:`main` 上的 CI
「Version consistency check」会红一次。照它给的命令本地跑一遍,补一个 PR 合掉即可。

### 手动补跑

Actions 页面 → 选 Release 工作流 → Run workflow → 填标签。推送用 `--skip-duplicate`,
对同一标签重复跑是幂等的。想只验不推,勾上 `dryRun`。

---

## 二、NuGet 可信发布(Trusted Publishing)怎么调

推送不存 API Key:工作流拿本次运行的 GitHub OIDC 令牌去 nuget.org 换一把 1 小时有效的
临时密钥。nuget.org 那边靠一条**策略**决定「哪个仓库的哪个工作流可以代表我推包」。

⚠️ **拆库之后需要三条策略,一个仓库一条** —— 策略按 (owner, repository, workflow file)
匹配,三个仓库这三项各不相同。策略的 owner 覆盖该账号名下**全部**包,所以不必按包开。

本仓库这一条填:

| 策略字段 | 值 |
| --- | --- |
| Policy name | `velashell-plugin-cli`(随意,能认出来就行) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-cli` |
| **Workflow File** | `release.yml` —— **只填文件名**,不要写 `.github/workflows/` 前缀 |
| Environment | 留空(工作流没用 GitHub Environments) |

建法:登录 nuget.org → 右上角用户名 → **Trusted Publishing** → **Add**。

### ⚠️ 新策略有 7 天窗口

私有仓库上新建的策略是「**临时激活**」状态,7 天内必须成功发布一次,否则自动失效
(可以随时重开窗口)。原因是 nuget.org 要在第一次成功发布时把 GitHub 的 repository ID
与 owner ID 记进策略,把它钉死在那个仓库上(防「删库重建同名仓库」的复活攻击)——
没有一次真实发布就拿不到那两个 ID。所以**建好策略就尽快发一次**,哪怕是 preview 版。

### 换不到密钥时先看这三样

`NuGet login` 那一步失败,九成是策略对不上:

* Repository 还写着 `velashell-plugin-toolchain`;
* Workflow File 写成了 `.github/workflows/release.yml`;
* `NUGET_USER` 填成了邮箱 —— 要的是 nuget.org 的**用户名**(profile name)。
  工作流默认取 `vars.NUGET_USER`,没配则回落到 `joes_du`。

另外 job 上的 `permissions: id-token: write` 不能少,否则 GitHub 根本不签发 OIDC 令牌。

---

## 三、本仓库不需要任何机密

程序集**不做强名称签名**。方向是对的:未签名程序集可以引用已签名的
(`vela-plugin` → `VelaShell.PluginSdk`),反过来才不行。契约 SDK 之所以必须签,
是因为**宿主**(已签名)要引用它 —— 那是 sdk 仓库的事。

推送走 OIDC。于是 fork PR 与主分支跑的是完全同一条 CI 路径,没有「拿不到密钥就降级」
的分支要维护。

---

## 四、唯一的跨仓库旋钮:`Set-Version.ps1` 刻意不碰

它不是「版本号」,是**依赖决定**:`src/VelaShell.Plugin.Cli` 的 csproj 里 `VelaShell.PluginSdk` 的
`PackageReference` —— `vela-plugin` **自己**读清单与 `.vpx` 容器用哪一版契约。

刻意写字面量而不抽成 `Directory.Build.props` 里的 MSBuild 属性:`Version="$(...)"` 会让 Dependabot 与
`dotnet add package` 认不出这条依赖,而这个包正是要靠它们来更新的 —— 合掉 Dependabot 的 PR 就是常规抬法。

抬它 = 让 `vela-plugin` 认得新契约(新的清单字段、容器格式)。插件作者的**编译目标**契约与本仓库无关,
由 velashell-plugin-sdk 的 `.Build` 决定。发一个只改了 `vela-plugin` 输出格式的补丁版时,不该顺手把这个也换掉
—— 所以它是一次独立的、需要想清楚的决定。抬完构建、测试一遍,再发一版本仓库的包,用户才吃得到;
不抬也完全正常,老契约照样能打包、能校验。

> 2026-09-11 之前本仓库同时发 `.Build`(两个包必须同版本 —— 当时 `.Build` 把 `vela-plugin` 收进 `tools/` 当打包器),
> 这里还有第二个旋钮 `VelaAvaloniaVersion` 及其 `VELA1000` / `VELA1005` / `VELA1006` 核对,发完还要考虑模板仓库要不要跟。
> 这些都随 `.Build` 去了 sdk 仓库,本仓库发版之后不需要动任何别的仓库。
