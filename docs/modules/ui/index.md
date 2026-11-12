# UI 模块

UI 模块负责窗口的创建、打开、关闭、层级、缓存、转场和生命周期，也提供 Widget、Tab 窗口、页面导航和 UI 事件自动解绑。

源码位置：

- 运行时代码：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI`
- 示例业务代码：`Client/Assets/Scripts/Hotfix/GameLogic/UI`

## 先跑起来

启动场景需要：

- `TimerComponent`
- `ResourceComponent`
- `UIComponent`

`UIComponent` 的 `UI Root Prefab` 指向一个包含 `Canvas` 且 `worldCamera` 已配置的 UI 根预制体。运行时它实例化根节点、按 `UILayer` 创建层级，并注册 `IUIService`。`TimerComponent` 必须先注册，否则 UI 初始化失败。

`UIComponent` 还提供：

| 字段 | 用途 |
| --- | --- |
| `UI Root Prefab` | UI 根预制体，必须包含 `Canvas` 和有效的 `worldCamera` |
| `Orthographic` | 是否使用正交相机；正交时沿用预制体上的裁剪面设置 |
| `Near Clip Plane` / `Far Clip Plane` | 透视相机的裁剪面 |

## 第一个窗口

窗口逻辑类继承 `UIWindow<Holder>`，用 `[Window]` 声明层级和缓存时间：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

[Window(UILayer.UI, UICacheTime.Forever)]
public sealed class HomeWindow : UIWindow<ui_HomeWindow>
{
    protected override void OnInitialize()
    {
        baseui.BtnShop.onClick.AddListener(() => GameApp.UI.ShowUI<ShopWindow>().Forget());
    }
}
```

Holder 由生成器生成，并在类上带 `[UIRes]`，不需要手写加载代码。

打开和关闭：

```csharp
await GameApp.UI.ShowUI<HomeWindow>();

GameApp.UI.CloseUI<HomeWindow>();

// 窗口逻辑类内部可以调用 CloseSelf()。
```

`ShowUI` 在窗口逻辑打开（`OnOpen` 执行完）后返回，不等开场转场；需要等动画时用 `AwaitTransition`：

```csharp
HomeWindow home = await GameApp.UI.ShowUI<HomeWindow>();
await home.AwaitTransition();
```

## 入口

```csharp
using AlicizaX.UI.Runtime;

IUIService ui = GameApp.UI;                       // 打开/关闭/查询/层级/遮挡
IUINavigationService nav = GameApp.UI.Navigation; // 页面导航
```

导航页面和普通弹窗分开用：

- 页面切换：`GameApp.UI.Navigation.NavigateTo<T>()`、`Back()` 等。
- 弹窗、Tips、独立面板：`GameApp.UI.ShowUI<T>()`、`CloseUI<T>()`。

不要把同一个页面同时交给导航和 `ShowUI` 管理。

## 层级

```csharp
public enum UILayer
{
    Background = 0,
    Scene = 1,
    UI = 2,
    Popup = 3,
    Tips = 4,
    Top = 5,
}
```

| 层 | 用途 |
| --- | --- |
| `Background` | 背景 |
| `Scene` | 2D 场景信息，例如血条、飘字 |
| `UI` | 普通主界面、全屏界面 |
| `Popup` | 弹窗，可同时存在多个 |
| `Tips` | Toast、确认框、跑马灯 |
| `Top` | 新手引导、最高层遮罩 |

每层最多 200 个窗口；超过会在编辑器中断言。窗口的 `sortingOrder` 由 UI 服务按层级和打开顺序维护，业务不要手动改 Canvas 排序。

## 调试

Play Mode 下选中 `UIComponent`：

- `UI RuntimeDebug`：Open / Cache / Update / Block 数量、层级窗口、缓存列表、运行时异常提示。
- `Navigation RuntimeDebug`：当前页面、`CanBack` 和 history。

这两个面板只读，不会修改运行状态。

## 本模块页面

| 页面 | 内容 |
| --- | --- |
| [窗口声明与生命周期](lifecycle.md) | `[Window]` / `[UIRes]`、缓存、生命周期钩子、Holder 事件、手动注册 |
| [打开、关闭与查询](window.md) | `ShowUI` / `CloseUI`、参数、取消、查询、全屏遮挡 |
| [路由与转场](router.md) | `Navigation` 前进/返回、路由结果、转场组件与自定义转场 |
| [Widget 与 Tab](widget.md) | Widget 创建与开关、Tab、UI 事件自动解绑 |
| [性能与缓存](performance.md) | 更新与缓存策略、避免分配、Profiler 观察 |

## 注意事项

1. `OnInitialize` 只在首次创建时执行一次；缓存复用只重新执行 `OnOpen`。
2. 想收到 `OnUpdate`，在窗口/Widget 中重写 `OnUpdate` 即可，不需要额外标签。
3. `ShowUI` 只传一个 `userData` 对象；多个字段请自己定义参数类。
4. `CloseSelf` 返回 `UIViewTransition`，需要等关场动画时再 await。
5. 页面导航使用 `GameApp.UI.Navigation`；`CloseUI` 从导航之外关闭当前页会清空 history。
6. 每帧更新和转场由 `AppServiceRoot` 的 `Tick` 驱动；自建测试环境需要显式调用 `IServiceTickable.Tick`。
