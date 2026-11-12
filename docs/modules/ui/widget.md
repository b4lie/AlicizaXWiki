# Widget 与 Tab

Widget 是挂在窗口内部的子视图，和窗口共用生命周期，但可以单独打开、关闭和销毁。需要独立层级、缓存或页面历史的界面使用 `UIWindow`；窗口内部的局部内容用 `UIWidget`。

## 创建 Widget

```csharp
using AlicizaX.UI.Runtime;

[Window(UILayer.UI, UICacheTime.Forever)]
public sealed class HomeWindow : UIWindow<ui_HomeWindow>
{
    private UITestAWidget _widget;

    protected override void OnInitialize()
    {
        _widget = CreateWidgetSync<UITestAWidget>(
            baseui.RectTransform,
            visible: false);
    }

    private void OnRoleClick()
    {
        _widget.Open();
    }
}
```

异步创建：

```csharp
using Cysharp.Threading.Tasks;

private async UniTaskVoid LoadWidgetAsync()
{
    UITestAWidget widget = await CreateWidgetAsync<UITestAWidget>(
        baseui.Content,
        visible: true,
        DestroyCancellationToken);

    if (widget != null)
    {
        widget.Open();
    }
}
```

`CreateWidgetSync` / `CreateWidgetAsync`：

- 父节点必须是当前视图 Holder 的子树，并且节点处于激活状态，否则创建失败并输出 Error。
- `visible: true` 时创建后直接打开；`false` 时只创建并保持关闭。
- 异步创建可以传 `DestroyCancellationToken`，窗口销毁时取消加载。
- 用已有 Holder 创建：`CreateWidgetSync<T>(holder)`；Holder 节点会随 Widget 一起销毁。

## 打开、关闭和销毁

```csharp
_widget.Open();                  // 打开；已打开时刷新参数
_widget.Open(userData);          // 传一个参数对象
_widget.Close();                 // 关闭，保留实例
await _widget.Destroy().AwaitTransition(); // 关闭后销毁
await _widget.AwaitTransition(); // 等当前开关转场结束
```

语义：

- `Open`：`Opening` / `Opened` 时只刷新参数；`Closed` 时重新打开；`Closing` 时从当前外观反向打开。
- `Close`：从未打开过的 Widget 是 no-op；其余状态走完整关闭流程。
- `Destroy`：播放关闭转场后销毁；已经 `Closed` 时立即销毁。
- 父窗口隐藏或关闭时，Widget 的逻辑状态不变，但更新和事件会暂停；父窗口重新启用后恢复。

## Tab 窗口

`UITabWindow<T>` 适合设置页、商店分类等页签结构：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

public sealed class SettingsWindow : UITabWindow<ui_SettingsWindow>
{
    protected override void OnInitialize()
    {
        InitTabVirtuallyView<AudioTab>(baseui.TabRoot);
        InitTabVirtuallyView<VideoTab>(baseui.TabRoot);
        InitTabVirtuallyView<ControlTab>(baseui.TabRoot);
    }

    private void SelectAudio()
    {
        SwitchTab(0).Forget();
    }
}
```

```csharp
public UniTask<UIWidget> SwitchTab(int index, object userData = null);
```

- 页签在第一次切换时才异步创建，之后复用实例。
- 已加载页签之间切换同步完成；未加载页签异步加载。
- 切换是 latest wins：被新切换取代的调用返回 `null`。
- `ActiveTab` 返回当前页签，未选中时为 `null`。
- 页签的父节点不传时默认使用窗口 Holder 根节点。

## UI 事件自动解绑

用 `EventProxy` 订阅事件，视图不可见时自动暂停，销毁时自动退订：

```csharp
using AlicizaX;
using AlicizaX.UI.Runtime;

public readonly struct ShopRefreshedEvent : IEmptyEventArgs
{
}

public sealed class ShopWindow : UIWindow<ui_ShopWindow>
{
    protected override void OnInitialize()
    {
        EventProxy.AddUIEvent<ShopRefreshedEvent>(OnShopRefreshed);
    }

    private void OnShopRefreshed()
    {
        Refresh();
    }
}
```

- `TEvent` 必须是 `struct`，实现 `IEmptyEventArgs`（无参数）或 `IPayloadEventArgs`（有参数）。
- 无参数事件使用 `AddUIEvent<T>(Action)`；有参数事件使用 `AddUIEvent<T>(InEventHandler<T>)`。
- 在 `OnInitialize` 里注册一次即可；不要每次 `OnOpen` 重复添加。
- `EventProxy` 由视图自动管理，销毁时归还内存池并释放全部订阅。

## 注意事项

1. Widget 不是窗口：没有 `[Window]` 层级，也不进入导航 history；层级由父窗口 Holder 决定。
2. Widget 必须创建在父视图 Holder 的激活子树内，不能在窗口还没绑定时创建。
3. 需要保留实例就调用 `Close()`；想彻底移除调用 `Destroy()`，不要只把节点 `SetActive(false)`。
4. 父视图隐藏时，Widget 的 `OnUpdate` 和事件会暂停，但 Widget 状态不会自动变成 `Closed`。
5. Tab 的 `SwitchTab` 只接收一个 `userData`；多个字段请自己定义参数类。
