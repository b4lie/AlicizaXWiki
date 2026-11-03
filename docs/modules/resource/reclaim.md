# 资源回收

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
