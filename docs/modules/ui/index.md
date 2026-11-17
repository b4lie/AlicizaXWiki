# UI 界面

UI 模块负责窗口的声明、打开关闭、层级与排序、缓存复用、开场/关闭转场，以及窗口内的 Widget 与页签；页面级前进/返回由 `Navigation` 提供。它不负责资源包与资源加载策略，也不负责页面里的业务数据。

源码位置：

- 运行时：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI`
- 编辑器：`Client/Packages/com.alicizax.unity.framework/Editor/Modules/UI`
- 模板示例：`Client/Assets/Scripts/Hotfix/GameLogic/UI`

## 适用场景

- 全屏页面（`UI` 层）、弹窗（`Popup`）、Tips/确认框（`Tips`）、2D 场景信息与引导遮罩（`Scene` / `Top`）。
- 需要「打开 → 关闭 → 缓存 → 复用」的界面：再次打开不重新加载、不重跑 `OnInitialize`。
- 需要统一的层级与排序，不想在业务里手改 Canvas 的 `sortingOrder`。
- 不适合：场景内的 3D 物件、纯数据/流程逻辑；这些交给场景、流程和业务层。

## 使用前提

UI 服务由 `UIComponent` 在 `Awake` 里创建并注册为 `IUIService`，它依赖下面几项：

| 依赖 | 用途 | 缺失后果 |
| --- | --- | --- |
| `RootModule` / `AppServices` | 服务容器 | 容器未创建时 `AppServices.Require` 抛异常 |
| `ITimerService` | 缓存计时、输入遮挡计时 | `UIComponent.Awake` 抛异常，UI 服务不会注册 |
| `IResourceService` | 实例化 Holder 预制体 | 打开窗口时报 `UI resource could not be loaded` |
| `com.unity.inputsystem` | `UIRoot.prefab` 的 EventSystem 输入模块 | UI 收不到点击（输入模块脚本缺失） |
| TextMeshPro / `com.alicizax.unity.ui.extension` | 生成代码里的 `Text` → `TextMeshProUGUI`、`Btn` → `UXButton` 绑定 | 只影响对应节点类型，UI 核心不依赖它们 |

挂载位置与初始化顺序：

- 模板通过 `Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab` 组装启动骨架，`UIComponent` 挂在它的子节点 `UI` 上；`Client/Assets/Scenes/Main.unity` 实例化 `Entry`。手动接入时直接给任意启动场景节点挂 `UIComponent`（`AddComponentMenu` 为 `Game Framework/UI`）。
- 执行顺序（`DefaultExecutionOrder`）：`RootModule`(-32000) → `MemoryPoolSetting`(-1000) → `TimerComponent`(-800) → `ResourceComponent`(-700) → `SceneComponent`(-600) → `UIComponent`(-500) → `AudioComponent`(-400)，所以 `UIComponent` 能取到 `ITimerService` 与 `IResourceService`。
- `UI Root Prefab` 必填，默认指向 `Client/Packages/com.alicizax.unity.framework/Editor/Modules/UI/Res/UIRoot.prefab`，Inspector 里为空时会标红并出现 `Default` 按钮。
- 该预制体必须包含 `Canvas`，且 `Canvas.worldCamera` 非空，否则构造 `UIService` 时抛 `InvalidOperationException`（`UI root must contain a Canvas.` / `UI Canvas must have a worldCamera.`）。
- 运行时会实例化根节点、改名为 `------UI Root------` 并 `DontDestroyOnLoad`。

## 最短接入

1. 准备根节点：确认启动场景（或 `Entry.prefab`）里有 `RootModule` + `TimerComponent` + `ResourceComponent` + `UIComponent`，并给 `UI Root Prefab` 填上 `UIRoot.prefab`。

2. 核对 `UIRoot.prefab` 的结构。运行时 `UIService` 只在 `UICanvas` 下补 `Layer0-Background` … `Layer5-Top` 六个全屏层和 `LayerBlock` 遮挡层：

   ```text
   UIRoot            Layer 0，位置 (1000, 1000, 0)，EventSystem + InputSystemUIInputModule
   ├─ UICanvas       Layer 5，Canvas = ScreenSpaceCamera（Camera 指向 UICamera，Plane Distance 100）
   │                 CanvasScaler = ScaleWithScreenSize 1920×1080（Match Width）
   │                 GraphicRaycaster；运行时在其下生成 Layer0…Layer5 与 LayerBlock
   └─ UICamera       Layer 0，正交相机，Culling Mask 只有 Layer 5，Depth 2，Clear Flags = Depth
   ```

3. 生成窗口的 Holder 与 `ui_*` 脚本：菜单 `AlicizaX/UISetting Window` 配置规则，选中预制体执行菜单 `GameObject/UI生成/绑定`。生成细节与字段含义见 [窗口声明与生命周期](lifecycle.md)。

4. 声明第一个窗口逻辑类，并绑定按钮：

   ```csharp
   using AlicizaX.UI.Runtime;
   using Cysharp.Threading.Tasks;
   using Game.UI;

   namespace GameLogic.UI
   {
       [Window(UILayer.UI, 30)]
       public class UIHomeWindow : UIWindow<ui_UIHomeWindow>
       {
           protected override void OnInitialize()
           {
               baseui.BtnShop.onClick.AddListener(OnBtnShopClick);
           }

           private void OnBtnShopClick()
           {
               GameApp.UI.ShowUI<UIShopWindow>().Forget();
           }
       }
   }
   ```

5. 从启动流程打开它。模板里的真实入口在 `Client/Assets/Scripts/Hotfix/GameLogic/HotfixEntry.cs`：

   ```csharp
   // HotfixEntry.EntranceAsync 的末尾
   await GameApp.UI.ShowUI<UIHomeWindow>();
   ```

6. 业务侧统一通过 `GameApp.UI` 拿服务，不要在业务代码里保存 `IUIService`：

   ```csharp
   IUIService ui = GameApp.UI;                       // 内部是 AppServices.Require<IUIService>()
   bool open = ui.IsOpen<UIHomeWindow>();
   RectTransform popupLayer = ui.GetLayer(UILayer.Popup);
   ```

## 核心用法

### 层级与排序

`UILayer`（源文件是 `Runtime/Modules/UI/Constant/EPanelLayer.cs`）从 0 到 5：

| 值 | 枚举 | 用途 |
| --- | --- | --- |
| 0 | `Background` | 最底层背景 |
| 1 | `Scene` | 用 2D 表现的血条、飘字、头像等场景信息 |
| 2 | `UI` | 普通全屏界面 |
| 3 | `Popup` | 非全屏弹窗，可同时存在多个 |
| 4 | `Tips` | 提示、确认框、跑马灯 |
| 5 | `Top` | 新手引导、最高层遮罩 |

`UIService` 在构造时按层创建全屏 `RectTransform`，每个窗口节点挂在所属层的根节点下。窗口排序值由层和层内位置算出：`sortingOrder = 层索引 × 4000 + 层内序号 × 20`，所以高层永远压在低层之上；同层内重新打开会提到最前。每层最多 200 个窗口，超过会在编辑器里触发断言（`Too many windows in one layer`）。

### 窗口根节点的运行时处理

窗口节点被绑定时，`UIWindow` 会把它设为全屏、移到 `UIComponent.UIShowLayer`（值是 5，相机只渲染这一层），并保证节点上有 `Canvas`（`overrideSorting = true`，`sortingOrder` 如上）和 `GraphicRaycaster`（未打开时禁用）。业务不需要、也不应该自己改这些值。

### 入口 API

`IUIService` 分四块：打开/关闭、查询、层级、遮挡；页面导航是它的 `Navigation` 属性。

| 成员 | 用途 |
| --- | --- |
| `ShowUI<T>()` / `ShowUI(string)` / `ShowUI(RuntimeTypeHandle)` | 异步打开，含取消重载 |
| `ShowUISync<T>()` | 同步加载并打开 |
| `CloseUI<T>()` / `CloseUI(RuntimeTypeHandle)` | 关闭，可等待转场 |
| `IsOpen<T>()` / `GetUI<T>()` | 查询打开/缓存实例 |
| `GetLayer(UILayer)` | 取某层根节点 |
| `TryGetTopVisibleHolder` / `TryCloseTopAsync` | 从最高层查找/关闭 |
| `SetUIBlock` / `ForceExitBlock` | 全屏输入遮挡 |
| `UICamera` / `UICanvasRoot` / `Navigation` | 相机、Canvas 根、页面导航 |

具体签名与行为见 [打开、关闭与查询](window.md)。

### 页面导航与窗口的关系

页面级前进/返回、路由 history 用 `GameApp.UI.Navigation`；弹窗、Tips、独立面板直接用 `ShowUI`。同一个页面不要同时交给导航和 `ShowUI` 管理，详见 [路由与转场](router.md)。

### 调试入口

Play Mode 下选中挂了 `UIComponent` 的节点，Inspector 会有两个只读卡片：

- `UI RuntimeDebug`：服务摘要（Initialized / Mode / Open / Cache / Update / Block）、`Runtime Alerts`（缓存过多、Holder 丢失、`sortingOrder` 冲突、缓存但仍可见）、可选的 `References`、按层分组的 `Open Windows` 和 `Cached Windows`；筛选条为 `Layers` / `Cache` / `References` / `Empty Layers`。
- `Navigation RuntimeDebug`：当前页、`CanBack`、history 条目与参数预览；筛选条为 `History`。

面板每 250 ms 刷新，只读，不会改运行状态；对应的 `IUIDebugService` 是 `internal`，仅编辑器与测试使用，业务代码不要调。

## 常见配置

| 位置 | 字段 | 说明 |
| --- | --- | --- |
| `UIComponent` | `UI Root Prefab` | 必填，通常填 `UIRoot.prefab` |
| `UIComponent` | `Orthographic` | 勾选时用预制体自带的正交参数；取消勾选才应用下面两个裁剪面 |
| `UIComponent` | `Near Clip Plane` / `Far Clip Plane` | 透视相机的裁剪面，正交模式下隐藏 |
| `UIRoot.prefab` | Canvas / CanvasScaler | `ScreenSpaceCamera`、参考分辨率 1920×1080、按宽度匹配 |
| `UIRoot.prefab` | Camera | 正交、`Culling Mask` 只留 Layer 5、`Depth` 2、只清深度 |
| `UIRoot.prefab` | EventSystem | 使用 `InputSystemUIInputModule`，改回旧输入模块需自己替换 |
| `AlicizaX/UISetting Window` | 生成配置 | 落在 `ProjectSettings/UIGenerateConfiguration.asset` |

## 本模块页面

| 页面 | 内容 |
| --- | --- |
| [窗口声明与生命周期](lifecycle.md) | `[Window]` / `[UIRes]`、UISetting 与生成流程、Holder 关系、钩子真实顺序、手动注册与手动创建 |
| [打开、关闭与查询](window.md) | `ShowUI` / `ShowUISync` / `CloseUI` / `CloseSelf`、参数、取消、并发规则、查询与遮挡 |
| [路由与转场](router.md) | `Navigation` 前进/返回、路由结果、预设与自定义转场 |
| [Widget 与 Tab](widget.md) | Widget 的创建与开关、Tab 窗口、UI 事件自动解绑 |
| [性能与缓存](performance.md) | 更新成本、缓存上限、每帧分配与检查工具 |

## 容易踩的坑

1. `UI Root Prefab` 为空，或预制体里没有带 `worldCamera` 的 `Canvas`，`UIComponent.Awake` 就会抛异常，`IUIService` 根本不会注册，后续所有 `GameApp.UI` 调用都会失败。
2. 不要额外放第二个 `EventSystem`：`UIRoot.prefab` 已自带；场景里再放一个会与它冲突并产生警告。
3. 不要手改窗口节点的 Layer、`overrideSorting`、`sortingOrder`：每次绑定 Holder 时都会被 `UIWindow` 覆盖。
4. 不要用 UI 模块做资源管理：`ShowUI` 只按 `[UIRes]` 的地址实例化预制体，贴图、音频、配置走 [加载与绑定](../resource/loading.md)；窗口内的 `Res` 作用域只覆盖当前视图。
5. 不要挂两个 `UIComponent`：会重复实例化根节点，并在注册 `IUIService` 时因契约已被占用而失败。
6. `Layer0-Background` … `Layer5-Top`、`LayerBlock` 都是运行时生成的，不要把它们当成预制体或场景对象去引用。

## 下一步

- 第一次接窗口：先读 [窗口声明与生命周期](lifecycle.md)，把 Holder 和 `ui_*` 脚本生成出来。
- 已经在接业务：读 [打开、关闭与查询](window.md) 确认重复打开、缓存与取消的规则。
