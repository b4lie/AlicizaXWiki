# RecyclerView 导航

`RecyclerViewNavigationController` 让虚拟列表参与手柄/键盘导航。列表项是可回收的 Prefab 实例，所以导航焦点不绑定 `ViewHolder` 或 item `GameObject`，而是绑定数据索引；控制器自己是 EventSystem 中唯一的选中对象。

源码位置：

- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView/Navigation`
- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView/ViewHolder`

依赖：UI extension + `com.alicizax.unity.input`。导航代码受 `INPUTSYSTEM_SUPPORT && UXNAVIGATION_SUPPORT` 保护，宏由 asmdef 自动生成。

## Prefab 配置

`RecyclerViewNavigationController` 必须和 `RecyclerView` 挂在同一个 GameObject 上：

```text
ScrollView
├── Content
├── Templates
└── Components
    ├── RecyclerView
    ├── Scroller
    └── RecyclerViewNavigationController
```

父级窗口或面板要有 `NavigationScope`。编辑器会从父节点向上找最近的 `NavigationScope` 并写入 `Navigation Scope` 字段；运行时不会自动查找。

Inspector：

| 字段 | 说明 |
| --- | --- |
| `Recycler View` | 同物体上的 `RecyclerView`，自动赋值 |
| `Navigation Scope` | 父级最近的 `NavigationScope`，自动赋值 |
| `Wrap Navigation` | 普通列表首尾是否环绕 |
| `Keep Focus Visible When Suppressed` | 被上层遮挡时是否保留焦点视觉 |
| `Smooth Scroll` | 焦点变化时是否平滑滚动 |
| `Smooth Scroll Duration` | 平滑滚动时长 |
| `Focus Alignment` | 焦点项滚动对齐：`Start`、`Center`、`End` |

边界出口使用 UGUI `Selectable.navigation`（推荐 `Explicit` + `Select On Left/Right/Up/Down`），不再有独立的 Exit 字段。

## 模板接入

普通 `ViewHolder<TData>` 不参与导航。需要导航的模板继承 `NavigationViewHolder<TData>` 或 `InteractiveViewHolder<TData>`：

```csharp
using AlicizaX.UI;
using UnityEngine;
using UnityEngine.EventSystems;

public sealed class BagItemHolder : InteractiveViewHolder<BagItemData>
{
    [SerializeField] private GameObject focusedFrame;

    protected override void OnBind(BagItemData data, int index)
    {
        // 正常绑定业务数据。
    }

    protected override void OnNavigationFocused(bool focused)
    {
        focusedFrame.SetActive(focused);
    }

    protected override void OnItemClick(BagItemData data, int index)
    {
        // 鼠标点击或导航提交。
    }
}
```

- `NavigationViewHolder<TData>`：只处理导航焦点和 Move/Submit，不处理鼠标点击。
- `InteractiveViewHolder<TData>`：在导航基础上处理指针点击、按压视觉和 `OnItemClick`，适合卡片、按钮式列表项。
- 不需要导航的模板继续继承 `ViewHolder<TData>`，会被跳过。

导航焦点和业务选中是两套状态：

- `OnNavigationFocused` 只画焦点视觉。
- `SetSelect()` / `ChoiceIndex` 表示业务选中。
- `InteractiveViewHolder` 默认 `SelectOnActivate = true`，点击或提交时同时提交业务选中。

## 进入列表

`RecyclerViewNavigationController` 本身是 `Selectable`。`NavigationScope` 选中它后，方向输入进入 `OnMove`，再由控制器计算下一个可导航数据索引。

业务也可以主动控制：

```csharp
using AlicizaX.UI;
using UnityEngine;

[SerializeField] private RecyclerViewNavigationController nav;

public void FocusItem(int dataIndex)
{
    nav.SetFocus(dataIndex, scroll: true);
}

public void RestoreLastFocus(int dataIndex)
{
    nav.RestoreFocus(dataIndex);
}
```

| API | 说明 |
| --- | --- |
| `FocusedDataIndex` | 当前焦点数据索引，无焦点为 -1 |
| `HasFocus` | 是否有焦点 |
| `IsSuppressed` | 是否被上层 Scope 抑制 |
| `SetFocus(int dataIndex, bool scroll = true)` | 设置焦点 |
| `RestoreFocus(int dataIndex)` | 恢复焦点 |
| `SetFocusFilter(Predicate<int>)` | 过滤可聚焦数据；返回 false 的项被跳过 |

## 数据刷新

不需要手动通知导航。继续使用列表 API：

```csharp
items.SetData(newItems);
items.Adapter.NotifyItemChanged(index);
```

`Adapter` 的结构变化会通知 `RecyclerView`，控制器再按数据索引修正焦点。数据减少或当前模板不可导航时，会找最近的可导航项；没有可导航项时清空焦点。

## 层级与遮挡

上层窗口的 `NavigationScope` 接管焦点后，底层列表的 `RecyclerViewNavigationController` 被抑制：

1. 隐藏当前焦点视觉（是否保留由 `Keep Focus Visible When Suppressed` 决定）。
2. 记住当前 `FocusedDataIndex`。
3. 上层关闭、Scope 恢复后重新选中控制器，并恢复数据索引和滚动位置。

恢复只依赖数据索引，不依赖被回收的 `ViewHolder` 或 item 实例。

## 注意事项

1. 控制器必须和 `RecyclerView` 同物体，不要挂到 `Content` 或模板上。
2. 父级必须有 `NavigationScope`；没有 Scope 时控制器不会注册。
3. 需要导航的模板继承 `NavigationViewHolder` / `InteractiveViewHolder`，不要直接实现内部接口。
4. `Wrap Navigation` 只控制首尾环绕，不是循环列表开关；`UGLoopList` 会被自动识别。
5. 布局支持线性、网格、混合和圆形；自定义布局需要实现对应的导航步进接口。
