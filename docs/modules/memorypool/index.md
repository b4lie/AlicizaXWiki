# MemoryPool 模块

`MemoryPool` 是框架唯一的 C# 引用对象池，用于复用频繁创建和释放的临时对象，例如事件参数、加载请求、战斗结算数据、UI 列表项数据，以及持有资源租约、需要跨多次使用缓存的包装对象。`GameObject` 实例请使用 `GameObjectPool`。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/MemoryPool`
- Inspector：`Client/Packages/com.alicizax.unity.framework/Editor/Common/Inspector/MemoryPoolComponentInspector.cs`

## 使用前提

池对象必须继承 `MemoryObject`，并且是带 public 无参构造函数的非抽象 `class`。

```csharp
using AlicizaX;

public sealed class BattleDamageInfo : MemoryObject
{
    public int AttackerId;
    public int TargetId;
    public int Damage;
    public bool Critical;

    public override void Clear()
    {
        AttackerId = 0;
        TargetId = 0;
        Damage = 0;
        Critical = false;
    }
}
```

约束：

- 只能在 Unity 主线程使用。Editor / Development Build 下其它线程调用会抛 `InvalidOperationException`。
- 使用完成后必须调用 `MemoryPool.Release` 归还。
- 只能归还从 `MemoryPool.Acquire<T>()` 取得的对象。

## 获取和归还

```csharp
BattleDamageInfo info = MemoryPool.Acquire<BattleDamageInfo>();
info.AttackerId = 1001;
info.TargetId = 2001;
info.Damage = 350;
info.Critical = true;

MemoryPool.Release(info);
```

`Acquire` 不会失败：有空闲对象时取最近归还的那个，没有时直接 `new T()`。

`Release` 的执行顺序：

1. 对象已在池中（重复归还、或已被淘汰）时抛 `InvalidOperationException`，池状态不变。
2. 调用 `Clear()`。`Clear()` 抛异常时异常直接抛给调用方，对象仍算借出状态，修复后可以再次 `Release`。
3. 计入空闲列表。空闲数已经达到 `Hard Free` 时不入列，直接淘汰。

`Clear()` 内可以归还其它对象（包括同类型对象），也可以 `Acquire`。

## 持有资源的对象

需要在多次复用之间保留昂贵资源（资源租约、`CancellationTokenSource`、缓存的 Unity 对象）时实现 `IPoolEvictable`：

```csharp
public sealed class CachedTextWrapper : MemoryObject, IPoolEvictable
{
    public AssetLease<TextAsset> Lease;

    public override void Clear()
    {
    }

    public void OnEvict()
    {
        Lease.Dispose();
        Lease = default;
    }
}
```

- `Clear()`：每次归还都会调用，只重置业务状态。
- `OnEvict()`：对象被池真正丢弃时调用一次（硬上限溢出、Tick 缩容、`TrimAll`、低内存、`MemoryPoolSetting` 销毁）。被淘汰的对象不能再归还。
- `OnEvict()` 抛异常时直接向上抛出，已经淘汰的对象不会回到池中，其余空闲对象保持原状，下一次维护继续处理。

不需要跨复用保留资源时，在 `Clear()` 里直接释放即可，不必实现 `IPoolEvictable`。

## 回收策略

每个类型一个池，空闲对象存放在环形数组里：

- 获取取最新归还的对象（LIFO），对 CPU 缓存友好。
- 缩容淘汰最早归还的对象（FIFO）。
- 获取、归还都是 O(1)，稳定状态零 GC。

每个池记录最近 `Peak Window` 秒内的借出峰值（8 个时间桶的滑动窗口，计时使用 `Time.unscaledTime`）。`MemoryPoolSetting.Update` 每帧只 Tick 有多余空闲对象的池：

```text
TargetFree = clamp(近期借出峰值, Retain Free, Soft Free)
```

空闲数超过 `TargetFree` 时，每帧最多淘汰 `Trim Per Tick` 个。空闲数降到 `Retain Free` 后该池退出调度，不再占用 Tick。

一次突发后，空闲对象会被保留整个 `Peak Window`，窗口滑过后才开始逐帧缩容。

## MemoryPoolSetting 配置

| 参数 | 默认值 | 作用 | 限制 |
|---|---:|---|---|
| `Retain Free` | `0` | 长期保留的空闲对象数，缩容不会低于该值。 | `>= 0` |
| `Soft Free` | `128` | 空闲对象软上限，超过近期峰值的部分逐帧淘汰。 | `>= Retain Free` |
| `Hard Free` | `512` | 空闲对象硬上限。归还时空闲数已达该值则直接淘汰。 | `>= Soft Free` |
| `Trim Per Tick` | `16` | 每帧每个池最多淘汰的空闲对象数。 | `>= 1` |
| `Peak Window (s)` | `30` | 峰值统计窗口，单位秒，不受 `timeScale` 影响。 | `> 0` |

`Hard Free` 只限制归还后最多缓存多少空闲对象，不限制借出数量。

运行时修改预算：

```csharp
MemoryPool.SetBudget(new MemoryPoolBudget(retainFreeCount: 0, softFreeCount: 64, hardFreeCount: 256, trimPerTick: 8, peakWindowSeconds: 20f));
```

参数非法时构造函数抛 `ArgumentOutOfRangeException`。新预算立即生效：超过新 `Hard Free` 的空闲对象会马上淘汰。

`MemoryPoolSetting` 的生命周期：

- `Awake`：应用 Inspector 中的预算。
- `Update`：Tick 需要缩容的池。
- `Application.lowMemory`：所有池缩到 `Retain Free`。
- `OnDestroy`：淘汰所有空闲对象。借出中的对象不受影响，之后归还会正常入池。

## 手动清理 API

```csharp
// 淘汰所有池的全部空闲对象。
MemoryPool.TrimAll();
```

业务代码通常不需要手动清理。

## 运行时统计

```csharp
var infos = new MemoryPoolInfo[MemoryPool.Count];
int count = MemoryPool.GetInfos(infos);
```

| 字段 | 含义 |
|---|---|
| `Type` | 池对象类型 |
| `FreeCount` | 当前空闲对象数 |
| `LeasedCount` | 当前借出未归还的对象数 |
| `CreatedCount` | 累计 `new T()` 次数 |
| `TargetFreeCount` | 最近一次 Tick 计算的目标空闲数 |
| `PeakLeasedCount` | 峰值窗口内的最大借出数 |

## Inspector 调试面板

Play 模式下选中场景里的 `MemoryPoolSetting`：

- `Runtime`：池类型数量、`Show Full Class Name`、`Show Empty Pools`（默认隐藏 `Free=0 && Leased=0` 的池）。
- `Budget`：当前生效的预算。
- `Overview`：`Total Cached`（空闲总数）、`Total In Use`（借出总数）、`Total Created`。
- `Actions`：`Trim All`，淘汰所有空闲对象。
- `Pools`：按程序集分组列出每个池的 `Free / Leased / Created / Target / Peak`，可导出 CSV。

运行时调试面板（[`DebuggerComponent`](../debugger/index.md)）的 `Profiler/Reference Pool` 页显示同样的数据。

## 如何判断是否泄露

- 优先看 `Leased`：业务结束后等待几帧仍大于 0，说明有对象没有归还。沿该类型的 `Acquire` 调用点检查异常分支、提前 return、取消流程。
- `Free > 0` 不是泄露，只是缓存。池不再使用时，窗口滑过后会逐帧缩到 `Retain Free`。
- `Created` 在稳定阶段持续增长，说明峰值超过了缓存或对象归还过晚，是性能问题而不是泄露。

## 已删除的 API

- `IMemory`、`MemoryPool<T>.Acquire/Release`（改用 `MemoryPool.Acquire<T>()` / `MemoryPool.Release`）
- `MemoryPool.Acquire(Type)`、`MemoryPoolHandle`、`MemoryPool.GetHandle`
- `MemoryPool.Add/Remove/RemoveAll/Compact/CompactAll/ClearAll`
- `MemoryPool.TrimAllNativeMetadata`、`MemoryPool.ResetAllStats`、`MemoryPool.GetAllMemoryPoolInfos`
- `MemoryPoolSetting` 的帧数衰减参数与 `Phase`
- `ObjectPool` 模块整体（`IObjectPoolService`、`ObjectBase`、`ObjectPoolComponent`、`GameApp.ObjectPool`）。普通引用对象改用 `MemoryPool`，需要跨复用持有资源时实现 `IPoolEvictable`；`GameObject` 实例改用 `GameObjectPool`。
