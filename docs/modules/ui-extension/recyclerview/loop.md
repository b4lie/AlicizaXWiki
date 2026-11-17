# 循环列表

循环列表由 `UGLoopList<T>` + `LoopAdapter<T>` 组成：适配器把真实数据扩展成一段有限的虚拟地址空间，滑到边界时静默跳到中段等价位置，视觉上可以一直绕。它适合轮播、循环选择器和短列表的“无限”滑动，但**不是**无界数据源。

源码位置：`Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView/Adapter/LoopAdapter.cs`、`UGList.cs`、`RecyclerView.cs`。

## 适用场景

- 横向或纵向轮播：少量卡片循环展示，用户可一直向同一方向滑。
- 循环选择器：例如日期、头像、奖励预览，滑到末尾后自然接回开头。
- 跑马灯式列表：视觉需要连续绕回，但真实数据量固定。
- 不适合：服务端分页的无限信息流（数据量会持续增长，应使用普通列表 + 追加）；混合模板或环形布局。

## 使用前提

- Prefab 结构与 [普通列表](list.md) 相同：`RecyclerView` + `Scroller` + `Content` + 一个 `ViewHolder` 模板。
- `Layout Type` 推荐 `AlicizaX.UI.LinearLayoutManager`；`LoopAdapter` **不能**配 `MixedLayoutManager` 或 `CircleLayoutManager`，构造 `UGLoopList` 时会直接抛异常。
- 模板建议定高/定宽；轮播通常只用一个模板，所有项共用同一尺寸。
- 数据仍实现 `ISimpleViewData`；循环只改变展示索引，不改变业务数据接口。
- 框架不提供自动播放计时器；自动轮播需要业务自己定时调用 `ScrollToIndex` / `ScrollToCenter`。

## 最短接入

1. 做一个普通列表 Prefab，`Layout Type` 选 `AlicizaX.UI.LinearLayoutManager`；轮播就设 `Direction = Horizontal`，需要吸附时打开 `Snap`。
2. `Content` 和 `Item Templates[0]` 按普通列表配好；模板尺寸固定。
3. 窗口首次打开时创建循环列表，并灌入真实数据：

```csharp
using System.Collections.Generic;
using AlicizaX.UI;
using UnityEngine;

// UIWindow<...> 内
private UGLoopList<RvTextData> _loop;

protected override void OnOpen()
{
    _loop ??= UGListCreateHelper.CreateLoop<RvTextData>(baseui.ScrollViewLoop);

    var items = new List<RvTextData>();
    for (int i = 0; i < 12; i++)
    {
        items.Add(new RvTextData { Title = "Loop " + i, Color = Color.white });
    }

    _loop.SetData(items);
    _loop.ScrollToCenter(0, 0f, true);
}
```

4. 业务跳转始终传真实索引（`0..DataCount-1`），不要自己乘圈数：

```csharp
_loop.ScrollToIndex(3, true);
_loop.ScrollToCenter(0, 0f, true);
```

## 核心用法

### 虚拟数据量与“无限”规则

`LoopAdapter.GetRealCount()` 返回真实数据量，`GetItemCount()` 返回布局使用的虚拟项数。`LoopVirtualRange.ComputeVirtualItemCount` 的规则是：

- 虚拟项数 = `realCount × cycles`，绑定仍按 `layoutIndex % realCount` 取真实数据。
- `cycles` 最多 21，且尽量取奇数，便于把中段作为重锚点。
- 圈数还受 `MaxVirtualItemCount = 50000` 约束；可容纳的最大圈数不足 3 时，结果退化为 `realCount`，不再循环。
- 12 条真实数据会得到 21 圈、共 252 个虚拟项；8 条会得到 168 个。
- 数据量为 0 时虚拟项数也是 0；只有 1 条时虽然会扩圈，但通常内容不足一屏，能否滚动取决于模板尺寸。

所以“无限滚动”只是在**有限虚拟窗口**内循环；超过窗口边界时由框架重锚，不是无限生成数据。

### 重锚

`RecyclerView` 在滚动回调里检查当前布局索引；当它进入虚拟空间边缘的一圈时，会把滚动位置平移到中段的等价索引，同时保留项内偏移。因为绑定取模，用户看到的还是同一批数据。

`UGLoopList` 在以下入口也会主动选择离当前位置最近的等价索引：

- `ScrollToIndex(realIndex)`：把真实索引解析成当前附近的虚拟索引；
- `ScrollTo(realIndex, alignment, ...)`：同上，再按对齐方式计算位置；
- `Data` setter：换一批数据后回到新数据的中间锚点。

### 数据变更

- 追加/插入/删除沿用 `Add`、`Insert`、`RemoveAt` 等普通列表方法；虚拟项数会随真实数量重新计算。
- 替换整批数据时，优先用 `_loop.Data = newList`，它会在 `SetData` 后重新建立循环锚点；直接 `SetData(newList)` 也会刷新，但不会走这个显式重锚逻辑。
- 循环列表里的 `ScrollDataIndex` 已经被归一到真实索引（`0..realCount-1`），不是虚拟布局索引。
- `TryGetVisibleRange` 返回的也是真实数据索引；在接缝附近它可能看起来从末尾跳回开头，这是正常的。

### 与导航、交互配合

`LoopAdapter` 绑定 Holder 时会把取模后的**真实索引**传给 `ViewHolder.Bind`，所以：

- `ViewHolder.DataIndex`、`CurrentIndex` 是真实索引；
- `SetSelect()` 提交的 `ChoiceIndex` 也是真实索引；
- 焦点导航的 `FocusedDataIndex`、`SyncChoiceToFocus()` 同样按真实索引工作。

模板可以照常继承 `NavigationViewHolder<T>` 或 `InteractiveViewHolder<T>`，不需要为循环写特殊逻辑。焦点和点击的完整约定见 [导航](navigation.md) 与 [数据更新与交互](interaction.md)。

### 限制

- 没有自动轮播 API；不调用滚动方法时列表不会自己动。
- 不能配 `MixedLayoutManager` / `CircleLayoutManager`；需要环形布局请直接用普通列表 + `CircleLayoutManager`，而不是循环列表。
- 真实数据量过大时不扩圈，行为退化为普通列表；做“无限加载”应使用普通列表并在接近末尾时追加数据。
- `IsAtEnd` 在循环列表里只表示虚拟滚动位置到达边界，不代表业务数据结束，不要用它判断“加载完毕”。

## 常见配置

| Inspector 字段 | 源码字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Layout Type | `layoutManager` | None | 选 `LinearLayoutManager` |
| Direction | `direction` | Vertical | 轮播通常改 `Horizontal` |
| Content | `content` | 空 | 列表项父节点，必须显式指定 |
| Item Templates | `templates` | 空 | 建议只配一个定高模板 |
| Spacing | `spacing` | (0, 0) | 项间距；轮播常用 `x` |
| Padding | `padding` | (0, 0) | 内容留白 |
| Snap | `snap` | false | 轮播可打开，停止后吸附到最近项 |
| Snap Duration | `snapDuration` | 0.12 | 吸附动画时长 |
| Scroll Mode | `scroll` | Always Enable | 设为 `Always Disable` 时 `ScrollToIndex` / `ScrollTo` 会被拒绝 |
| Refresh Mode | `refreshMode` | Immediate | 数据频繁变化时可设 `AfterSettle` |
| Scroller Type | `scroller` | None | 同节点上的 Scroller 组件 |

## 容易踩的坑

- 把虚拟索引或布局索引传给 `GetItem`、`ChoiceIndex`：这些入口只接受真实索引，循环列表不会帮你反算。
- 以为 `IsAtEnd` 是“数据到底了”：它只反映虚拟滚动窗口的末尾，循环列表没有业务意义上的终点。
- 数据量很大（例如 3 万条）时虚拟窗口不会扩圈，列表行为和普通列表一样；循环只对较小数据量有意义。
- `UGLoopList` 第二次创建会撞上“一个 RecyclerView 只能绑一个适配器”的异常；窗口缓存复用时只创建一次。
- 直接改 `Data` 指向的 `List` 而不调用 `NotifyDataChanged`，虚拟项数和布局不会更新。
- 用 `PageLayoutManager` 或 `GridLayoutManager` 做循环不在示例路径内；需要翻页/网格时优先用普通列表，循环场景保持 Linear。

## API 速查

| 成员 | 用途 |
| --- | --- |
| `UGListCreateHelper.CreateLoop<T>` | 创建循环列表并绑定 `LoopAdapter<T>` |
| `SetData(list, policy)` / `Data` | 替换真实数据；`Data` setter 会重锚 |
| `Add` / `AddRange` / `Insert` / `InsertRange` | 追加或插入真实数据 |
| `ScrollToIndex(index, smooth)` | 按真实索引滚动 |
| `ScrollTo` / `ScrollToCenter` / `ScrollToStart` / `ScrollToEnd` | 按对齐方式滚动 |
| `ScrollToChoice` | 滚到当前选中项 |
| `ScrollDataIndex` | 当前滚动位置的真实索引 |
| `ChoiceIndex` | 当前选中项的真实索引 |
| `GetItem(index)` | 按真实索引取数据 |
| `TryGetVisibleRange` | 当前可见的真实数据区间 |
| `IsAtEnd` | 仅表示虚拟滚动到边界，不用于业务结束判断 |

数据接口、选中态与局部刷新的通用规则见 [普通列表](list.md) 和 [配置与 API](config.md)。
