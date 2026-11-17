# Widget 与 Tab

`UIWidget` 是挂在窗口内部、可以单独打开和关闭的子视图，和窗口共用 `UIBase` 状态机，但没有独立层级、缓存和 history。`UITabWindow<T>` 则把一组 Widget 当作页签，在第一次切换时按需创建。

源码位置：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI/UIBase/UIWidget.cs`、`UIBase.Widget.cs`、`UITabWindow.cs`、`EventListenerProxy.cs`。

## 适用场景

- 窗口内部的局部区块或浮层：角色属性、提示框、奖励预览等。
- 同一个窗口里多块内容各自独立开关，但共享窗口的加载与生命周期。
- 设置页、商店分类这类页签结构（`UITabWindow<T>`）。
- 列表项不要用 Widget：列表项由 RecyclerView 的 ViewHolder 管理（例如 `UIShopGoodsItemViewHolder`），它的创建、复用和销毁归列表，不是 `UIWidget`。

## 使用前提

- 父窗口已经绑定 Holder，且创建 Widget 时父节点在当前视图 Holder 的激活子树内。
- Widget 逻辑类继承 `UIWidget<Holder>`；Holder 是 `UIHolderObjectBase` 子类，类型已注册（生成器通常会生成 `[UIRes]`，也可手动调用 `UIResRegistry.Register`，见 [窗口声明与生命周期](lifecycle.md)）。
- Widget 的 Prefab 根节点必须是 `RectTransform`，并且根节点上挂着对应的 Holder 组件；`UIHolderFactory` 只在实例化根节点上 `GetComponent(HolderType)`。
- 逻辑类不是 MonoBehaviour，不需要挂到 Prefab 上；它由 UI 注册表构造，再绑定到 Holder。

## 最短接入

1. 准备 Widget 逻辑类和它的 Holder。本仓库已有 `UITestAWidget : UIWidget<ui_UITestAWidget>`。
2. 在窗口的 `OnInitialize` 里创建，父节点用 `baseui.RectTransform` 或 Holder 下的激活节点。
3. 用 `Open` / `Close` 控制显示，不要手动 `SetActive`。
4. 需要彻底移除时调用 `Destroy`，而不是只关闭。

```csharp
// UIHomeWindow 中的真实写法
private UITestAWidget _aWidget;

protected override void OnInitialize()
{
    _aWidget = CreateWidgetSync<UITestAWidget>(baseui.RectTransform, visible: false);
}

private void OnBtnRoleClick()
{
    _aWidget.Open();
}

private void OnBtnTestTipsClick()
{
    _aWidget.Close();
}
```

`CreateWidgetSync` 的 `visible: false` 表示加载后保持关闭；传 `true` 会在 Holder 绑定完成、`OnInitialize` 执行后直接打开。

## 核心用法

### 创建 Widget

`UIBase` 提供的创建入口（窗口和 Widget 内都能调用）：

| 方法 | 说明 |
| --- | --- |
| `CreateWidgetSync<T>(Transform parent, bool visible = true)` | 同步加载并绑定；返回 `null` 表示创建失败 |
| `CreateWidgetAsync<T>(Transform parent, bool visible = true, CancellationToken cancellationToken = default)` | 异步加载；取消令牌在加载中取消会销毁该 Widget |
| `CreateWidgetSync<T>(UIHolderObjectBase holder, bool visible = true)` | 用已存在的 Holder 创建，节点随 Widget 一起销毁 |
| `CreateWidgetSync(string typeName, Transform parent, bool visible = true)` | 按类型全名创建；传入窗口类型会报错 |
| `CreateWidgetAsync(string typeName, Transform parent, bool visible = true, CancellationToken cancellationToken = default)` | 同上，异步版本 |

```csharp
private async UniTaskVoid ShowTestWidgetAsync()
{
    UITestAWidget widget = await CreateWidgetAsync<UITestAWidget>(
        baseui.RectTransform, visible: false, DestroyCancellationToken);

    if (widget != null) widget.Open();
}
```

父子校验在创建时一次完成：父节点为空、父节点不在当前视图 Holder 下、父节点未激活，或父视图处于 `Loading` / `Destroyed`，都会记录 Error 并返回 `null`。

### 打开、关闭与销毁

```csharp
_widget.Open();                 // 打开；已打开时更新参数并触发 OnRefresh
_widget.Open(userData);         // 传一个参数对象
_widget.Close();                // 关闭，保留实例
await _widget.Destroy().AwaitTransition(); // 关闭转场结束后销毁
await _widget.AwaitTransition();           // 等当前开关转场结束
```

| 调用 | 行为 |
| --- | --- |
| `Open(userData)` | `Opening` / `Opened`：更新参数并 `OnRefresh`；`Closed`：`OnOpen` + 转场；`Closing`：从当前外观反向打开 |
| `Close()` | `Opening` / `Opened` 才生效；只关闭，不销毁实例；`Loading` / `Closed` 为 no-op |
| `Destroy()` | 播放关闭转场后销毁；已经 `Closed` 时立即销毁；没有转场时同步完成销毁 |
| `AwaitTransition()` | 等到状态不再处于 `Opening` / `Closing` |

`Destroy()` 返回 `UIViewTransition`；销毁时 Widget 会从父视图的 Widget 列表移除，嵌套 Widget 一并销毁。

### 挂载与生命周期

- 创建顺序：构造逻辑实例 → 实例化 Prefab → 在根节点解析 Holder → `holder.Bind(view)` → 状态设为 `Closed` → `BindVisuals` → `OnInitialize` → 隐藏 → 按 `visible` 决定是否 `Open`。
- `OnInitialize` 每个 Widget 实例只执行一次；没有缓存复用，`Close` 后再 `Open` 只重新执行 `OnOpen`。
- 显示/隐藏不切节点激活状态：Holder 有 `Canvas` 时切 `Canvas.enabled` 和 `GraphicRaycaster.enabled`；没有 `Canvas` 时自动补 `CanvasGroup`，改 `alpha`、`blocksRaycasts`、`interactable`。
- 父视图隐藏或关闭时，Widget 的逻辑状态不变，但 `Holder.IsEnabled` 变 false，`OnUpdate` 和事件订阅会暂停；父视图重新启用后恢复。
- `Visible` 是视图自身的逻辑状态，不反映父级是否可见；判断“现在真的能看到”用 `Holder.IsEnabled`。

### 嵌套 Widget 的所有权

Widget 的父级是“创建它的视图”，不是 Transform 上的父节点：

- 窗口销毁时按列表逐个销毁自己的 Widget，Widget 再销毁自己创建的嵌套 Widget，形成整棵树。
- Widget 被销毁时只影响它自己和它的子树，不会销毁兄弟或父窗口。
- Holder 被外部销毁时，`UIHolderObjectBase.OnDestroy` 会通知绑定视图走销毁流程；父窗口还活着时只销毁该 Widget。
- 不要手动 `Destroy` Holder 节点来“关闭” Widget；用 `Close()` 或 `Destroy()`。

### Tab 窗口

`UITabWindow<T>` 继承 `UIWindow<T>`，页签本身是 Widget。下面是页签宿主示例（类名和 Holder 换成你的实际类型）：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;
using Game.UI;

[Window(UILayer.UI, UICacheTime.Forever)]
public sealed class HomeTabWindow : UITabWindow<ui_UIHomeWindow>
{
    protected override void OnInitialize()
    {
        InitTabVirtuallyView<UITestAWidget>(baseui.RectTransform);
    }

    private void SelectFirstTab()
    {
        SwitchTab(0).Forget();
    }
}
```

| 成员 | 说明 |
| --- | --- |
| `InitTabVirtuallyView<TTab>(Transform parent = null)` | 登记页签；`parent` 为 null 时用 `baseui.RectTransform`；调用顺序就是页签索引 |
| `InitTabVirtuallyView(string typeName, Transform parent = null)` | 按类型全名登记；必须是 Widget |
| `SwitchTab(int index, object userData = null)` | 切到指定页签，返回该 Widget；被更新的切换取代时返回 `null` |
| `ActiveTab` | 当前页签；还没切换过时为 `null` |

- 页签在第一次 `SwitchTab` 时才创建 Holder，之后复用同一个 Widget 实例。
- 已加载页签之间切换同步完成；前一个页签的关闭转场与本次切换并行，不阻塞返回。
- 索引越界会记录 `Invalid tab index` 错误并返回 `null`。
- 页签切换是 latest wins：旧调用返回 `null`，只有最后一次结果生效。
- 页签 Widget 被 `Destroy` 后，下一次切到该索引会重新创建。
- 页签不会单独进缓存或 history；父窗口销毁时整组页签一起销毁。

### 事件自动解绑

窗口和 Widget 共用 `UIBase.EventProxy`。在本仓库的 `UIShopWindow` 中：

```csharp
using AlicizaX;
using AlicizaX.UI.Runtime;
using GameLogic.Player;

protected override void OnInitialize()
{
    EventProxy.AddUIEvent<PlayerDataChangedEvent>(OnPlayerDataChanged);
}

private void OnPlayerDataChanged(in PlayerDataChangedEvent evt)
{
    RefreshCurrencyText();
}
```

- 无参事件用 `AddUIEvent<T>(Action)`，`T` 实现 `IEmptyEventArgs`；有参事件用 `AddUIEvent<T>(InEventHandler<T>)`，`T` 实现 `IPayloadEventArgs`。
- 视图启用时订阅、停用时退订：窗口关闭或父级隐藏，Widget 的事件自动暂停，重新打开后恢复。
- 在 `OnInitialize` 里注册一次即可；`AddUIEvent` 不查重，每次 `OnOpen` 都加会累积重复回调。
- 视图销毁时代理归还 `MemoryPool` 并退订全部事件；销毁后访问 `EventProxy` 会抛 `InvalidOperationException`，不要把它存到静态字段。
- 手工 C# 事件（例如 `UIBuyAlertWindow` 里的 `LocalizationKit.Changed`）不经过 `EventProxy`，仍需在 `OnDestroy` 里 `-=`。

## 常见配置

| 需求 | 写法 | 说明 |
| --- | --- | --- |
| 创建后先不显示 | `CreateWidgetSync<T>(parent, visible: false)` | `UIHomeWindow` 的用法；之后手动 `Open` |
| 异步加载并跟随视图销毁 | `CreateWidgetAsync<T>(parent, true, DestroyCancellationToken)` | 加载中取消会销毁 Widget |
| 复用现成 Holder | `CreateWidgetSync<T>(holder)` | Holder 类型必须匹配，节点随 Widget 销毁 |
| 按类型名创建 | `CreateWidgetSync(typeName, parent)` | 只能传 Widget 类型；窗口类型会报错 |
| 切页签 | `SwitchTab(index, userData).Forget()` | 懒创建；参数传给该页签的 `Open` |
| 订阅 UI 事件 | `EventProxy.AddUIEvent<T>(handler)` | 只在 `OnInitialize` 订阅一次 |
| 彻底移除 | `await widget.Destroy().AwaitTransition()` | 只关闭不移除用 `Close()` |

## 容易踩的坑

1. **父节点必须属于当前视图且处于激活状态。** 把 Widget 建在窗口外、未激活节点或 Holder 还没绑定时，都会失败并返回 `null`；`CreateWidgetSync<T>(holder)` 借用的 Holder 也一样要满足这个条件。
2. **Widget 不是窗口。** 没有 `[Window]`、层级、缓存和 history，不能用 `GetUI<T>` 查；`NavigateTo<T>` 的泛型约束也不接受 Widget。
3. **`Close()` 不销毁，`SetActive(false)` 也不算关闭。** 关闭只是隐藏并保留实例；要释放节点和订阅用 `Destroy()`，否则父窗口销毁前会一直存在。
4. **父窗口隐藏后 Widget 仍可能报 `Visible == true`。** 它只是逻辑上还开着；用 `Holder.IsEnabled` 判断当前是否真正启用，更新和事件已经停了。
5. **加载中的 Widget `Close()` 是 no-op。** 异步创建还没绑定时关闭不会取消加载；要取消就给 `CreateWidgetAsync` 传令牌，或销毁父视图。
6. **页签切换是异步的。** `ActiveTab` 在第一次切换前是 `null`，被后续切换取代的 `SwitchTab` 返回 `null`；不要在返回值上无条件解引用。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `UIWidget.Open(object userData = null)` | 打开或刷新；参数为 null 时保留原参数 |
| `UIWidget.Close()` | 关闭并保留实例 |
| `UIWidget.Destroy()` | 关闭后销毁，返回可等待的 `UIViewTransition` |
| `UIBase.AwaitTransition()` | 等当前开关转场结束 |
| `UIBase.State` / `Visible` | 当前状态、逻辑可见性 |
| `UIBase.DestroyCancellationToken` | 视图销毁时取消，传给业务异步 |
| `UIBase.EventProxy` | 视图级事件订阅代理，自动暂停/退订 |
| `CreateWidgetSync<T>` / `CreateWidgetAsync<T>` | 创建 Widget（含 Holder、类型名重载） |
| `UITabWindow<T>.InitTabVirtuallyView<TTab>` | 登记页签 |
| `UITabWindow<T>.SwitchTab(int, object)` | 切换页签，返回 `UniTask<UIWidget>` |
| `UITabWindow<T>.ActiveTab` | 当前页签 |
| `EventListenerProxy.AddUIEvent<T>` | 订阅无参或有参 UI 事件 |
