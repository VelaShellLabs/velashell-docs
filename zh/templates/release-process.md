# 发版流程(dotnet new 模板)

> 本篇只讲**本仓库**怎么发版。契约 SDK 见
> [velashell-plugin-sdk](../sdk/release-process.md),
> `vela-plugin` 与 `VelaShell.PluginSdk.Build` 见
> [velashell-plugin-cli](../cli/release-process.md)。

本仓库一次发布产出一个包:`VelaShell.Plugin.Templates`。

**上游都不必跟着发。** 反过来也一样:契约 SDK 或 `.Build` 包发新版**不要求**本仓库跟着发。

---

## 一、两个版本号,先分清

单仓库时代它们是同一个数字,拆库(2026-08-27)之后必须分开 —— 「改了模板文案」与
「换了插件作者的构建工具链」是性质完全不同的两种变更,合成一个数字之后就没法只做前一件。

| 属性(`Directory.Build.props`) | 含义 | 落点 |
| --- | --- | --- |
| `VelaTemplatesVersion` | **模板包自己的版本**,就是 Release 标签里那个 | 1 处:`Directory.Build.props` |
| `VelaBuildPackageVersion` | 生成的工程要引用哪一版 `VelaShell.PluginSdk.Build` | 5 处,见下 |

`VelaBuildPackageVersion` 的五处落点:

| 落点 | 漏改的后果 |
| --- | --- |
| `Directory.Build.props` | 核对基准不对,下面几处的漏改就查不出来了 |
| `content/velaplugin/.template.config/template.json` | 新建的工程去还原一个旧版包 —— 构建期由 `VELA1004` 拦下 |
| `content/velaplugin-ui/.template.config/template.json` | 同上 |
| `docs/dev-guide.md` 的 `PackageReference` 片段 | **不报错**,但文档是给人照抄的,过期版本号会被原样粘进别人的工程 |
| `docs-en/dev-guide.md` 的同一片段 | 同上 |

一条命令全改:

```powershell
pwsh scripts/Set-Version.ps1 -BuildPackageVersion 1.5.2
```

不给 `-BuildPackageVersion` 时脚本会读 `Directory.Build.props` 里的当前值沿用,所以
「只发一版模板」不必重复输入它。

---

## 二、怎么发

两步:

1. 本地落版本号,连同改动一起合进 `main`:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.5.1
   ```

2. 在 GitHub 上**发 Release**,标签填 `v1.5.1`(带不带 `v` 都行,流水线会 `TrimStart`,
   但建议统一带)。预发布勾 prerelease,标签用 `v1.6.0-preview.1`。

流水线的 Stamp 步骤**只传模板包版本**,`VelaBuildPackageVersion` 由脚本从
`Directory.Build.props` 读出来沿用 —— 发版不该顺手换掉插件作者的构建工具链版本。
它只改 runner 上的工作区、**不回写仓库**。

### 忘了在发版前落版本号怎么办

不影响这一次发布(Stamp 步骤已经兜住了),但 `main` 落后了:`main` 上的 CI
「Version consistency check」会红一次。照它给的命令本地跑一遍,补一个 PR 合掉即可。

### 手动补跑

Actions 页面 → 选 Release 工作流 → Run workflow → 填标签。推送用 `--skip-duplicate`,
对同一标签重复跑是幂等的。想只验不推,勾上 `dryRun`。

---

## 三、什么时候该抬 `VelaBuildPackageVersion`

`velashell-plugin-cli` 发了新版 `VelaShell.PluginSdk.Build`,而你希望
`dotnet new velaplugin` 生成的工程指过去 —— 典型场景是那一版带来了插件作者用得上的
东西(新的 MSBuild 目标、修了打包器的某个行为、或者它引用的契约 SDK 变新了)。

```powershell
pwsh scripts/Set-Version.ps1 -BuildPackageVersion 1.5.2
pwsh scripts/Set-Version.ps1 1.5.1        # 再发一版模板把它带出去
```

⚠️ **抬到一个还没发布的版本上会在 CI 里红。** 冒烟会真的去 nuget.org 还原那一版,
所以「.Build 还没发就先改模板」会当场被拦下,而不是等插件作者新建工程时才发现。
顺序永远是:先在 cli 仓库发布,再来这里抬。

**不抬也完全正常。** 新建的工程继续引用上一版 `.Build`,那是可用的,不是遗漏 ——
这正是拆库之后的常态。

---

## 四、NuGet 可信发布(Trusted Publishing)怎么调

推送不存 API Key:工作流拿本次运行的 GitHub OIDC 令牌去 nuget.org 换一把 1 小时有效的
临时密钥。nuget.org 那边靠一条**策略**决定「哪个仓库的哪个工作流可以代表我推包」。

⚠️ **拆库之后需要三条策略,一个仓库一条** —— 策略按 (owner, repository, workflow file)
匹配,三个仓库这三项各不相同。策略的 owner 覆盖该账号名下**全部**包,所以不必按包开。

本仓库这一条填:

| 策略字段 | 值 |
| --- | --- |
| Policy name | `velashell-plugin-templates`(随意,能认出来就行) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShellLabs` |
| **Repository** | `velashell-plugin-templates` |
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

另外 job 上的 `permissions: id-token: write` 不能少,否则 GitHub 根本不签发 OIDC 令牌。

---

## 五、本仓库不需要任何机密

不产出程序集,因此既不签名也不需要 `STRONG_NAME_KEY`;推送走 OIDC。
于是 fork PR 与主分支跑的是完全同一条 CI 路径。

---

## 六、端到端冒烟

发布前会跑 `scripts/Invoke-Smoke.ps1`:装上刚打出的模板包,对**两个模板**各走一遍
`dotnet new` → 还原 → `dotnet build -t:PackVpx` → 读回容器 → 查共享程序集有没有漏进
插件输出目录。

**这一步不是形式。** 模板包是纯内容包,`dotnet pack` 成功几乎什么都不能说明 ——
真正会出问题的地方全在包装好之后:

* `.template.config` 以点开头,没开 `NoDefaultExcludes` 的话根本不会进包 ——
  而且**不报错**,只是 `dotnet new` 列表里看不见它;
* `sdkVersion` 的默认值指向一个不存在(或还没发布)的 `.Build` 版本;
* 模板里的示例代码用到了新版契约的 API,而它引用的 `.Build` 包还是老的。

三种都只有"真的装一次、真的生成一次、真的构建一次"才会显形。
