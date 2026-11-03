# Resources 模块


Resources 模块是框架对 YooAsset 的资源服务封装。它把「谁在用这份资源」显式化：每次加载产出**租约**（`AssetLease<T>`）或落进**作用域**（`ResourceScope`），引用归零后进入空闲缓存，到期再卸底层句柄。

底层 YooAsset 句柄只活在资源记录里，业务不要直接拿 `AssetHandle`。同一地址的同步/异步加载会合流到同一条记录。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Resource`

运行时主要入口：

- `GameApp.Resource` → `IResourceService`：加载、实例化、回收
- `GameApp.ResourcePackage` → `IResourcePackageService`：包初始化、下载器、版本、清单、缓存清理

## 本模块页面

| 页面 | 内容 |
| :--- | :--- |
| [加载与绑定](loading.md) | 租约加载、作用域批量加载、Prefab 实例化、组件资源绑定 |
| [资源包与下载](packages.md) | 包初始化、资源查询、多包、版本/清单/下载器、缓存清理 |
| [回收与调试](reclaim.md) | `UnloadUnusedAssets`、强制回收、调试面板、API 速查 |

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
