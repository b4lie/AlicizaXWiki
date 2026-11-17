# Service 服务容器

Service 是框架唯一的全局服务容器：`AppServiceRoot` 在 `Awake` 里创建 `ServiceRegistry`，模块组件把自己的服务注册进去，业务代码通过静态门面 `AppServices` 注册、解析、注销，容器再按注册顺序驱动 tick。它只解决"进程级单实例 + 生命周期回调 + 每帧驱动"三件事，没有作用域层级，也没有优先级或排序字段。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Core/Service`
- `Client/Packages/com.alicizax.unity.framework/Runtime/Core/Module/RootModule.cs`
- `Client/Packages/com.alicizax.unity.framework/Runtime/Core/App/GameApp.cs`
- `Client/Packages/com.alicizax.unity.framework/Runtime/Common/Utility/Utility.Unity.cs`
- `Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab`

## 适用场景

在任意脚本按契约拿到同一实例的全局能力（资源、音频、UI、计时器、配置等）；需要每帧/每物理帧更新但不想挂 `MonoBehaviour`；需要在退出时按反向注册顺序统一释放。不适合一份代码要跑多套实例的玩法，也不支持同一契约多实现按优先级解析——容器是进程级的，注册顺序就是唯一顺序。

## 使用前提

启动场景必须存在一个 `AppServiceRoot` 实例。模板直接使用 `Entry.prefab`（`Client/Packages/.../Runtime/Prefabs/Entry.prefab`），由 `Client/Assets/Scenes/Main.unity` 以预制体实例放到根级。

`Entry` 根节点挂 `RootModule`（`AppServiceRoot` 唯一的生产实现，`sealed`），7 个子节点各挂一个模块组件。层级顺序是 `Memory Pool`/`Resources`/`Audio`/`UI`/`Scene`/`Timer`/`GameObjectPool`，实际 `Awake` 顺序由 `[DefaultExecutionOrder]` 决定：

| 节点 | 组件 | 提供的契约 | `DefaultExecutionOrder` |
| --- | --- | --- | --- |
| Entry | `RootModule` | 无（只创建容器） | -32000（定义在 `AppServiceRoot`） |
| Memory Pool | `MemoryPoolSetting` | 无（配置 `MemoryPool` 预算） | -1000 |
| Timer | `TimerComponent` | `ITimerService` | -800 |
| Resources | `ResourceComponent` | `IResourceService` / `IResourcePackageService` | -700 |
| Scene | `SceneComponent` | `ISceneService` | -600 |
| UI | `UIComponent` | `IUIService`（构造时 `Require<ITimerService>()`） | -500 |
| Audio | `AudioComponent` | `IAudioService` | -400 |
| GameObjectPool | `GameObjectPoolComponent` | `IGameObjectPoolService` / `IGameObjectPoolDebugService` | 0（未标注） |

执行顺序就是这些数值决定的：容器先于一切创建，Timer 早于 UI，所以 `UIComponent.Awake` 里的 `Require<ITimerService>()` 才能成功。

`RootModule.Awake()` 内部顺序：

1. `base.Awake()`：重复根节点检查（打印 `Debug.LogWarning` 并 `enabled = false`）→ 记录静态实例 → 按 `_dontDestroyOnLoad` 调 `DontDestroyOnLoad` → `AppServices.GetOrCreateRegistry()`。
2. `_instance = this`、`DontDestroyOnLoad(this)`。
3. `Utility.Unity.MakeEntity(transform)`：创建子节点 `[Unity.Utility]`，并注册内部 `UtilityLoopService`（同时实现 4 个 tick 接口的 `ServiceBase`）。
4. `Log.Init()` 并打印版本；计算 `Utility.Converter.ScreenDpi`（`Screen.dpi <= 0` 时取 96）。
5. 应用 `frameRate` / `gameSpeed` / `runInBackground` / `neverSleep`。
6. 订阅 `Application.lowMemory`。

`OnApplicationQuit` 的关闭顺序（`RootModule.Shutdown()`）：

1. 退订 `Application.lowMemory`，`StopAllCoroutines()`。
2. `AppServices.TryGet(out IResourcePackageService resources)` 先把资源包服务留住。
3. `AppServices.Shutdown()`：按注册逆序 `Dispose` 所有服务。
4. `GameApp.ResetCachedServices()`：清空 `GameApp` 的 7 个静态缓存。
5. `resources?.ShutdownBackend()`：资源服务释放完再关 YooAsset 后端，反序会让清理落在失效的包上。
6. `Destroy(gameObject)`：Entry 与子组件一起销毁（此时子组件的 `AppServices.Unregister` 已是安全空操作）。
7. `Utility.Unity.Shutdown()`、`Utility.Marshal.FreeCachedHGlobal()`。

## 最短接入

1. 确认启动场景里有 `Entry.prefab`，或自己挂一个 `RootModule`。
2. 定义契约与服务类型：

```csharp
using AlicizaX;

public interface IPlayerProfileService : IService
{
    int Level { get; }
    void SetLevel(int level);
}

public sealed class PlayerProfileService : ServiceBase, IPlayerProfileService
{
    public int Level { get; private set; }

    public void SetLevel(int level) => Level = level;
}
```

3. 在早于业务的注册点注册，之后任意位置解析：

```csharp
AppServices.Register<IPlayerProfileService>(new PlayerProfileService());

IPlayerProfileService profile = AppServices.Require<IPlayerProfileService>();
profile.SetLevel(12);
```

## 核心用法：注册、解析、注销

| 成员 | 行为 |
| --- | --- |
| `Register(IService)` / `Register(IService, Type[])` | 只按具体类型 / 具体类型 + 显式契约数组 |
| `Register<TContract>(IService)` | 具体类型 + `TContract`，返回 `TContract` |
| `Register<TContract, TExtraContract>(IService)` | 具体类型 + 两个契约 |
| `TryGet<T>(out T service)` | 查不到返回 `false`；容器不存在也返回 `false` |
| `Require<T>()` | 查不到抛 `InvalidOperationException` |
| `Unregister(IService service)` | 注销并 `Dispose`；容器已销毁返回 `false` |

```csharp
// 注册返回主契约实例；具体类型同时可解析
IPlayerProfileService profile = AppServices.Register<IPlayerProfileService>(new PlayerProfileService());

// 注销：容器已关闭时返回 false，不抛异常
AppServices.Unregister(profile);
```

契约在注册边界一次性校验：

- 具体类型必然可解析，不必再把它写进契约列表。
- 契约必须是该实例实现的类型，否则 `ArgumentException`。
- 同一个契约不能绑定两个实例、同一个实例不能注册两次，否则 `InvalidOperationException`。
- `OnInitialize` 抛异常时注册整体回滚（契约与 tick 列表都移除），异常抛回调用方。

容器未创建时 `Register` 抛 `InvalidOperationException("ServiceRegistry has not been created yet.")`；`Unregister`、`TryGet` 则不抛。

## 核心用法：ServiceBase 生命周期

服务生命周期接口 `IServiceLifecycle` 是 internal 的，业务程序集无法直接实现，所以普通服务统一继承 `ServiceBase`；它还提供 `protected T Require<T>()` 与 `protected bool TryGet<T>(out T)`，比直接写 `AppServices` 更省事：

```csharp
public sealed class VibrationService : ServiceBase, IVibrationService
{
    private IAudioService _audio;

    protected override void OnInitialize()
    {
        // Register 内部同步调用，此时服务已在容器里，可以安全解析依赖
        _audio = Require<IAudioService>();
    }

    protected override void OnDispose()
    {
        // Unregister 或容器 Dispose 时调用，注册表引用还没清空，仍能解析
        _audio = null;
    }
}
```

## 核心用法：Tick 生命周期

| 接口 | 方法 | 驱动方 | 时机 |
| --- | --- | --- | --- |
| `IServiceTickable` | `Tick(float deltaTime)` | `AppServiceRoot.Update()` | 每帧，`Time.deltaTime` |
| `IServiceLateTickable` | `LateTick(float deltaTime)` | `AppServiceRoot.LateUpdate()` | 每帧 LateUpdate |
| `IServiceFixedTickable` | `FixedTick(float fixedDeltaTime)` | `AppServiceRoot.FixedUpdate()` | 每物理帧，`Time.fixedDeltaTime` |
| `IServiceGizmoDrawable` | `DrawGizmos()` | `AppServiceRoot.OnDrawGizmos()` | 仅编辑器绘制 Gizmos 时 |

```csharp
public sealed class SessionClockService : ServiceBase, ISessionClockService, IServiceTickable, IServiceLateTickable
{
    private float _sessionSeconds;

    public void Tick(float deltaTime) => _sessionSeconds += deltaTime;

    public void LateTick(float deltaTime)
    {
        if (_sessionSeconds < 60f) return;
        _sessionSeconds = 0f;
    }
}
```

- tick 中注册的服务当帧不参与（循环用进入时的 `Count` 快照），下一个阶段或下一帧才开始。
- tick 中注销是延迟的：服务先进待删除列表，当前阶段循环结束后才 `Dispose`；若它排在注销者之后，本帧仍会收到一次 tick。
- 单个服务 tick 抛异常只记录第一次，循环继续，其他服务不受影响；该服务下一帧仍会被调用。

## 核心用法：Mono 服务

服务本身需要 `MonoBehaviour` 能力（序列化配置、协程、Unity 回调）时继承 `MonoServiceBehaviour`：

```csharp
using AlicizaX;
using UnityEngine;

[DefaultExecutionOrder(-30000)] // 必须晚于 AppServiceRoot(-32000)，早于业务代码
public sealed class CameraRigService : MonoServiceBehaviour<ICameraRigService>, ICameraRigService
{
    [SerializeField] private Camera _camera;

    public Camera Camera => _camera;

    protected override void Awake()
    {
        base.Awake();                 // 注册（并按 _dontDestroyOnLoad 常驻）
        if (_camera == null) _camera = Camera.main;
    }

    protected override void OnDestroy()
    {
        _camera = null;               // 先清理，再让基类注销
        base.OnDestroy();
    }
}
```

- `Awake` 注册、`OnDestroy` 注销，解析方式与普通服务完全一致。
- 默认只按具体类型可解析；要暴露接口契约就继承 `MonoServiceBehaviour<TContract>`，或重写 `GetServiceContracts()`。
- 不能再实现 `IServiceTickable` / `IServiceLateTickable` / `IServiceFixedTickable` / `IServiceGizmoDrawable`：注册时直接抛 `InvalidOperationException`，因为 Unity 已经在驱动它。
- 重写 `Awake` / `OnDestroy` 必须调用 `base`，否则不会注册/注销。
- 真实用法见 `Client/Packages/com.alicizax.unity.input/Runtime/InputService.cs`：`MonoServiceBehaviour` + `[DefaultExecutionOrder(-30000)]`，`Awake` 中先 `base.Awake()` 再校验配置。

## 核心用法：GameApp 快捷入口

`GameApp` 缓存 7 个常用服务，首次访问时 `Require` 并缓存：

```csharp
RootModule root = GameApp.Base;   // 就是 RootModule.Instance
root.FrameRate = 60;
root.PauseGame();

ITimerService timer = GameApp.Timer;
IAudioService audio = GameApp.Audio;
IUIService ui = GameApp.UI;
```

`Base` / `Audio` / `Resource` / `ResourcePackage` / `Scene` / `Timer` / `UI` / `GameObjectPool` 覆盖全部内置模块。缓存是静态字段，`RootModule.Shutdown` 会通过 `GameApp.ResetCachedServices()` 清空。

## 容器关闭与静态缓存重置

```csharp
AppServices.Shutdown();   // 逆序 Dispose 全部服务，然后置空容器
```

- 只由 `RootModule.OnApplicationQuit` 调用；`AppServiceRoot.OnDestroy` 会兜底，且仅当容器仍是它创建的那个实例。
- 单个服务 `Dispose` 抛异常只记日志，不阻断其余服务销毁。
- 销毁期间 `Register` 抛 `ObjectDisposedException`，避免新服务被追加到正在倒序遍历的列表尾部。
- 关闭后 `IsAvailable` 为 `false`，`TryGet` 返回 `false`，`Register` 抛 `InvalidOperationException`；手动 `Shutdown` 不经过 `ResetCachedServices`，`GameApp` 会继续返回旧实例。

## 常见配置

`Entry` 节点的 Inspector（`RootModuleInspector`）字段：

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `frameRate` | 120 | `Application.targetFrameRate`，Inspector 滑杆限制 1–120 |
| `gameSpeed` | 1 | `Time.timeScale`，设置负值会被钳到 0 |
| `runInBackground` | true | `Application.runInBackground` |
| `neverSleep` | true | `Screen.sleepTimeout`：`NeverSleep` / `SystemSetting` |
| `_dontDestroyOnLoad` | true | 定义在 `AppServiceRoot`；关闭后 Entry 随场景卸载 |
| Log Defines | — | Info/Warning/Error/Assert/Exception 开关，写入各平台宏并触发重编译 |

## 容易踩的坑

1. `Register` 抛 `ServiceRegistry has not been created yet.`，说明 `RootModule` 还没 `Awake`（或容器已被关闭）。让脚本的执行顺序大于 -32000，或把注册推迟到 `Start` 之后。
2. `MonoServiceBehaviour` 上多实现一个 tick 接口，会在 `Awake` 抛 `InvalidOperationException`，而且服务不会注册成功。
3. 在 `Tick` 里 `Unregister` 自己后不要继续用已释放的依赖：`Dispose` 在本阶段结束才执行，但下一次 tick 不会再来。
4. 一个契约只能绑定一个实例。重复注册会抛异常，初始化流程重入前先 `TryGet` 判断。
5. `OnDispose` 之后再调用 `ServiceBase.Require` 会 `NullReferenceException`：那时内部注册表引用已被清空，延迟回调里不要解析服务。

## API 速查

| API | 说明 |
| --- | --- |
| `AppServices.Register(IService)` / `Register(IService, Type[])` | 按具体类型 / 显式契约注册 |
| `AppServices.Register<TContract>(IService)` / `Register<TContract, TExtraContract>(IService)` | 带一个 / 两个契约注册 |
| `AppServices.TryGet<T>(out T)` / `Require<T>()` | 尝试解析 / 必须解析 |
| `AppServices.Unregister(IService)` / `IsAvailable` / `Shutdown()` | 注销 / 容器状态 / 销毁容器 |
| `ServiceBase.OnInitialize()` / `OnDispose()` | 服务生命周期钩子 |
| `IServiceTickable` / `IServiceLateTickable` / `IServiceFixedTickable` / `IServiceGizmoDrawable` | tick 契约 |
| `MonoServiceBehaviour` / `MonoServiceBehaviour<TContract>` | Mono 服务基类 |
| `GameApp.Base` / `AppServices` | 根节点与容器门面 |
