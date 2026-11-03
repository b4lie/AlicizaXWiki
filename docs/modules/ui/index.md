# UI 模块

UI 模块负责窗口创建、显示、关闭、层级、缓存、生命周期、Widget、Tab 窗口、页面导航和 UI 事件自动解绑。

主要代码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI`
- 示例业务代码：`Client/Assets/Scripts/Hotfix/GameLogic/UI`

常用入口：

```csharp
using AlicizaX;
using AlicizaX.UI.Runtime;

IUIService ui = GameApp.UI;          // 门面（IService + 下列 4 组细分接口）
IUIPanelService panel = GameApp.UI;   // 面板开关/查询
IUINavigationService router = GameApp.UI; // 页面导航
IUIRootService root = GameApp.UI;     // 根节点/相机/层级
IUIBlockService block = GameApp.UI;   // 全屏遮挡层
```

## 使用前提

启动场景需要挂载并初始化这些组件：

- `TimerComponent`
- `ResourceComponent`
- `UIComponent`

`UIComponent` Inspector 中需要配置 `uiRoot` 预制体。运行时会实例化 UI 根节点（必须含 `Canvas` 且 `worldCamera` 有效），注册 `IUIService`，并按 `UILayer` 创建层级节点；`TimerComponent` 必须已注册，否则 UI 初始化失败。

UI 模块没有自己的 MonoBehaviour 驱动，**所有每帧工作由 `AppServiceRoot.Update → ServiceWorld.Tick` 驱动**（窗口更新与转场）。脱离该驱动（例如自建测试环境）时需要显式调用 `IServiceTickable.Tick(deltaTime)`。

## UI 层级

```csharp
public enum UILayer
{
    Background = 0,
    Scene = 1,
    UI = 2,
    Popup = 3,
    Tips = 4,
    Top = 5,
    All = 6,
}
```

约定：

- `Background`：背景层。
- `Scene`：场景 2D 信息，例如血条、飘字。
- `UI`：普通主界面、全屏界面。
- `Popup`：弹窗层。
- `Tips`：提示、Toast、跑马灯。
- `Top`：最高层，例如新手引导遮罩。
- `UILayer.All` 不是可显示层，只用于数组长度。

`UIMetaRegistry.Register` 在注册期校验层级，传入 `UILayer.All` 会抛 `ArgumentOutOfRangeException`。

每个层最多 **200** 个窗口（`LAYER_DEEP / WINDOW_DEEP`）；超出会在编辑器中断言，避免 Canvas `sortingOrder` 与下一层重叠。置顶只给目标窗口分配新序号，关闭中间窗口不改变其他窗口的 `sortingOrder`；序号用尽时整层重编号一次，因此层内窗口接近 200 个时置顶会退化为 O(层内窗口数)。

## 元数据

窗口逻辑类使用 `WindowAttribute` 描述显示层级和缓存时间：

```csharp
[Window(UILayer.UI, 30)]
public sealed class HomeWindow : UIWindow<ui_HomeWindow>
{
    protected override void OnInitialize()
    {
        baseui.BtnClose.onClick.AddListener(() => CloseSelf().Forget());
    }
}
```

```csharp
public WindowAttribute(UILayer windowLayer, int cacheTime = 0);
```

- `windowLayer`：窗口层级，不能是 `UILayer.All`。
- `cacheTime`：`-1` 永久缓存，`0` 不缓存，`>= 1` 按秒缓存（关场动画结束后开始计时）。

需要每帧更新时添加 `[UIUpdate]`：

```csharp
[UIUpdate]
[Window(UILayer.UI)]
public sealed class BattleHudWindow : UIWindow<ui_BattleHudWindow>
{
    protected override void OnUpdate(float deltaTime) => RefreshHpBar(deltaTime);
}
```

- 只有带 `[UIUpdate]` 的窗口/Widget 才会收到 `OnUpdate`，每帧成本只与这些视图的数量有关。视图必须 `Effective`，自身和所在窗口都必须处于 `Opened`。
- 父节点先于子节点更新；同级 Widget 按进入更新链表的顺序更新，不是创建顺序。不要依赖同级之间的顺序。
- 本帧新进入更新或转场链表的视图从下一帧开始执行；本帧移除的视图不再执行。
- `deltaTime` 来自 `AppServiceRoot.Update`（`Time.deltaTime`）。

Holder 的资源位置使用 `[UIRes]`（写在 Holder 类上，不写在窗口逻辑类上）：

```csharp
[UIRes(ui_LoginWindow.ResTag, EUIResLoadType.AssetBundle)]
public class ui_LoginWindow : UIHolderObjectBase { ... }
```

注册入口：

- 启动期集中注册：`UIMetaRegistry.Register(uiType, holderType, layer, cacheTime, needUpdate, create)` 与 `UIResRegistry.Register(holderType, location, loadType)`。`create` 可省略，省略时用反射创建实例。
- 生成器（`Plugins/SourceGenerators/UI`）在每个程序集生成 `UIGeneratedRegistry.RegisterAll()`，包含 Holder 资源、窗口元数据和实例工厂。生成器修改后要运行 `Build.ps1` 更新 DLL。
- `RegisterAll` 默认由 `RuntimeInitializeOnLoadMethod` 调用。特殊小游戏平台不会调用它时，未注册类型走反射兜底（首次打开打 `UI not pre-registered` Warning）；需要关注性能时，在所属程序集入口手动调用 `AlicizaX.UI.Runtime.UIGeneratedRegistry.RegisterAll()`。
- 生成器诊断：`UI004`（缺少可访问的无参构造函数）、`UI005`（`[Window]` 使用 `UILayer.All`）是编译错误；`UI003`（生成代码无法访问的类型）是 Warning，该类型走反射。

## 生命周期

初始化生命周期内联在 `UIBase` 中，通过虚方法重写：

```csharp
protected virtual void OnInitialize();
protected virtual void OnOpen();
protected virtual void OnRefresh();
protected virtual void OnClose();
protected virtual void OnDestroy();
protected virtual void OnUpdate(float deltaTime);
protected virtual void OnRegisterEvent(EventListenerProxy proxy);
```

语义：

- `OnInitialize`：资源绑定后、首次打开前执行一次；缓存复用不会再次执行。
- `OnOpen`：逻辑打开。钩子执行期间 `IsOpen == false`，返回后才变为 true；开场动画尚未开始。
- `OnRefresh`：已 `Opened` 后再次 `Show` 传参时调用，调用前参数已更新。
- `OnClose`：逻辑关闭（`IsOpen` 已为 false），关场动画尚未完成。
- `OnDestroy`：实例销毁前调用一次。
- `OnUpdate`：仅 `[UIUpdate]` + `Opened` + `Effective` 时每帧调用。
- 业务钩子抛异常会被捕获并记录，生命周期继续推进。

## 打开 / 关闭契约（摘要）

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

## UIRouter（页面导航）

入口：`IUINavigationService router = GameApp.UI;`（`IUIService` 也直接暴露这些方法）。

### 建立 root

```csharp
await GameApp.UI.NavigateTo<UIHomeWindow>();          // 第一条 history
GameApp.UI.ResetHistory();                            // 只清 history，不关 UI
GameApp.UI.SyncFromCurrentUI(typeof(UIHomeWindow));   // 把当前已 Opened 的 UI 设为 root
```

### 前进导航

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

### 返回

```csharp
await GameApp.UI.Back();                 // history.Count < 2 -> NotFound
await GameApp.UI.BackToRoot();
await GameApp.UI.BackTo<UIHomeWindow>();               // 未命中且 openIfMissing=true -> ResetTo
await GameApp.UI.BackTo<UIHomeWindow>(openIfMissing: false);
await GameApp.UI.CloseCurrent(force: false);           // 关闭当前页
```

`CloseCurrent` 规则：关闭当前页并清空 history。需要回到上一页时使用 `Back`。

### 状态

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

## Widget

```csharp
protected T CreateWidgetSync<T>(Transform parent, bool visible = true) where T : UIWidget;
protected T CreateWidgetSync<T>(UIHolderObjectBase holder, bool destroyHolderOnDispose = true, bool visible = true) where T : UIWidget;
protected UniTask<T> CreateWidgetAsync<T>(Transform parent, bool visible = true, CancellationToken cancellationToken = default);
protected UniTask<UIWidget> CreateWidgetAsync(string typeName, Transform parent, bool visible = true, CancellationToken cancellationToken = default);
public UniTask RemoveWidget(UIWidget widget);

widget.Open(params object[] userDatas);
widget.Close();
await widget.Destroy();
await widget.AwaitTransition();
```

语义：

- `CreateWidget*`：资源 + Init +（`visible` 时）逻辑 `OnOpen`；开场转场后台推进。
- Widget 必须属于父窗口 Holder 的 Transform 子树，否则创建失败并报 Error。
- 每个 Widget 自己持有加载取消源：父窗口销毁会取消所有子创建；Widget 自身销毁也会立即取消自己的加载。
- `Open`：已 `Opened` 只刷新参数；`Opening` 返回成功；`Closing` 返回 false（需要重开请在关闭完成后再调）。
- `Close`：`Initialized`（从未打开）时是 no-op；其余状态走完整关闭流程。
- 可见性：`IsVisible = Visible && Parent.IsVisible`；父级不可见时子 Widget 逻辑状态不变，只暂停事件/更新。

## Tab 窗口

```csharp
public abstract class UITabWindow<T> : UIWindow<T> where T : UIHolderObjectBase
{
    protected void InitTabVirtuallyView<TTab>(Transform parent = null) where TTab : UIWidget;
    public UniTask<UIWidget> SwitchTab(int index, params object[] userDatas);
}
```

- Tab 首次选中时异步创建，之后复用实例。
- 切换是 latest wins：旧请求返回 `null`，旧 Tab 关闭完成后才打开新 Tab。
- Tab 加载中窗口被关闭/销毁时请求会以 `null` 结束，迟到资源被销毁。

## UI 事件自动解绑

```csharp
protected override void OnRegisterEvent(EventListenerProxy proxy)
{
    proxy.AddUIEvent<LocalizationChangeEvent>(OnLanguageChanged);
}

private void OnLanguageChanged(in LocalizationChangeEvent evt) => RefreshText();
```

- `OnRegisterEvent` 在每个视图的生命周期内只调用一次，发生在首次 `Effective` 时。之后不可见或关闭只暂停订阅，重新 `Effective` 时恢复，不再调用它。依赖每次打开的逻辑写在 `OnOpen` 或 `OnRefresh`。
- 销毁时释放订阅并把代理归还 `MemoryPool`。
- 在 `OnRegisterEvent` 内销毁自身是允许的：事件代理会延迟到回调返回后释放，不会泄漏句柄。

## Holder 生命周期事件

Holder 暴露 6 个事件：

```csharp
OnWindowInitEvent / OnWindowBeforeShowEvent / OnWindowAfterShowEvent
OnWindowBeforeClosedEvent / OnWindowAfterClosedEvent / OnWindowDestroyEvent
```

- 顺序与调用点固定；单个订阅者抛异常只记录日志，不影响其余订阅者。
- 增删订阅时复制回调数组，invoke 路径 0 B。回调内增删订阅从下一次触发开始生效。
- 输入系统等外部模块通过它感知窗口显示/隐藏；不要在回调里做重活。

## 手动创建 Holder

少数场景可手动创建 Holder，例如临时 Tips 多实例，生命周期需要业务自行维护：

```csharp
ui_UILogicTestAlert holder = await UIHolderFactory.CreateUIHolderAsync<ui_UILogicTestAlert>(parent);
ui_UILogicTestAlert sync = UIHolderFactory.CreateUIHolderSync<ui_UILogicTestAlert>(parent);
```

- 资源加载失败或预制体缺 Holder 时返回 `null`，已实例化的对象会被销毁。
- 手动 Holder 没有窗口 owner，不会自动解绑事件或驱动转场。

## 全屏遮挡层

```csharp
GameApp.UI.SetUIBlock(2f);   // 显示 2 秒，期间拦截点击
GameApp.UI.ForceExitBlock();
```

- 遮挡节点由 UI 服务创建，`sortingOrder = short.MaxValue`，位于所有层级之上。
- 到期或强制退出会移除计时器；重复 `SetUIBlock` 会重置计时。

## 分配预算

以下数据只来自 Unity 2022.3.62f3 编辑器（EditMode 和 PlayMode，测试 `UIAllocationCases`、`UIPerformanceTests`、`UIHotPathTests`），不代表 IL2CPP、HybridCLR 和小游戏平台。

常规热路径：

| 路径 | 实测 | 用例 |
| --- | --- | --- |
| 稳态 `Tick`（1、16、512 个更新视图） | **0 B** | `tick-steady-*` |
| 活动 tick 转场（1、16、64 个） | **0 B** | `tick-transitions-*` |
| 缓存窗口关闭、重开，首次和稳态 | **0 B** | `cached-*-first`、`cached-cycle-*` |
| 带缓存时长的窗口开关，稳态 | **0 B**（首次关闭见下表） | `timed-cache-cycle` |
| tick 转场开关，首次和稳态 | **0 B** | `transition-cycle-*` |
| 同步完成的 `ShowUI<T>`（已打开、已缓存），新 `T` 首次调用和稳态 | **0 B**（进程内首次见下表） | `async-show-*` |
| 关闭时有 1–2 个等待者 | **0 B** | `close-two-waiters` |
| Widget 无参 `Open`、`Close`，有无 tick 转场 | **0 B** | `widget-toggle-*` |
| 已加载 Tab 切换 | **0 B** | `tab-switch-loaded` |
| 方法组注册后的事件暂停和恢复 | **0 B** | `event-resume-method-group` |
| 复用代理容量、业务缓存委托时的 `AddUIEvent` | **0 B** | `event-add-cached-delegate` |
| Holder 事件触发、只读查询 | **0 B** | `holder-event-invoke`、`query-paths` |
| 全部 `UITransitionPreset` 的 `Advance` | **0 B** | `preset-advance-*` |

低频路径：

| 路径 | 实测或对象 | 触发条件 | 释放责任 |
| --- | --- | --- | --- |
| 进程内首次 `ShowUI` | 最多 244 B，4 次分配；PlayMode 实测 0 B，随首次调用的时机变化 | 进程内一次，UniTask 和泛型静态初始化 | 进程持有 |
| 创建 Widget（不含 Prefab 实例化成本的差异） | 约 1.5 KB，12–13 次分配，含测试夹具创建 GameObject | 每实例一次 | 随视图回收 |
| 首次异步加载 | 约 1.0–4.8 KB，16–54 次分配；高值出现在它先于其他异步路径运行时 | 每次实际加载 | 加载结束释放；runner 由 UniTask 池持有 |
| 带缓存时长的窗口首次关闭 | 646 B，8 次分配，来源未逐个确认；`AddTimer` 路径没有实例级 `new` | 实测首次；之后 0 B | 未确认 |
| 视图首次有等待者 | 64 B，1 次分配（2 项等待数组） | 每个视图首次 `await` 转场 | 随视图回收 |
| 事件代理首次注册 | 160 B 的订阅数组（4 项，不够时翻倍）；每种事件类型另有一个静态委托；业务写方法组时再加委托 | 代理首次注册；每种事件类型一次 | 数组随代理留在对象池；静态委托进程持有 |
| 带一个值类型参数打开 | 60 B，2 次分配（数组和装箱） | 用户操作级别 | 关闭或替换时释放 |
| 导航 | `NavigationCommand`、UCS、CTS、`UIRouteEntry`，有参数时再加数组 | 每次导航 | 命令结束释放，条目随 history |
| 异步转场源 | CTS 和 UniTask runner | 非 tick 转场每次播放 | 播放结束释放 |
| 未完成任务上调用 `Forget()` | UniTask 池化的 `StateTuple` | 业务调用 `CloseSelf().Forget()` 且有转场 | UniTask 池 |
| Tab 排队切换 | `TabRequest`、UCS | 目标未加载，或上一 Tab 还在关闭 | 请求结束释放 |
| 钩子内重入打开 | 一个 `UniTaskCompletionSource<bool>` | 在钩子内调用 `ShowUI` | 逻辑打开完成释放 |
| 同一转场超过 2 个等待者 | 一次数组扩容 | 第 3 个等待者 | 随视图回收 |
| 反射注册兜底 | 特性读取、注册表项和创建委托，打 Warning | 未注册类型首次使用 | 注册表持有到进程结束 |

每帧或每次高频调用不应出现新的托管分配；任何无法消除的分配都必须在上表中登记触发条件、数量、频率与释放责任。

Holder 所在 GameObject 被外部销毁时，所属视图立即按 `DestroyNow` 释放。例外：从未激活过的 Holder 不会收到 `OnDestroy`，视图收不到通知，之后访问会抛 `MissingReferenceException`。Prefab 根节点应保持激活。

隐藏只切显示状态：窗口切 layer 和 `Canvas.enabled`，Widget 切 `CanvasGroup`。隐藏和缓存期间，节点上的 MonoBehaviour 继续运行，也不会触发 `OnEnable`、`OnDisable`，进出缓存也不触发 `OnTransformParentChanged`。带 `Update`、粒子或 Animator 的组件，要在 `OnClose` 和 `OnOpen` 里自行停止和恢复。

## 并发模型

互斥粒度是**单个 UI 类型（per-type）**，不是整层队列：

- 同层不同类型：`Show` / `Close` 可并行。
- 同类型：已打开/打开中再次 `Show` 只刷 latest userData；关闭中 `Show` 返回 `null` + Warning；打开中 `Close` 取消加载并走关闭路径。
- 导航是单队列事务：同一时刻只有一个命令执行，最多排一个 pending；新命令取消未提交的旧命令。

## 日志与告警语义

原则：**取消静默，故障告警**。

- 预期取消/打断（加载中被 Close、Opening 被新 Show 顶替、导航被新导航顶替）不打 Warning。
- 资源加载失败、预制体缺 Holder、类型未注册且无法反射、层级非法、历史超限等真实错误才 `Error`/`Warning`。
- `[UI] UIService is not running.`：在未初始化或已销毁后调用 UI 服务。
- `[UI] UI not pre-registered`：该类型未由生成器注册，走反射兜底（应修复生成器配置）。

## 注意事项

1. `WindowAttribute` 写在窗口逻辑类上；`UIResAttribute` 写在 Holder 类上。
2. `OnInitialize` 只执行一次（缓存复用不再执行）；`OnUpdate` 需要 `[UIUpdate]`。
3. `ShowUI` / `ShowUISync` 以逻辑打开完成（`IsOpen`）为完成点；开场动画是后台任务，需要时 `await AwaitTransition()`。
4. 页面用导航 API，弹窗/Tips 用 `ShowUI/CloseUI`；关闭当前页会清空 history。
5. `CurrentEntry` 与 `UserDatas` 都是只读视图，不要写入。
6. 每层最多 200 个窗口；每个窗口的 Canvas 使用 `overrideSorting`，深度由服务排序。
7. 转场默认跟随 unscaled time；自定源必须实现 `Snap`，否则强制关闭/回滚会停在半途。
8. UI 的每帧更新与转场依赖 `AppServiceRoot` 的 `Tick`；测试或自建环境需要显式驱动。
9. UI 模块依赖 `TimerComponent`（缓存计时、遮挡层）；启动顺序错误会初始化失败。
10. 不要在 `OnOpen`/`OnClose` 里另起一套动画，会绕过 `skipTransition`、回滚与缓存前的关态钉死。
