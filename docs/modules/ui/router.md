# UIRouter（页面导航）

入口：`IUINavigationService router = GameApp.UI;`（`IUIService` 也直接暴露这些方法）。

## 建立 root

```csharp
await GameApp.UI.NavigateTo<UIHomeWindow>();          // 第一条 history
GameApp.UI.ResetHistory();                            // 只清 history，不关 UI
GameApp.UI.SyncFromCurrentUI(typeof(UIHomeWindow));   // 把当前已 Opened 的 UI 设为 root
```

## 前进导航

```csharp
UIRouteResult result = await GameApp.UI.NavigateTo<UIShopWindow>(shopId);
if (!result.Success) { /* result.Status */ }
```

规则：

- 每次成功导航写入一条 history；history 上限 64，超出返回 `RejectedLimit`。
- **history 中同类型最多一条**：导航到已存在类型时按该条截断（回到该页并丢弃其上的页），不会产生重复条目。
- 参数在入 history 时浅拷贝一份，之后以只读视图暴露；业务修改传入数组不会影响 history。
- `Replace` 替换当前页（`keep = 当前页 index`），`ResetTo` 重建 history 只保留目标页。
- 打开失败不改 history、不关闭当前页；`Back` 恢复的页面使用 history 中保存的参数。
- 新导航会取消进行中且未提交的旧导航（旧调用返回 `Cancelled`）；`Back` 在打开过程中会直接取消打开并返回 `Ok`。

## 返回

```csharp
await GameApp.UI.Back();                 // history.Count < 2 -> NotFound
await GameApp.UI.BackToRoot();
await GameApp.UI.BackTo<UIHomeWindow>();               // 未命中且 openIfMissing=true -> ResetTo
await GameApp.UI.BackTo<UIHomeWindow>(openIfMissing: false);
await GameApp.UI.CloseCurrent(force: false);           // 关闭当前页
```

`CloseCurrent` 规则：关闭当前页并清空 history。需要回到上一页时使用 `Back`。

## 状态

```csharp
bool canBack = GameApp.UI.CanBack;
Type current = GameApp.UI.Current;
UIRouteEntry entry = GameApp.UI.CurrentEntry;   // 内部只读实例，不做拷贝
var firstArg = entry.Arguments[0];              // IReadOnlyList<object>
```

`UIRouteResult.Status`：

| Status | 含义 |
|---|---|
| `Success` | 成功 |
| `RejectedLimit` | history 已满 |
| `NotFound` | 无目标/无 history |
| `OpenFailed` | 开关窗失败 |
| `Cancelled` | 被后续导航/关闭/关服取消 |

> 导航 API 全部是异步事务；按钮回调里可以 `.Forget()`，关键流程请 `await` 并检查 `Status`。

## 转场

转场由 Holder 驱动。一次播放只有一个所有者：

```csharp
public interface IUITransitionSource
{
    UniTask Play(bool open, CancellationToken cancellationToken);
    void Snap(bool open);
}

// 可选：由 UI 服务逐帧推进，避免每帧调度 PlayerLoop
public interface ITickTransitionSource : IUITransitionSource
{
    bool Begin(bool open);          // 建立起止状态；false = 0 时长、已应用终态
    bool Advance(float deltaTime);  // true = 已到终态
}
```

- 转场源实现 `ITickTransitionSource` 时走 tick 路径：由 `UIService.Tick` 调用 `Advance`，完成后 Holder 直接通知所属视图，不分配等待对象。
- 只实现 `IUITransitionSource` 时走异步 `Play` 路径，每次播放分配一个 CTS 和一个 UniTask runner。
- 手动创建的 Holder（`UIHolderFactory`）没有所属视图，不会播放转场。
- `UIPresetTransition` 同时实现两种接口；tick 路径使用 `useUnscaledTime` 选择时间源（默认 unscaled）。
- `timeScale = 0` 且 `useUnscaledTime = false` 时，按缩放时间的转场会随游戏暂停而冻结（不会死循环）；这是明确的时间语义。
- 取消时只取消，不补终态；终态由下一次完整 `Play` 或 `Snap` 负责。被取消的异步 `Play` 抛出的 `OperationCanceledException` 由 Holder 吞掉。
- `skipTransition`、销毁子树、`RemoveWidget`、Router 关下层走 `Snap`。
- 自定义：

```csharp
protected override void OnInitialize()
{
    SetTransition(PlayMine, SnapMine);
}

private UniTask PlayMine(bool open, CancellationToken ct) { ... }
private void SnapMine(bool open) { ... }
```

`SetTransition(null)` 取消当前播放并回到 Inspector 上的组件源。播放中替换转场源时，当前打开或关闭阶段立即结束，不会重新播放。
