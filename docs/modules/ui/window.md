# 打开、关闭与查询

窗口通过 `IUIService` 打开和关闭。页面跳转请使用 [路由与转场](router.md) 的 `Navigation`，弹窗和 Tips 直接用这里的 `ShowUI` / `CloseUI`。

## 打开窗口

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

// 不传参数
HomeWindow home = await GameApp.UI.ShowUI<HomeWindow>();

// 传一个参数对象
ShopWindow shop = await GameApp.UI.ShowUI<ShopWindow>(shopArgs);

// 按钮回调里不关心结果
GameApp.UI.ShowUI<ShopWindow>(shopArgs).Forget();

// 同步加载
HomeWindow same = GameApp.UI.ShowUISync<HomeWindow>();
```

`ShowUI` 在资源加载、`OnInitialize` 和 `OnOpen` 完成后返回；开场转场还在后台播放。需要等转场时：

```csharp
HomeWindow home = await GameApp.UI.ShowUI<HomeWindow>();
await home.AwaitTransition();

// 或者直接在 Show 任务上等
await GameApp.UI.ShowUI<HomeWindow>().AwaitTransition();
```

| 调用结果 | 含义 |
| --- | --- |
| `null` | 类型未知、资源加载失败，或本次调用被取消；需要区分原因时看日志 |
| 已有实例 | 已打开/打开中的窗口复用同一个实例，只更新参数并 `OnRefresh` |
| 关闭中再次打开 | 从当前外观反向打开，不返回 `null` |
| 加载中再次打开 | 只更新参数，加载完成后打开 |

### 取消等待

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;

async UniTask OpenWithCancel(CancellationToken token)
{
    HomeWindow home = await GameApp.UI.ShowUI<HomeWindow>(token);
    if (home == null)
    {
        // 只有本次等待被取消，窗口仍会继续加载和打开。
    }
}
```

令牌只影响本次调用：调用前已经取消则不发起打开；加载过程中取消只结束本次等待，窗口仍会继续加载和打开。要取消加载，使用 `CloseUI`。

### 传入参数

`ShowUI` 只接收一个 `userData`：

```csharp
public sealed class ShopOpenArgs
{
    public int ShopId;
    public string From;
}

await GameApp.UI.ShowUI<ShopWindow>(new ShopOpenArgs { ShopId = 1001, From = "bag" });
```

窗口里读取：

```csharp
protected override void OnOpen()
{
    if (UserData is ShopOpenArgs args)
    {
        Refresh(args.ShopId);
    }
}
```

同一个窗口已打开时再次 `Show`，`userData` 非空才替换原参数；传 `null` 保留原参数。

## 关闭窗口

`CloseSelf` 是窗口逻辑类的 `protected` 方法，下面片段写在窗口类内；从外部关闭使用 `CloseUI<T>`。

```csharp
// 窗口内关闭自己
CloseSelf();

// 等待关场动画结束
await CloseSelf().AwaitTransition();

// 跳过关场动画
CloseSelf(snap: true);

// 关闭并直接销毁，不进入缓存
CloseSelf(force: true);

// 从外部关闭
GameApp.UI.CloseUI<HomeWindow>();
await GameApp.UI.CloseUI<HomeWindow>(snap: true).AwaitTransition();
```

| 参数 | 作用 |
| --- | --- |
| `force: true` | 关闭后直接销毁，不进入缓存 |
| `snap: true` | 关闭转场直接落到终态 |

- 关闭完成后才进入缓存；缓存时长从关场转场结束开始计算。
- 缓存中的窗口再次 `CloseUI(force: false)` 是 no-op；`force: true` 会销毁缓存实例。
- 关闭类型没有打开且不在缓存时，`CloseUI` 输出 Warning 并返回默认过渡。
- `CloseSelf` 在 `OnOpen` 等钩子内调用会立即改变状态，原流程不再继续；在 `OnClose` 中调用不会重复关闭。

## 查询窗口

```csharp
bool open = GameApp.UI.IsOpen<HomeWindow>();
HomeWindow cached = GameApp.UI.GetUI<HomeWindow>();
RectTransform uiLayer = GameApp.UI.GetLayer(UILayer.UI);

if (GameApp.UI.TryGetTopVisibleHolder(null, out UIHolderObjectBase holder))
{
    // holder 是当前最高的 Opening/Opened 窗口节点。
}
```

- `IsOpen`：`Opening` 或 `Opened`，不表示开场动画已经结束。
- `GetUI`：`Opening`、`Opened` 或缓存中的 `Closed`；`Loading`、`Closing`、已销毁返回 `null`。
- `TryGetTopVisibleHolder`：从最高层向下查找第一个 `Opening/Opened` 且满足谓词的 Holder；谓词传 `null` 表示不过滤。

从最高层关闭窗口：

```csharp
bool closed = await GameApp.UI.TryCloseTopAsync(
    handle => handle.Equals(typeof(SettingsWindow).TypeHandle));
```

`TryCloseTopAsync` 只关闭第一个匹配的逻辑打开窗口，并等待关场转场结束；没有匹配返回 `false`。关闭导航当前页会清空导航 history。

## 重复打开与并发

互斥粒度是单个 UI 类型，不是整层队列：

- 同层不同类型可以并行打开/关闭。
- 同一类型已打开或打开中：复用实例，只更新参数并刷新。
- 同一类型正在关闭：反向打开；如果你希望等它完全关闭后再打开，先 `await CloseUI<T>().AwaitTransition()`。
- 同一类型加载中：只记录参数，加载完成后打开。
- 导航是另一套事务，见 [路由与转场](router.md)；不要用 `ShowUI` 绕开导航来打开导航页面。

## 全屏遮挡

```csharp
// 显示 2 秒，期间拦截点击
GameApp.UI.SetUIBlock(2f);

// 立即解除
GameApp.UI.ForceExitBlock();
```

遮挡节点由 UI 服务创建，显示在所有层级之上；重复 `SetUIBlock` 会重置计时，计时使用 unscaled time。

## 注意事项

1. `ShowUI` 返回代表“逻辑打开完成”，不代表转场结束；需要等动画时用 `AwaitTransition`。
2. `ShowUISync` 只能同步加载；目标窗口正在异步加载时返回 `null` 并输出 Warning。
3. `CloseSelf` 是窗口自身的关闭入口；从外部关闭优先用 `CloseUI<T>`。
4. `GetUI<T>` 可能返回缓存中的关闭实例，不要把它当成“当前可见”。
5. 关闭导航当前页会清空 history；页面返回请用 `GameApp.UI.Navigation.Back()`。
