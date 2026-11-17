# Resources 资源

Resources 是 YooAsset 之上的资源服务层：按 location 拿资源，用 `AssetLease<T>` / `ResourceScope` 明确「谁还在用」，再把没人用的记录放进 Idle 缓存，到期才真正释放底层句柄。
它只负责加载、所有权与包级运维，不负责编辑器资源收集规则、CDN 部署和业务缓存策略。

源码：

- 运行时：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Resource`
- 编辑器：`Client/Packages/com.alicizax.unity.framework/Editor/Modules/Resource`
- 挂载点：`Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab` 的 `Resources` 节点

## 适用场景

- 读配置表、字体、图标、材质、音频 Clip 这类「一份资源」的加载。
- 一次界面、一局战斗、一个流程内成组加载、成组释放。
- 把 Sprite / Material 绑到 `Image`、`SpriteRenderer`、`Renderer`，并跟随物体生命周期释放。
- 加载并实例化 UI 预制体、道具预制体。
- 资源包初始化、版本与清单、补丁下载（见[资源包与更新](packages.md)）。

不适合：资源收集规则、CDN 上传、业务数据缓存、对象池。对象池有自己的 Prefab 租约策略，见 [GameObjectPool](../gameobjectpool/index.md)。

## 本模块页面

| 页面 | 内容 |
| :--- | :--- |
| [加载、租赁与绑定](loading.md) | 租约、作用域、Prefab 实例化、组件资源绑定、失败返回值 |
| [资源包与更新](packages.md) | 播放模式、包初始化、版本与清单、下载器、多包、缓存清理、打包入口 |
| [回收、统计与调试](reclaim.md) | 引用计数与回收时机、手动回收、Inspector 诊断面板、泄漏排查 |
| [Scene 场景](../scene/index.md) | 主场景 / 附加场景切换，与资源包的关系 |

## 使用前提

1. 启动场景里有 `RootModule`（服务容器，`DefaultExecutionOrder(-32000)`）。没有它 `AppServices` 还没创建，`GameApp.Resource` 会抛异常。
2. `Entry.prefab` 的 `Resources` 节点上挂 `ResourceComponent`。它在 `Awake` 里 `new ResourceService()`，并以 `IResourceService` + `IResourcePackageService` 两个契约注册到 `AppServices`。
3. `ResourceComponent.Awake` 已经完成 `YooAssets.Initialize(new ResourceLogger())`、`YooAssets.SetAsyncOperationMaxTimeSlice(milliseconds)` 和默认包创建。**不要**在业务里再调一次 `YooAssets.Initialize`。
4. 资源地址必须是 Collector 生成的 location。本工程 Collector 用 `AddressByFileName`，即**不带扩展名的文件名**：`Assets/Bundles/Configs/bytes/localization/Game_ChineseSimplified.bytes` 的 location 是 `Game_ChineseSimplified`。
5. 包必须先 `InitPackageAsync` 成功、再 `LoadPackageManifestAsync` 成功，才能读到任何资源（见 [资源包与更新](packages.md)）。

## 最短接入

1. 取服务。`GameApp.Resource` 只是 `AppServices.Require<IResourceService>()` 的缓存入口：

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;

IResourceService resources = GameApp.Resource;
```

2. 确认可以加载。`CanLoadResources` 为 true 表示服务在跑、YooAssets 已初始化：

```csharp
if (!GameApp.Resource.CanLoadResources)
{
    return; // 包还没初始化，或正在 ForceUnloadAllAssets 的重置过程中
}
```

3. 加载时用 `AssetLease<T>` 明确所有权，`using` 包住作用域：

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;
using Cysharp.Threading.Tasks;

public static async UniTask<byte[]> ReadTableAsync(string location)
{
    using AssetLease<TextAsset> lease = await GameApp.Resource.LoadLeaseAsync<TextAsset>(location);
    if (!lease.IsValid)
    {
        throw new System.InvalidOperationException($"Missing config asset: {location}");
    }

    return lease.Asset.bytes;
}
```

4. 一批资源同生共死时改用一个作用域，退出 `using` 一次性释放：

```csharp
using AlicizaX.Resource.Runtime;

using ResourceScope scope = GameApp.Resource.CreateScope("Battle");
var atlas = scope.Load<Texture2D>("battle_atlas");
var config = await scope.LoadAsync<TextAsset>("battle_config");
```

5. 要世界物体就用 `Instantiate` / `InstantiateAsync`：实例上会自动挂 `ResourceOwner`，`Destroy(instance)` 即释放源 Prefab 引用。

## 核心用法

### 能力边界

| 你要做的事 | 这一层提供 | 不要指望它 |
| :--- | :--- | :--- |
| 按 location 取资源 | `LoadLease` / `LoadLeaseAsync` / `ResourceScope.Load*` | 拼路径、猜后缀（地址由 Collector 决定） |
| 复用与释放 | `AssetLease.Dispose()`、`ResourceScope.Dispose()` | 立即把对象从内存抠掉（先进 Idle） |
| 世界对象 | `Instantiate` / `InstantiateAsync` | 对象池（用 `GameObjectPool`） |
| 组件贴图/材质 | `SetSprite` / `SetSubSprite` / `SetMaterial` / `SetSharedMaterial` / `SetMaterialInstance` | 手动维护引用计数 |
| 包与下载 | `IResourcePackageService`（见 [资源包与更新](packages.md)） | 断点续传 UI、灰度、CDN 上传 |
| 回收策略 | Idle TTL + `UnloadUnusedAssets` | 按业务优先级淘汰 |

### 所有权模型：Owner / Scope / Lease

YooAsset 不知道业务还在不在用一份资源，所以框架把「谁在用」显式记在资源记录上：

- `AssetLease<T>`：一条引用的凭据，`readonly struct`，只由加载接口产出；`Dispose()` 交还这条引用。
- `ResourceScope`：一组引用的容器，同样 `readonly struct`；`Dispose()` 一次性释放容器内所有条目。
- `ResourceOwner`：挂在 `GameObject` 上的 `MonoBehaviour`（`[DisallowMultipleComponent]`），持有一个 `ResourceScope`，在 `OnDestroy` 里 `Scope.Dispose()`。Prefab 实例与组件绑定目标都由它兜底释放。

| 谁在用 | 类型 | 什么时候释放 | 典型场景 |
| :--- | :--- | :--- | :--- |
| 单条资源 | `AssetLease<T>` | `Dispose()` / `using` 结束 | 临时读表、音频 Clip |
| 一组资源 | `ResourceScope` | `Dispose()` | 界面、战斗、启动流程 |
| Prefab 实例 | `ResourceOwner`（`Instantiate` 自动挂） | 实例被 `Destroy` | 场景对象、UI 节点 |
| 组件属性 | `ResourceOwner`（绑定扩展自动挂） | 目标销毁、重新绑定或清空 | `Image`、`Renderer` |
| 全局常驻 | `GameApp.Resource.App` | 服务 `Shutdown` | 极少数跨场景资源 |

引用计数按记录算，租约、作用域条目、绑定各记一份。作用域内部还会按 (scope, asset) 去重：同一个作用域里重复 `Load` 同一资源只算一条引用，不需要成对释放。作用域来源分三类——手动创建、Prefab 实例、绑定目标；Inspector 的 Scope 卡片会按来源标注。

### 资源记录状态

| 状态 | 含义 | 出口 |
| :--- | :--- | :--- |
| Loading | 底层句柄已发起，尚未完成 | 完成后进 Active 或 Idle |
| Active | 至少还有一条引用 | 引用归零进 Idle |
| Idle | 无引用，句柄仍在，等待过期 | 重新加载会命中 Idle；过期或被回收则 Released |
| Released | 底层句柄已释放 | 下次加载重新走 YooAsset |

Idle 的 TTL 由 `idleAssetExpireTime` 控制，用 `Time.unscaledTime` 计时；每帧最多处理 `expireProcessCountPerFrame` 条。完整时机见 [回收、统计与调试](reclaim.md)。

### 初始化顺序

`ResourceComponent` 必须在 Scene / UI / Audio 之前完成注册，因为这些模块在各自的 `OnInitialize` 或首次使用时就会 `Require<IResourceService>()`：

| 顺序 | 组件 | `DefaultExecutionOrder` | 与资源的关系 |
| :--- | :--- | :--- | :--- |
| 0 | `RootModule` / `AppServiceRoot` | -32000 | 创建服务容器，提供 `Update` 驱动服务 tick |
| 1 | `MemoryPool` 设置 | -1000 | 无直接依赖 |
| 2 | `TimerComponent` | -800 | 无直接依赖 |
| 3 | `ResourceComponent` | -700 | 初始化 YooAssets，注册两个资源契约 |
| 4 | `SceneComponent` | -600 | `OnInitialize` 里 `Require` 资源两个契约 |
| 5 | `UIComponent` | -500 | `UIBase.Res`、`UIHolderFactory` 走 `Instantiate` |
| 6 | `AudioComponent` | -400 | `AudioService` 用 `LoadLease<AudioClip>` |
| 7 | 热更 `ConfigService` | 运行时注册 | `IConfigBytesLoader` 用 `LoadLeaseAsync<TextAsset>` |

顺序由 `DefaultExecutionOrder` 固定，改动挂载关系前先确认这条链没断。另外，异步加载的完成回调由服务 tick 推进（`IServiceTickable`，经 `RootModule.Update` 转发），根节点失活或销毁后 `LoadLeaseAsync` 不会再完成。

### ResourceComponent 关键字段

自定义 Inspector 只绑定下面这些序列化字段，改字段名要同步改 `ResourceComponentInspector` 与 `ResourceComponentBuildProcessor`（后者按 `_playMode` / `decryptionServices` 两个字段名查找）：

| Inspector | 序列化字段 | 默认值 | 作用 |
| :--- | :--- | :--- | :--- |
| Play Mode | `_playMode` | `EditorSimulateMode` | YooAsset 播放模式，取值见 [资源包与更新](packages.md) |
| Decryption Service | `decryptionServices` | 空 | `IBundleDecryptor` 实现类型全名，留空表示不加密 |
| Package Name | `packageName` | `DefaultPackage` | 默认包名，同时作为 `IResourcePackageService.DefaultPackageName` |
| Auto Unload Bundle | `autoUnloadBundleWhenUnused` | `false` | 传给 YooAsset 初始化选项，引用归零即卸 bundle |
| Milliseconds | `milliseconds` | `30` | `YooAssets.SetAsyncOperationMaxTimeSlice` 的时间片 |
| Max Downloading | `downloadingMaxNum` | `10` | 下载器并发上限（Inspector 滑杆 1–48） |
| Max Retry Count | `failedTryAgain` | `3` | 单个文件失败重试次数（滑杆 1–48） |
| Use System Unload | `useSystemUnloadUnusedAssets` | `true` | 强制回收后是否调 `Resources.UnloadUnusedAssets()` |
| Max Unload Interval | `maxUnloadUnusedAssetsInterval` | `300` | 自动 `UnloadUnusedAssets(false)` 的周期（秒） |
| Expire Per Frame | `expireProcessCountPerFrame` | `16` | 每帧最多释放多少条过期 Idle |
| Idle Asset Capacity | `idleAssetCapacity` | `256` | Idle 记录数量上限 |
| Idle Expire Time | `idleAssetExpireTime` | `60` | Idle 记录过期秒数 |
| （未在自定义 Inspector 暴露） | `minGCCollectInterval` | `30` | `GC.Collect()` 的最小间隔 |

编辑器下 `Play Mode` 优先读 `EditorPrefs`（键 `ResourceComponent.PrefsKey`，按 `Application.dataPath` 哈希区分工程），并在 `Awake` 里覆盖序列化值；Player 下没有 `EditorPrefs`，用序列化值，出包时由构建处理器写入。`Entry.prefab` 里存的是 `None`，非 BuildCLI 出包只会打印警告，运行到 `InitPackageAsync` 会直接返回 false。

## 常见配置

| 目标 | 关键字段 | 说明 |
| :--- | :--- | :--- |
| 编辑器里直接跑，不打包 | `_playMode = EditorSimulateMode` | 运行时通过反射调用 YooAsset 编辑器模拟构建，仅编辑器可用 |
| 单机包 | `_playMode = OfflinePlayMode` | 只读 Builtin 文件系统，不需要 CDN |
| 联机包 | `_playMode = HostPlayMode` | 必须有 host URL，缓存用 Sandbox 文件系统 |
| WebGL | `_playMode = WebPlayMode` | 同样需要 host URL，走 WebServer 文件系统 |
| 大包缓解卡顿 | `milliseconds` 调小 | 每帧异步操作时间片变小，加载更平滑但更慢 |
| 补丁下载慢 | `downloadingMaxNum` 调大 | 并发上限受下载器内部 32 限制 |
| 内存吃紧 | `idleAssetExpireTime` / `idleAssetCapacity` 调小 | 无引用资源更早卸句柄 |
| 加密包 | `decryptionServices` + 构建侧加密 | 两边必须成对，见 [资源包与更新](packages.md) |

## 容易踩的坑

1. **`Asset` 字段不是所有权**。`lease.Asset` 只是当前拿到的对象引用；`Dispose()` 之后 `IsValid` 变 false，后续是否还能用取决于 Idle TTL，不要把它存进静态字段当常驻资源。
2. **不要自己创建 `ResourceService`**。服务由 `ResourceComponent` 注册，重复 `new` 会绕过包初始化，`AppServices` 里也查不到。
3. **`Destroy(Entry)` 等于停服**。`ResourceComponent.OnDestroy` 会 `AppServices.Unregister` 触发 `Shutdown`，所有租约与作用域失效。
4. **location 不是路径**。写成 `Assets/Bundles/...` 或带 `.prefab` 后缀都找不到；以 Collector 的地址规则为准。
5. **包没初始化时先别加载**。`LoadLease*` 会返回空租约并静默失败，`Instantiate` 返回 `null`，看起来像「资源丢了」。
6. **异步加载要靠服务 tick**。把框架根节点禁用或销毁，挂起的 `LoadLeaseAsync` 永远不完成。
