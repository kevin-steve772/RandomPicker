# AGENTS.md

ClassIsland 插件「随机抽选」：单项目 C# / .NET 8 / Avalonia，没有 solution、测试、lint、CI 或 pre-commit。
唯一能验证改动的手段是编译。

## 构建

- `dotnet build -c Release`
- **本仓库不能独立编译。** csproj 里 `ProjectReference` 指向 `..\..\ClassIsland.Core\ClassIsland.Core.csproj`，
  必须把仓库克隆到 ClassIsland 源码树的 `plugins/` 下（README 也这么写）。
  独立检出直接编译会报约 36 个 CS0246/CS0234（缺 `ClassIsland.Core`、`ClassIsland.Shared`、
  `Microsoft.Extensions.*`、`Windows.*` 等）。这是预期行为，不是代码坏了 ——
  不要为了「让编译过」改 using 或删功能。
- 还需要 `microsoft.windows.sdk.net.ref` 10.0.19041.57 在 NuGet 缓存里：csproj 用
  `HintPath` 硬路径引用 `Microsoft.Windows.SDK.NET` / `WinRT.Runtime`，**不是** PackageReference，
  本仓库的 restore 不会去拉它（得靠宿主树那边的 restore 先放进缓存）。
- 没有测试套件。改动后只能靠编译 + 人工验证。

## 打包引用规则（最容易改错的地方）

csproj 注释是硬结论的来源，动 `Reference`/打包之前先读。

- **宿主已有的**（`ClassIsland.Core`、Avalonia、FluentAvaloniaUI、WinRT 两件套）：
  引用上标 `Private="false"` / `ExcludeAssets="runtime"`（见 csproj，Avalonia 主包是普通
  PackageReference，靠打包清单把它排除），**绝不能进插件包**。
  宿主 `PluginLoadContext` 有 WinRT 白名单，会强制解析到宿主那份。
- **插件自带的**（ONNX Runtime）：`Private="true"`，dll + 原生库 + 模型平铺到输出根目录，
  且必须在 `pack-include.txt` 里列一行 —— 那份清单决定哪些文件进 `.cipx`。
  新增任何随包依赖 = csproj 引用 + `None Include CopyToOutputDirectory` + `pack-include.txt` 三处同步。
- `GenerateDependencyFile=false`（不生成 deps.json）是刻意的：生成了宿主的
  `AssemblyDependencyResolver` 会把本该用宿主那份的 Avalonia 一起拖下水。
  代价是没有 deps.json，程序集/原生库要手动挂解析回调：
  `RandomPickerPlugin.EnsureAssemblyResolvable`、`FaceModel.EnsureManagedResolvable`、
  `FaceModel.EnsureNativeResolvable`。**新增随包托管依赖时这几处要一起改。**
- 不要把 TFM 切成 `net8.0-windows10.0.19041.0`（会触发框架引用还原，注释里说明了为什么走不通）。
- `Native/` 下的 dll 和 onnx 是**提交进仓库的二进制**，不是生成物，别删别重新生成。

## 版本号

`ClassIsland.RandomPicker.csproj` 的 `<Version>` 和 `manifest.yml` 的 `version` 必须一致
（当前 1.1.0.0）；`manifest.yml` 的 `apiVersion` 是宿主 API 级别，非必要不改。

## 运行时布局

- 设置与名单在宿主给的 `PluginConfigFolder` 下：`settings.json`、`名单.txt`、
  可选 `名单-文字.txt`，存原图时还有照片。名单是纯文本、`FileSystemWatcher` 热加载。
- 入口 `RandomPickerPlugin` → 注册 `PickerHostService`（IHostedService，悬浮窗与抽选主流程）、
  `PickerNotificationProvider`、`PickerSettingsPage`。
- 拍照抽人走 WinRT `MediaCapture`/`MediaFrameReader`（Windows 专属），失败时静默退回按名单抽；
  `TopmostEnforcer` 用 `OperatingSystem.IsWindows()` 门控。manifest 声称支持三平台，
  改平台相关代码时别破坏非 Windows 下的编译与降级路径。

## 约定

- 注释、UI 字符串、README、commit message 全部是**中文**，新写的也保持中文。
  注释风格是「记录来之不易的实测结论/为什么不能那样改」，不是复述代码在干什么。
- 随机数一律 `RandomNumberGenerator`（操作系统熵源），不要用 `Random`。
- 提交历史只有 `main`，仓库里没有 CI workflow。

## 参考

- `README.md`：用户视角的完整行为说明（三种模式、阈值实测数据、置顶实现），与代码冲突时以代码为准。
