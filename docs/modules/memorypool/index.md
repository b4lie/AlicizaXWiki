# MemoryPool 内存池

`MemoryPool` 是框架唯一的普通 C# 引用对象池：按类型分池、借出与归还零分配，归还时自动 `Clear()` 重置，空闲对象由预算策略逐帧淘汰。`GameObject` 实例请用 [GameObjectPool](../gameobjectpool/index.md)。

源码：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/MemoryPool`，调试面板 `Client/Packages/com.alicizax.unity.framework/Editor/Common/Inspector/MemoryPoolComponentInspector.cs`。

## 适用场景

- 高频创建/释放的小对象：事件参数、加载请求、战斗结算数据、UI 列表项数据。
- 需要跨多次使用保留昂贵资源（资源租约、`CancellationTokenSource`、缓存对象）的包装类，例如音频模块的 `AudioClipCacheEntry`、`AudioLoadRequest`。
- 想消除稳定期 GC 分配的热路径。

不适用：`GameObject` / `Component`（用 `GameObjectPool`）、超大数组、需要长期存活且所有权不清的对象。

## 使用前提

1. 池化类型必须是**非抽象 class**，继承 `MemoryObject`，有 public 无参构造函数（`Acquire<T>` 的 `new()` 约束）。
2. 必须实现 `public override void Clear()`：归还时调用，只做状态重置。
3. 只在 Unity 主线程使用。`Acquire` / `Release` 会走 `MemoryPoolRegistry.AssertMainThread`，Editor 与 Development Build 下其它线程调用抛 `InvalidOperationException("MemoryPool must be used from the Unity main thread.")`。
4. 想要"空闲对象随时间缩容"，场景里必须有 `MemoryPoolSetting`（模板 `Entry.prefab` 的 `Memory Pool` 节点，`[DefaultExecutionOrder(-1000)]`）。**没有这个组件，预算取 `MemoryPoolBudget.Default`，且没人驱动 `TickAll`、也没人订阅 `Application.lowMemory`**，空闲对象不会随时间缩容，只有硬上限溢出、`SetBudget`、`TrimAll` 会淘汰。

## 最短接入

1. 定义池化类型：

```csharp
using AlicizaX;

public sealed class DamageInfo : MemoryObject
{
    public int AttackerId;
    public int Damage;
    public bool Critical;

    public override void Clear()
    {
        AttackerId = 0;
        Damage = 0;
        Critical = false;
    }
}
```

2. 借出与归还：

```csharp
DamageInfo info = MemoryPool.Acquire<DamageInfo>();
try
{
    info.AttackerId = 1001;
    info.Damage = 350;
    // ... 使用
}
finally
{
    MemoryPool.Release(info);
}
```

3. 在服务根节点放 `MemoryPoolSetting`（菜单 `Add Component` 里搜类名，模板 `Entry.prefab` 已挂），按需调预算。

4. 需要跨复用持有资源时再实现 `IPoolEvictable`（见下）。

## 核心用法

### 生命周期：只有 `Clear()` 和 `OnEvict()`

`MemoryObject` 的全部成员就是 `Clear()`（抽象方法）加两个 `internal` 字段；除 `Clear()` 与 `IPoolEvictable.OnEvict()` 外没有其它生命周期钩子：

| 时机 | 调用 | 说明 |
| --- | --- | --- |
| `Acquire<T>()` | 无回调 | 取最近归还的对象（LIFO）；没有就 `new T()` |
| `Release(obj)` | `Clear()` | 每次归还都调，抛异常则对象仍算借出 |
| 被淘汰 | `IPoolEvictable.OnEvict()` | 对象被真正丢弃，之后不能再归还 |

`Release` 的完整顺序：

1. 主线程断言。
2. `InPool == true` 时抛 `MemoryPool<T>: object is already released.`，池状态不变（此时**还没**调 `Clear()`，不会重复清）。
3. 调用 `Clear()`；异常直接抛给调用方，对象保持借出状态，修好后可再次 `Release`。
4. 借出计数减一；若空闲数已达 `HardFreeCount`，直接淘汰该对象，不入空闲列表。
5. 否则入空闲环形数组，必要时扩容，并请求下一次 Tick。

`Release` 按 `memory.Pool` 路由到**运行时类型**的池，所以用 `MemoryObject` 或基类变量归还也安全；反过来 `Acquire<Base>` 与 `Acquire<Derived>` 是两个不同的池。

`Clear()` 里可以归还其它对象（同池、跨池都允许），也可以 `Acquire`。

### `IPoolEvictable`：持有资源时必写

```csharp
public sealed class CachedTextWrapper : MemoryObject, IPoolEvictable
{
    public AssetLease<TextAsset> Lease;

    public override void Clear()
    {
        // 只重置业务状态，租约要跨复用保留
    }

    public void OnEvict()
    {
        Lease.Dispose();
        Lease = default;
    }
}
```

- 不需要跨复用保留资源时，直接在 `Clear()` 里释放即可，不必实现 `IPoolEvictable`。
- `OnEvict()` 抛异常会向上抛，已淘汰的对象不回池，其余空闲对象保持原状，下一次维护继续处理。
- 池化类型未实现 `IPoolEvictable` 时，淘汰不会产生任何通知——这是最容易漏释放的地方。

### 预算与缩容

```csharp
MemoryPool.SetBudget(new MemoryPoolBudget(
    retainFreeCount: 0,
    softFreeCount: 64,
    hardFreeCount: 256,
    trimPerTick: 8,
    peakWindowSeconds: 20f));
```

- 构造函数校验参数：`retain >= 0`、`soft >= retain`、`hard >= soft`、`trimPerTick > 0`、`peakWindowSeconds > 0`，否则抛 `ArgumentOutOfRangeException`（`ParamName` 指向越界的那个）。
- `MemoryPoolBudget.Default` = `(0, 128, 512, 16, 30f)`。
- `SetBudget` 立即生效：超过新 `HardFreeCount` 的空闲对象当场淘汰，超 `RetainFreeCount` 的池重新进入 Tick 调度。
- 每个池的缩容目标：`TargetFreeCount = clamp(近期借出峰值, RetainFreeCount, SoftFreeCount)`，每帧最多淘汰 `TrimPerTick` 个；空闲数降到 `RetainFreeCount` 后退出调度。
- 近期峰值用 8 桶滑动窗口（`PeakWindow`）统计，`Peak = max(当前借出数, 窗口内各桶峰值)`，计时用 `Time.unscaledTime`。一次突发后空闲对象会被保留整个窗口，窗口滑过才开始缩容。

### GC 友好的使用约束

- 稳态（空闲列表已扩容到位）下 `Acquire` / `Release` 不产生托管分配；`Clear()` 里若写了分配，那是业务自己的开销。
- 不要为了"用一下"而 `new` 池化类型：既绕开了池，`CreatedCount` 也会一直涨；也不要把池化对象长期挂在静态字段里。
- 统计要给缓冲：自己复用 `MemoryPoolInfo[]`（或 `stackalloc` 的 `Span`），不要每帧 `new`。
- 每种 `T` 的池是静态单例，首次 `Acquire<T>` 才创建并注册（一次性开销），别把它放在极端热路径的首次调用上。

### 清理与统计

```csharp
MemoryPool.TrimAll();                       // 淘汰所有池的全部空闲对象
int poolCount = MemoryPool.Count;           // 已注册的池类型数
var infos = new MemoryPoolInfo[MemoryPool.Count];
int count = MemoryPool.GetInfos(infos);     // 最多写入 destination.Length 条，返回写入条数
```

`GetInfos` 接受 `Span<MemoryPoolInfo>`，目标长度不够时按长度截断（不会扩容）。

### `MemoryPoolSetting` 生命周期

| 时机 | 行为 |
| --- | --- |
| `Awake` | 用 Inspector 字段 `SetBudget`，并订阅 `Application.lowMemory` |
| `Update` | `MemoryPoolRegistry.TickAll(Time.unscaledTime)` 驱动所有待缩容的池 |
| `Application.lowMemory` | `TrimAll(RetainFreeCount)`，即所有池缩到保留下限 |
| `OnDestroy` | 取消订阅并 `TrimAll(0)`；借出中的对象不受影响，之后归还会正常入池 |

## 常见配置

`MemoryPoolSetting` 的 Inspector 字段（序列化名即 `m_` 前缀字段，`OnValidate` 会钳制）：

| Inspector | 字段 | 默认 | 作用 | 钳制 |
| --- | --- | ---: | --- | --- |
| `Retain Free` | `m_RetainFreeCount` | `0` | 长期保留的空闲对象数，缩容不会低于它 | `>= 0` |
| `Soft Free` | `m_SoftFreeCount` | `128` | 空闲软上限；超过近期峰值的部分逐帧淘汰 | `>= Retain Free` |
| `Hard Free` | `m_HardFreeCount` | `512` | 空闲硬上限；归还时已达该值就直接丢弃 | `>= Soft Free` |
| `Trim Per Tick` | `m_TrimPerTick` | `16` | 每帧每个池最多淘汰几个空闲对象 | `>= 1` |
| `Peak Window (s)` | `m_PeakWindowSeconds` | `30` | 峰值统计窗口（秒），不受 `timeScale` 影响 | `>= 0.1` |

`Hard Free` 只限制"归还后最多缓存多少空闲对象"，**不限制借出数量**；借出多少由业务决定。

## 容易踩的坑

1. `new DamageInfo()` 之后直接 `MemoryPool.Release(obj)`：`obj.Pool` 是 `null`，抛 `NullReferenceException`。只能归还 `Acquire` 来的对象。
2. 重复 `Release` 或归还已被淘汰的对象：抛 `already released`。此时 `Clear()` 还没执行，所以不会把已清空的状态再清一遍，但异常必须处理掉。
3. `Clear()` 抛异常时对象**仍在借出**，不要吞掉异常当作已归还；否则这批对象会一直挂在 `LeasedCount` 上。
4. 场景里没有 `MemoryPoolSetting`：预算停在默认值且没有任何 Tick，空闲对象只增不减（除硬上限/手工清理），表现很像"内存泄漏"。
5. 泛型类型是一个独立池：`Acquire<MyItem<int>>` 与 `Acquire<MyItem<string>>` 互不相干，`Count` 会把它们各算一个。
6. `Clear()` / `OnEvict()` 里不要留下未释放的 `IDisposable`、事件订阅、静态引用——`Clear()` 只负责重置字段，池不做反射清扫。

## 如何判断是否泄漏

- 看 `Info.LeasedCount`：业务结束后等几帧仍大于 0，说明对象没归还。沿该类型的 `Acquire` 调用点查异常分支、提前 `return`、取消流程。
- 恒等式：`CreatedCount == FreeCount + 累计淘汰数`。`CreatedCount` 在稳定阶段持续增长，说明峰值确实超过缓存或归还太晚，是性能问题而不是泄漏。
- `FreeCount > 0` 不是泄漏，只是缓存；窗口滑过后会逐帧缩到 `RetainFreeCount`。

```csharp
var infos = new MemoryPoolInfo[MemoryPool.Count];
int count = MemoryPool.GetInfos(infos);
for (int i = 0; i < count; i++)
{
    MemoryPoolInfo info = infos[i];
    // info.Type / FreeCount / LeasedCount / CreatedCount / TargetFreeCount / PeakLeasedCount
}
```

## 编辑器调试面板

Play 模式选中场景里的 `MemoryPoolSetting`（`MemoryPoolComponentInspector`）：

- `Runtime`：`Memory Pool Count`、`Show Full Class Name`、`Show Empty Pools`（默认隐藏 `Free == 0 && Leased == 0` 的池）。
- `Budget`：当前生效的 `Retain / Soft / Hard / Trim Per Tick / Peak Window`。
- `Overview`：`Total Cached`、`Total In Use`、`Total Created`。
- `Actions`：`Trim All`（等价 `MemoryPool.TrimAll()`）。
- `Pools`：按 `Type.Assembly` 分组，每个池显示 `Free / Leased / Created` 与 `Target / Peak`，程序集卡片可 `Export CSV`。

非 Play 模式只显示 `Configuration` 与提示；选中 Prefab 资产（不在场景里）时提示 "Select a scene instance to inspect memory pool runtime data."；Play 模式下且选中的是场景实例时 `Configuration` 段会隐藏。

## API 速查

```csharp
// MemoryPool（静态入口）
static int Count { get; }
static T Acquire<T>() where T : MemoryObject, new();
static void Release(MemoryObject memory);
static void SetBudget(in MemoryPoolBudget budget);
static void TrimAll();
static int GetInfos(Span<MemoryPoolInfo> destination);

// MemoryObject
abstract void Clear();

// IPoolEvictable
void OnEvict();

// MemoryPoolBudget
readonly int RetainFreeCount, SoftFreeCount, HardFreeCount, TrimPerTick;
readonly float PeakWindowSeconds;
static readonly MemoryPoolBudget Default;

// MemoryPoolInfo
Type Type { get; }
int FreeCount { get; } int LeasedCount { get; } int CreatedCount { get; }
int TargetFreeCount { get; } int PeakLeasedCount { get; }
```
