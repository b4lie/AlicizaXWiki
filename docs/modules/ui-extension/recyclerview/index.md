# RecyclerView 列表

RecyclerView 是 `com.alicizax.unity.ui.extension` 中的虚拟滚动列表：只创建视口附近的列表项，由 `Adapter` 绑定数据、`LayoutManager` 计算位置、`Scroller` 处理拖拽与惯性。业务层通常使用 `UGList`、`UGMixedList`、`UGLoopList`、`UGGroupList`，不需要直接操作内部的 `ViewProvider`。

源码位置：

- 运行时代码：`Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView`
- 编辑器：`Client/Packages/com.alicizax.unity.ui.extension/Editor/RecyclerView/RecyclerViewEditor.cs`
- 真实示例：`Client/Assets/Scripts/Hotfix/GameLogic/UI/RecyclerViewTest/UIRecyclerViewExampleWindow.cs`

## 适用场景

- 数据量大、需要回收复用的列表，例如背包、任务、聊天、排行榜。
- 同一份数据需要多种列表形态：定高/变高、网格、分页、环形、分组、循环。
- 需要按索引定位、平滑滚动、吸附、局部刷新、选中态，而不是自己管理 `ScrollRect` 的内容节点。
- 不适合：只有几个静态子节点、且不需要数据驱动的界面；这种界面直接用 Unity `ScrollRect` + `LayoutGroup` 更简单。

与 Unity `ScrollRect` 的分工：

| 维度 | Unity ScrollRect | RecyclerView |
| --- | --- | --- |
| 列表项 | 内容节点下通常全部存在 | 只创建可见范围及少量缓冲，回收复用 |
| 布局 | 依赖 `LayoutGroup` / 手写位置 | `LayoutManager` 决定线性、网格、分页、环形等 |
| 数据 | 业务自己增删子节点 | `Adapter` 根据数据绑定 `ViewHolder` |
| 扩展 | 分组、循环、混合模板要自己做 | 已有 `UGMixedList` / `UGGroupList` / `UGLoopList` |

**同一层级不要同时挂 `ScrollRect` 和 `RecyclerView`**：两者都会响应拖拽和滚轮，输入会互相抢。

## 使用前提

- 项目包含 `com.alicizax.unity.ui.extension`（当前包版本 `4.0.2`，最低 Unity 2022.3）。
- 滚动区域根节点挂 `RecyclerView` 与 `Scroller`；`RecyclerView` 所在节点就是视口。
- 必须显式配置 `Content`（列表项父节点）和至少一个 `ViewHolder` 模板；当前源码不会用第一个子节点自动兜底。
- 必须选择 `Layout Type`。创建 `UGList` 时适配器会立即绑定，缺少布局管理器会直接抛异常。
- 若要显示滚动条，滚动条节点还要挂 `ScrollbarEx`；`Scrollbar` 组件本身不够。

## 最短接入

1. 建根节点：在滚动区域根节点挂 `RecyclerView`，再通过 Inspector 的 `Scroller Type` 添加 `AlicizaX.UI.Scroller`；不要加 `ScrollRect`。需要裁剪时，可在根节点或 Content 上加 `RectMask2D`。
2. 建 Content：在根节点下建一个空 `RectTransform`，铺满视口；把 `Content` 字段指向它。列表项会在运行时实例化到它下面。
3. 建模板：做一个 Prefab，根节点挂 `ViewHolder<T>` 子类，设置固定主轴尺寸；拖到 **Item Templates**，顺序下标就是模板 ID。
4. 配布局：在 `Layout Type` 选择 `AlicizaX.UI.LinearLayoutManager`，设置 `Direction`、`Spacing`、`Padding`；需要滚动条时设置 `Scrollbar Visibility`。
5. 写窗口：在首次打开时创建 `UGList`，之后只更新数据。

```csharp
using System.Collections.Generic;
using AlicizaX.UI;
using GameLogic.UI;

// UIWindow<...> 内；baseui.ScrollViewLinear 是示例窗口的生成字段，实际换成你的字段。
private UGList<RvTextData> _list;

protected override void OnOpen()
{
    _list ??= UGListCreateHelper.Create<RvTextData>(baseui.ScrollViewLinear);
    _list.SetData(new List<RvTextData>
    {
        new() { Title = "Item 1", Color = UnityEngine.Color.white },
        new() { Title = "Item 2", Color = UnityEngine.Color.gray }
    });
}
```

## 核心用法

### 列表类型总表

| 业务封装 | 适配器 | 布局管理器 | 数据接口 | 典型用途 |
| --- | --- | --- | --- | --- |
| `UGList<T>` | `Adapter<T>` | `LinearLayoutManager` / `GridLayoutManager` / `PageLayoutManager` / `CircleLayoutManager` | `ISimpleViewData` | 普通列表、网格、翻页、环形 |
| `UGMixedList<T>` | `MixedAdapter<T>` | `MixedLayoutManager`（强制） | `IMixedViewData` | 多种高度/多种模板混排 |
| `UGLoopList<T>` | `LoopAdapter<T>` | 推荐 `LinearLayoutManager`；不能配 `MixedLayoutManager` / `CircleLayoutManager` | `ISimpleViewData` | 轮播、循环选择、跑马灯 |
| `UGGroupList<T>` | `GroupAdapter<T>` | 示例用 `MixedLayoutManager`；模板同高时也可用 Linear | `IGroupViewData` | 可展开/折叠的分组列表 |

`UGListBase.ValidateAdapterLayoutCompatibility` 会在创建封装时校验组合：`MixedAdapter` 只能配 `MixedLayoutManager`；`LoopAdapter` 不能配 `MixedLayoutManager` 或 `CircleLayoutManager`；变长项只支持 `LinearLayoutManager` / `MixedLayoutManager`。

### 基本结构

```text
ScrollView (RecyclerView + Scroller)
├─ Content (RectTransform，运行时列表项的父节点)
│  ├─ RvTextItem(Clone)
│  └─ ...
└─ Scrollbar (可选，必须带 ScrollbarEx)
```

```text
RvTextItem.prefab (RectTransform + RvTextItemViewHolder)
-> RecyclerView / Item Templates[0]
```

`Templates` 不是强制放在场景里的子节点，可以直接引用外部 Prefab；运行时会隐藏模板自身，再用它实例化可见项。

### 数据接口

| 接口 | 作用 | 需要实现的成员 |
| --- | --- | --- |
| `ISimpleViewData` | 普通列表的最小标记接口 | 无 |
| `IMeasuredViewData` | 为变长列表声明主轴长度 | `bool TryGetItemLength(out float length)` |
| `IMixedViewData` | 指定该项使用哪套模板 | `int TemplateId { get; set; }` |
| `IGroupViewData` | 分组列表的组信息 | `Expanded`、`Type`、`TemplateId` |

`TemplateId` 是 `Item Templates` 数组下标，不是模板名；当前源码没有按名称匹配模板的机制。

### ViewHolder 如何创建

业务只需要写 `ViewHolder<TData>` 子类并拖进 `Item Templates`。运行时流程是内部的：

```text
Adapter.GetTemplateId(dataIndex)
  -> ViewProvider 按模板 ID 从池中取/创建可见项
  -> ViewHolderFactory 在 Content 下 Instantiate 模板
  -> Adapter.OnBindViewHolder -> ViewHolder<TData>.OnBind
  -> 变长布局按需测量并回写长度
```

`ViewProvider`、`ViewHolderFactory` 都是 `internal`，没有公开的 Holder 工厂注册入口；不要自己 `Instantiate` 列表项，也不要在运行时改 `Templates`。需要多模板时，改的是模板数组和数据的 `TemplateId`，不是注册新工厂。

### 页面导航

| 页面 | 内容 |
| --- | --- |
| [普通列表](list.md) | `UGList` 的 Prefab 配置、增删改、滚动、变高测量 |
| [多模板列表](mixed.md) | `UGMixedList`、`TemplateId`、多套模板的高度与测量 |
| [循环列表](loop.md) | `UGLoopList` 的虚拟地址空间、重锚与限制 |
| [分组列表](group.md) | `UGGroupList`、组头、展开/折叠 |
| [数据更新与交互](interaction.md) | 局部刷新、选中、点击与滚动事件 |
| [配置与 API](config.md) | 完整 Inspector 字段与 API 表 |
| [导航](navigation.md) | 手柄/键盘焦点导航 |

## 常见配置

| Inspector 字段 | 源码字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Layout Type | `layoutManager` | None | 布局管理器；运行期不可换 |
| Direction | `direction` | Vertical | 主轴方向 |
| Content | `content` | 空 | 列表项父节点，必须显式指定 |
| Item Templates | `templates` | 空（null） | 模板列表，下标即 TemplateId |
| Spacing | `spacing` | (0, 0) | 列表项间距；主轴取对应分量 |
| Padding | `padding` | (0, 0) | 内容内边距；主轴取对应分量 |
| Scroll Mode | `scroll` | Always Enable | 是否允许滚动 |
| Refresh Mode | `refreshMode` | Immediate | `AfterSettle` 会推迟到滚动停止后刷新 |
| Snap | `snap` | false | 停止后是否吸附到最近项 |
| Scrollbar Visibility | `scrollbarVisibility` | Always Hide | 滚动条显示策略 |
| Scroller Type | `scroller` | None | 同节点上的 Scroller 组件 |

## 容易踩的坑

- `Content` 为空不会被自动修复；`RecyclerView` 会判定配置无效，列表不显示。
- 一个 `RecyclerView` 只允许绑定一个适配器。重复 `new UGList<T>(recyclerView)` 会抛 `InvalidOperationException`；窗口缓存复用时只创建一次。
- 模板主轴方向不能使用拉伸锚点（`anchorMin != anchorMax`），否则启动校验失败。
- 普通列表、网格、分页、环形都用 **模板 #0** 的统一尺寸计算格长；多套不同尺寸模板必须走 `UGMixedList`。
- `UGList` 没有公开的 `Refresh()` 或 `Clear()`。全量刷新用 `Adapter.NotifyDataChanged()`，清空用 `SetData(null, ListScrollPolicy.ResetScroll)`。
- `SetData` 默认 `KeepAnchor` 保留的是滚动像素，不是按数据身份锚定；换一批高度不同的数据时位置可能跳动。
- 变长项只在 `LinearLayoutManager` / `MixedLayoutManager` 打开 `Variable Item Length` 后生效；改高后还要 `RefreshItem(index, true)` 或 `RequestResize()`。

## API 速查

| 成员 | 用途 |
| --- | --- |
| `UGListCreateHelper.Create<T>` | 创建普通列表 |
| `CreateMixed<T>` / `CreateLoop<T>` / `CreateGroup<T>` | 创建多模板/循环/分组列表 |
| `SetData(list, policy)` / `Data` | 替换数据源并刷新 |
| `Add` / `AddRange` / `Insert` / `InsertRange` | 增量插入 |
| `ReplaceAt` / `RemoveAt` / `RemoveRange` / `RemoveAll` / `Move` / `Sort` | 单条或批量修改 |
| `RefreshItem(index, relayout)` | 重绑一项；尺寸变化时 `relayout: true` |
| `Adapter.NotifyDataChanged()` | 全量失效并重绑可见项 |
| `ScrollToIndex` / `ScrollTo` / `ScrollToCenter` / `StickToEnd` | 定位与平滑滚动 |
| `ChoiceIndex` / `ClearChoice()` / `OnChoiceIndexChanged` | 选中态 |
| `TryGetVisibleRange` / `TryGetViewHolder` | 查询可见数据 |

窗口生命周期和 `baseui` 字段的绑定方式见 [UI 模块](../../ui/index.md)；第一次接入工程的步骤见 [快速开始](../../../quickstart.md)。
