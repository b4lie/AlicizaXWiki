# 路由与转场

页面级导航由 `IUINavigationService` 提供，入口是 `GameApp.UI.Navigation`。它维护有顺序的 history，并按 history 打开/关闭页面；弹窗、Tips 等不需要返回栈的界面仍用 `ShowUI` / `CloseUI`（见 [打开、关闭与查询](window.md)）。

源码位置：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI/Manager/IUINavigationService.cs`、`UIService.Navigation.cs`。

## 适用场景

- 主界面 → 子页面 → 详情页这类需要“返回上一页”的层级切换。
- 需要 `CanBack`、当前页面类型、页面参数等路由状态的界面。
- 需要统一页面转场，或只对页面而不是弹窗播放转场。

不适用：弹窗、Toast、确认框、独立工具面板。它们没有返回栈语义，直接 `ShowUI` 即可。

## 使用前提

- 页面必须是 `UIWindow` 子类（通常是 `UIWindow<Holder>`），并有 `[Window]` 声明的层级与缓存时间，见 [窗口声明与生命周期](lifecycle.md)。
- 页面类型已由生成器注册；导航只接受窗口类型，Widget 不能作为页面。
- UI 服务已初始化（`UIComponent`），见 [UI 模块](index.md)。
- 引入命名空间并取得入口：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;
using GameLogic.UI;

IUINavigationService nav = GameApp.UI.Navigation;
```

## 最短接入

1. 准备一个页面类型，例如本仓库的 `UIHomeWindow`、`UIShopWindow`、`UIRecyclerViewExampleWindow`。
2. 在按钮回调里前进，并检查 `UIRouteResult`。
3. 需要等开场转场结束时调用 `AwaitTransition`。
4. 返回上一页时用 `Back`，不要用 `CloseSelf` 关页面。

```csharp
private async UniTaskVoid OpenChatPageAsync()
{
    UIRouteResult result = await GameApp.UI.Navigation
        .NavigateTo<UIRecyclerViewExampleWindow>(RecyclerViewExampleKind.Chat);

    if (!result) return;          // NotFound / OpenFailed / Cancelled
    await result.AwaitTransition(); // 逻辑已打开，这里等转场稳定
}

private async UniTaskVoid BackAsync()
{
    if (!GameApp.UI.Navigation.CanBack) return;

    UIRouteResult result = await GameApp.UI.Navigation.Back();
    if (!result.Success) return;
}
```

`NavigateTo` 返回时页面已执行完 `OnOpen`，但开场转场可能还在播放；`UIRouteResult.AwaitTransition()` 等待当前转场结束。

## 核心用法

### 导航方法

| 方法 | 对 history 的影响 | 对窗口的影响 |
| --- | --- | --- |
| `NavigateTo<T>()` | `T` 已在 history 中时截断到该条；否则压入栈顶 | 打开/刷新 `T`，提交后关闭旧栈顶 |
| `Replace<T>()` | `T` 已在 history 中时截断到该条；否则去掉栈顶再压入 | 打开/刷新 `T`，提交后关闭旧栈顶 |
| `ResetTo<T>()` | 清空后只保留 `T` | 打开 `T`，提交后关闭旧栈顶 |
| `Back()` | 至少两条时回到倒数第二条；加载中撤销打开、history 不变 | 关闭当前页，打开上一页 |
| `BackToRoot()` | 多于一条时截断到第一条；只有一条时不变 | 打开/刷新根页面 |
| `BackTo<T>(openIfMissing)` | 找到 `T` 时截断到该条并沿用原参数；找不到且 `openIfMissing` 为 true 时按 `ResetTo` 处理 | 打开 `T` |
| `CloseCurrent(force)` | 立即清空 | 关闭当前页，`force` 决定是否进缓存 |
| `ResetHistory()` | 立刻清空 | 不打开、不关闭任何窗口 |
| `SyncFromCurrentUI(Type)` | 清空后把已打开的窗口登记为唯一条目 | 不打开、不关闭 |

`CanBack` 等价于 `History.Count > 1`。history 没有页数上限；每条 `UIRouteEntry` 持有一个 `UserData` 引用，长期不清空会一起常驻。

### history 规则

- history 中同一类型最多一条；导航到已存在类型时截断它上面的所有条目。
- 同一时刻只保留一个页面打开；新页面 `OnOpen` 执行、history 提交后才关闭旧页面。
- 页面参数保存在 `UIRouteEntry.UserData` 中，是引用不是快照；`Back` / `BackTo` 会复用该条目原来的参数。
- `CurrentEntry` 在 history 为空时是 `default`，`IsValid` 为 false。
- 页面离场时按 `[Window]` 的 `cacheTime` 决定关闭后缓存还是销毁：`0` 会在离开后销毁，`Back` 时重新加载并重新执行 `OnInitialize`。

```csharp
UIRouteEntry entry = GameApp.UI.Navigation.CurrentEntry;
if (entry.IsValid)
{
    Type pageType = entry.UIType;
    object args = entry.UserData;
}
```

### 页面与窗口

导航不经过 `ShowUI`：`Navigator` 直接调用 UI 服务内部的打开方法，因此不会重复触发 `ShowUI` 的等待逻辑。由此有三条实际规则：

- `ShowUI<T>` 打开的页面不会写 history；需要接管时调用 `SyncFromCurrentUI(typeof(T))`，且窗口必须处于 `Opening` / `Opened`。
- 如果 `ShowUI` 恰好打开了导航正在提交的同一个窗口，导航会把该窗口交给 `ShowUI` 管理；之后被新导航取代时，不会把这个窗口关掉。
- 导航没有 `snap` 参数，转场始终由 Holder 上的 `IUITransition` 决定；要跳过动画只能改转场配置。

### 关闭当前页与失败

- 从导航之外关闭当前页（`CloseSelf`、`CloseUI`、Holder 被外部销毁）会清空 history；关闭的不是当前页时 history 不变。
- `CloseCurrent` 先清 history 再播放关闭；history 为空或当前窗口已不存在时返回 `NotFound`。
- 未注册类型、资源加载失败、目标在加载完成前被销毁：返回 `OpenFailed`，history 与当前页保持原样，可重试。
- 被后续导航、`Back`、`ResetHistory`、`CloseCurrent` 或服务停止取代：返回 `Cancelled`，history 不变。
- 同一目标加载中重复导航只加载一次；较早的调用返回 `Cancelled`，最后一次提交，参数以最后一次为准。
- 不同目标并发导航时“最新一次赢”；未提交的页面会被后一次导航关闭。
- 加载过程中按 `Back` 会撤销这次打开并直接返回 `Success`，history 不动，也不会切页。

### 转场

Holder 通过 `UIHolderObjectBase.UITransition` 取得转场：运行时 `SetTransition` 优先，否则使用编辑器维护的序列化默认转场。默认值由编辑器工具从 Holder 自身开始、深度优先查找子节点上的 `IUITransition`，遇到嵌套 Holder 停止；Inspector 里的 `Transition` 字段只读。

`UIPresetTransition` 是内置实现。把组件挂到 Holder 根节点（或需要单独动画的子节点）上即可，运行时也可以替换：

```csharp
protected override void OnInitialize()
{
    // 传 null 恢复预制体/编辑器绑定的默认转场
    baseui.SetTransition(new MyTransition());
}
```

自定义转场实现 `IUITransition`：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

public sealed class MyTransition : IUITransition
{
    public UniTask Open(bool snap)
    {
        // 从当前外观推进到打开终态；snap 为 true 时立即落到终态。
        return UniTask.CompletedTask;
    }

    public UniTask Close(bool snap)
    {
        return UniTask.CompletedTask;
    }
}
```

转场播放中再次 `Open` / `Close` 会结束上一次播放，让上一次返回的任务完成；`snap: true` 同步落终态。转场任务只应由视图 await 一次。

## 常见配置

`UIPresetTransition` 的 Inspector 字段（序列化字段名见括号）：

| Inspector | 字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Open Preset / Close Preset | `openPreset` / `closePreset` | `FadeScale` | 打开/关闭使用的预设 |
| Open Ease / Close Ease | `openEase` / `closeEase` | `OutCubic` / `InCubic` | 缓动曲线 |
| Open Duration / Close Duration | `openDuration` / `closeDuration` | `0.22` / `0.18` 秒 | 时长为 0 时立即完成 |
| Slide Distance | `slideDistance` | `120` | 四个 `SlideFrom*` 的位移距离 |
| Toast Distance | `toastDistance` | `40` | `Toast` 的上浮距离 |
| Closed Scale | `closedScale` | `0.94` | `Scale` / `FadeScale` 的关闭缩放，范围 0.5–1 |
| Unscaled Time | `useUnscaledTime` | 开启 | 暂停游戏时继续播放 |
| Target Rect | `targetRect` | 组件自身 | 位置/缩放的动画目标 |
| Canvas Group | `canvasGroup` | 组件所在节点 | alpha 预设的透明度载体 |

预设与 alpha 的关系：

| 预设 | 需要 CanvasGroup | 说明 |
| --- | --- | --- |
| `None` | 否 | 直接落终态 |
| `Fade` / `FadeScale` | 是 | 透明度，`FadeScale` 同时缩放 |
| `Scale` | 否 | 只缩放 |
| `SlideFromBottom` / `Top` / `Left` / `Right` | 是 | 位移 + 透明度 |
| `Toast` | 是 | 位移 + 缩放 + 透明度 |

`UITransitionEase` 可选 `Linear`、`InQuad`、`OutQuad`、`InCubic`、`OutCubic`、`InOutCubic`、`OutBack`。

- Widget Holder 没有 Canvas 时，绑定期会自动补一个 `CanvasGroup`；窗口不会自动补。窗口要用 alpha 预设，必须在窗口预制体上自行挂 `CanvasGroup`（本仓库 `UIHomeWindow.prefab` 就是这种配置）。
- Inspector 的 `Canvas Group` 只查组件所在节点，不是 Holder 根节点；把组件放在子节点又想要整体淡入淡出时，手动把 `Target Rect` / `Canvas Group` 指到根节点。
- Inspector 的 `Editor Preview` 只在编辑模式、且只选中一个组件时可用；预览会在 Inspector 关闭时自动还原，不影响运行时状态。

## 容易踩的坑

1. **用 `CloseSelf` / `CloseUI` 关导航页会清空 history。** 本仓库 `UIShopWindow` 的关闭按钮就是 `CloseSelf`；如果把它作为导航页，返回栈会一起消失。导航页关闭请走 `Back` / `CloseCurrent`。
2. **`NavigateTo` 返回时视觉还没稳定。** history 已提交、旧页面的关闭转场与新页面的开场转场并行；需要动画完成再操作 UI 时 `await result.AwaitTransition()`。
3. **`cacheTime` 为 0 的页面 `Back` 时会重建。** 离开即销毁，返回时重新加载资源并再次执行 `OnInitialize`，不要假设页面字段还保留着。
4. **同类型页面只有一条 history。** 从 A 前进到 B 再 `NavigateTo<A>` 会截断 B，`CanBack` 变回 false；这不是“再压一层 A”。
5. **加载中按 `Back` 返回 `Success` 但页面没变。** 它撤销的是尚未提交的打开，history 不动；不要用这个结果判断“已经回到上一页”。
6. **窗口 alpha 预设不会自动生效。** 窗口不会自动加 `CanvasGroup`；`UIPresetTransition` 的 `canvasGroup` 为空时 alpha 通道会被跳过，只剩缩放或位移。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `IUIService.Navigation` | 取得 `IUINavigationService`，业务侧通常写 `GameApp.UI.Navigation` |
| `NavigateTo<T>(object userData = null)` | 压入页面；`T` 已存在时截断到该条并刷新参数 |
| `Replace<T>(object userData = null)` | 替换栈顶；`T` 已存在时截断到该条 |
| `ResetTo<T>(object userData = null)` | 清空 history 后只保留 `T` |
| `Back()` | 返回上一页；加载中调用会撤销打开 |
| `BackToRoot()` | 回到第一条 history |
| `BackTo<T>(bool openIfMissing = true, object userData = null)` | 回到 history 中的 `T`；找不到时按参数决定是否新建 |
| `CloseCurrent(bool force = false)` | 关闭当前页并结束 history |
| `ResetHistory()` | 只清 history，不动窗口 |
| `SyncFromCurrentUI(Type currentPageType, object userData = null)` | 把已打开窗口登记为唯一 history 条目 |
| `CanBack` | history 至少两条时为 true |
| `CurrentEntry` | 当前 `UIRouteEntry`；为空时 `IsValid` 为 false |
| `UIRouteResult.Status` | `Success`、`NotFound`、`OpenFailed`、`Cancelled` |
| `UIRouteResult.Success` / 隐式 `bool` | 判断是否成功 |
| `UIRouteResult.AwaitTransition()` | 等待当前页面转场结束 |
| `UIRouteEntry.UIType` / `UserData` / `IsValid` | 页面类型、参数引用、是否为空条目 |
