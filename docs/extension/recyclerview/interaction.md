# 数据更新与交互

`UGList` / `UGMixedList` / `UGLoopList` / `UGGroupList` 已直接暴露增删改接口，业务层优先走这些方法，不要再绕到 `Adapter`。

```csharp
_items.Add(new BagItemData
{
    ItemId = 1003,
    Name = "Gem",
    Count = 3,
});

_items.RemoveAt(0);
_items.RefreshItem(1);
_items.RefreshItem(1, relayout: true);
_items.SetData(newList, ListScrollPolicy.KeepAnchor);
```

常用策略：

| 场景 | 推荐 API |
| --- | --- |
| 替换整份数据 | `_items.SetData(newList)` 或 `_items.Data = newList` |
| 替换时保持当前可见位置 | `_items.SetData(newList, ListScrollPolicy.KeepAnchor)` |
| 替换时滚回起点 | `_items.SetData(newList, ListScrollPolicy.ResetScroll)` |
| 可见项内容变化且尺寸不变 | `_items.RefreshItem(index)` |
| 可见项尺寸或模板变化 | `_items.RefreshItem(index, relayout: true)` |
| 添加单项 | `_items.Add(data)` |
| 批量添加 | `_items.AddRange(items)` |
| 插入单项 | `_items.Insert(index, data)` |
| 顶部插入历史 | `_items.InsertRange(0, older)` |
| 删除单项 | `_items.RemoveAt(index)` |
| 清空列表 | `_items.Clear()` |

`ListRefreshMode.Immediate` 是默认刷新。聊天列表用 `AfterSettle`：拖拽或惯性未停稳时先记脏，停稳后再刷。

## 选择、滚动和事件

`ChoiceIndex` 是业务选择索引。它只会在显式设置、`SetSelect()` 或分组子项 `Activate(index)` 时变化。

```csharp
_items.ChoiceIndex = 3;
_items.ClearChoice();

_items.ScrollToIndex(10);
_items.ScrollToStart(10, offset: 0f, smooth: true);
_items.ScrollToCenter(10, offset: 0f, smooth: true, duration: 0.3f);
_items.ScrollToEnd(10);
_items.ScrollTo(10, ScrollAlignment.Center, offset: 0f, smooth: true);
_items.ScrollToChoice(ScrollAlignment.Center, smooth: true);
```

事件：

```csharp
_items.OnChoiceIndexChanged += index => { };
_items.ScrollValueChanged += position => { };
_items.ScrollStopped += () => { };
_items.ScrollDraggingChanged += dragging => { };
```

`ChoiceIndex`、`ScrollDataIndex`、`FocusedDataIndex` 是三套索引。业务选择、当前滚动项、导航焦点不要混用。未接导航包时 `FocusedDataIndex` 为 `-1`。
