# Timer 计时器

Timer 提供全局的延迟执行能力：一次性/循环计时、暂停与恢复、按句柄取消、可选择是否跟随 `Time.timeScale`，并支持查询剩余时间与容量统计。它面向"按秒计时"，不是逐帧调度器：每帧到期回调有固定预算，超出的部分顺延到下一帧。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Timer`
- `Client/Packages/com.alicizax.unity.framework/Editor/Modules/Timer/TimerComponentInspector.cs`

## 适用场景

技能冷却、延迟弹窗、引导超时、防沉迷倒计时、断线重连等待、节流刷新等"隔一段时间做一次"的逻辑。需要每帧执行的逻辑请用 Service 的 tick 接口，而不是 0 间隔计时器。

## 使用前提

场景里必须同时具备两样东西：

1. `TimerComponent`：挂在 `Entry.prefab` 的 `Timer` 子节点上（`[DefaultExecutionOrder(-800)]`）。它只做一件事——`Awake` 里检查容器，没有 `ITimerService` 就注册一个 `TimerService`；已经存在时直接返回，所以重复挂 `TimerComponent` 不会替换已有服务。
2. 一个 `AppServiceRoot`（模板里是 `Entry` 上的 `RootModule`）：计时器的推进发生在 `AppServiceRoot.Update()` → `ServiceRegistry.Tick()` → `TimerService.Tick()`。只有 `TimerComponent` 而没有服务根节点，计时器永远不会触发。

自己搭场景时，把 `RootModule` 与 `TimerComponent` 一起挂到常驻节点即可；`TimerComponent` 不需要任何 Inspector 配置。

获取服务两种方式等价：

```csharp
using AlicizaX;
using AlicizaX.Timer.Runtime;

ITimerService timer = AppServices.Require<ITimerService>();
ITimerService viaGameApp = GameApp.Timer;   // 内部同样走 Require 并缓存
```

## 最短接入

1. 确认 `Entry.prefab`（或自己的 `RootModule` + `TimerComponent`）在启动场景里。
2. 拿服务并创建计时器，保存返回的句柄。
3. 生命周期结束时用 `RemoveTimer` 释放。

```csharp
using AlicizaX;
using AlicizaX.Timer.Runtime;
using UnityEngine;

public sealed class TutorialTip : MonoBehaviour
{
    private ulong _handle;

    private void Start()
    {
        _handle = GameApp.Timer.AddTimer(ShowTip, 3f);
    }

    private void OnDestroy()
    {
        if (AppServices.TryGet<ITimerService>(out var timer)) timer.RemoveTimer(_handle);
    }

    private void ShowTip() => Debug.Log("3 秒后显示提示");
}
```

`AddTimer` 返回 `ulong` 句柄。返回 `0` 表示创建失败（回调为空、`time` 非有限值、服务已销毁或没有空闲槽位），不要拿它继续操作。

## 核心用法：一次性与循环

```csharp
// 一次性：time 秒后执行一次，然后句柄自动失效
ulong once = timer.AddTimer(OnTimeout, 5f);

// 循环：每隔 time 秒执行一次，直到 RemoveTimer
ulong loop = timer.AddTimer(OnHeartbeat, 1f, isLoop: true);

// 循环 + 不受 timeScale 影响
ulong realtimeLoop = timer.AddTimer(OnPing, 1f, isLoop: true, isUnscaled: true);
```

循环计时器按创建时的间隔重复；如果某一轮被长时间卡住，多出来的周期会被合并，不会连发补偿。

## 核心用法：取消、暂停、恢复、重启

```csharp
ulong handle = timer.AddTimer(OnTimeout, 10f);

timer.Stop(handle);            // 暂停：剩余时间被记住
float left = timer.GetLeftTime(handle);
timer.Resume(handle);          // 从剩余时间继续
timer.Restart(handle);         // 回到完整时长并重新开始
bool running = timer.IsRunning(handle);
timer.RemoveTimer(handle);     // 彻底释放槽位
```

- `Stop` 是暂停，不是销毁；循环计时器暂停后，剩余时间记为完整时长。
- `Resume` 对已在运行的句柄是空操作。
- `Restart` 对一次性计时器等价于"重新计时"，句柄保持不变，可以在自己的回调里调用它来延长时间。
- `RemoveTimer` 之后句柄失效，再 `Stop`/`Resume`/`IsRunning` 都是安全空操作（`IsRunning` 返回 `false`，`GetLeftTime` 返回 `0`）。
- 槽位会复用，但句柄带版本号，旧句柄不会误伤同槽位的新计时器。

## 核心用法：与时间缩放解耦

默认计时器使用 `Time.timeAsDouble`，会被 `Time.timeScale` 影响；`isUnscaled: true` 改用 `Time.unscaledTimeAsDouble`。

```csharp
GameApp.Base.PauseGame();       // Time.timeScale = 0

ulong paused = timer.AddTimer(OnResumeTip, 5f);                    // 暂停期间不推进
ulong network = timer.AddTimer(OnReconnectTimeout, 5f, isUnscaled: true); // 照常推进

GameApp.Base.ResumeGame();
```

`timeScale = 0` 时，scaled 计时器的 `GetLeftTime` 会保持在暂停前的剩余值；恢复后从该值继续。整帧卡顿（例如长时间 GC）后，过期的计时器在下一次 tick 触发一次，不会按错过的周期补发。

## 核心用法：向回调传参

三种重载，按分配需求选。先定义参数类型与结构体回调：

```csharp
using AlicizaX.Timer.Runtime;

public sealed class RewardContext
{
    public int PlayerId;
}

// 结构体回调：调用点不会创建委托
public readonly struct RewardToast : ITimerCallback<RewardContext>
{
    public void Invoke(RewardContext context) { /* ... */ }
}
```

调用侧：

```csharp
timer.AddTimer(OnReward, 2f);                                          // 无参
timer.AddTimer<RewardContext>(GiveReward, context, 3f);                // 引用类型参数
timer.AddTimer<RewardToast, RewardContext>(context, 3f);               // 结构体回调
```

`AddTimer<T>` 的委托由框架按 `T` 缓存，调用本身不分配；但如果传的是捕获外部变量的 lambda，闭包对象仍由调用方分配。`AddTimer<T>` 的 `T` 必须是 `class`（引用类型），不能直接传 `int`、`float`；要传多个字段就定义一个上下文类。

## 容量与统计

计时器槽位按页分配（每页 256 个），不够时自动加页，上限 4096 页（约 104 万槽位）。大量计时器被移除后调用 `TrimExcess()` 回收尾部空页；包含正在执行回调的页会被保留。

每个时钟（scaled / unscaled）每帧最多执行 256 个到期回调，超出的留到下一帧，游标停在本帧未处理完的位置。

统计接口不是注册契约，只能从服务实例做接口转换：

```csharp
using AlicizaX.Timer.Runtime;

if (timer is ITimerDebugService debug)
{
    debug.GetStatistics(out int active, out int capacity, out int peak, out int free);
    debug.GetAllTimers(buffer);   // buffer 是 TimerDebugInfo[]，返回写入条数
    timer.TrimExcess();
}
```

`TimerDebugInfo` 字段：`TimerHandle`、`LeftTime`、`Duration`、`Age`、`Flags`（`TimerDebugFlags.Running` / `Loop` / `Unscaled`）。

## 编辑器 Inspector 调试

选中挂了 `TimerComponent` 的节点，Play 模式下 Inspector 每 250ms 刷新一次：

- `Runtime Debug`：Active Timers、Pool Capacity、Peak Active Count、Free Slots，以及 Active/Peak 使用率进度条。
- `Active Timer Sample`：最多 32 条，显示 `ID`、`Loop/Once`、`Scaled/Unscaled`、`Running/Paused`、剩余时间与总时长。
- `Stale One-shot Timers`：创建超过 300 秒仍未触发的一次性计时器，用来发现忘记 `RemoveTimer` 的泄漏。

非 Play 模式只显示 "Available during runtime only."。

## 常见配置

`TimerComponent` 没有序列化字段，全部行为由 `AddTimer` 参数决定：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `time` | 必填 | 秒；小于等于 `1e-6` 时按 `1e-6` 处理 |
| `isLoop` | `false` | 是否循环触发 |
| `isUnscaled` | `false` | `true` 时用 `Time.unscaledTimeAsDouble` |
| `callback` | 必填 | 为空时直接返回 `0` |

## 容易踩的坑

1. 只挂 `TimerComponent` 而场景里没有 `AppServiceRoot`，计时器不会推进；反过来没有 `TimerComponent` 则 `Require<ITimerService>()` 直接抛异常。
2. `AddTimer` 返回 `0` 是失败信号，不要把它当有效句柄传入 `Stop`/`Resume`（虽然不会崩，但什么都不会发生）。
3. 热路径每次传捕获 lambda 会产生闭包分配；需要严格控制分配时用 `ITimerCallback` 结构体重载或缓存委托。
4. Timer 不感知 `UnityEngine.Object` 生命周期：场景对象销毁不会自动取消计时器，回调触发时参数里的对象可能已经 `== null`。离开场景或销毁对象前先 `RemoveTimer`。
5. `GameApp.Base.PauseGame()` 会把 `timeScale` 置 0，所有 scaled 计时器一起冻结；UI 超时、重连倒计时这类必须在暂停时继续走的逻辑要加 `isUnscaled: true`。
6. 集中到期时会受每帧 256 个回调的预算限制，实际触发时间可能晚几帧；不要把计时器当逐帧时钟用。

## API 速查

```csharp
ulong AddTimer(TimerHandlerNoArgs callback, float time, bool isLoop = false, bool isUnscaled = false);
ulong AddTimer<T>(Action<T> callback, T arg, float time, bool isLoop = false, bool isUnscaled = false) where T : class;
ulong AddTimer<TCallback, TArg>(TArg arg, float time, bool isLoop = false, bool isUnscaled = false)
    where TCallback : struct, ITimerCallback<TArg> where TArg : class;

void Stop(ulong timerHandle);          // 暂停
void Resume(ulong timerHandle);        // 从剩余时间继续
void Restart(ulong timerHandle);       // 回到完整时长重新开始
void RemoveTimer(ulong timerHandle);   // 释放槽位
bool IsRunning(ulong timerHandle);
float GetLeftTime(ulong timerHandle);
void TrimExcess();                     // 回收尾部空页

// ITimerDebugService
int GetAllTimers(TimerDebugInfo[] results);
void GetStatistics(out int activeCount, out int poolCapacity, out int peakActiveCount, out int freeCount);
```

相关页面：[Service 服务容器](../service/index.md)、[UI 模块](../ui/index.md)。
