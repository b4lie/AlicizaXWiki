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

## 本模块页面

| 页面 | 内容 |
| :--- | :--- |
| [元数据与生命周期](lifecycle.md) | `UIMetaRegistry` / `UIResRegistry`、生命周期钩子、Holder 事件、手动创建 Holder |
| [打开、关闭与查询](window.md) | 打开/关闭契约、`ShowUI` 系列、查询、并发模型、全屏遮挡、日志语义 |
| [路由与转场](router.md) | `UIRouter` 导航、`NavigateTo` / `Back`、转场源与 `Snap` |
| [Widget 与 Tab](widget.md) | `UIWidget`、Tab 窗口、UI 事件自动解绑 |
| [性能预算](performance.md) | 热路径 0 B 分配清单、低频路径与 Profiler 观察 |
| [编辑器工具](lifecycle.md#uisetting) | `UISetting` 三个页签、生成菜单、生成器诊断 |

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
