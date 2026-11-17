# 资源包与更新

这一页讲包级运维：播放模式、包初始化、版本与清单、补丁下载、多包、资源查询、缓存清理，以及构建侧的收集规则与打包入口。
单条资源的加载与释放看[加载、租赁与绑定](loading.md)。

## 适用场景

- 启动流程里初始化资源包并校验版本、清单。
- 做补丁更新界面：统计待下载数量与体积、显示进度、失败重试。
- 一个工程拆多个资源包（基础包 + DLC + 活动包）。
- 上线前清理沙盒缓存，或按标签清理指定资源。
- 出包：配置 Collector、打 AssetBundle、把播放模式写进 Player 包。

不适用：单资源加载（用 `IResourceService`）、场景加载（见 [Scene](../scene/index.md)）、CDN 上传。

## 使用前提

- `ResourceComponent` 已 `Awake`。它调用 `Initialize()` 完成 `YooAssets.Initialize(new ResourceLogger())`、`YooAssets.SetAsyncOperationMaxTimeSlice(milliseconds)` 并创建默认包。没有这一步，`InitPackageAsync` 直接返回 false。
- `PlayMode` 已确定：编辑器下取 `EditorPrefs`，Player 下取序列化值（正式出包由 `ResourceComponentBuildProcessor` 写入）。
- `HostPlayMode` / `WebPlayMode` 需要可用的 host 地址，结尾是否带 `/` 由 `RemoteServices.NormalizeHostPrefix` 补齐。
- 默认包名来自 `ResourceComponent.packageName`（默认 `DefaultPackage`），要与 Collector 里的 `PackageName` 一致。

## 最短接入

1. 取服务，按模式准备 host 地址。只有联机 / WebGL 需要 CDN：

```csharp
using AlicizaX;
using YooAsset;

IResourcePackageService packages = GameApp.ResourcePackage;

string hostUrl = string.Empty;
if (packages.PlayMode == EPlayMode.HostPlayMode || packages.PlayMode == EPlayMode.WebPlayMode)
{
    hostUrl = StartupSetting.CDNUrl;
}
```

2. 初始化包。空字符串表示用默认包名；`false` 表示这次没成功（模式未指定、YooAssets 未初始化、初始化失败）：

```csharp
bool initialized = await packages.InitPackageAsync(string.Empty, hostUrl, hostUrl);
if (!initialized)
{
    Log.Error("Resource package initialization failed.");
    return;
}
```

3. 请求版本号。版本号在 operation 上，不会自动写回属性：

```csharp
RequestPackageVersionOperation versionOperation = packages.RequestPackageVersionAsync();
await versionOperation.ToUniTask();
if (versionOperation.Status != EOperationStatus.Succeeded)
{
    Log.Error(versionOperation.Error);
    return;
}

packages.PackageVersion = versionOperation.PackageVersion;
```

4. 加载清单。清单成功之前任何资源都读不到；成功后 `PackageVersion` / `ApplicableGameVersion` / `InternalResourceVersion` 才会被框架刷新：

```csharp
LoadPackageManifestOperation manifestOperation =
    packages.LoadPackageManifestAsync(versionOperation.PackageVersion);
await manifestOperation.ToUniTask();
if (manifestOperation.Status != EOperationStatus.Succeeded)
{
    Log.Error(manifestOperation.Error);
    return;
}
```

5. 有补丁就下载，没有就进游戏（下载器见下一节）。

## 核心用法

### 播放模式

`PlayMode` 类型是 YooAsset 的 `EPlayMode`，真实取值与框架行为如下：

| 值 | 名称 | 框架用的初始化选项 | 文件系统 | 需要 host |
| :--- | :--- | :--- | :--- | :--- |
| 0 | `None` | 无（`InitPackageAsync` 返回 false） | — | — |
| 1 | `EditorSimulateMode` | `EditorSimulateModeOptions` | 编辑器模拟构建产物（反射调用 `EditorSimulateBuildInvoker`） | 否 |
| 2 | `OfflinePlayMode` | `OfflinePlayModeOptions` | Builtin | 否 |
| 3 | `HostPlayMode` | `HostPlayModeOptions` | Builtin + Sandbox 缓存 | 是 |
| 4 | `WebPlayMode` | `WebPlayModeOptions` | WebServer（WebGL；微信小游戏另加 WebNetwork） | 是 |
| 5 | `CustomPlayMode` | 无（返回 false） | — | — |

- `HostPlayMode` / `WebPlayMode` 下 host 为空会抛 `ArgumentNullException`（`GameFrameworkGuard.NotNullOrEmpty`），不会静默失败。
- `autoUnloadBundleWhenUnused` 会写进所有模式的初始化选项。
- `EditorSimulateMode` 的构建入口通过反射调用 YooAsset 编辑器程序集，只能在编辑器里跑。
- 框架自带 `RemoteServices` 作为 `IRemoteService`：主 host 拼文件名，配了 fallback 时返回两个候选 URL，由 YooAsset 依次尝试。

### 版本与清单

- `InitPackageAsync` 只建包、装文件系统；清单还没加载时 `RefreshPackageVersion` 会因为 `PackageValid == false` 直接返回，所以三个版本属性此时仍是空值 / 0。
- `LoadPackageManifestAsync` 会先清空资源信息缓存，成功后刷新默认包版本：

| 属性 | 来源 | 何时有值 |
| :--- | :--- | :--- |
| `PackageVersion` | `ResourcePackage.GetPackageVersion()` | 默认包清单加载成功后 |
| `ApplicableGameVersion` | `Application.version` | 同上 |
| `InternalResourceVersion` | 包版本字符串的 32 位哈希 | 同上；只适合判断「变没变」 |

- `GetPackageVersion(customPackageName)` 直接问 YooAsset：包不存在返回空字符串；包存在但未初始化或清单未就绪时 YooAsset 的 `CheckInitialized` 会抛 `YooPackageInvalidException`。
- `PackageVersion` 是可读写属性，`RequestPackageVersionAsync` 不会自动赋值；工程做法是 `packages.PackageVersion = versionOperation.PackageVersion;` 再传给清单加载。

```csharp
string packageVersion = GameApp.ResourcePackage.GetPackageVersion();
string gameVersion = GameApp.ResourcePackage.ApplicableGameVersion;
int internalVersion = GameApp.ResourcePackage.InternalResourceVersion;
Log.Info($"package {packageVersion} / game {gameVersion} / internal {internalVersion}");
```

### 下载器

`CreateResourceDownloader(customPackageName)` 用组件上的 `downloadingMaxNum` 与 `failedTryAgain` 构造 `ResourceDownloaderOptions`（并发会被 YooAsset 夹到 1–32）。

```csharp
ResourceDownloaderOperation downloader = GameApp.ResourcePackage.CreateResourceDownloader();
if (downloader.TotalDownloadCount == 0)
{
    Log.Info("No asset needs to be downloaded.");
    return;
}

downloader.DownloadProgressChanged += data =>
    Log.Info($"{data.CurrentDownloadCount}/{data.TotalDownloadCount} ({data.Progress:P1})");
downloader.DownloadError += data =>
    Log.Error($"Package:{data.PackageName} File:{data.FileName} Error:{data.ErrorInfo}");

downloader.StartDownload();
await downloader;

if (downloader.Status != EOperationStatus.Succeeded)
{
    Log.Error(downloader.Error);
}
```

可读成员：`TotalDownloadCount`、`TotalDownloadBytes`、`CurrentDownloadCount`、`CurrentDownloadBytes`、`Progress`、`Status`、`Error`。事件有 `DownloadProgressChanged`、`DownloadError`、`DownloadCompleted`、`DownloadFileStarted`。暂停 / 恢复 / 取消用 `PauseDownload()`、`ResumeDownload()`、`CancelDownload()`，取消会把状态置为失败。工程里的失败重试做法是在 `DownloadError` 里重建下载器再走一遍。

### 多包

- `DefaultPackageName` 只影响「没显式传包名」的入口：加载、查询、版本、清单、下载器、缓存清理。它不会重命名已创建的包。
- `InitPackageAsync(packageName, host, fallbackHost)` 会复用已有的同名 YooAsset 包或新建一个；同包并发调用共享同一次初始化任务；已初始化成功的包直接返回 true 并刷新版本。
- 加载时把包名传进去即可：

```csharp
bool dlcReady = await GameApp.ResourcePackage.InitPackageAsync("DLC", cdnUrl, cdnUrl);
using AssetLease<Sprite> lease =
    await GameApp.Resource.LoadLeaseAsync<Sprite>("icon_dlc", cancellationToken, "DLC");
```

- 场景加载固定使用默认包（`SceneService` 读 `DefaultPackageName`），多包资源不能给 `LoadMainAsync` / `LoadAdditiveAsync` 用。
- 初始化失败时该包会从内部包表里移除并返回 false，不会留下半初始化的包。

### 查询与校验

| API | 返回 | 说明 |
| :--- | :--- | :--- |
| `IsLocationValid(location, packageName)` | `bool` | location 是否在清单里 |
| `HasAsset(location, packageName)` | `HasAssetResult` | `NotExist` / `AssetOnline`（下载大小 > 0）/ `AssetOnDisk` |
| `GetAssetInfo(location, packageName)` | `AssetInfo` | 带 256 条缓存，清单重新加载时清空 |
| `GetAssetInfos(resTag, packageName)` | `AssetInfo[]` | 按单个标签查 |
| `GetAssetInfos(tags, packageName)` | `AssetInfo[]` | 按标签数组查 |

```csharp
HasAssetResult state = GameApp.ResourcePackage.HasAsset("UIShopWindow");
if (state == HasAssetResult.AssetOnline)
{
    // 需要下载后才保证能同步读到
}
```

包不存在时这些入口走 `GetPackageOrThrow`，抛 `GameFrameworkException`（`The package does not exist. Package Name :…`），不是返回 null。

### 缓存清理

```csharp
await GameApp.ResourcePackage.ClearAllBundleFiles().ToUniTask();
await GameApp.ResourcePackage.ClearCacheAsync(
    new ClearCacheOptions(ClearCacheMethods.ClearBundleFilesByTags, "Launch")).ToUniTask();
```

`ClearCacheMethods` 的可用取值：`ClearAllBundleFiles`、`ClearUnusedBundleFiles`、`ClearBundleFilesByLocations`、`ClearBundleFilesByTags`、`ClearAllManifestFiles`、`ClearUnusedManifestFiles`。清理只作用于沙盒缓存，Builtin 里的随包资源不受影响，也不会删除 CDN 上的文件。

### 关闭后端

`ShutdownBackend()` 会清空内部包表并 `YooAssets.Destroy()`。它唯一该被调用的地方是进程退出路径：`RootModule.Shutdown` 先 `AppServices.Shutdown()`（服务 `Dispose` → `ResourceService.Shutdown` 释放记录、租约、作用域、包初始化任务），**之后**才调 `resources?.ShutdownBackend()`。顺序反了会让资源服务在已经销毁的包上做清理。业务代码不要调这个接口。

### 构建侧：收集规则与打包入口

收集规则在 `Client/Assets/YooAsset/AssetBundleCollectorSetting.asset`：

| 项 | 值 |
| :--- | :--- |
| 包名 | `DefaultPackage` |
| `EnableAddressable` | 1（地址可寻址） |
| `SupportExtensionless` | 1（地址不带扩展名） |
| `AutoCollectShaders` | 1 |
| `IgnoreRuleName` | `LocalizationAssetIgnoreRule` |
| 分组 | Entity、UIRaw、Audios、Configs、DLL、Effects、Materials、Scenes、Shaders、UI、ShaderVariants |
| `AddressRuleName` | 全部 `AddressByFileName` |
| `PackRuleName` | `PackDirectory` / `PackCollector` / `PackSeparately` / `PackShader` / `PackShaderVariants` |

`AddressByFileName` 意味着代码里的 location 是**不带扩展名的文件名**：`Assets/Bundles/UI/Window/UIShopWindow.prefab` → `UIShopWindow`。同名文件跨目录会撞地址，新增资源前先确认文件名唯一。

打包入口：

| 目的 | 入口 | 关键参数 |
| :--- | :--- | :--- |
| 命令行打 AB | `BuildCli.RunResources()` | `-uclParams <json>`，环境变量 `UCL_PARAM_PACKAGE_VERSION`、`UCL_PARAM_ENCRYPTION` |
| 菜单打 AB | `BuildCliMenu` 的 `Build/Offline-Mode/Build_AB`、`Build/Online-Mode/Build_AB` | 输出 `../Bundle` |
| 实际构建 | `ResourceBuildHelper.BuildResourcePackage` → `ScriptableBuildPipeline.Run` | `ScriptableBuildParameters` |
| 出 Player 包 | `BuildCli.Execute()` → `AppBuildHelper` | 环境变量 `UCL_PARAM_RES_MODE`、`UCL_PARAM_DECRYPTION` |

- 版本号：未显式传时用 `yyyy-MM-dd-<当天分钟数>` 生成。
- 输出流程：先写 `<output>.building`，成功后把 `report.OutputPackageDirectory` 拷到 `<output>.next`，再整体替换正式目录；中途失败不会留下半个包。
- Online 模式只复制 tag 为 `Launch` 的 bundle，Offline 模式复制全部。
- 加密：构建侧 `EncryptionServiceType` 指向 `IBundleEncryptor`；框架自带 `FileStreamEncryption`（按字节异或 `64`）与 `FileOffsetEncryption`（前置 32 字节偏移）。运行时对应 `FileStreamDecryption`、`FileOffsetDecryption`，在 Inspector 的 Decryption Service 下拉里按类型全名选择，两边必须配对。
- Player 包的播放模式：`ResourcePlayerBuildConfig.Set(playMode, decryptionServices)` 在 `BuildPipeline.BuildPlayer` 之前调用，`ResourceComponentBuildProcessor` 把它写进构建副本里的 `ResourceComponent`；`InjectedCount == 0`（命中场景数据缓存）时会带 `BuildOptions.CleanBuildCache` 重打一次，仍为 0 则构建失败。

## 常见配置

| 场景 | 配置 | 说明 |
| :--- | :--- | :--- |
| 编辑器日常开发 | `PlayMode = EditorSimulateMode` | 改资源不用打 AB，仅编辑器 |
| 单机版 | `PlayMode = OfflinePlayMode` | 无 CDN，包内资源直读 |
| 联机版 | `PlayMode = HostPlayMode` + CDN | 首次启动要下载，沙盒缓存 |
| WebGL / 小游戏 | `PlayMode = WebPlayMode` + CDN | 走 WebServer 文件系统 |
| 下载并发 | `downloadingMaxNum` | 组件默认 10，Inspector 滑杆上限 48，YooAsset 内部上限 32 |
| 下载重试 | `failedTryAgain` | 组件默认 3，按文件计 |
| 引用归零即卸 bundle | `autoUnloadBundleWhenUnused` | 省内存但会提高重复加载概率 |
| 包体加密 | `decryptionServices` + 构建加密 | 类型名写错会在 `InitPackageAsync` 抛异常 |

## 容易踩的坑

1. **`InitPackageAsync` 返回 false 不是异常**。先查 `PlayMode` 是不是 `None` / `CustomPlayMode`、host 有没有传、`ResourceComponent` 是否已经跑过 `Awake`。
2. **版本号不会自动写回**。`RequestPackageVersionAsync` 只产出 operation；要用 `operation.PackageVersion`，或自己赋给 `packages.PackageVersion`。
3. **默认包是个隐式参数**。所有带 `customPackageName` / `packageName` 默认值的入口都落到 `DefaultPackageName`；多包项目里漏传就是「资源明明下载了却读不到」。
4. **清单没加载成功就等于没有资源**。`InitPackageAsync` 只建包，不加载清单；跳过 `LoadPackageManifestAsync` 会一直加载失败。
5. **包不存在时查询会抛异常**。`GetAssetInfo` / `HasAsset` / `IsLocationValid` / 下载器 / 清理都走 `GetPackageOrThrow`，先 `InitPackageAsync` 再查。
6. **`EditorSimulateMode` 不能出包**。Player 包里没有 YooAsset 编辑器程序集，必须由构建注入 `OfflinePlayMode` / `HostPlayMode` / `WebPlayMode`。

## API 速查

`IResourcePackageService`（`GameApp.ResourcePackage`）：

```csharp
string DefaultPackageName { get; set; }
EPlayMode PlayMode { get; set; }
string DecryptionServices { get; set; }
bool AutoUnloadBundleWhenUnused { get; set; }
long Milliseconds { get; set; }
int DownloadingMaxNum { get; set; }
int FailedTryAgain { get; set; }
string PackageVersion { get; set; }
string ApplicableGameVersion { get; }
int InternalResourceVersion { get; }

void Initialize();
UniTask<bool> InitPackageAsync(string packageName = "", string hostServerURL = "", string fallbackHostServerURL = "");

string GetPackageVersion(string customPackageName = "");
RequestPackageVersionOperation RequestPackageVersionAsync(bool appendTimeTicks = false, int timeout = 60, string customPackageName = "");
LoadPackageManifestOperation LoadPackageManifestAsync(string packageVersion, int timeout = 60, string customPackageName = "");

ResourceDownloaderOperation CreateResourceDownloader(string customPackageName = "");
ClearCacheOperation ClearCacheAsync(ClearCacheOptions options, string customPackageName = "");
ClearCacheOperation ClearAllBundleFiles(string customPackageName = "");

HasAssetResult HasAsset(string location, string packageName = "");
bool IsLocationValid(string location, string packageName = "");
AssetInfo GetAssetInfo(string location, string packageName = "");
AssetInfo[] GetAssetInfos(string resTag, string packageName = "");
AssetInfo[] GetAssetInfos(string[] tags, string packageName = "");

void ShutdownBackend();
```
