# 初始化资源包

`ResourceComponent.Awake()` 只创建并注册默认包对象，真正初始化要调用 `InitPackageAsync()`：

```csharp
using AlicizaX;
using Cysharp.Threading.Tasks;
using UnityEngine;

public sealed class ResourceInitExample : MonoBehaviour
{
    private async UniTaskVoid Start()
    {
        bool succeed = await GameApp.ResourcePackage.InitPackageAsync(
            packageName: "DefaultPackage",
            hostServerURL: "https://cdn.example.com/Android",
            fallbackHostServerURL: "https://fallback-cdn.example.com/Android");

        Debug.Log($"Init package succeed: {succeed}");
    }
}
```

不同运行模式：

- `EditorSimulateMode`：编辑器模拟构建 + 编辑器文件系统，收集路径以 `AssetBundleCollectorSetting` 为准（当前工程是 `Assets/Bundles/...`）。
- `OfflinePlayMode`：内置文件系统，可配置解密服务。
- `HostPlayMode`：内置文件系统 + 远端缓存文件系统，`hostServerURL` 必须非空。
- `WebPlayMode`：WebServer 文件系统；WebGL 微信小游戏宏下会创建微信缓存文件系统，`hostServerURL` 必须非空。

YooAsset 3 注意：

- `InitPackageAsync` 成功不等于 ActiveManifest 就绪。`GetPackageVersion()` 在无清单时会抛 `Active package manifest not found.`，要等 `LoadPackageManifestAsync` 成功后再读。
- 同一包的并发 `InitPackageAsync` 会合流到同一个 `TaskCompletionSource<bool>`，可以多次等待。

启动流程通常是：`InitPackageAsync` → `RequestPackageVersionAsync` → `LoadPackageManifestAsync` → 创建下载器。

## 查询资源

`HasAsset` / `IsLocationValid` / `GetAssetInfo` / `GetAssetInfos` 都在 `IResourcePackageService` 上：

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;
using UnityEngine;

public sealed class ResourceCheckExample : MonoBehaviour
{
    private void Start()
    {
        IResourcePackageService package = GameApp.ResourcePackage;

        string location = "UIHomeWindow";
        if (!package.IsLocationValid(location))
        {
            Debug.LogError($"Invalid location: {location}");
            return;
        }

        HasAssetResult result = package.HasAsset(location);
        Debug.Log($"Asset state: {result}");
    }
}
```

`HasAssetResult` 取值：

- `NotExist`：资源不存在。
- `Valid`：资源可用（本地或已下载）。
- `AssetOnline`：资源存在，但需要从远端更新下载。
- `AssetOnDisk`：资源存在并位于磁盘。
- `AssetOnFileSystem`：资源存在并位于文件系统。
- `BinaryOnDisk` / `BinaryOnFileSystem`：原生二进制资源在磁盘 / 文件系统。

## 多资源包

大多数 API 的最后一个参数是 `packageName` / `customPackageName`，不传时用 `ResourceComponent.PackageName`。

```csharp
await GameApp.ResourcePackage.InitPackageAsync(
    packageName: "DlcPackage",
    hostServerURL: "https://cdn.example.com/DLC",
    fallbackHostServerURL: "https://fallback-cdn.example.com/DLC");

using AssetLease<Texture2D> lease =
    await GameApp.Resource.LoadLeaseAsync<Texture2D>("icon_dlc", packageName: "DlcPackage");
```

## 下载、版本和清单

下载器使用 Inspector 里的 `Downloading Max Num` 和 `Failed Try Again`：

```csharp
using YooAsset;

ResourceDownloaderOperation downloader = GameApp.ResourcePackage.CreateResourceDownloader();
if (downloader.TotalDownloadCount > 0)
{
    downloader.DownloadErrorCallback = data =>
        Debug.LogError($"Download error: {data.PackageName}/{data.FileName}, {data.ErrorInfo}");

    downloader.DownloadUpdateCallback = data =>
        Debug.Log($"Download: {data.CurrentDownloadCount}/{data.TotalDownloadCount}");

    downloader.BeginDownload();
    await downloader;
}
```

版本与清单：

```csharp
RequestPackageVersionOperation versionOperation = GameApp.ResourcePackage.RequestPackageVersionAsync();
await versionOperation.ToUniTask();

if (versionOperation.Status == EOperationStatus.Succeeded)
{
    GameApp.ResourcePackage.PackageVersion = versionOperation.PackageVersion;

    LoadPackageManifestOperation manifestOperation =
        GameApp.ResourcePackage.LoadPackageManifestAsync(versionOperation.PackageVersion);
    await manifestOperation.ToUniTask();
}
```

编辑器模拟模式下清单版本通常是 `"Simulate"`，不要拿远端版本号去加载模拟清单。

```csharp
string version = GameApp.ResourcePackage.GetPackageVersion();
```

只在 ActiveManifest 就绪后调用；初始化刚成功时用 `RequestPackageVersionAsync` / `PackageVersion`。

## 缓存清理

```csharp
using YooAsset;

ClearCacheOperation clearUnused =
    await GameApp.ResourcePackage.ClearCacheAsync(
        new ClearCacheOptions(ClearCacheMethods.ClearUnusedBundleFiles));

ClearCacheOperation clearAll = GameApp.ResourcePackage.ClearAllBundleFiles("DlcPackage");
```

`ClearAllBundleFiles()` 只是发起操作，不会等待完成；需要等待结果时用 `ClearCacheAsync()`。
