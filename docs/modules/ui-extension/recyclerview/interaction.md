# 数据更新与交互

这一页讲数据的增删改各自触发哪种刷新、`ListRefreshMode` 与 `ListScrollPolicy` 管什么、点击/长按/选中怎么接，以及滚动定位与回调。列表怎么搭见 [普通列表](list.md)，分组索引规则见 [分组列表](group.md)。

## 适用场景

- 列表要追加消息、插入历史、删除某项、改某条文本或高度。
- 要做点击、长按、单选高亮。
- 要滚到指定项、贴底、停滚，或按滚动状态做业务；想避免拖拽/惯性期间反复重建。

## 使用前提

- 已有 `UGList` / `UGMixedList` / `UGGroupList` / `UGLoopList` 实例。列表只能建一次（内部 `SetAdapter` 二次绑定会抛异常），后续都改数据。
- 数据实现 `ISimpleViewData`；变高还要 `IMeasuredViewData`，且布局管理器支持变长。
- 要点击的模板继承 `InteractiveViewHolder<TData>`；只做导航继承 `NavigationViewHolder<TData>`；普通 `ViewHolder<TData>` 不感知指针。

## 最短接入

1. 建列表并灌数据。
2. 小改动走增量 API，别整表重设。
3. 内容或高度变了，用 `RefreshItem(index, relayout: true)`。
4. 点击在模板里重写 `OnItemClick`。

```csharp
using AlicizaX.UI;

UGList<ItemData> list = UGListCreateHelper.Create<ItemData>(recyclerView);
list.SetData(items);            // 整表替换，默认保留滚动像素
list.Add(newItem);              // 尾部追加，增量
list.RefreshItem(0, true);      // 第 0 条内容/高度变了，需要重新测量
list.ChoiceIndex = 2;           // 业务选中（-1 清空）
list.OnChoiceIndexChanged += index => Debug.Log(index);
```

```csharp
public sealed class BagItemHolder : InteractiveViewHolder<BagItemData>
{
    protected override void OnBind(BagItemData data, int index) { /* 绑定 */ }

    // 鼠标左键点击与手柄提交都会走到这里
    protected override void OnItemClick(BagItemData data, int index) { /* 开详情弹窗 */ }
}
```

## 核心用法

### 数据更新：整表还是增量

| API | 索引空间 | 刷新方式 | 说明 |
| :--- | :--- | :--- | :--- |
| `SetData(List<TData>, ListScrollPolicy)` / `Data = ...` | — | 清池 + 全量重建 | 换整份数据；默认 `KeepAnchor` |
| `SetData(null)` / `Adapter.Clear()` | — | 全量重建为空 | `UGList` 没有 `Clear()` |
| `Add` / `AddRange` | 源列表 | 尾部插入增量 | 唯一能并入变长偏移表的路径 |
| `Insert` / `InsertRange` | 源列表 | 结构增量 + 锚点补偿 | 顶部插历史不会把视口顶走 |
| `RemoveAt` / `RemoveRange` / `RemoveAll` | 源列表 | 删除增量 / 整体重建 | `RemoveAll` 是散点删除，只发一次整体刷新 |
| `ReplaceAt(index, item, relayout)` | 源列表 | 单行重绑 | `relayout: true` 时重新测量 |
| `Move` / `Sort` | 源列表 | 整体重建 | 顺序变化无法用索引平移表达 |
| `RefreshItem(index, relayout)` | 显示索引 | 重绑 / 重新测量 | 改文本、改高度用 `true` |
| `Adapter.NotifyDataChanged()` | — | 全量重建 | 最重的一档，能不用就不用 |

三档粒度：`RefreshItem(i)` 只重绑可见行；`RefreshItem(i, true)` 重绑并在变长布局下按实测长度重算内容尺寸；`NotifyDataChanged` / `SetData` 连可见项一起重建。变高列表在 `OnBind` 里改完自身高度后调 `ViewHolder.RequestResize()` 也能写回长度——`RvChatItemViewHolder` 就是绑定后 `RequestResize()`，窗口侧再用 `RefreshItem(0, true)` 触发改高。

### `ListRefreshMode`：什么时候可以推迟刷新

| 值 | 行为 |
| :--- | :--- |
| `Immediate`（默认） | 每次数据变化立刻重建/重排 |
| `AfterSettle` | 拖拽或惯性期间只登记，落定后刷新一次 |

- 只在 `Scroller.IsDragging || IsMoving` 期间推迟，且只作用于走 `MarkDirty` 的路径：`SetData`、`Add/Insert/Remove` 等结构变化、`NotifyDataChanged`。
- `RefreshItem(i, true)` 不经过 `MarkDirty`，**当帧生效**，不会被推迟。
- 推迟期间 `ContentSize`、可见区间、`IsAtEnd` 都是旧值，别在滚动中拿它们判断。
- 补刷发生在 `LateUpdate`、`OnEnable` 与滚动落定（早于 `OnScrollStopped`）；`SetActive(false)` 再启用会强制补一次，`Tests/EditMode/RecyclerRefreshTests.cs` 就是验证这一点。

```csharp
chatList.ApplyUpdateOptions(new ListUpdateOptions
{
    ApplyRefreshMode = true,
    RefreshMode = ListRefreshMode.AfterSettle
});
```

`ListUpdateOptions` 还有 `Relayout`、`Scroll`、`ScrollPolicy`，`UGListBase.ApplyUpdateOptions` 只处理两件事：设 `RefreshMode`；当 `Scroll == true` 且 `ScrollPolicy == ResetScroll` 时回顶。其余字段不生效。

### `ListScrollPolicy`：换数据后停在哪

| 值 | 行为 |
| :--- | :--- |
| `KeepAnchor`（默认） | 保留当前滚动像素位置，越界时夹到新的 `MaxPosition` |
| `ResetScroll` | 位置归零、停掉当前运动、滚动条归零、`ScrollDataIndex` 变 `-1` |

`KeepAnchor` 保的是**像素**，不是「同一条数据」，换整表后同一像素通常对应别的项。要保住阅读位置：先记 `ScrollDataIndex`，`SetData` 之后再 `ScrollToIndex(anchor, false)`；或者干脆用 `InsertRange(0, older)`，锚点补偿会保住当前视口。

### 点击、长按、选中

| 回调 | 时机 | 默认行为 |
| :--- | :--- | :--- |
| `OnPointerDown` / `OnPointerUp` | 左键按下/抬起 | 记按下标记；抬起时未拖拽则把焦点转给该行 |
| `OnPointerClick` | 左键点击 | `TryActivate()`：`CanActivate` → 可能 `SetSelect` → 播放反馈 → `OnItemClick` |
| `OnPointerEnter` / `OnPointerExit` | 悬停进出 | 视觉切 `Highlighted` / `Normal` |
| `HandleNavigationSubmit` | 手柄提交 | 同样走 `TryActivate()`，且恒返回 `true` |

可重写项：`SelectOnActivate`（默认 `true`，点击/提交是否同时写业务选中）、`PlayInteractionClickFeedback`（默认 `true`）、`CanActivate(data, index)`（默认 `true`，每次视觉刷新都调，必须 O(1)）、`OnNavigationFocused(bool)`、`OnSelectionChange(bool)`。

未绑定的项不会激活，右键也不触发。`Interaction` 字段可选挂 `UXSelectable`/`UXButton`：挂上由 Holder 驱动外部视觉状态、原生点击被屏蔽，见 [UXButton](../uxbutton.md)。

**长按没有内置事件**，自己做「按下计时 + 抬起判定 + 吞掉随后的 click」：

```csharp
// 字段：[SerializeField] private float longPressSeconds = 0.5f;
//       private float pressTime; private bool longPressed;
public override void OnPointerDown(PointerEventData e)
{
    base.OnPointerDown(e);                 // 保留按下视觉与 pointerPress 认领
    pressTime = Time.unscaledTime;
    longPressed = false;
}

public override void OnPointerUp(PointerEventData e)
{
    longPressed = !longPressed && e.eligibleForClick &&
                  Time.unscaledTime - pressTime >= longPressSeconds;
    if (longPressed && CurrentData != null && CanActivate(CurrentData, CurrentIndex))
    {
        OnLongPress(CurrentData, CurrentIndex);
    }

    base.OnPointerUp(e);
}

public override void OnPointerClick(PointerEventData e)
{
    if (longPressed) { longPressed = false; return; }   // 长按不再触发短按
    base.OnPointerClick(e);
}
```

业务选中与导航焦点是两套状态：`ChoiceIndex` / `HasChoice` / `ClearChoice()` / `OnChoiceIndexChanged` 管选中，`SetSelect()` 把当前行提交为选中，`SyncChoiceToScroll()` / `SyncChoiceToFocus()` 把选中同步到停下的位置或导航焦点，`ScrollToChoice(alignment = Center, ...)` 滚到选中项。`Adapter.SetChoiceIndex` 会把索引夹到 `0..GetRealCount()-1`；只有可见行收到 `OnSelectionChange`，滚回来靠绑定时的 `ApplySelection` 重画。

### 滚动定位

| API | 对齐 | 说明 |
| :--- | :--- | :--- |
| `ScrollToIndex(index, smooth)` | 顶部 | 最常用；`smooth` 时长由距离与 `ScrollSpeed` 推算 |
| `ScrollTo(index, ScrollAlignment, offset, smooth, duration)` | 指定 | `UGList` 层入口，圆形布局抛 `NotSupportedException` |
| `ScrollToStart/Center/End(index, offset, smooth, duration)` | 固定 | 上一行的快捷写法 |
| `ScrollToAligned(index, alignment, offset, smooth, duration)` | 指定 | `RecyclerView` 层入口 |
| `StickToEnd(smooth)` / `StopScroll()` | 底部 / — | 贴底；停滚原因是 `Interrupted` |

- 定位不受可见性限制：目标在屏内也会滚。`smooth: false` 立即跳并重建可见项。
- `ScrollAlignment` 只有 `Start` / `Center` / `End`；`offset` 是叠加在对齐结果上的像素偏移，最终被 `ClampPosition` 夹进合法范围。
- 想「移动到某项」传数据索引即可；没有按像素移动的接口，也不该自己算像素。

### 滚动回调与原因

| 事件 | 参数 | 时机 |
| :--- | :--- | :--- |
| `OnScrollValueChanged` | `float position` | 每帧位置变化（含立即跳） |
| `OnScrollStopped` | `ScrollStopReason` | 运动结束；已停止时再 `StopScroll` 不触发 |
| `OnScrollDraggingChanged` | `bool dragging` | 拖拽开始/结束 |
| `OnItemResized` | `int dataIndex, float length` | 变高测量结果写回后 |

`ScrollStopReason`：`Completed`（运动自然结束）、`Interrupted`（新运动抢占、`StopScroll`、开始拖拽）、`Disabled`（`Scroller` 被禁用）、`Reset`（`Reset(ResetScroll)` 内部复位）。`Scroller.MotionKind`（`None`/`Smooth`/`Duration`/`Inertia`/`Wheel`）可查当前运动类型。开 `Snap` 时只有 `Completed` 触发吸附，吸附落定后才发 `OnScrollStopped(Completed)`。

## 常见配置

| 项目 | 建议 | 说明 |
| :--- | :--- | :--- |
| `Refresh Mode` | 聊天/日志流 `AfterSettle`，其余 `Immediate` | 推迟重建，落定后补一次 |
| `Inertia` / `Deceleration Rate` | `true` / `0.135` | 关惯性后松手立即停 |
| `Snap` / `Snap Duration` | 普通列表关，翻页列表开 / `0.12` | 吸附只对 `Completed` 生效 |
| `Movement Type` | `Elastic` | 要硬边界（贴底聊天）用 `Clamped` |

## 容易踩的坑

1. **变高列表改高只调 `RefreshItem(i)` 不生效**：`relayout: false` 只重绑不测量，必须 `true`（或 `OnBind` 里 `RequestResize()`）。
2. **`SetData` 会清视图池**：所有可见项重建，`KeepAnchor` 只保像素；频繁小改动用 `Add`/`Insert`/`ReplaceAt`。
3. **`AfterSettle` 期间读到旧几何**：内容高度、可见区间、`IsAtEnd` 都可能滞后；要实时值就设 `Immediate`，且别指望它推迟 `RefreshItem(i, true)`。
4. **已停止时 `StopScroll()` 没有回调**：`StopMovement` 在 Idle 直接返回。
5. **开 `Snap` 后 `OnScrollStopped(Completed)` 晚一拍**：先吸附再发事件，依赖「停稳」的 UI 要容忍延迟。
6. **圆形布局不能用对齐滚动**：`ScrollTo`/`ScrollToStart`/`ScrollToCenter`/`ScrollToEnd` 抛 `NotSupportedException`，改用 `ScrollToIndex`。
7. **未绑定项不激活但仍收到指针事件**：`CurrentData == null` 时 `TryActivate` 返回 `false`，自定义 `OnPointerDown`/长按照样进，里面要自己判空（绑定失败时 `CurrentIndex` 是 `-1`）。
8. **`CanActivate` 每次视觉刷新都调**：里面查表、`GetComponent`、分配字符串都会拖慢滚动。
9. **选中索引会被夹**：`SetChoiceIndex` 超出 `GetRealCount()` 会夹到末项，`-1` 才是「无选中」。
10. **不要自己订阅 `Scroller.OnValueChanged` 再改数据**：每个滚动帧都会触发一次全量布局，容易递归刷新。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `SetData(List<TData>, ListScrollPolicy = KeepAnchor)` | 整表替换 |
| `Add` / `AddRange` / `Insert` / `InsertRange` | 插入 |
| `RemoveAt` / `RemoveRange` / `RemoveAll` / `ReplaceAt` / `Move` / `Sort` | 删除、替换、重排 |
| `RefreshItem(int index, bool relayout = false)` | 单行刷新 |
| `ApplyUpdateOptions(in ListUpdateOptions)` | 刷新模式 / 可选回顶 |
| `ChoiceIndex` / `ClearChoice` / `OnChoiceIndexChanged` | 选中 |
| `ScrollToIndex` / `ScrollTo` / `StickToEnd` / `StopScroll` | 定位 |
| `ScrollPosition` / `ScrollDataIndex` / `IsAtEnd` / `IsScrolling` / `TryGetVisibleRange` | 状态查询 |
| `RecyclerView.TrimInactivePool()` | 裁剪空闲池，配置见 [配置与 API](config.md) |
