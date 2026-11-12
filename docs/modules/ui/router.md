# 路由与转场

页面导航从 `GameApp.UI.Navigation` 取得：

```csharp
using System;
using AlicizaX.UI.Runtime;

IUINavigationService nav = GameApp.UI.Navigation;
```

导航负责维护 history 和页面切换；实际打开/关闭窗口仍由 `IUIService` 完成。弹窗、Tips 不要走导航，继续使用 `ShowUI` / `CloseUI`。

## 前进

```csharp
UIRouteResult result = await GameApp.UI.Navigation.NavigateTo<ShopWindow>(shopArgs);

if (!result)
{
    // result.Status: NotFound / OpenFailed / Cancelled
    return;
}

await result.AwaitTransition(); // 需要等页面转场时
```

`NavigateTo` 在目标页面逻辑打开后返回，转场还在后台播放。`UIRouteResult` 可以隐式转 `bool`，也可以读 `Status`。

常用导航：

```csharp
await GameApp.UI.Navigation.NavigateTo<ShopWindow>(shopArgs); // 压入新页面
await GameApp.UI.Navigation.Replace<ShopWindow>(shopArgs);    // 替换栈顶
await GameApp.UI.Navigation.ResetTo<HomeWindow>();            // 清空 history，只保留目标
await GameApp.UI.Navigation.Back();                           // 返回上一页
await GameApp.UI.Navigation.BackToRoot();                     // 回到第一条 history
await GameApp.UI.Navigation.BackTo<HomeWindow>();             // 回到指定页面
await GameApp.UI.Navigation.CloseCurrent();                   // 关闭当前页并结束 history
```

## history 规则

- history 中同类型最多一条；导航到已存在类型时截断到该条，并用新参数刷新。
- 同一时刻只保留一个页面打开；新页面逻辑打开后才关闭旧页面。
- 新的导航会取代尚未提交的上一次导航，被取代的调用返回 `Cancelled`。
- `Back` 在页面打开过程中会撤销那次打开，并按返回处理。
- `UserData` 保存的是传入对象引用，不会自动复制；需要快照就在传入前自己复制。
- `CloseUI`、`CloseSelf` 从导航之外关闭当前页时，history 会清空；页面返回请走 `Back`。
- `ResetHistory` 只清 history，不关闭窗口；`SyncFromCurrentUI` 把已打开的窗口登记为唯一 history 条目。

建立/清理 history：

```csharp
await GameApp.UI.Navigation.ResetHistory();
await GameApp.UI.Navigation.SyncFromCurrentUI(typeof(HomeWindow));
```

`SyncFromCurrentUI` 要求窗口已经是 `Opening/Opened`；否则返回 `NotFound`。

## 返回与状态

```csharp
bool canBack = GameApp.UI.Navigation.CanBack;

UIRouteEntry current = GameApp.UI.Navigation.CurrentEntry;
if (current.IsValid)
{
    Type pageType = current.UIType;
    object args = current.UserData;
}
```

`UIRouteStatus`：

| 状态 | 含义 |
| --- | --- |
| `Success` | 成功 |
| `NotFound` | 没有目标或没有 history |
| `OpenFailed` | 窗口打开失败 |
| `Cancelled` | 被后续导航或关闭取代 |

`BackTo<T>(openIfMissing: false)` 在 history 中找不到 T 时返回 `NotFound`；`openIfMissing: true` 会把 T 作为新页面打开。

## 转场

转场由 Holder 上的 `IUITransition` 驱动。框架自带 `UIPresetTransition`，也可以自己实现。

### 使用预设转场

在 Holder 预制体根节点或子节点上添加 `UIPresetTransition` 组件。编辑器会自动把最近的 `UIPresetTransition` 绑定到 Holder 的 `Transition` 字段，不需要手动拖引用。

常用配置：

| 字段 | 说明 |
| --- | --- |
| `Open Preset` / `Close Preset` | 打开/关闭动画，例如 `Fade`、`Scale`、`FadeScale`、`SlideFromBottom`、`Toast` |
| `Open Ease` / `Close Ease` | 缓动曲线 |
| `Open Duration` / `Close Duration` | 动画时长 |
| `Unscaled Time` | 默认开启，暂停游戏时动画继续 |
| `Target Rect` / `Canvas Group` | 默认自动取 Holder 自身；为空时编辑器/运行时自动解析 |

注意：

- `Fade`、`FadeScale`、`Slide*`、`Toast` 带 alpha 效果，需要 `CanvasGroup`，适合 Widget Holder。
- 窗口的显隐由 `Canvas.enabled` 控制，窗口根节点没有自动添加 `CanvasGroup`；窗口优先用 `Scale` 或自定义非 alpha 转场，不要依赖 alpha 预设。
- `snap: true` 会直接落到终态，不播放动画。

### 自定义转场

实现 `IUITransition`：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

public sealed class MyTransition : IUITransition
{
    public UniTask Open(bool snap)
    {
        // 从当前外观播放到打开终态；snap 为 true 时直接设置终态。
        return UniTask.CompletedTask;
    }

    public UniTask Close(bool snap)
    {
        // 从当前外观播放到关闭终态。
        return UniTask.CompletedTask;
    }
}
```

运行时替换 Holder 的转场：

```csharp
baseui.SetTransition(new MyTransition());
baseui.SetTransition(null); // 恢复预制体上序列化的默认转场
```

`UIPresetTransition` 已经实现 `IUITransition`；自定义转场可以只负责位置、缩放或透明度，不必关心窗口状态机。

## 注意事项

1. 页面用 `Navigation`，弹窗用 `ShowUI`；同一页面不要混用两条路径。
2. `NavigateTo` 返回时逻辑页面已打开，但转场可能还没结束；需要动画完成再操作 UI 时 `await result.AwaitTransition()`。
3. `UIRouteEntry.UserData` 是引用，不是快照；不要把之后会被修改的对象直接交给 history。
4. 窗口的 alpha 转场不可靠，使用 `Scale` 或自定义非 alpha 转场。
5. 关闭导航当前页会清空 history；需要“返回上一页”时使用 `Back`，不要直接 `CloseUI`。
