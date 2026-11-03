# Resources 模块

Resources 模块是框架对 YooAsset 的资源服务封装。它把「谁在用这份资源」显式化：每次加载产出**租约**（`AssetLease<T>`）或落进**作用域**（`ResourceScope`），引用归零后进入空闲缓存，到期再卸底层句柄。

底层 YooAsset 句柄只活在资源记录里，业务不要直接拿 `AssetHandle`。同一地址的同步/异步加载会合流到同一条记录。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Resource`

运行时主要入口：

- `GameApp.Resource` → `IResourceService`：加载、实例化、回收
- `GameApp.ResourcePackage` → `IResourcePackageService`：包初始化、下载器、版本、清单、缓存清理

## 使用前提

场景中的框架根节点需要挂载 `ResourceComponent`。

## ResourceComponent Inspector 参数

| 字段 | 默认值 | 作用 |
| --- | --- | --- |
| `Play Mode` | `EditorSimulateMode` | YooAsset 运行模式。编辑器中从 `EditorPrefs` 的 `GamePlayMode` 读。 |
| `Package Name` | `DefaultPackage` | 默认包名，`IResourcePackageService.DefaultPackageName` 与 YooAsset 默认包共用。 |
| `Milliseconds` | `30` | YooAsset 异步操作每帧时间片，内部调用 `YooAssets.SetAsyncOperationMaxTimeSlice()`。 |
| `Downloading Max Num` | `10` | 下载器最大并发数。 |
| `Failed Try Again` | `3` | 单个文件下载失败的重试次数。 |
| `Max Unload Unused Assets Interval` | `300` | 自动回收的最大间隔，超过后调用 `UnloadUnusedAssets(releaseIdleAssets: false)`。 |
| `Use System Unload Unused Assets` | `true` | 强制回收并请求 GC 时，是否顺带调用 Unity 的 `Resources.UnloadUnusedAssets()`。 |
| `Min GC Collect Interval` | `30` | `GC.Collect()` 最小间隔，防止短时间重复触发。 |
| `Idle Asset Expire Time` | `60` | 无引用资源进入 Idle 后的过期秒数，是释放底层句柄的权威 TTL。 |
| `Idle Asset Capacity` | `256` | Idle 记录的数量上限，超出后按最少使用回收。 |
| `Expire Process Count Per Frame` | `16` | 每帧处理过期记录的上限。 |
| `Decryption Services` | 空 | YooAsset 解密服务类型名，需实现 `IDecryptionServices`。 |
| `Auto Unload Bundle When Unused` | `false` | 传给 YooAsset 的 `AutoUnloadBundleWhenUnused`。 |

`Application.lowMemory` 会触发 `ResourceService.OnLowMemory()`，请求强制释放未使用资源并执行 GC。

## 所有权模型

YooAsset / Unity 都不知道业务还在不在用这份资源。句柄必须有明确 Owner，不能靠 `Resources.UnloadUnusedAssets()` 去猜。

| 场景 | 推荐 API | 谁释放 |
| --- | --- | --- |
| 临时读表、配置、音频 Clip | `LoadLease<T>` / `LoadLeaseAsync<T>` | `lease.Dispose()`。引用归零后进 Idle，等 `IdleAssetExpireTime` 再卸句柄 |
| 一批资源共享同一生命周期 | `CreateScope()` / `GameApp.Resource.App` | `scope.Dispose()` 一次性释放该作用域内所有条目 |
| 绑到 Image / Renderer | `SetSprite` / `SetMaterial` 等扩展方法 | 物体销毁或重新绑定时由 `ResourceOwner` 自动放 |
| 实例化到场景的 Prefab | `Instantiate` / `InstantiateAsync` | `Destroy(instance)`，实例上的 `ResourceOwner` 解除源 Prefab 引用 |
| 对象池 Prefab 源 | `IGameObjectPoolService` | 池内部持有租约，按池策略 `Dispose` |

资源记录状态：

- `Loading`：正在加载。
- `Active`：还有引用（租约、作用域条目、绑定）。
- `Idle`：没有引用，等待 `IdleAssetExpireTime` 到期。
- `Released`：底层句柄已释放。

`UnloadUnusedAssets(releaseIdleAssets: false)` 只回收已过期的 Idle 记录；传 `true` 忽略 TTL 立即回收。`ForceUnloadAllAssets()` 清空全部记录。

## 初始化资源包

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

## 加载资源（租约）

新代码一律用 `LoadLease` / `LoadLeaseAsync`。租约是一次明确的所有权；`Dispose()` 只交还所有权，不会立刻把资源从内存里抠掉，引用归零后走 Idle TTL。

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;
using UnityEngine;

Texture2D icon;

using (AssetLease<Texture2D> lease = GameApp.Resource.LoadLease<Texture2D>("icon_start"))
{
    icon = lease.Asset;
}
```

异步版本建议传 `CancellationToken`：

```csharp
using System.Threading;
using AlicizaX;
using AlicizaX.Resource.Runtime;
using Cysharp.Threading.Tasks;
using UnityEngine;

public sealed class LoadSpriteExample : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private AssetLease<Sprite> _avatar;

    private async UniTaskVoid OnEnable()
    {
        _cts = new CancellationTokenSource();
        _avatar = await GameApp.Resource.LoadLeaseAsync<Sprite>("avatar_default", _cts.Token);
    }

    private void OnDisable()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;

        if (_avatar.IsValid)
        {
            _avatar.Dispose();
        }
    }
}
```

需要复用一个地址时，先取 `AssetRef<T>`，再反复按 key 加载，省掉字符串查表：

```csharp
AssetRef<Sprite> key = GameApp.Resource.Key<Sprite>("avatar_default");
AssetLease<Sprite> lease = GameApp.Resource.LoadLease(key);
```

同一地址正在异步加载时，同步 `LoadLease` 会加入同一个 Provider，最终只保留一条记录。

## 作用域加载

一批资源如果同生共死（一次战斗、一个界面），用作用域比逐个 `Dispose` 更省心：

```csharp
using AlicizaX.Resource.Runtime;
using UnityEngine;

using ResourceScope scope = GameApp.Resource.CreateScope("Battle");

var atlas = scope.Load<Texture2D>("battle_atlas");
var config = await scope.LoadAsync<TextAsset>("battle_config");
// 离开 using 时，scope 内所有条目一起释放
```

`GameApp.Resource.App` 是进程级的全局作用域，框架自身也挂在上面。

## 加载并实例化 Prefab

`Instantiate()` / `InstantiateAsync()` 加载 Prefab 并实例化。实例会自动挂载或复用 `ResourceOwner`，由它持有源 Prefab 的引用；`Destroy(instance)` 时自动解除。

```csharp
using AlicizaX;
using UnityEngine;

GameObject window = GameApp.Resource.Instantiate("UIHomeWindow", parent);
```

```csharp
GameObject window = await GameApp.Resource.InstantiateAsync("UIHomeWindow", parent, cancellationToken);
```

两个方法在服务未就绪、取消或加载失败时返回 `null`，调用方要判空。

对象池不要走这两条 API。`GameObjectPool` 内部用 `LoadLease<GameObject>` 持有源 Prefab，按池策略 `Dispose`。

## 组件资源绑定

绑定服务把资源绑到组件属性上，生命周期交给 `ResourceOwner`。物体销毁或重新绑定时自动释放，不需要手动 `Dispose`。

```csharp
using AlicizaX.Resource.Runtime;
using UnityEngine.UI;

image.SetSprite("icon_start", ResourceBindingOptions.SetNativeSize);
image.SetSubSprite("CommonAtlas", "btn_start");
image.SetMaterial("ui_mask");

spriteRenderer.SetSprite("player");

renderer.SetSharedMaterial("shared");
renderer.SetMaterialInstance("role", ResourceBindingOptions.Synchronous);
```

可选项 `ResourceBindingOptions`（可组合）：

- `None`：默认，异步加载。
- `SetNativeSize`：绑定成功后对 `Image` 调 `SetNativeSize()`。
- `Synchronous`：要求同步完成绑定。

绑定槽类型：

- `ImageSprite`：绑定 `Image.sprite`。
- `SubSprite`：从图集子资源取 Sprite，绑到 `Image.sprite` 或 `SpriteRenderer.sprite`。
- `SpriteRendererSprite`：绑定 `SpriteRenderer.sprite`。
- `ImageMaterial`：绑定 `Image.material`。
- `RendererSharedMaterial`：绑定 `Renderer.sharedMaterial`。
- `RendererMaterialInstance`：从材质创建运行时实例，绑到 `Renderer.sharedMaterial`。
- `PrefabSource`：Prefab 实例和源 Prefab 之间的绑定。

需要显式回收时，销毁挂 `ResourceOwner` 的物体，或直接 `owner.Scope.Dispose()`。

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

## 资源回收

```csharp
IResourceService resources = GameApp.Resource;

resources.UnloadUnusedAssets(releaseIdleAssets: false); // 只回收已过期的 Idle
resources.UnloadUnusedAssets(releaseIdleAssets: true);  // 忽略 TTL，立即回收无引用记录

resources.ForceUnloadUnusedAssets(performGCCollect: true);
resources.ForceUnloadAllAssets();
```

自动回收流程：

1. `ResourceComponent.Update()` 每帧按 `Expire Process Count Per Frame` 处理过期记录。
2. 达到 `Max Unload Unused Assets Interval` 后调用 `UnloadUnusedAssets(releaseIdleAssets: false)`。
3. 低内存或显式 `ForceUnloadUnusedAssets(true)` 会请求强制回收、Unity 系统卸载和 GC。
4. GC 受 `Min GC Collect Interval` 限制。

## 调试信息

运行时挂载 `DebuggerComponent`，在 `Profiler / Resource` 页可以看到资源记录状态、引用计数、Idle 过期时间和作用域条目。

资源/绑定/作用域的明细结构（`ResourceAssetInfo`、`ResourceScopeInfo`、`ResourceStats`）是服务内部诊断接口，不对外开放；需要扩展调试面板时在框架内加窗口，不要从业务代码直接读。

## API 速查

`IResourceService`（`GameApp.Resource`）：

```csharp
ResourceScope App { get; }
ResourceScope CreateScope(string debugName = null);

AssetRef<T> Key<T>(string location, string packageName = null) where T : Object;
AssetLease<T> LoadLease<T>(AssetRef<T> key) where T : Object;
AssetLease<T> LoadLease<T>(string location, string packageName = null) where T : Object;
UniTask<AssetLease<T>> LoadLeaseAsync<T>(AssetRef<T> key, CancellationToken cancellationToken = default) where T : Object;
UniTask<AssetLease<T>> LoadLeaseAsync<T>(string location, CancellationToken cancellationToken = default, string packageName = null) where T : Object;

GameObject Instantiate(string location, Transform parent = null, string packageName = null);
UniTask<GameObject> InstantiateAsync(string location, Transform parent = null, CancellationToken cancellationToken = default, string packageName = null);

float IdleAssetExpireTime { get; set; }
int IdleAssetCapacity { get; set; }
int IdleReleaseCountPerTick { get; set; }

void UnloadUnusedAssets(bool releaseIdleAssets);
void ForceUnloadUnusedAssets(bool performGCCollect);
void ForceUnloadAllAssets();
void OnLowMemory();
```

`IResourcePackageService`（`GameApp.ResourcePackage`）：

```csharp
void Initialize();
UniTask<bool> InitPackageAsync(string packageName = "", string hostServerURL = "", string fallbackHostServerURL = "");
string GetPackageVersion(string customPackageName = "");
RequestPackageVersionOperation RequestPackageVersionAsync(bool appendTimeTicks = false, int timeout = 60, string customPackageName = "");
LoadPackageManifestOperation LoadPackageManifestAsync(string packageVersion, int timeout = 60, string customPackageName = "");
ResourceDownloaderOperation CreateResourceDownloader(string customPackageName = "");
ClearCacheOperation ClearCacheAsync(ClearCacheOptions options, string customPackageName = "");
ClearCacheOperation ClearAllBundleFiles(string customPackageName = "");

bool IsLocationValid(string location, string packageName = "");
HasAssetResult HasAsset(string location, string packageName = "");
AssetInfo GetAssetInfo(string location, string packageName = "");
AssetInfo[] GetAssetInfos(string resTag, string packageName = "");
AssetInfo[] GetAssetInfos(string[] tags, string packageName = "");
```

`ResourceScope`：

```csharp
bool IsAlive { get; }
T Load<T>(AssetRef<T> key) where T : Object;
T Load<T>(string location, string packageName = null) where T : Object;
UniTask<T> LoadAsync<T>(AssetRef<T> key, CancellationToken cancellationToken = default) where T : Object;
UniTask<T> LoadAsync<T>(string location, CancellationToken cancellationToken = default, string packageName = null) where T : Object;
void Dispose();
```

## 注意事项

- 新代码优先 `LoadLease` / 作用域 / 绑定扩展；不要再找 `LoadAsset` / `LoadGameObject` / `ResourceLeaseHandle`，这些入口已经移除。
- 租约是 `readonly struct`，用 `using` 或手动 `Dispose()` 释放；`Dispose()` 之后 `IsValid` 变 false，`Asset` 字段可能仍指向已释放的对象，不要再用。
- 实例化出来的 Prefab 通过 `Destroy(instance)` 释放，实例上的 `ResourceOwner` 会自动解除源 Prefab 引用。
- 组件图片、材质优先用绑定扩展，避免手动维护引用。
- 异步加载建议传 `CancellationToken`，对象销毁或界面关闭时取消。
- 多包资源必须显式传 `packageName` / `customPackageName`。
- `Decryption Services` 必须填可被框架程序集工具找到的完整类型名，并实现 `IDecryptionServices`。
- `InitPackageAsync` 成功后仍要 `LoadPackageManifestAsync`，再读 `GetPackageVersion()`。
- GameObjectPool、音频缓存内部已持有租约，业务继续走各自模块 API 即可。
