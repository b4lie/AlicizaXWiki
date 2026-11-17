# Scene 场景

`ISceneService` 封装 YooAsset 的 `SceneHandle`：主场景（Single）、附加场景（Additive）、挂起与放行、激活切换、卸载。它只管场景本身，场景里的资源仍然走 `IResourceService`。

源码：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Scene`（`ISceneService`、`SceneService`、`SceneComponent`）
- 挂载点：`Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab` 的 `Scene` 节点

## 适用场景

- 启动完成后从资源包场景进入主玩法。
- 分块加载附加场景：光照、区块、副本、临时战斗场景。
- 想先把场景读完再决定何时切屏（`suspend` + `AllowActivation`）。
- 查询某个场景是否由本服务加载、切换激活场景。

不适用：首场景启动（那是 Build Settings 与 `Assets/Scenes/Main.unity` 的事）、场景内资源加载（用 [Resources](../resource/index.md)）、Unity 原生多场景管理细节。

## 使用前提

1. `ResourceComponent` 必须先于 `SceneComponent` 完成注册：`ResourceComponent` 是 `DefaultExecutionOrder(-700)`，`SceneComponent` 是 `-600`，顺序由特性保证。`SceneService.OnInitialize` 会 `Require<IResourcePackageService>()` 与 `Require<IResourceService>()`。
2. 默认包必须已 `InitPackageAsync` 成功并 `LoadPackageManifestAsync` 成功。场景固定从默认包加载，没有传包名的入口。
3. location 必须是 Collector 收集过的场景。本工程 Collector 的 `Scenes` 组指向 `Assets/Bundles/Scenes`，地址规则是 `AddressByFileName`（场景文件名）；该目录当前为空，直接把 `Assets/Scenes/Main.unity` 传给 `LoadMainAsync` 是找不到的。
4. 地址不能为空：四个加载 / 卸载入口都会 `GameFrameworkGuard.NotNullOrEmpty`，`null` 或空串**同步**抛 `ArgumentNullException`（不用 await 就能捕获）。
5. 不要在场景加载时销毁 `Entry` 或 `SceneComponent`：服务销毁会取消在途操作并卸载已完成的附加场景。

## 最短接入

1. 取服务：

```csharp
using AlicizaX;
using AlicizaX.Scene.Runtime;

ISceneService scenes = GameApp.Scene; // 等价于 AppServices.Require<ISceneService>()
```

2. 等默认包与清单就绪（见[资源包与更新](../resource/packages.md)），否则加载会失败。
3. 加载主场景并接进度：

```csharp
using AlicizaX;
using AlicizaX.Scene.Runtime;
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.SceneManagement;

public static class SceneFlow
{
    public static async UniTask<Scene> EnterGameAsync(IProgress<float> progress)
    {
        return await GameApp.Scene.LoadMainAsync("Game", progress);
    }
}
```

4. 需要附加场景时成对加载 / 卸载：

```csharp
Scene battle = await GameApp.Scene.LoadAdditiveAsync("BattleAdditive");
// ...
await GameApp.Scene.UnloadAdditiveAsync("BattleAdditive");
```

5. 需要「加载完先不切」就挂起，随后放行：

```csharp
UniTask<Scene> loading = GameApp.Scene.LoadAdditiveAsync("BattleAdditive", suspend: true);
// 这里可以播过场、挡 UI……
GameApp.Scene.AllowActivation("BattleAdditive");
Scene scene = await loading;
```

## 核心用法

### 主场景

```csharp
UniTask<Scene> LoadMainAsync(string location, IProgress<float> progress = null,
    bool suspend = false, CancellationToken cancellationToken = default);
UniTask<Scene> ReloadMainAsync(string location, IProgress<float> progress = null,
    bool suspend = false, CancellationToken cancellationToken = default);

Scene MainScene { get; }
```

- `LoadMainAsync` 走 `LoadSceneMode.Single`：会关闭其他所有场景，包括已经加载的附加场景。提交成功后服务清空自己的场景表，只保留这个主场景。
- 该 location 已经是当前主场景且已加载完成时，直接返回现有 `Scene`，不重复加载；需要重建同一个主场景用 `ReloadMainAsync`。
- `MainScene` 在主场景尚未由本服务加载、或句柄已失效（被外部卸载）时回退到 `SceneManager.GetActiveScene()`。
- 主场景提交后框架会调用 `ForceUnloadUnusedAssets(false)`，让各包卸载未使用的 bundle；它不会清 Idle，也不碰你的租约。
- 另一个主场景切换在途、该 location 正在加载 / 挂起 / 卸载、加载失败，都会抛 `GameFrameworkException`。

### 附加场景

```csharp
UniTask<Scene> LoadAdditiveAsync(string location, IProgress<float> progress = null,
    bool suspend = false, CancellationToken cancellationToken = default);
UniTask UnloadAdditiveAsync(string location, CancellationToken cancellationToken = default);
```

- `LoadAdditiveAsync` 已是加载完成的附加场景时直接返回现有 `Scene`；加载中或挂起中的重复请求抛异常。
- `UnloadAdditiveAsync` 只接受「已加载完成的附加场景」：
  - 传主场景 → 抛 `GameFrameworkException`（主场景只能靠下一次 Single 替换）。
  - 正在卸载 → 抛异常，不会排队。
  - 没加载过或还没加载完 → 抛异常。
- 卸载开始后 `Contains` 仍为 true，直到操作完成才移除记录；这期间对同一 location 的加载 / 卸载都会被拒绝。
- 取消**等待**不会撤销卸载：内部用 `AttachExternalCancellation`，`await` 抛 `OperationCanceledException`，但场景仍会被卸载完。
- 取消附加场景的**加载**会回滚：句柄被释放，场景不会残留。

### 挂起与放行

- `suspend: true` 时框架用 `allowSceneActivation: false` 发起加载（优先级固定 100），加载停在激活前，`await` 不会完成。
- `AllowActivation(location)` 放行一个挂起中的场景：挂起的主场景，或追踪中尚未完成的附加场景；不存在、已完成或已失效返回 false。
- 挂起的主场景**不会**替换已提交的 `MainScene`，`SetActive` 也只作用于已提交实例。
- 挂起期间新的主场景加载和附加场景加载都会被拒绝（`GameFrameworkException`），直到放行完成或服务销毁。
- 取消挂起的主场景加载只是停止等待：场景仍停在激活前，`_pendingMainHandle` 仍被占用，后续仍要 `AllowActivation` 才能继续；放行后它照样会成为主场景。

### 状态查询

| API | 返回 | 说明 |
| :--- | :--- | :--- |
| `MainScene` | `Scene` | 已提交主场景；没有则回退当前激活场景 |
| `Contains(location)` | `bool` | 挂起中的主场景、加载中 / 挂起 / 卸载中的场景都算；已被外部卸载的不算 |
| `SetActive(location)` | `bool` | 对已加载完成的追踪场景调 `SceneManager.SetActiveScene`；挂起 / 未完成返回 false |
| `AllowActivation(location)` | `bool` | 放行挂起场景 |

`SetActive` 不是「激活挂起加载」——它只切换激活场景。挂起场景要先用 `AllowActivation` 放行。

### 与 Resources 的关系

- 场景从默认包加载，无法指定其他包；多包资源不要指望 `LoadMainAsync` 去别的包找。
- 场景文件本身不走 `IResourceService`，不要用 `LoadLeaseAsync<SceneAsset>` 或 `LoadLeaseAsync<GameObject>` 去加载 `.unity`。
- 主场景提交后只做一次 `ForceUnloadUnusedAssets(false)`；如果你想连 Idle 一起清，参考[回收、统计与调试](../resource/reclaim.md)自己调用。
- 场景里的资源该用租约 / 作用域照旧。切主场景时旧场景物体会被销毁，挂在它们身上的 `ResourceOwner` 会随之释放绑定与实例引用；但你自己的租约、静态字段里的资源引用不会被自动回收。
- 场景加载期间如果界面还在用某批资源，别在同一时间做 `ForceUnloadAllAssets()`：那会取消所有等待者，包括正在加载的场景依赖。

### 进度语义

`progress` 是 `IProgress<float>`，由 YooAsset 句柄的 `Progress` 每帧上报：

- 起始强制上报 `0`。
- 资源包阶段推进到 `0.5`，场景阶段推进到 `0.9` 后停住（YooAsset 不上报 0.5 以下的场景阶段进度），所以进度条是台阶状而不是连续曲线。
- 激活完成后上报 `1`；重复加载已就绪场景时只上报一次 `1`。

### 失败与析构

| 情况 | 表现 |
| :--- | :--- |
| location 为 `null` / 空串 | `ArgumentNullException`，同步抛出 |
| 主场景切换在途、location 正在加载或卸载 | `GameFrameworkException`，`await` 时抛出 |
| 加载失败（句柄状态失败或 `isLoaded == false`） | `GameFrameworkException`，消息含 location 与 `handle.Error` |
| 卸载的不是已加载附加场景 / 是主场景 / 正在卸载 | `GameFrameworkException`，同步抛出 |
| 取消 | `OperationCanceledException` |
| 服务已销毁后调用 | `ObjectDisposedException` |

服务销毁（`SceneComponent.OnDestroy` / `AppServices.Shutdown`）时：

- 取消内部操作令牌，清空挂起状态与场景表。
- 已加载完成的附加场景会被卸载；在途的加载被取消并自行回滚，不留下半加载场景。
- 主场景**保留**——它是当前世界，比服务活得久。
- 之后 `Contains` 全部为 false，`MainScene` 回退 `SceneManager.GetActiveScene()`；再调用加载入口抛 `ObjectDisposedException`。

## 常见配置

| 项 | 取值 | 说明 |
| :--- | :--- | :--- |
| `location` | YooAsset 地址 | Collector 已收集的场景，`AddressByFileName` 下就是场景文件名 |
| `progress` | `IProgress<float>` 或 `null` | 传 `null` 不上报；台阶式进度见上 |
| `suspend` | `false` / `true` | `true` 时停在激活前，等 `AllowActivation` |
| `cancellationToken` | 默认 / 自定义 | 主场景只停止等待，附加场景加载会回滚 |
| 场景包 | 固定默认包 | 由 `DefaultPackageName` 决定，入口不可覆盖 |
| 加载优先级 | 固定 100 | 框架内部常量，不对外暴露 |

## 容易踩的坑

1. **`SetActive` 不能放行挂起场景**。放行只有 `AllowActivation`；`SetActive` 是 `SceneManager.SetActiveScene`，对未完成的场景返回 false。
2. **主场景是 Single，会连带关掉附加场景**。切主场景后再去 `UnloadAdditiveAsync` 旧附加场景会抛「不是已加载的附加场景」，因为它已经被关掉了。
3. **挂起会阻塞所有后续场景加载**。放行前新的 `LoadMainAsync` / `LoadAdditiveAsync` 一律抛异常；挂起不是「后台慢慢加载」。
4. **取消主场景加载不等于没加载**。场景仍会提交成为主场景（挂起的除外，它会停在激活前等你放行）。
5. **附加场景取消加载会回滚，取消卸载不会**。卸载一旦开始就由 Unity 执行，不可撤销。
6. **`Assets/Scenes` 下的场景不能直接加载**。它们走 Build Settings 启动，不在 YooAsset 清单里；要 `LoadMainAsync` 先把场景放进 Collector 收集的目录。

## API 速查

`ISceneService`（`GameApp.Scene`）：

```csharp
Scene MainScene { get; }

UniTask<Scene> LoadMainAsync(string location, IProgress<float> progress = null, bool suspend = false,
    CancellationToken cancellationToken = default);
UniTask<Scene> ReloadMainAsync(string location, IProgress<float> progress = null, bool suspend = false,
    CancellationToken cancellationToken = default);
UniTask<Scene> LoadAdditiveAsync(string location, IProgress<float> progress = null, bool suspend = false,
    CancellationToken cancellationToken = default);
UniTask UnloadAdditiveAsync(string location, CancellationToken cancellationToken = default);

bool AllowActivation(string location);
bool SetActive(string location);
bool Contains(string location);
```

`SceneComponent`：

```csharp
// 无序列化字段，只在 Awake 里注册 ISceneService、OnDestroy 里注销
[AddComponentMenu("Game Framework/Scene")]
[DefaultExecutionOrder(-600)]
public sealed class SceneComponent : MonoBehaviour
```
