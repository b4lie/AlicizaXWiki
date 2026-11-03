# RecyclerView 列表


`RecyclerView` 是 `com.alicizax.unity.ui.extension` 中的虚拟滚动列表组件，负责列表项复用、滚动、布局、吸附、滚动条和局部刷新。业务层通常通过 `UGList`、`UGMixedList`、`UGLoopList`、`UGGroupList` 使用它，而不是直接操作内部 `IAdapter`。

对应包版本：`com.alicizax.unity.ui.extension` `2.1.11`。

源码位置：

```text
Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView
```

## 列表类型

按用途拆成独立页面，每页都带可运行示例：

| 页面 | 适用场景 |
| :--- | :--- |
| [普通列表 UGList](list.md) | 定高列表、变高列表、聊天/消息流 |
| [混合模板 UGMixedList](mixed.md) | 一行多种模板混排（商品流、动态卡片） |
| [循环列表 UGLoopList](loop.md) | 轮播、无限滚动、环形选择 |
| [分组列表 UGGroupList](group.md) | 带表头、可折叠分组 |
| [数据更新与交互](interaction.md) | 增量刷新、选中、滚动、点击事件 |
| [配置与 API](config.md) | Inspector 配置项、接口速查 |
| [导航](navigation.md) | 手柄/键盘焦点导航（依赖 input 扩展包） |

## 基本结构

一个可用的 `RecyclerView` 需要以下配置：

| 配置 | 说明 |
| --- | --- |
| `RecyclerView` | 挂在滚动区域根节点上 |
| `Content` | 承载运行时列表项实例的 `RectTransform`；未配置时会尝试使用第一个子节点 |
| `Templates` | 一个或多个列表项模板；模板根节点必须挂 `ViewHolder` 子类 |
| `LayoutManager` | 布局管理器，例如线性、网格、分页、混合尺寸、圆形布局 |
| `Scroller` | 处理拖拽、滚轮、惯性、回弹和平滑滚动 |
| `Scrollbar` | 可选滚动条组件 |

运行时模板会被隐藏，实例由对象池创建和回收。不要直接把 `Templates` 中的模板对象当作真实显示项操作。

## 数据接口

普通列表数据只需要实现空接口 `ISimpleViewData`：

```csharp
public interface ISimpleViewData
{
}
```

多模板和分组列表使用模板下标，而不是模板名称：

```csharp
public interface IMixedViewData : ISimpleViewData
{
    int TemplateId { get; set; }
}

public interface IGroupViewData : IMixedViewData
{
    bool Expanded { get; set; }
    int Type { get; set; }
}
```

`TemplateId` 对应 `RecyclerView.Templates` 数组下标。当前代码中不存在 `TemplateName` 匹配机制。

变高列表数据额外实现 `IMeasuredViewData`，用来预声明滚动轴长度。它不替代绑定后实测：

```csharp
public interface IMeasuredViewData : ISimpleViewData
{
    bool TryGetItemLength(out float length);
}
```

只有 `LinearLayoutManager` / `MixedLayoutManager` 打开 `Variable Item Length` 后才支持变高。`Grid` / `Page` / `Circle` 必须保持定高；变高开关打开时，编辑器和列表启动都会拒绝。

## ViewHolder

列表项渲染逻辑写在 `ViewHolder<TData>` 子类中。当前模块没有独立的 `ItemRender<TData, THolder>` 类。

常用成员：

| 成员 | 说明 |
| --- | --- |
| `CurrentData` | 当前绑定的数据 |
| `CurrentIndex` | 当前数据索引 |
| `CurrentBindingVersion` | 当前绑定版本，可用于异步加载回调校验 |
| `IsBindingCurrent(version)` | 判断异步回调是否仍对应当前绑定 |
| `RequestResize()` | 绑定后文本或资源改高时，按当前 Holder 实测长度写回偏移表 |
| `SetSelect()` | 将当前 `DataIndex` 提交为列表选择项 |
| `OnSelectionChange(bool)` | 选择状态变化回调 |
| `OnClear()` | Holder 被回收或重新绑定前的清理回调 |

如果异步加载图片，建议在 `OnBind` 记录 `CurrentBindingVersion`，回调时用 `IsBindingCurrent(version)` 判断当前 Holder 是否仍绑定同一份数据。

## 注意事项

1. `Templates` 必须非空，且每个模板都必须挂 `ViewHolder` 子类。
2. 混合模板和分组列表的 `TemplateId` 必须是合法的模板数组下标。
3. 分组列表的 `groupTemplateId` 必须是组头模板下标，子项不要使用同一个模板下标，除非业务确实希望它被识别为组头。
4. 动态改变列表项尺寸后，Holder 里改完高度再 `RequestResize()`，或调用 `RefreshItem(index, relayout: true)`。
5. `Content` / `Templates` / `Adapter` / `Layout` 运行期不可更换；`SetAdapter` 只允许首次绑定。
6. 一个 `.cs` 里只能有一个可挂到 Prefab 的 `MonoBehaviour`，类名必须和文件名一致，否则模板会变成 Missing Script。
7. `ScrollMode.AlwaysDisable` 会阻止 `ScrollTo` 定位。
8. 所有 Unity 对象和 UI 操作都应在 Unity 主线程执行。
