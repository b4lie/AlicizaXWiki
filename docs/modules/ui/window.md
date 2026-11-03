# 打开 / 关闭契约（摘要）

| 路径 | 完成点（await / 返回） | 开场动画 | 关场动画 |
| --- | --- | --- | --- |
| `ShowUI` | 资源 + Init + 逻辑 `OnOpen`（进入 `Opening`，`IsOpen` 已是 true） | 后台并行 | - |
| `ShowUISync` | 同步资源 + `OnInitialize` + 逻辑 `OnOpen`，返回时已 `Opening/Opened` | 后台并行 | - |
| `CloseUI` | 逻辑 `OnClose` + 关场转场完成后再 Finalize/Cache | - | await 关场后再入缓存 |
| `CloseSelf` | 同 `CloseUI`（窗口自身） | - | 同上 |
| `AwaitTransition` | 当前开/关转场结束 | opt-in | opt-in |

补充规则：

- **同类型**已在 `Opening` / `Opened` / 进行中：只刷新 userData（latest wins），不重新走完整打开；`Opening` 期间不会反复 `OnRefresh`，开动画结束后合并刷新一次。
- **同类型**正在 `Closing`：`ShowUI` 和 `ShowUISync` 都返回 `null` 并打 Warning。导航 API 会等待关闭完成后再打开目标页。
- **同层不同类型**可并行打开/关闭；没有整层 FIFO 命令队列。
- 页面导航用 `NavigateTo/Back`；叠加弹窗/独立面板用 `ShowUI/CloseUI`。同一逻辑页不要混用两条路径。
- 缓存复用不会重新触发 `OnInitialize`，只重新 `OnOpen` 并按需 `OnRefresh`；`CacheTime == 0` 或 `force` 关闭直接销毁。

## 打开 UI

```csharp
HomeWindow home = await GameApp.UI.ShowUI<HomeWindow>();
ItemTipsWindow tips = await GameApp.UI.ShowUI<ItemTipsWindow>(10001, "from_bag");
HomeWindow same = GameApp.UI.ShowUISync<HomeWindow>("startup");
GameApp.UI.ShowUI<UIRecyclerViewExampleWindow>(kind); // 值类型参数会装箱，见「分配预算」
```

可用重载：

```csharp
UniTask<T> ShowUI<T>(params object[] userDatas) where T : UIWindow;
UniTask<T> ShowUI<T>(CancellationToken cancellationToken, params object[] userDatas) where T : UIWindow;
UniTask<UIWindow> ShowUI(string type, params object[] userDatas);
UniTask<UIWindow> ShowUI(string type, CancellationToken cancellationToken, params object[] userDatas);
UniTask<UIWindow> ShowUI(RuntimeTypeHandle handle, params object[] userDatas);
UniTask<UIWindow> ShowUI(RuntimeTypeHandle handle, CancellationToken cancellationToken, params object[] userDatas);
T ShowUISync<T>(params object[] userDatas) where T : UIWindow;
```

- `await ShowUI<T>()` 在失败/取消时返回 `null`；需要区分原因时看日志（资源失败 -> Error）。
- 每个 `Show` 请求有独立取消：最后一个等待者取消后才撤销资源加载；其余等待者继续共享同一次加载，参数取“最后一个非空参数”。
- `ShowUISync` 只做同步加载与同步初始化；依赖异步资源的 UI 请用 `ShowUI`。

接收参数：

```csharp
public sealed class ItemTipsWindow : UIWindow<ui_ItemTipsWindow>
{
    protected override void OnOpen()
    {
        int itemId = UserData is int value ? value : 0;
        ReadOnlyMemory<object> args = UserDatas;  // 只读视图，禁止写入
    }
}
```

## 关闭 UI

```csharp
// 窗口自身
protected UniTask CloseSelf(bool force = false, bool skipTransition = false);

// 外部
UniTask CloseUI<T>(bool force = false, bool skipTransition = false) where T : UIWindow;
UniTask CloseUI(RuntimeTypeHandle handle, bool force = false, bool skipTransition = false);
bool closed = await TryCloseTopAsync(handle => handle.Equals(typeof(SettingsWindow).TypeHandle), force: false);
```

- `force: true` 跳过缓存直接销毁。
- `skipTransition: true` 仅对尚未开始的关场动画生效；已开始的转场会被取消。
- 关闭完成后才入缓存；缓存实例不会带着未播完的关场动画进 Cache。
- 对缓存中的窗口再次 `CloseUI(force: false)` 是 no-op；`force: true` 会销毁缓存实例。
- 关闭当前导航页会清空 history（与 Router 语义一致）。

## 查询 UI

```csharp
bool opened = GameApp.UI.IsOpen<LoginWindow>();
LoginWindow login = GameApp.UI.GetUI<LoginWindow>();   // Opening/Opened/Cached；Closing 或已销毁返回 null
RectTransform uiLayer = GameApp.UI.GetLayer(UILayer.UI);
bool found = GameApp.UI.TryGetTopVisibleHolder(null, out UIHolderObjectBase holder);
```

- `IsOpen` 只表示逻辑打开（`OnOpen` 后至 `Closing` 前），不表示开场动画已结束。
- `TryGetTopVisibleHolder` 从最高层、最高栈位开始查找可见且满足谓词的 Holder；谓词异常直接抛出。

## 并发模型

互斥粒度是**单个 UI 类型（per-type）**，不是整层队列：

- 同层不同类型：`Show` / `Close` 可并行。
- 同类型：已打开/打开中再次 `Show` 只刷 latest userData；关闭中 `Show` 返回 `null` + Warning；打开中 `Close` 取消加载并走关闭路径。
- 导航是单队列事务：同一时刻只有一个命令执行，最多排一个 pending；新命令取消未提交的旧命令。

## 全屏遮挡层

```csharp
GameApp.UI.SetUIBlock(2f);   // 显示 2 秒，期间拦截点击
GameApp.UI.ForceExitBlock();
```

- 遮挡节点由 UI 服务创建，`sortingOrder = short.MaxValue`，位于所有层级之上。
- 到期或强制退出会移除计时器；重复 `SetUIBlock` 会重置计时。

## 日志与告警语义

原则：**取消静默，故障告警**。

- 预期取消/打断（加载中被 Close、Opening 被新 Show 顶替、导航被新导航顶替）不打 Warning。
- 资源加载失败、预制体缺 Holder、类型未注册且无法反射、层级非法、历史超限等真实错误才 `Error`/`Warning`。
- `[UI] UIService is not running.`：在未初始化或已销毁后调用 UI 服务。
- `[UI] UI not pre-registered`：该类型未由生成器注册，走反射兜底（应修复生成器配置）。
