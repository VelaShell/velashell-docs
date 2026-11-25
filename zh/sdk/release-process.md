# 发版流程(契约 SDK)

> 本篇只讲**本仓库**(velashell-plugin-sdk)怎么发版。`vela-plugin` 见
> [velashell-plugin-cli](../cli/release-process.md),
> `dotnet new` 模板见
> [velashell-plugin-templates](../templates/release-process.md)。

本仓库一次发布产出三个包,共用 Release 标签里的版本号:

| 包 | 内容 |
| --- | --- |
| `VelaShell.PluginSdk` | 契约程序集 |
| `VelaShell.PluginSdk.Testing` | 测试替身 |
| `VelaShell.PluginSdk.Build` | 插件工程**只需引用的那一个包**:targets、随包分发的打包器、共享运行时排除与清单/版本核对 |

`.Build` 于 2026-09-11 从 velashell-plugin-cli 搬来;`.vpx` 打包器(`src/VelaShell.PluginSdk.Packer`,
`IsPackable=false`)也在本仓库,不单独发包,只以构建产物的形式躺在 `.Build` 包的 `tools/` 下。
`.Build` 对契约与打包器都是 `ProjectReference`,所以发给插件作者的契约与打包器永远就是同一次发布的这一版。

**下游不必跟着发。** cli 与 templates 两个仓库各有各的版本号:cli 只在想吃到新契约时才把自己引用的
`VelaShell.PluginSdk` 版本抬上来,模板仓库只在想让新建工程指向新版时才抬 `VelaBuildPackageVersion`
(生成的工程引用哪一版 `.Build`)。SDK 发 1.6.0 不代表 `vela-plugin` 要变成 1.6.0 —— 那正是拆库要的效果。

---

## 一、怎么发

三步:

1. **破坏性变更**才需要:手工把 `src/VelaShell.PluginSdk/VelaPluginApi.cs` 里的
   `VelaPluginApi.Level` +1。脚本会核对但**不代改** —— 「契约破没破」是人的判断,
   不是版本号的推论。

2. 本地落版本号,连同功能改动一起合进 `main`:

   ```powershell
   pwsh scripts/Set-Version.ps1 1.6.0
   ```

3. 在 GitHub 上**发 Release**,标签填 `v1.6.0`(带不带 `v` 都行,流水线会 `TrimStart`,
   但建议统一带)。预发布勾 prerelease,标签用 `v1.6.0-preview.1`。

流水线在解析出标签之后**第一件事**也会跑一遍 `Set-Version.ps1`,只改 runner 上的工作区、
**不回写仓库** —— 于是产物版本号永远等于 Release 标签,与仓库里当时提交了什么无关。

### 忘了在发版前落版本号怎么办

不影响这一次发布(Stamp 步骤已经兜住了),但 `main` 落后了:`main` 上的 CI
「Version consistency check」会红一次。照它给的命令本地跑一遍 `Set-Version.ps1`,
补一个 PR 合掉即可。

### 手动补跑

`release` 事件偶尔不触发。Actions 页面 → 选 Release 工作流 → Run workflow → 填标签。
推送用 `--skip-duplicate`,对同一标签重复跑是幂等的。想只验不推,勾上 `dryRun`。

---

## 二、NuGet 可信发布(Trusted Publishing)怎么调

推送不存 API Key:工作流拿本次运行的 GitHub OIDC 令牌去 nuget.org 换一把 1 小时有效的
临时密钥。nuget.org 那边靠一条**策略**决定「哪个仓库的哪个工作流可以代表我推包」。

⚠️ **拆库之后需要三条策略,一个仓库一条** —— 策略是按 (owner, repository, workflow file)
匹配的,三个仓库这三项各不相同。策略的 owner 覆盖该账号名下**全部**包,所以不必按包开。

本仓库这一条填:

| 策略字段 | 值 |
| --- | --- |
| Policy name | `velashell-plugin-sdk`(随意,能认出来就行) |
| Policy owner | `joes_du` |
| Repository Owner | `VelaShell` |
| **Repository** | `velashell-plugin-sdk` |
| **Workflow File** | `release.yml` —— **只填文件名**,不要写 `.github/workflows/` 前缀 |
| Environment | 留空(工作流没用 GitHub Environments) |

建法:登录 nuget.org → 右上角用户名 → **Trusted Publishing** → **Add**。

顺带把拆库前那条指向 `velashell-plugin-toolchain` 的策略删掉 —— 那个仓库不再推包,
留着就是一条多余的信任面。

### ⚠️ 新策略有 7 天窗口

私有仓库上新建的策略是「**临时激活**」状态,7 天内必须成功发布一次,否则自动失效
(可以随时重开窗口)。原因是 nuget.org 要在第一次成功发布时把 GitHub 的 repository ID
与 owner ID 记进策略,用来把它钉死在那个仓库上(防「删库重建同名仓库」的复活攻击)——
没有一次真实发布就拿不到那两个 ID。所以**建好策略就尽快发一次**,哪怕是 preview 版。

公开仓库通常直接是永久激活状态,但发一次验证仍然是省事的做法。

### 换不到密钥时先看这三样

`NuGet login` 那一步失败,九成是策略对不上:

* Repository 还写着 `velashell-plugin-toolchain`;
* Workflow File 写成了 `.github/workflows/release.yml`;
* `NUGET_USER` 填成了邮箱 —— 要的是 nuget.org 的**用户名**(profile name)。
  工作流默认取 `vars.NUGET_USER`,没配则回落到 `joes_du`。

另外 job 上的 `permissions: id-token: write` 不能少,否则 GitHub 根本不签发 OIDC 令牌。

---

## 三、版本号纪律

`Directory.Build.props` 的 `VelaSdkVersion` 是本仓库的默认版本,发布时由标签覆盖。

### 三个版本号各司其职,别混

| 属性 | 值 | 作用 |
| --- | --- | --- |
| `AssemblyVersion` | `<主版本>.0.0.0` | **绑定标识**,只随主版本动。插件是编译期绑到这个标识上的 |
| `FileVersion` | 完整数字版(`1.6.0`) | 资源管理器属性页看到的那个 |
| `InformationalVersion` | 完整版本含预发布后缀 | `vela-plugin` 报的版本 |

`AssemblyVersion` 钉在主版本上,是因为让它跟着补丁号动等于每发一次补丁就要所有已编译
插件重新绑定,毫无收益。而主版本变了意味着契约破了 —— 那一刻 **apiLevel 必须同步 +1**,
于是老宿主在**发现期**就按 apiLevel 干净拒载,而不是等装载时抛一个看不懂的绑定异常。

### 「SDK 主版本 == apiLevel」

这条纪律由 `Set-Version.ps1` 硬核对:要发 `2.x.x` 就必须先把 `VelaPluginApi.Level` 改成 `2`,
反之亦然。apiLevel 是**契约的属性**,所以这条检查只在本仓库有落点 —— CLI 和模板都无权动它。

### 版本号的落点(由脚本统一维护)

拆库之后只剩四处,其中两处在 velashell-docs:

| 落点 | 漏改的后果 |
| --- | --- |
| `Directory.Build.props` 的 `VelaSdkVersion` | 包版本不对 |
| `src/VelaShell.PluginSdk/VelaPluginApi.cs` 的 `SdkVersion` | **什么都不会报错**,只是 `vela-plugin doctor` 从此汇报一个错的宿主 SDK 版本,插件的 `minSdkVersion` 门槛跟着判错 |
| velashell-docs 的 `zh/sdk/sdk-reference.md` 版本横幅 | 给人照抄的过期数字 |
| velashell-docs 的 `en/sdk/sdk-reference.md` 版本横幅 | 同上 |

最后两处在另一个仓库,是**可选**落点:脚本按 `-DocsRoot` / `$env:VELASHELL_DOCS` / 同级 `../velashell-docs` 找到它才改,找不到就跳过并警告。

跑 `pwsh scripts/Set-Version.ps1 <版本>` 一次全改。CI 用 `-Check` 做体检。

---

## 四、Avalonia 依赖边界:宿主定版本,插件显式引用

`VelaShell.PluginSdk` 与 `VelaShell.PluginSdk.Build` **不依赖 Avalonia,也不导出任何 Avalonia pin**。
这让不带 UI 的插件无需还原 UI 框架,也避免把某个 SDK 发布版本误当成所有宿主的 UI 版本。

准确的 Avalonia 版本由目标宿主维护在 `VelaShell/src/Directory.Packages.props`。插件工程若使用 UI,
要把目标宿主的值写成自己的属性,并显式引用对应包:

```xml
<PropertyGroup>
  <VelaAvaloniaVersion>12.2.0</VelaAvaloniaVersion>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="Avalonia" Version="$(VelaAvaloniaVersion)" ExcludeAssets="runtime" />
</ItemGroup>
```

若插件使用 `Avalonia.AvaloniaEdit`,还要添加同一宿主的版本与直接引用:

```xml
<PropertyGroup>
  <VelaAvaloniaEditVersion>12.0.0</VelaAvaloniaEditVersion>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="Avalonia.AvaloniaEdit" Version="$(VelaAvaloniaEditVersion)" ExcludeAssets="runtime" />
</ItemGroup>
```

上面的 `12.2.0` 和 `12.0.0` 只是示例,必须替换为目标宿主 `src/Directory.Packages.props` 中的准确值,不能从
SDK 或文档横幅推断。不使用这些包时不要声明对应属性或 PackageReference。采用中央包管理时,版本可在插件
自己的 `Directory.Packages.props` 中设为对应属性,但仍要保留直接 `PackageReference` 和运行时资产排除。

`.Build` 在解析引用后核对实际的 `Avalonia` / `Avalonia.AvaloniaEdit` 包版本与插件的目标属性;
版本不一致会在构建期失败。直接引用同时提供编译与 AXAML/compiler 的 build 资产,而
`ExcludeAssets="runtime"` 加上 `.Build` 的打包规则确保共享运行时 DLL 不进入输出和 `.vpx`。
`VelaShell.PluginSdk.dll` 与名字以 `Avalonia` 开头的共享运行时程序集都不会随包分发;宿主未提供的
同前缀第三方包不能靠插件自带来绕过装载器规则。

### 从旧 `.Build` 迁移:只做一次

插件首次采用不再传递 Avalonia 的 SDK/.Build 组合时,对照目标宿主做一次迁移:

1. 从 `VelaShell/src/Directory.Packages.props` 读取准确版本;
2. 只为实际使用的 `Avalonia` / `Avalonia.AvaloniaEdit` 设置对应属性并添加直接引用;
3. 运行构建与 `PackVpx`,确认 AXAML 编译资产可用且 `.vpx` 没有共享运行时 DLL。

已有发布的 SDK/.Build 包不会因这份文档而追溯改变;尚未迁移的工程按其所引用包的既有行为处理。
之后宿主升级 Avalonia 时,插件若要针对该宿主重编,只需同步属性和直接引用的版本;这**不要求**重新发布
SDK/.Build,也不要求仅因 Avalonia 更新而递增插件版本或 `minHostVersion`。`minHostVersion` 只填写
插件首次依赖的那个**宿主版本**(能提供所用新宿主 API 的第一个发布),绝不填写 Avalonia 版本或
`minSdkVersion`。

---

## 五、为什么测试必须 Debug

`src/Directory.Build.props` 在 Release 下打开强名称签名(宿主已签名,已签名程序集不能
引用未签名程序集,所以 SDK 必须用与宿主同一把钥匙签)。而签名程序集的
`InternalsVisibleTo` 必须带友元公钥、友元本身也得用同一把钥匙签 —— 测试程序集两条都不满足。

这些用例验的是契约,与优化级别无关,Debug 等价;且 Debug 的 `obj/bin` 与 `pack` 用的
Release 目录彼此隔离,不会把未签名产物混进要发布的包里。

密钥不入库(`.gitignore` 里 `*.snk`),CI 从 `STRONG_NAME_KEY` 机密还原到仓库根。
这是本仓库**唯一**需要的机密。

> cli 与 templates 两个仓库都不签名,因此**不需要**这把钥匙 ——
> 未签名程序集可以引用已签名的,方向是对的。

---

## 六、端到端冒烟

发布前(CI 与 Release 都跑)会跑 `scripts/Invoke-Smoke.ps1 -Feed ./artifacts/nuget -Version <版本>`:拿刚打出的包,
把 `tests/smoke/` 下那个手写的最小插件工程(与 `velaplugin-ui` 模板同形:命令 + 面板 + 编译期 AXAML)复制到临时目录、
从本地源还原、`dotnet build -t:PackVpx`、确认真出了 `.vpx`、用 `.Build` 包 `tools/` 里的打包器把容器读回来、
查共享程序集有没有漏进插件输出目录。本地源只装本仓库刚打出的三个包,产物不出本仓库 —— 这是 CI 的常规步骤,
不是给别的仓库造本地包的办法。

**这一步不是形式。** 插件工程是仓库外环境,仓库内的 `Directory.Build.props` 一条也吃不到
(夹具里那两个空的 `Directory.Build.props`/`.targets` 就是为了切断向上查找)。
历史上正是它先发现了两个坑:

* SDK 带 `RequiresPreviewFeatures`,插件工程不开预览开关就全线 `CA2252`;
* NuGet 默认不传递 `build` 资产,Avalonia 的 AXAML 编译器到不了插件工程 —— 现象是
  运行时找不到控件,而不是构建报错。

两个坑都只有在「什么都不继承」的前提下才会显形。它还守着「打包器真的进了包」:读回用的打包器刻意从包里取,
漏打了在这里就红,而不是等插件作者 `-t:PackVpx` 时拿到 `VELA1003`。**改了 `.Build` 的 targets 或打包器的命令面,必须跑它。**

> 这一步 2026-09-11 随 `.Build` 从 cli 仓库搬来。更早(拆库之前)它用的是 `dotnet new velaplugin-ui`;模板搬走之后
> 不能再那样做:那会让模板仓库出问题时本仓库的 CI 无端变红,而且发本仓库的包还得先有模板包。
> 模板本身的端到端验证由模板仓库自己的 CI 负责。
