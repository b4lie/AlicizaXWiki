# 加载、租赁与绑定

这一页只讲四条拿到资源的路：单条租约、成组作用域、加载并实例化 Prefab、把资源绑到组件属性上。
资源包怎么初始化、补丁怎么下，看[资源包与更新](packages.md)；什么时候真正卸载，看[回收、统计与调试](reclaim.md)。

## 适用场景

- 读一份配置、字体、图标、材质球，用完马上放。
- 一个界面 / 一局战斗里的一批资源共享生命周期。
- 从 AssetBundle 里实例化 UI 节点或场景物体。
- 让 `Image` / `SpriteRenderer` / `Renderer` 的贴图、材质跟随物体销毁自动释放。

不适用：对象池内的 Prefab 源（池自己持有租约，见 [GameObjectPool](../gameobjectpool/index.md)）、纯 `Resources` 目录加载、跨进程资源。

## 使用前提

1. `ResourceComponent` 已注册服务，`GameApp.Resource.CanLoadResources` 为 true。
2. 默认包已 `InitPackageAsync` 成功，并 `LoadPackageManifestAsync` 成功——否则底层一定找不到资源。
3. 需要 `using AlicizaX.Resource.Runtime;`：`AssetLease`、`AssetRef`、`ResourceScope`、`ResourceBindingOptions`、绑定扩展方法都在这个命名空间。
4. 绑定到 UGUI 时再加 `using UnityEngine.UI;`。
5. 异步方法用 `Cysharp.Threading.Tasks` 的 `UniTask`，取消用 `System.Threading.CancellationToken`。

## 最短接入

1. 取服务，别自己 `new`：

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;

IResourceService resources = GameApp.Resource;
```

2. 异步加载单条资源，`using` 保证释放：

```csharp
using System.Threading;
using AlicizaX;
using AlicizaX.Resource.Runtime;
using Cysharp.Threading.Tasks;
using UnityEngine;

public static class ConfigReader
{
    public static async UniTask<string> ReadTextAsync(string location, CancellationToken cancellationToken)
    {
        using AssetLease<TextAsset> lease =
            await GameApp.Resource.LoadLeaseAsync<TextAsset>(location, cancellationToken);
        return lease.IsValid ? lease.Asset.text : null;
    }
}
```

3. 判断结果。`IsValid` 为 false 表示这次没拿到（未初始化、取消、加载失败、类型不匹配都是这个结果），`Asset` 为 null。
4. 结束使用就 `Dispose()`，或让 `using` 作用域结束；不释放的租约会把引用一直挂在资源记录上。
5. 一批资源同生共死时换成 `CreateScope`，见下一节。

## 核心用法

### 租约：LoadLease / LoadLeaseAsync

```csharp
AssetLease<T> LoadLease<T>(AssetRef<T> key);
AssetLease<T> LoadLease<T>(string location, string packageName = null);
UniTask<AssetLease<T>> LoadLeaseAsync<T>(AssetRef<T> key, CancellationToken cancellationToken = default);
UniTask<AssetLease<T>> LoadLeaseAsync<T>(string location, CancellationToken cancellationToken = default, string packageName = null);
```

- 同步版本会阻塞到加载完成；如果同一个资源正在异步加载，它会加入那次加载并等它完成，不会重复发起底层请求。
- 异步版本在命中 Active 或 Idle 记录时同步完成；未命中才需要等服务 tick 推进。
- 地址要反复用时先取 `AssetRef<T>`，省掉字符串查表：

```csharp
AssetRef<Sprite> iconKey = GameApp.Resource.Key<Sprite>("icon_start");
using AssetLease<Sprite> lease = GameApp.Resource.LoadLease(iconKey);
```

- 多包时把包名传进去；注意异步重载的 `packageName` 在 `cancellationToken` 之后：

```csharp
using AssetLease<Sprite> dlcIcon =
    await GameApp.Resource.LoadLeaseAsync<Sprite>("icon_start", cancellationToken, "DLC");
```

取消与失败的真实语义：

| 情况 | 返回 | 副作用 |
| :--- | :--- | :--- |
| token 已取消 | 空租约（`IsValid == false`） | 不发起任何底层加载 |
| 等待中被取消 | 空租约 | 只移除自己这个等待者，其他等待者照常完成 |
| 所有等待者都取消 | 空租约 | 立即释放底层句柄，记录回到 Released |
| 加载失败 / 类型不匹配 | 空租约 | `Log.Warning`：`Resource load failed. Location: …, Type: …, Error: …`，句柄已释放 |
| 服务未运行 / YooAssets 未初始化 | 空租约 | 不发起加载，不抛异常 |
| 加载抛异常 | 异常向上抛 | 记录状态被复原，可以重试 |

`Dispose()` 是幂等的；把租约复制一份后两个副本共享同一条引用，任意一个 `Dispose()` 都会让两个副本的 `IsValid` 一起变 false。释放之后再 `Dispose()` 不会误伤复用同一槽位的新租约。

### 作用域：CreateScope / App

```csharp
ResourceScope CreateScope(string debugName = null);
ResourceScope App { get; }

T Load<T>(string location, string packageName = null);
T Load<T>(AssetRef<T> key);
UniTask<T> LoadAsync<T>(string location, CancellationToken cancellationToken = default, string packageName = null);
UniTask<T> LoadAsync<T>(AssetRef<T> key, CancellationToken cancellationToken = default);
```

```csharp
using AlicizaX.Resource.Runtime;

using ResourceScope scope = GameApp.Resource.CreateScope("Shop");
Sprite banner = scope.Load<Sprite>("ShopBanner");
TextAsset table = await scope.LoadAsync<TextAsset>("tables_shop");
// 离开 using：scope 内所有条目一次性释放
```

- 作用域内按 (scope, asset) 去重：同一资源 `Load` 多次只占一条引用，不需要成对释放。
- 已 `Dispose` 的作用域再 `Load` 返回 `null`，不抛异常；槽位被复用时新旧作用域靠代际号区分，旧句柄不会影响新作用域。
- 作用域加载失败或作用域在加载中途被销毁，`LoadAsync` 返回 `null`。
- `GameApp.Resource.App` 是进程级作用域，服务 `Shutdown` 时才失效；`ForceUnloadAllAssets` 只清空条目，作用域本身仍可用。
- 作用域记住创建它的服务实例，加载与绑定都走那个服务；不要把它传给另一个 `ResourceService` 的入口。
- 框架已经替你建好了两个：`UIBase.Res`（视图销毁时 `Dispose`）和 `ProcedureBase.Res`（流程 `Leave` 时 `Dispose`）。业务里别重复建，直接用 `Res` 即可：

```csharp
baseui.ImgItemIcon.SetSprite(Res, _goodsData.Icon); // UIBuyAlertWindow 的真实写法
```

### 加载并实例化 Prefab

```csharp
GameObject Instantiate(string location, Transform parent = null, string packageName = null);
UniTask<GameObject> InstantiateAsync(string location, Transform parent = null,
    CancellationToken cancellationToken = default, string packageName = null);
```

```csharp
GameObject window = GameApp.Resource.Instantiate("UIBuyAlertWindow", transform);
GameObject shop = await GameApp.Resource.InstantiateAsync("UIShopWindow", transform, cancellationToken);
```

- 内部先按 `GameObject` 拿资源，再 `Object.Instantiate`，然后给实例加 `ResourceOwner` 并把资源记进它的作用域。所以 **释放方式是 `Destroy(instance)`**，不是手动找租约。
- 同一个 Prefab 实例化多次就有多条引用，逐个 `Destroy` 逐个归还。
- 失败一律返回 `null`：服务未运行、取消、加载失败、`parent` 在等待期间被销毁、等待期间发生了 `ForceUnloadAllAssets`。
- 对象池不要走这两个 API：`GameObjectPool` 内部用 `LoadLease<GameObject>` 管 Prefab 源，`Instantiate` 会给实例挂 `ResourceOwner`，与池的生命周期冲突。
- UI 预制体也不用自己调：`UIHolderFactory` 按 `UIResRegistry` 登记的 location 走 `Instantiate` / `InstantiateAsync`（见 [UI](../ui/index.md)）。

### 组件资源绑定

绑定扩展把「资源 → 组件属性」的引用交给目标物体的 `ResourceOwner` 管理：目标销毁、重新绑定或显式清空时自动释放。

```csharp
image.SetSprite("icon_start", ResourceBindingOptions.SetNativeSize);
image.SetSubSprite("CommonAtlas", "btn_start");
image.SetMaterial("ui_mask");

spriteRenderer.SetSprite("player");

renderer.SetSharedMaterial("shared");
renderer.SetMaterialInstance("role", ResourceBindingOptions.Synchronous);
```

两种形态：

- **显式作用域**：`SetSprite(in ResourceScope scope, …)`，引用记进给定作用域。不会给目标加 `ResourceOwner`，适合 `UIBase.Res`、`ProcedureBase.Res` 这种已有生命周期的地方。
- **隐式归属**：不传 `scope`，框架取出目标所在物体的作用域（没有就自动加一个 `ResourceOwner`），引用随物体销毁释放。适合凭 Item 自己生存的控件。

| API | 写入的属性 | 说明 |
| :--- | :--- | :--- |
| `Image.SetSprite` | `Image.sprite` | 支持 `SetNativeSize` |
| `Image.SetSubSprite` | `Image.sprite` | 从图集子资源按名字取 Sprite |
| `Image.SetMaterial` | `Image.material` | 材质按 `Material` 类型加载 |
| `SpriteRenderer.SetSprite` | `SpriteRenderer.sprite` | — |
| `SpriteRenderer.SetSubSprite` | `SpriteRenderer.sprite` | 子资源名不存在时打警告并保留旧贴图 |
| `Renderer.SetSharedMaterial` | `Renderer.sharedMaterial` | 共享材质，不加实例 |
| `Renderer.SetMaterialInstance` | `Renderer.sharedMaterial` | `Instantiate` 一份运行时材质，替换/清空时销毁旧实例 |

`ResourceBindingOptions`（`[Flags]`，可组合）：

| 值 | 作用 |
| :--- | :--- |
| `None` | 默认，未命中时异步加载，加载完成后写入属性 |
| `Synchronous` | 未命中时同步等待加载，返回时属性已写好 |
| `SetNativeSize` | 绑定成功后对 `Image` 调 `SetNativeSize()` |

行为细节：

- 传 `""` 或 `null` 作为 location 只清它自己写过的属性；之前没绑定过就什么也不做。`Image.sprite` 和 `Image.material` 是两个独立槽位，清一个不影响另一个。
- 同一目标同一槽位再次绑定会替换并释放旧引用；重复绑同一个资源只保留一条引用。
- `SetSharedMaterial` 与 `SetMaterialInstance` 共用同一个槽位：先实例化再换共享材质，会销毁那份运行时实例。
- 子资源（`SetSubSprite`）走 `LoadSubAssets*<Sprite>`，是独立于普通 `Sprite` 记录的一条资源记录，按名字取子资源。
- 异步绑定未命中时属性保持原样；如果期间目标被销毁、槽位被重新绑定或作用域被 `Dispose`，这次加载结果会被丢弃，不会写回。
- 作用域 `Dispose` 会清空它名下所有绑定属性；把同一个绑定改到另一个作用域等于转移归属。
- 隐式重载的字符串版本只走**默认包**；多包资源必须用带 `AssetRef<T>` 的重载。

### 框架已经替你加载的地方

| 场景 | 入口 | 内部用的加载方式 |
| :--- | :--- | :--- |
| UI 窗口 / 控件 | `[UIRes]` + `UIHolderFactory` | `Instantiate` / `InstantiateAsync` |
| 对象池 Prefab | `GameObjectPoolService` → `IPrefabLoader` | `LoadLease<GameObject>`，池内计数后 `Dispose` |
| 音频 Clip | `AudioService` | `LoadLease<AudioClip>` / `LoadLeaseAsync<AudioClip>` |
| 配置表 | `IConfigBytesLoader` 实现 | `LoadLeaseAsync<TextAsset>` 后取 `bytes` |
| 启动流程 | `ProcedureBase.Res` | `CreateScope(流程类型名)`，流程 `Leave` 时释放 |

这些入口已经管好了引用，业务侧不要重复加载同一份资源再自己 `Dispose`，否则会出现「明明池里还在用，引用却被放掉」的错配。

## 常见配置

| 需求 | 用什么 | 释放方式 | 常见误用 |
| :--- | :--- | :--- | :--- |
| 临时读一份表 | `LoadLeaseAsync<T>` + `using` | `Dispose()` | 把 `lease.Asset` 存成字段后租约被释放 |
| 界面内一批资源 | `CreateScope` 或 `UIBase.Res` | `scope.Dispose()` / 视图销毁 | 手动逐个 `Dispose` 漏掉几个 |
| 一组道具图标 | 隐式 `SetSprite` | 物体销毁自动释放 | 又拿租约又绑定，引用记两份 |
| 场景物体 | `Instantiate` / `InstantiateAsync` | `Destroy(instance)` | 用对象池 API 管实例化结果 |
| 需要立即写上属性 | `ResourceBindingOptions.Synchronous` | 同上 | 在主线程大批量同步加载造成卡顿 |
| 多包资源 | 带 `packageName` 或 `AssetRef` 的重载 | 同上 | 隐式字符串重载只认默认包 |

## 容易踩的坑

1. **`using` 里 `await` 没问题，`using` 外再用 `Asset` 就有问题**。真正需要长期持有就保留租约字段，并在确定不需要时 `Dispose`。
2. **同步 `LoadLease` 是阻塞的**。它内部会 `WaitForAsyncComplete`，在加载大资源时会把当帧卡住；只在确定资源已在内存（或必须同步拿到）的场景用。
3. **绑定不会替你等异步结果**。默认 `None` 下属性可能在下一帧之后才出现，别在绑定后立刻读 `image.sprite` 做判断，需要确定值就传 `Synchronous`。
4. **同一物体只认一个作用域**。`ResourceOwner` 只存一个 `ResourceScope`；同一物体被另一个服务或另一处绑定接管后，旧作用域可能变成孤儿，只能等扫描回收或手动释放。
5. **子资源不是普通 Sprite**。`SetSubSprite` 必须给图集地址 + 子资源名，名字对不上只会警告，不会回退到整张图。
6. **`InstantiateAsync` 的 `parent` 要先活到加载完成**。父物体中途被销毁时返回 `null`，实例不会创建。
