# Event 事件

Event 是进程内的同步事件总线：订阅返回一个 `EventRuntimeHandle`，`Publish` 时按订阅顺序同步调用，单个监听抛异常不影响其余监听。事件类型必须是 `struct`，并实现 `IEmptyEventArgs`（无参）或 `IPayloadEventArgs`（带参）中的一个。

边界：没有跨线程保护、没有延迟/排队派发、不绑定 `UnityEngine.Object` 生命周期——取消订阅必须由持有句柄的一方显式完成。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Event`
- `Client/Packages/com.alicizax.unity.framework/Editor/Modules/Event/EventMonitorWindow.cs`
- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI/EventListenerProxy.cs`

## 适用场景

模块之间解耦：玩家数据变化后刷新多个界面、下载进度广播给进度条、设置项变更通知音量和语言。也适合"一次通知多个互不相识的监听者"的场合。

不适合跨线程通信、需要持久化或重放的事件流，以及必须在某个确定帧之后才投递的延迟消息（用 Timer 自己排程）。

## 使用前提

- 不需要挂任何组件，也没有初始化步骤：`EventContainer<TEvent, THandler>` 是静态泛型容器，第一次访问某个事件类型时才创建内部 `EventChannel`。
- 事件类型必须是 `struct`，并且只实现一种标记接口。两个接口都实现会同时匹配两个 `Subscribe` 重载，调用点无法确定派发模式。
- 订阅和派发都在主线程完成，源码里没有任何锁；不要从子线程 `Publish`。
- `Publish` 是同步的：订阅者里的耗时逻辑会直接算在发布者的调用栈上。

## 最短接入

1. 定义事件结构体。
2. 在 `OnEnable`/`OnInitialize` 里订阅并保存句柄。
3. 在合适的位置 `Publish`。
4. 在 `OnDisable`/`OnDestroy` 里 `Dispose` 句柄。

```csharp
using AlicizaX;

// 无参事件
public readonly struct InventoryChangedEvent : IEmptyEventArgs { }

// 带参事件
public readonly struct PlayerLevelUpEvent : IPayloadEventArgs
{
    public readonly int PlayerId;
    public readonly int NewLevel;

    public PlayerLevelUpEvent(int playerId, int newLevel)
    {
        PlayerId = playerId;
        NewLevel = newLevel;
    }
}

public sealed class LevelPresenter : UnityEngine.MonoBehaviour
{
    private EventRuntimeHandle _inventory;
    private EventRuntimeHandle _levelUp;

    private void OnEnable()
    {
        _inventory = EventBus.Subscribe<InventoryChangedEvent>(RefreshInventory);
        _levelUp = EventBus.Subscribe<PlayerLevelUpEvent>(OnLevelUp);
    }

    private void OnDisable()
    {
        _inventory.Dispose();
        _levelUp.Dispose();
    }

    private void RefreshInventory() { /* 刷新背包 */ }

    private void OnLevelUp(in PlayerLevelUpEvent evt) { /* 播放升级表现 */ }
}
```

派发：

```csharp
EventBus.Publish<InventoryChangedEvent>();

var evt = new PlayerLevelUpEvent(1001, 10);
EventBus.Publish(in evt);
```

## 核心用法：无参与带参事件

| 事件接口 | 订阅签名 | 派发签名 |
| --- | --- | --- |
| `IEmptyEventArgs` | `Subscribe<T>(Action handler)` | `Publish<T>()` |
| `IPayloadEventArgs` | `Subscribe<T>(InEventHandler<T> handler)` | `Publish<T>(in T payload)` |

`InEventHandler<T>` 的声明是 `delegate void InEventHandler<T>(in T evt)`，所以带参回调也必须写 `in`：

```csharp
private void OnLevelUp(in PlayerLevelUpEvent evt)
{
    Debug.Log($"{evt.PlayerId} -> Lv.{evt.NewLevel}");
}
```

带参数据从 `Publish` 到回调全程按 `in` 传递，不装箱；但业务若访问非 readonly 结构体的成员，C# 会做防御性拷贝，所以事件结构体建议全部字段 `readonly`。

## 核心用法：句柄与取消订阅

`EventRuntimeHandle` 是只读结构体，内部记录频道、槽位与版本号：

```csharp
EventRuntimeHandle handle = EventBus.Subscribe<InventoryChangedEvent>(RefreshInventory);
handle.Dispose();   // 取消这一个订阅

default(EventRuntimeHandle).Dispose();   // 空句柄，安全空操作
```

- 槽位会被复用，但句柄比较版本号，旧句柄不会误删同槽位的新订阅。
- `Dispose` 幂等，重复调用无害。
- 同一事件、同一目标、同一方法只能订阅一次：重复订阅会打印 `Duplicate event handler subscription: ...` 警告并返回 `default` 句柄，原订阅仍然有效。

## 核心用法：派发期间的时序

`Publish` 开始时对订阅数拍快照，之后逐个读取槽位，读到的已被取消项直接跳过：

| 派发中的操作 | 效果 |
| --- | --- |
| `Dispose` 某个还没轮到的句柄 | 该监听本轮不再被调用 |
| `ClearEmpty<T>()` / `ClearPayload<T>()` | 本轮剩余监听全部跳过，逻辑订阅立即清空 |
| `Subscribe` 新监听 | 不参与当前这一轮，但会被同类型的嵌套 `Publish` 调用 |
| 监听里抛异常 | `Debug.LogException` 记录后继续调用后面的监听 |

最外层 `Publish` 返回时才压缩空洞、恢复紧凑数组，所以"发布过程中取消"不会打乱仍在派发的顺序。

## 核心用法：容量与清理

```csharp
EventBus.ReserveEmptyCapacity<InventoryChangedEvent>(16);
EventBus.ReservePayloadCapacity<PlayerLevelUpEvent>(32);

int emptyCount = EventBus.GetEmptySubscriberCount<InventoryChangedEvent>();
int payloadCount = EventBus.GetPayloadSubscriberCount<PlayerLevelUpEvent>();

EventBus.ClearEmpty<InventoryChangedEvent>();
EventBus.ClearPayload<PlayerLevelUpEvent>();
```

初始容量 4，数组满时按倍数扩容；`Reserve*` 只能扩容，传负数抛 `ArgumentOutOfRangeException`。`Clear*` 只清订阅，不销毁频道、不回收容量、不重置类型。

内部结构：每个"事件类型 + 派发模式"组合对应一个 `EventContainer<TEvent, THandler>`，它持有 `EventChannel<THandler>`（紧凑委托数组 + 槽位版本表 + 去重集合 + 派发深度）。这些类型都是 internal，业务只能通过 `EventBus` 访问；频道列表另外供编辑器统计和进入 Play 模式时统一清空。

## 核心用法：Unity 对象与 UI 自动解绑

总线不感知 Unity 生命周期：对象被销毁后订阅仍在，`Publish` 时访问已销毁对象会抛异常（被逐监听隔离并记录），句柄只有 `Dispose` 才真正移除。

UI 视图不需要手写这套逻辑：`UIBase` 暴露 `protected EventListenerProxy EventProxy`，`AddUIEvent<T>` 会登记订阅，视图禁用时统一 `Suspend`、启用时 `Resume`，销毁归还内存池时 `Clear` 并释放全部句柄：

```csharp
// 在 UIBase 派生类（窗口或 Widget）里
protected override void OnInitialize()
{
    EventProxy.AddUIEvent<PlayerDataChangedEvent>(OnPlayerDataChanged);
}

private void OnPlayerDataChanged(in PlayerDataChangedEvent evt) { /* 刷新货币显示 */ }
```

详见 [UI 生命周期](../ui/lifecycle.md)。非 UI 的对象请自己在 `OnDisable`/`OnDestroy` 里 `Dispose`。

## 编辑器：Event Monitor

菜单 `AlicizaX/Event Monitor` 打开"事件监视器"（`EventMonitorWindow`），仅编辑器可用：

- 左侧事件列表：订阅数、派发模式（无参 / in）、容量、发布次数、是否已初始化；支持搜索与"仅已初始化"过滤。
- 右侧详情：订阅数、峰值、容量利用率、发布/订阅/取消/扩容/清空/回调异常计数、最后操作帧与时间。
- 订阅者列表：声明类型、方法名、槽位、版本、目标类型、静态/实例；已销毁的 Unity 目标与 lambda/闭包会单独告警，可"定位目标"。
- 告警与"快照对比"：拍摄快照后对比订阅数、发布次数、容量与订阅者集合的变化。
- 最近操作：固定 128 条环形记录（订阅/取消/扩容/清空/回调异常）。

统计只在窗口打开期间记录（`EventDebugRegistry.BeginRecording`/`EndRecording`），关掉窗口就没有开销。

## 常见配置

事件系统没有 Inspector 字段，可调的只有容量与清理：

| 项 | 默认值 | 说明 |
| --- | --- | --- |
| 初始容量 | 4 | 首次 `Subscribe` 时分配 |
| 扩容策略 | ×2 | 数组满时翻倍，旧句柄保持有效 |
| 预分配 | — | `ReserveEmptyCapacity<T>` / `ReservePayloadCapacity<T>` |
| 操作历史 | 128 条 | 仅编辑器，环形覆盖 |
| 统计记录 | 关闭 | 打开 Event Monitor 才开启 |

## 容易踩的坑

1. 事件类型必须是 `struct` 且只实现一种标记接口：泛型约束 `where T : struct, IEmptyEventArgs` 会挡下 class，两个接口都实现则重载解析失败。
2. 重复订阅会返回 `default` 句柄，对它 `Dispose` 什么都不会发生——看起来取消了，实际订阅还在，很容易变成"回调执行两次"。
3. 派发过程中新订阅的监听收不到当前这一轮，只会在下一次（或同类型嵌套 `Publish`）收到；不要在 `Publish` 前后假设订阅立即生效。
4. 销毁 `GameObject` 不会自动退订。长期存活的静态字段里挂着的句柄必须显式 `Dispose`，否则每次重建都会多一份回调。
5. `Publish` 是同步调用，订阅者里的加载、循环、等待全部压在发布者的栈上；重活请转成异步或在下一帧处理。
6. 关闭 Domain Reload 进入 Play 时，`SubsystemRegistration` 阶段会清空所有频道；跨场景静态缓存里的句柄虽然 `Dispose` 安全，但已经收不到事件，需要在初始化时重新订阅。

## API 速查

```csharp
// 订阅，返回可 Dispose 的句柄
EventRuntimeHandle Subscribe<T>(Action handler) where T : struct, IEmptyEventArgs;
EventRuntimeHandle Subscribe<T>(InEventHandler<T> handler) where T : struct, IPayloadEventArgs;

// 派发
void Publish<T>() where T : struct, IEmptyEventArgs;
void Publish<T>(in T payload) where T : struct, IPayloadEventArgs;

// 计数与容量
int GetEmptySubscriberCount<T>() where T : struct, IEmptyEventArgs;
int GetPayloadSubscriberCount<T>() where T : struct, IPayloadEventArgs;
void ReserveEmptyCapacity<T>(int capacity) where T : struct, IEmptyEventArgs;
void ReservePayloadCapacity<T>(int capacity) where T : struct, IPayloadEventArgs;

// 清理
void ClearEmpty<T>() where T : struct, IEmptyEventArgs;
void ClearPayload<T>() where T : struct, IPayloadEventArgs;

// 句柄
void EventRuntimeHandle.Dispose();

// UI 视图内（UIBase.EventProxy）
void EventListenerProxy.AddUIEvent<T>(Action handler) where T : struct, IEmptyEventArgs;
void EventListenerProxy.AddUIEvent<T>(InEventHandler<T> handler) where T : struct, IPayloadEventArgs;
```

性能说明：容量足够且委托被复用的情况下，稳定路径的订阅、派发、退订不产生托管分配（框架性能用例按 0 B 断言）；首次访问事件类型、扩容、重复订阅告警与异常日志会分配。

相关页面：[UI 生命周期](../ui/lifecycle.md)、[Service 服务容器](../service/index.md)。
