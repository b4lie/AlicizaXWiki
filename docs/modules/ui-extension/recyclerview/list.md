# 普通列表

普通列表由 `UGList<T>` 封装：一条业务数据对应一个 `ViewHolder<T>` 模板，`Adapter<T>` 负责增删改和绑定，`LinearLayoutManager` 负责逐行排列；需要网格、翻页或环形时换布局管理器即可。下面以仓库里的 `RvTextData` / `RvTextItemViewHolder` 为例。

源码位置：`Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView/UGList.cs`、`Adapter/Adapter.cs`、`Layout/LinearLayoutManager.cs`。

## 适用场景

- 背包、任务、邮件、排行榜、聊天记录等一维数据列表。
- 固定行高，或需要按内容测量的变高列表；需要两列/多列时换 `GridLayoutManager`。
- 列表项结构统一，不需要每项切换模板；需要混排请看 [多模板列表](mixed.md)。

## 使用前提

- 根节点挂 `RecyclerView` 和 `Scroller`，并且 `Layout Type` 已选择；普通列表常用 `AlicizaX.UI.LinearLayoutManager`。
- `Content` 已指向一个铺满视口的 `RectTransform`；`Item Templates` 至少有一项，模板根节点挂 `ViewHolder<T>` 子类。
- 模板主轴方向的锚点不要拉伸；垂直列表的模板应使用固定高度，宽度可以拉伸或固定。
- 同一 `RecyclerView` 只创建一个 `UGList`；窗口被缓存时不要在下一次 `OnOpen` 重新创建。

## 最短接入

1. 在滚动区域根节点挂 `RecyclerView`，用 Inspector 的 `Scroller Type` 添加 `AlicizaX.UI.Scroller`；不要加 `ScrollRect`。
2. 在根节点下建空 `RectTransform` 命名为 `Content`，四边铺满，拖到 `Content` 字段。
3. 在 `Layout Type` 选择 `AlicizaX.UI.LinearLayoutManager`；`Direction` 设为 `Vertical`，按需设置 `Spacing`、`Padding` 和 `Snap`。
4. 做一个带 `RvTextItemViewHolder` 的 Prefab，设置固定高度，拖进 `Item Templates` 的 `#0`。
5. 在窗口首次打开时创建列表并灌入数据：

```csharp
using System.Collections.Generic;
using AlicizaX.UI;
using UnityEngine;

// UIWindow<...> 内；baseui.ScrollViewLinear 是示例窗口的生成字段。
private UGList<RvTextData> _list;

protected override void OnOpen()
{
    _list ??= UGListCreateHelper.Create<RvTextData>(baseui.ScrollViewLinear);
    _list.SetData(new List<RvTextData>
    {
        new() { Title = "Item 1", Color = Color.white },
        new() { Title = "Item 2", Color = Color.gray }
    });
}
```

`UGListCreateHelper.Create` 内部会 `new Adapter<T>(recyclerView)` 并立即绑定；第二次创建会抛异常，所以窗口复用时要用 `_list != null` 或 `??=` 保护。

## 核心用法

### 数据类

普通列表的数据类只需要实现空标记接口 `ISimpleViewData`，字段由业务自己定义：

```csharp
using AlicizaX.UI;

namespace GameLogic.UI
{
    public sealed class RvTextData : ISimpleViewData
    {
        public string Title;
        public UnityEngine.Color Color;
    }
}
```

### ViewHolder 写法

模板根节点挂一个 `ViewHolder<TData>` 子类；`OnBind` 里只做数据到视图的赋值。点击并提交选中时，调用基类的 `SetSelect()`：

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;

namespace GameLogic.UI
{
    public sealed class RvTextItemViewHolder : ViewHolder<RvTextData>, IPointerClickHandler
    {
        [SerializeField] private Image background;
        [SerializeField] private TextMeshProUGUI title;

        protected override void OnBind(RvTextData data, int index)
        {
            title.text = data.Title;
            background.color = data.Color;
        }

        public void OnPointerClick(PointerEventData eventData)
        {
            SetSelect();
        }
    }
}
```

需要统一处理鼠标点击、导航提交和按压反馈时，继承 `InteractiveViewHolder<TData>` 并实现 `OnItemClick`，见 [数据更新与交互](interaction.md) 和 [导航](navigation.md)。

### 增删改

`UGList` 的方法会同步修改它持有的 `List<T>` 并通知列表：

```csharp
_list.Add(new RvTextData { Title = "new" });
_list.Insert(0, new RvTextData { Title = "top" });
_list.ReplaceAt(3, new RvTextData { Title = "replaced" });
_list.RemoveAt(0);
_list.RemoveAll(data => data.Title == "expired");
_list.Move(0, 2);
_list.Sort((a, b) => string.CompareOrdinal(a.Title, b.Title));
```

### 刷新与清空

- 换一整批数据：`SetData(list)`；默认 `ListScrollPolicy.KeepAnchor` 保留当前滚动像素。
- 回到顶部换数据：`SetData(list, ListScrollPolicy.ResetScroll)`。
- 单条内容变化：`RefreshItem(index)`；该项尺寸可能变化时传 `true`。
- 全量重绑但不换列表：`_list.Adapter.NotifyDataChanged()`。
- 清空：`_list.SetData(null, ListScrollPolicy.ResetScroll)`。

`UGList` 当前没有公开的 `Refresh()` / `Clear()`；换数据、重绑单项和全量重建分别用上面的 `SetData`、`RefreshItem`、`Adapter.NotifyDataChanged()`。

### 滚动与定位

```csharp
_list.ScrollToIndex(2, true);                          // 滚到索引 2，按起始对齐
_list.ScrollToCenter(2, 0f, true);                     // 居中
_list.StickToEnd(true);                                 // 贴到末尾（聊天）
_list.StopScroll();
```

`ScrollDataIndex` 是当前滚动位置对应的数据索引，`TryGetVisibleRange` 返回可见数据区间；空列表时索引可能是 `-1`，使用前先判空。

### 变高列表与测量

先给 `LinearLayoutManager` 打开 `Variable Item Length`，否则 `ViewHolder` 改高不会触发布局更新。数据可选实现 `IMeasuredViewData`，在绑定前给出一个预估长度：

```csharp
public sealed class RvChatData : IMeasuredViewData
{
    public float DeclaredLength;

    public bool TryGetItemLength(out float length)
    {
        length = DeclaredLength;
        return DeclaredLength > 0f;
    }
}
```

绑定后实际测量值优先。`OnBind` 里改完 Holder 根节点高度后调用 `RequestResize()`，让列表回写新长度：

```csharp
RectTransform root = (RectTransform)transform;
root.SetSizeWithCurrentAnchors(RectTransform.Axis.Vertical, bubbleHeight);
RequestResize();
```

## 常见配置

| Inspector 字段 | 源码字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Layout Type | `layoutManager` | None | 选 `LinearLayoutManager`；也可换 Grid/Page/Circle |
| Direction | `direction` | Vertical | 垂直或水平主轴 |
| Content | `content` | 空 | 列表项父节点，必须显式指定 |
| Item Templates | `templates` | 空 | 普通列表用 `#0` 的尺寸作为统一格长 |
| Spacing | `spacing` | (0, 0) | 垂直列表用 `y`，水平列表用 `x` |
| Padding | `padding` | (0, 0) | 内容四周留白，主轴/交叉轴都参与布局 |
| Scroll Mode | `scroll` | Always Enable | 设为 `Always Disable` 时 `ScrollToIndex` / `ScrollTo` 直接返回 |
| Refresh Mode | `refreshMode` | Immediate | 聊天可设 `AfterSettle` 避免滚动中重建 |
| Scroller Type | `scroller` | None | 同节点上的 Scroller 组件 |
| Variable Item Length | `variableItemLength` | false | 仅 Linear/Mixed 有效；变高列表必须打开 |

## 容易踩的坑

- 只配了 `RecyclerView` 没配 `Scroller`：列表能显示，但拖拽和滚轮不工作；Inspector 的 Setup 区会报 `Scroller is missing`。
- `Content` 为空不会被自动兜底；当前源码会判定配置无效，列表直接不显示。
- 模板在主轴方向使用拉伸锚点会被启动校验拒绝；给模板一个固定高度/宽度。
- 普通列表即使有多个模板，尺寸仍取模板 `#0`；要按模板混排请用 `UGMixedList`。
- 数据高度变了却只 `RefreshItem(index)` 不传 `relayout: true`，内容会重绑但位置和总高度不变。
- 未打开 `Variable Item Length` 时，`RequestResize()` 不会生效，列表仍按模板固定高度排版。
- `SetData` 的 `KeepAnchor` 保留滚动像素，不保证停在原来那条数据上；换数据要定位就显式 `ScrollToIndex` 或用 `ResetScroll`。

## API 速查

| 成员 | 用途 |
| --- | --- |
| `UGListCreateHelper.Create<T>` | 创建普通列表并绑定 `Adapter<T>` |
| `SetData(list, policy)` | 替换数据源；策略默认 `KeepAnchor` |
| `Data` / `DataCount` | 当前列表引用与数量 |
| `Add` / `AddRange` / `Insert` / `InsertRange` | 插入数据 |
| `ReplaceAt` / `RemoveAt` / `RemoveRange` / `RemoveAll` / `Move` / `Sort` | 修改数据 |
| `RefreshItem(index, relayout)` | 重绑单项，尺寸变化传 `true` |
| `Adapter.NotifyDataChanged()` | 全量失效并重绑可见项 |
| `ScrollToIndex` / `ScrollTo` / `ScrollToCenter` / `StickToEnd` / `StopScroll` | 滚动控制 |
| `ChoiceIndex` / `ClearChoice` / `OnChoiceIndexChanged` | 选中态 |
| `TryGetVisibleRange` / `TryGetViewHolder` | 查询可见项 |
