# 打开、关闭与查询

窗口的开关和查询都走 `IUIService`，业务里通过 `GameApp.UI` 取到它。这一页按 `IUIService` 的真实签名讲打开重载、返回 `null` 的条件、重复打开与并发规则、关闭的缓存与 `force`/`snap` 语义、取消，以及查询与全屏遮挡。页面级前进/返回在 [路由与转场](router.md)，不在这里。

## 适用场景

- 在按钮、流程或启动入口打开一个界面，并决定要不要等开场动画。
- 判断「已打开」「关闭中」「加载中」再次打开会发生什么。
- 关闭后想复用实例（缓存）还是立即释放（`force`），以及关掉缓存中的旧实例。
- 查询界面是否打开、拿运行中的实例、找最高层窗口、从最高层关闭。
- 加载或切场景期间用全屏遮挡挡住输入。

## 使用前提

- 窗口已在 `[Window]` + `[UIRes]` 下声明并注册，见 [窗口声明与生命周期](lifecycle.md)。
- 服务入口用 `GameApp.UI`（内部 `AppServices.Require<IUIService>()`），不要自己缓存 `IUIService` 实例。
- 需要 `using AlicizaX.UI.Runtime;`；异步与取消还要 `using Cysharp.Threading.Tasks;` 和 `using System.Threading;`。
- 关闭后要复用实例，`[Window]` 的 `cacheTime` 必须非 0；要立即释放就用 `force: true`。

## 最短接入

1. 打开窗口并拿到实例。返回值代表 **逻辑打开完成**（`OnOpen` 已执行），不等于开场转场结束：

   ```csharp
   UIHomeWindow home = await GameApp.UI.ShowUI<UIHomeWindow>();
   ```

2. 需要等动画时，再等一次转场：

   ```csharp
   await home.AwaitTransition();

   // 等价的一行写法（UIShowAwaitExtensions）
   await GameApp.UI.ShowUI<UIHomeWindow>().AwaitTransition();
   ```

3. 关闭窗口。窗口内部用 `CloseSelf`，外部用 `CloseUI<T>`：

   ```csharp
   // 写在 UIWindow 子类里
   CloseSelf();

   // 外部关闭，并等关闭转场结束
   await GameApp.UI.CloseUI<UIHomeWindow>().AwaitTransition();
   ```

4. 查询当前状态：

   ```csharp
   bool open = GameApp.UI.IsOpen<UIHomeWindow>();
   UIHomeWindow current = GameApp.UI.GetUI<UIHomeWindow>(); // 可能是缓存中的 Closed 实例
   RectTransform layer = GameApp.UI.GetLayer(UILayer.UI);
   ```

5. 加载遮罩：

   ```csharp
   GameApp.UI.SetUIBlock(2f);   // 2 秒后自动解除
   GameApp.UI.ForceExitBlock(); // 立即解除
   ```

## 核心用法

### 打开：`ShowUI` 的重载

| 签名 | 说明 |
| --- | --- |
| `UniTask<T> ShowUI<T>(object userData = null)` | 泛型打开，最常用 |
| `UniTask<T> ShowUI<T>(CancellationToken, object userData = null)` | 可独立取消的打开 |
| `UniTask<UIWindow> ShowUI(string type, object userData = null)` | 按类型全名打开 |
| `UniTask<UIWindow> ShowUI(string type, CancellationToken, object userData = null)` | 同上，带取消 |
| `UniTask<UIWindow> ShowUI(RuntimeTypeHandle, object userData = null)` | 按运行时类型句柄打开 |
| `UniTask<UIWindow> ShowUI(RuntimeTypeHandle, CancellationToken, object userData = null)` | 同上，带取消 |
| `T ShowUISync<T>(object userData = null)` | 同步加载并打开 |

返回 `null` 的真实条件：

- 类型未注册：`Unknown UI type`（按全名）或 `Unknown UI type handle`（按句柄），需要先注册。
- 类型不是窗口（例如拿 Widget 的类型来开）：`The UI type must be a window`。
- 资源加载失败：`UI resource could not be loaded` / `Failed to load`；预制体上没有 Holder 组件也会失败。
- 加载完成前窗口已被销毁（例如加载中调了 `CloseUI`）。
- 本次调用的 `CancellationToken` 被取消。
- 在 `OnInitialize` / `OnOpen` 里把窗口强制关闭（`CloseSelf(force: true)`），这次 `ShowUI` 就返回 `null`。
- `ShowUISync` 目标窗口正在异步加载：只记录 Warning（`is loading asynchronously, ShowUISync returns null`），返回 `null`，那次异步加载继续。
- `UIService` 已释放：`UIService is not running.`。

### 重复打开与并发

互斥粒度是「单个窗口类型」，不是队列：同层不同类型可以同时打开、关闭。

| 当前状态 | 再次 `ShowUI` | 实际行为 |
| --- | --- | --- |
| 不存在 | 创建并加载 | 多个并发 `ShowUI` 共用同一次加载，返回同一个实例 |
| `Loading` | 只记录参数 | 加载完成后按最新参数打开；`ShowUISync` 此时返回 `null` |
| `Closed`（缓存中） | 复用实例 | 不重新加载、不重跑 `OnInitialize`，只走 `OnOpen` |
| `Opening` | 更新参数 + `OnRefresh` | 不重播开场转场 |
| `Opened` | 更新参数 + `OnRefresh` | 同上 |
| `Closing` | 从当前外观反向打开 | 状态回到 `Opening`；之前那次关闭的 `AwaitTransition()` 会在重新打开完成后结束 |

### 传入参数

`ShowUI` 只接收一个 `object userData`，多个字段请自己定义参数类；传 `null` 时保留原参数。

```csharp
// 弹窗带数据（UIShopGoodsItemViewHolder 里的真实写法）
GameApp.UI.ShowUI<UIBuyAlertWindow>(data).Forget();

// 传枚举/任意对象（UIRecyclerViewTestHubWindow 里的真实写法）
GameApp.UI.ShowUI<UIRecyclerViewExampleWindow>(kind);
```

窗口侧在 `OnOpen` 里读（`UIBuyAlertWindow`）：

```csharp
protected override void OnOpen()
{
    _goodsData = UserData as ShopGoodsData;
    RefreshView();
}
```

### 取消是「取消等待」，不是取消加载

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;

async UniTask<UIHomeWindow> OpenAsync(CancellationToken token)
{
    UIHomeWindow window = await GameApp.UI.ShowUI<UIHomeWindow>(token);
    // 取消时 window == null，但窗口仍会继续加载并打开
    return window;
}
```

- 调用前令牌已取消：直接返回 `null`，不会发起加载。
- 加载中取消：只有本次调用返回 `null`，窗口照常加载与打开，其他等待者不受影响。
- 想真正取消加载：用 `CloseUI`，加载中关闭会取消加载并销毁窗口。

### 关闭：`force` / `snap` / 缓存

```csharp
// 窗口内部（UIWindow 的 protected 方法）
CloseSelf();
await CloseSelf().AwaitTransition();
CloseSelf(snap: true);   // 转场直接落终态
CloseSelf(force: true);  // 关完即销毁

// 外部
GameApp.UI.CloseUI<UIHomeWindow>();
GameApp.UI.CloseUI(typeof(UIHomeWindow).TypeHandle);
await GameApp.UI.CloseUI<UIHomeWindow>(force: true, snap: true).AwaitTransition();
```

| 参数 | 含义 |
| --- | --- |
| `force: true` | 关闭后直接销毁，不进缓存；对缓存中的窗口则直接销毁 |
| `snap: true` | 关闭转场直接落到终态（仍会触发一次关闭流程） |
| 默认 | 播关闭转场，结束后按 `cacheTime` 决定进缓存或销毁 |

关闭时各状态的行为：

| 当前状态 | `CloseUI` 行为 |
| --- | --- |
| `Loading` | 取消加载并销毁（`force`/`snap` 无意义），进行中的 `ShowUI` 返回 `null` |
| `Opening` / `Opened` | `OnClose` → 关闭转场 → `Closed` → 进缓存或销毁 |
| `Closing` | 重复调用不会重播 `OnClose`，只把 `force` 标记合并进去 |
| `Closed`（缓存中） | 默认无操作；`force: true` 会销毁缓存实例 |
| 不存在 / 已销毁 | 记录 `Close ignored because the window is not open and not cached.`，返回默认 `UIViewTransition`（`await` 立即完成） |

- 缓存时长从 **关闭转场结束** 开始计算，正数按秒（unscaled 计时），负数常驻，0 关闭即销毁。
- 关闭导航当前页会清空导航 history，见 [路由与转场](router.md)。
- 关闭中重新打开会取消待销毁标记（`force` 只在本次关闭流程有效）。

### 查询

| API | 真实语义 |
| --- | --- |
| `IsOpen<T>()` / `IsOpen(RuntimeTypeHandle)` | 仅 `Opening` / `Opened` 为 `true`，不代表开场动画播完 |
| `GetUI<T>()` | `Opening` / `Opened` / 缓存的 `Closed` 返回实例；`Loading`、`Closing`、已销毁返回 `null` |
| `GetLayer(UILayer)` | 返回该层的根 `RectTransform`，窗口 Holder 的父节点 |
| `TryGetTopVisibleHolder(predicate, out holder)` | 从最高层向下找第一个 `Opening`/`Opened` 且满足谓词的 Holder；`predicate` 为 `null` 表示不过滤 |
| `TryCloseTopAsync(predicate, force)` | 从最高层关闭第一个匹配的逻辑打开窗口，**等关闭转场结束**后返回 `true`；没有匹配返回 `false`；`predicate` 为 `null` 表示任意窗口 |

```csharp
if (GameApp.UI.TryGetTopVisibleHolder(null, out UIHolderObjectBase top))
{
    // top 是当前最高层的 Holder
}

bool closed = await GameApp.UI.TryCloseTopAsync(
    handle => handle.Equals(typeof(UIBuyAlertWindow).TypeHandle),
    force: true);
```

### 全屏输入遮挡

```csharp
GameApp.UI.SetUIBlock(2f);    // 显示并开始计时，期间拦截点击
GameApp.UI.SetUIBlock(5f);    // 重复调用会重置计时
GameApp.UI.ForceExitBlock();  // 立即解除
```

遮挡节点是 `UIService` 构造时创建在 Canvas 下的 `LayerBlock`（嵌套 `Canvas`，排序值 `short.MaxValue`，挂只拦射线的 `UIBlock`），默认不激活。计时走 unscaled 时间，到点自动隐藏；它只挡输入，不挡渲染。

## 常见配置

| 需求 | 选法 |
| --- | --- |
| 普通弹窗，关掉就释放 | `[Window(UILayer.Popup, 0)]` + 默认 `CloseUI` |
| 需要反复开关、保留状态 | `cacheTime` 设正数（如 30）或 `-1` 常驻 |
| 手动释放缓存实例 | `GameApp.UI.CloseUI<T>(force: true)` |
| 跳过动画直接收尾 | `snap: true` |
| 加载期间同步打开 | `ShowUISync<T>()`，注意目标窗口不能在异步加载中 |
| 只关最高层某个窗口 | `TryCloseTopAsync(handle => ...)` |
| 临时挡输入 | `SetUIBlock(duration)`，默认时间到了自动解除 |

## 容易踩的坑

1. `ShowUI` 返回不等于转场结束，`await GameApp.UI.ShowUI<T>()` 之后画面可能还在动画中；要等画面稳定就再 `await window.AwaitTransition()`。
2. `await CloseUI<T>().AwaitTransition()` 的落点不一定是「已关闭」：关闭中重新打开时，这个等待会在窗口重新变成 `Opened` 后结束；需要确认结果请用 `IsOpen` / `GetUI` 复核。
3. `ShowUISync` 在目标窗口正在异步加载时返回 `null` 并只打 Warning，不代表资源失败；应等那次异步 `ShowUI`。
4. `GetUI<T>()` 可能返回缓存中 `Closed` 的实例，不能当成「可见」；可见性判断用 `IsOpen` 或 `Visible`。
5. 缓存中的窗口再次关闭是 no-op，不释放内存；要释放必须 `force: true`，否则会一直留到缓存到期。
6. 加载中关闭会导致这次 `ShowUI` 返回 `null`——这是正常路径（取消加载并销毁），先 `await` 再判空即可，不要当成异常处理。

## API 速查

| 成员 | 返回 | 说明 |
| --- | --- | --- |
| `ShowUI<T>(object)` | `UniTask<T>` | 异步打开，`OnOpen` 后返回 |
| `ShowUI<T>(CancellationToken, object)` | `UniTask<T>` | 令牌只取消本次等待 |
| `ShowUI(string, object)` / `ShowUI(RuntimeTypeHandle, object)` | `UniTask<UIWindow>` | 按全名/句柄打开 |
| `ShowUISync<T>(object)` | `T` | 同步加载并打开 |
| `CloseUI<T>(bool force, bool snap)` | `UIViewTransition` | 关闭，可等待转场 |
| `CloseUI(RuntimeTypeHandle, bool, bool)` | `UIViewTransition` | 同上 |
| `IsOpen<T>()` / `IsOpen(RuntimeTypeHandle)` | `bool` | `Opening`/`Opened` |
| `GetUI<T>()` | `T` | 打开中或缓存中的实例 |
| `GetLayer(UILayer)` | `RectTransform` | 层根节点 |
| `TryGetTopVisibleHolder(Predicate<UIHolderObjectBase>, out UIHolderObjectBase)` | `bool` | 最高层可见 Holder |
| `TryCloseTopAsync(Predicate<RuntimeTypeHandle>, bool force)` | `UniTask<bool>` | 关最高层匹配窗口并等转场 |
| `SetUIBlock(float)` / `ForceExitBlock()` | `void` | 全屏遮挡 |
| `UICamera` / `UICanvasRoot` / `Navigation` | — | 相机、Canvas 根、页面导航 |
