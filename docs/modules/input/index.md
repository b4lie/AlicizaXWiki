# Input 输入

`com.alicizax.unity.input`（当前版本 4.0.2，Unity 2022.3）在 Unity Input System 之上提供一层输入组合：`InputService` 负责启动与关闭，`UXInput` 暴露八个模块实例，`InputReader` 是唯一持有 `InputActionAsset` 的模块。包本身不实现设备抽象层——设备枚举、输入 State 与 Action 求值都来自 `UnityEngine.InputSystem`；这一层做的是设备分类与上下文缓存、读取封装、重绑定与持久化、图标解析、震动、UI 导航、热键和交互反馈。

源码位置：

| 内容 | 路径 |
| --- | --- |
| 启动与门面 | `Client/Packages/com.alicizax.unity.input/Runtime/InputService.cs`、`Runtime/Input/UXInput.cs` |
| 读取、设备、重绑定、震动 | `Client/Packages/com.alicizax.unity.input/Runtime/Input` |
| 图标数据库与组件 | `Client/Packages/com.alicizax.unity.input/Runtime/InputGlyph` |
| UI 导航 | `Client/Packages/com.alicizax.unity.input/Runtime/Navigation` |
| 热键 | `Client/Packages/com.alicizax.unity.input/Runtime/Hotkey` |
| 交互反馈 | `Client/Packages/com.alicizax.unity.input/Runtime/Feedback` |

## 适用场景

- 需要全局唯一的输入入口：一份 `InputActionAsset` 驱动读取、重绑定、图标、导航、热键和反馈。
- 需要按「玩家当前用手柄、键鼠还是触屏」切换 UI 提示、图标和焦点行为。
- 需要玩家可改键、改键结果落盘，且绑定变化能自动刷新界面。
- 需要 UI 导航与热键共用同一套层级优先级规则（Canvas 排序 → 显示序号 → 层级深度）。

## 使用前提

- 工程必须安装 `com.unity.inputsystem`（当前 `Client/Packages/manifest.json` 为 1.14.0），并在 Player Settings 中启用新输入后端（工程当前 Active Input Handling 为 Both）。
- 包以 embedded package 形式放在 `Client/Packages/com.alicizax.unity.input`；运行时代码依赖框架运行时（`AppServices`、`MonoServiceBehaviour`、`AlicizaX.UI.Runtime` 的 Holder）与 TextMeshPro。
- `Runtime/AlicizaX.Input.asmdef` 与 `Editor/AlicizaX.Input.Editor.asmdef` 都在 `com.unity.inputsystem` 版本 ≥ 1.0.0 时定义 `INPUTSYSTEM_SUPPORT`。包里所有运行时类型都包在 `#if INPUTSYSTEM_SUPPORT` 内：没有 Input System 时这些类型整体不存在，是编译期缺失，不是运行时关闭。
- 反向也有宏：`com.alicizax.unity.ui.extension` 在 `com.alicizax.unity.input` ≥ 1.0.0 时定义 `UXNAVIGATION_SUPPORT`，它的 RecyclerView 导航代码（`RecyclerViewNavigationController` 等）依赖本包。
- 纯读取输入只需要框架 + Input System；导航与热键额外要求 UI 节点处于框架 UI 的 Holder（`AlicizaX.UI.Runtime.UIHolderObjectBase`）之下；交互反馈需要框架 `IAudioService`（缺失时静默跳过播放）。

## 最短接入

1. 在启动场景挂一个常驻 `InputService`。本工程 `Client/Assets/Scenes/Main.unity` 已有根节点 `InputService`：挂了 `InputService` 组件，`Actions` 指向 `Client/Assets/Art/Input.inputactions`，`Glyph Database` 为空。新场景用 Add Component → Input → Input Service 添加。
2. 在 Inspector 填写字段：`Actions` 必填，为空时 `Awake` 记录 `InputService requires an InputActionAsset. Input stays disabled.` 并放弃初始化；`Glyph Database` 可空，之后可用 `UXInput.Glyph.SetDatabase` 替换。
3. 所有读取都使用完整的 `Map/Action` 路径，例如 `Gameplay/Jump`。只有 `MapName/ActionName` 会被注册，单写动作名解析不到。
4. 业务侧先用 `UXInput.IsInitialized` 守卫，再访问模块属性并缓存自己订阅的实例：

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class InputProbe : MonoBehaviour
{
    private InputWatch _watch;
    private InputAction _jump;

    private void OnEnable()
    {
        if (!UXInput.IsInitialized)
            return;

        _watch = UXInput.Watch;                       // 缓存订阅的实例
        _watch.OnInputTypeChanged += OnInputTypeChanged;
        _jump = UXInput.Reader.ResolveAction("Gameplay/Jump");
    }

    private void OnDisable()
    {
        if (_watch == null)
            return;

        _watch.OnInputTypeChanged -= OnInputTypeChanged;  // 对同一实例退订
        _watch = null;
    }

    private void Update()
    {
        if (!UXInput.IsInitialized || _jump == null)
            return;

        if (UXInput.Reader.IsPressed(_jump))
        {
            // 本帧处于按下状态。
        }
    }

    private void OnInputTypeChanged(InputWatch.InputType inputType)
    {
        // Gamepad / KeyboardMouse / Touch / Unknown
    }
}
```

5. 需要代码创建（例如没有场景节点）时调用静态入口；`Initialize` 会先查 `AppServices`，已有 `InputService` 时直接返回：

```csharp
InputService.Initialize(actions, glyphDatabase);
```

## 核心用法

### 门面 `UXInput`

`UXInput` 只有只读属性和 `IsInitialized`，没有公开的初始化/关闭方法；`Initialize` 与 `Shutdown` 都是 `internal`，由 `InputService` 驱动。

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `UXInput.IsInitialized` | `bool` | 会话是否已初始化；为 `false` 时下面八个属性全为 `null` |
| `UXInput.Watch` | `InputWatch` | 当前设备上下文、输入类型与 Profile、设备活动事件 |
| `UXInput.Reader` | `InputReader` | 动作解析、值读取、按下状态、组合绑定部件、`ButtonState` |
| `UXInput.Rebind` | `InputRebind` | 交互式重绑定、冲突处理、绑定覆盖持久化 |
| `UXInput.Glyph` | `InputGlyph` | 控制路径到图标、TMP 标签、显示名 |
| `UXInput.Haptics` | `InputHaptics` | 手柄马达震动 |
| `UXInput.Navigation` | `InputNavigation` | 导航域、焦点补选与压制 |
| `UXInput.Hotkey` | `InputHotkey` | 热键注册与按 Holder 焦点链分发 |
| `UXInput.Feedback` | `InputFeedback` | 按设备和交互提示播放 UI 音效 |

### 启动与关闭顺序

1. `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]`：`InputService.ResetForPlayMode` 调用 `UXInput.Shutdown()`，清掉可能残留的静态模块。
2. 场景加载后 `InputService.Awake`（`[DefaultExecutionOrder(-30000)]`）：`base.Awake()` 把组件注册进 `AppServices` → 校验 `actions` → `UXInput.Initialize(actions, glyphDatabase)` → `DontDestroyOnLoad(gameObject)`。
3. `UXInput.Initialize` 一次性构造全部模块：`Watch`、`Reader`、`Rebind`、`Glyph`、`Haptics`、`Navigation`、`Hotkey`、`Feedback`；然后依次执行 `BindingVersion.Attach` → `Watch.Initialize` → `Rebind.Initialize` → `Reader.Initialize` → `Navigation.Initialize`。`Rebind` 必须早于 `Reader`：存档里的绑定覆盖要在资产整份 `Enable()` 之前加载。
4. 运行期失焦/暂停：`InputService` 只调用 `Hotkey.ResetPressTargets()` 和 `Haptics.Stop()`，不关闭输入层。
5. `InputService.OnDestroy` → `UXInput.Shutdown()` → `base.OnDestroy()` 注销服务。关闭顺序是 `Hotkey` → `Navigation` → `Haptics` → `Rebind` → `Reader` → `Glyph` → `Watch`，最后解绑版本监听；`Feedback` 直接置空。
6. 关闭后八个模块属性全为 `null`、`IsInitialized` 为 `false`；旧实例收不到新会话的输入，但业务仍应重新获取模块引用。

### 场景切换与 Play Mode 重载

- `DontDestroyOnLoad` 让输入层跨场景存活；切场景不需要也不应该再初始化一次。
- 场景里出现第二个 `InputService` 时，第二个组件的 `Awake` 会调用 `UXInput.Initialize` 并抛 `InvalidOperationException`（`UXInput is already initialized...`）。用 `InputService.Initialize` 代码创建则因为先查 `AppServices` 而安全返回。
- 关闭 Domain Reload 时静态字段会跨 Play Mode 保留，`SubsystemRegistration` 的 reset 是让新会话从干净状态开始的关键；`InputPlayModeReloadTests` 对 `EnterPlayModeOptions.None` 和 `DisableDomainReload` 两种情况都做了验证，并断言上一会话的模块收不到新会话输入。
- 输入层重建后 `Watch`/`Reader`/`Rebind`/`Glyph`/`Haptics`/`Navigation`/`Hotkey`/`Feedback` 都是新实例。业务侧绑在模块上的状态（例如 `UXInput.Feedback.Profile`）需要重新设置，工程里 `HotfixEntry` 就是在启动时赋 `Resources.Load<InputFeedbackProfile>("UX/InputFeedbackProfile")`。

### 模块与文档

| 模块 | 提供者 | 用途 | 文档 |
| --- | --- | --- | --- |
| `UXInput.Reader` / `Watch` / `Rebind` / `Haptics` | `InputReader`、`InputWatch`、`InputRebind`、`InputHaptics` | 设备上下文、读取、重绑定、震动 | [设备、读取与重绑定](uxinput.md) |
| `UXInput.Glyph` | `InputGlyph` | 图标数据库查询、TMP 标签、显示名 | [输入图标](input-glyph.md) |
| `UXInput.Hotkey` | `InputHotkey` | `HotkeyComponent` 注册与 Holder 焦点链分发 | [快捷键](hotkey-component.md) |
| `UXInput.Navigation` | `InputNavigation` | `NavigationScope` 焦点域与补选 | [UI 导航](navigation.md) |
| `UXInput.Feedback` | `InputFeedback` | Cue + 设备 → UI 音效 | [交互反馈](input-feedback.md) |
| 分配与缓存边界 | — | 热路径分配、绑定缓存、诊断工具 | [性能与分配](performance.md) |

## 常见配置

| 配置 | 位置 | 说明 |
| --- | --- | --- |
| `Actions` | `InputService` Inspector | 必填；为空时不初始化输入层 |
| `Glyph Database` | `InputService` Inspector | 可空；运行时可用 `UXInput.Glyph.SetDatabase` 替换 |
| `Dont Destroy On Load` | `InputService` Inspector（继承自 `MonoServiceBehaviour`） | 常驻场景勾选；`Awake` 里还会再调一次 `DontDestroyOnLoad` |
| Active Input Handling | Player Settings | 必须包含 Input System 后端，工程为 Both |
| `com.unity.inputsystem` | `Client/Packages/manifest.json` | 1.14.0 |
| `INPUTSYSTEM_SUPPORT` | `Runtime/AlicizaX.Input.asmdef`、`Editor/AlicizaX.Input.Editor.asmdef` | `com.unity.inputsystem` ≥ 1.0.0 时自动定义 |
| 绑定覆盖文件 | 编辑器 `Library/AlicizaXInput/input_bindings.json`，运行时 `Application.persistentDataPath/input_bindings.json` | 读写与损坏处理见[设备、读取与重绑定](uxinput.md) |

## 容易踩的坑

1. **场景里放两个 `InputService`。** 第二个会在 `Awake` 抛异常。另外 `Awake` 只执行一次，禁用组件或失活节点并不会关闭已经初始化的输入层（`ProviderDisableKeepsEnabledGlyphAndHotkeyConnected` 用例覆盖这一点）。
2. **以为 `actions` 可以后补。** 为空时不是延迟初始化，而是直接放弃；之后也不会自动重试。
3. **在更早的 `Awake` 里访问门面。** 只有执行顺序晚于 -30000 的脚本才保证能看到已初始化的输入层，统一用 `UXInput.IsInitialized` 守卫。
4. **把模块实例或 `InputAction` 放进静态字段跨会话复用。** 输入层重建后这些引用属于旧资产，读取会一直返回默认值。
5. **没有 Input System 时去找运行时开关。** `AlicizaX.Input` 命名空间下的类型会整体消失，报的是编译错误；不要试图用运行时兜底。
6. **以为纯读取也需要 UI Holder。** 只有导航和热键依赖 Holder，`InputService` + `UXInput.Reader` 就能读输入。

## API 速查

| 类型 | 成员 | 说明 |
| --- | --- | --- |
| `InputService` | `Initialize(InputActionAsset, InputGlyphDatabase)` | 代码创建输入层，已存在时直接返回 |
| `InputService` | `actions`、`glyphDatabase` | Inspector 序列化字段 |
| `UXInput` | `Watch`、`Reader`、`Rebind`、`Glyph`、`Haptics`、`Navigation`、`Hotkey`、`Feedback` | 模块属性，未初始化时为 `null` |
| `UXInput` | `IsInitialized` | 会话是否已初始化 |
| `AlicizaX.Input` | `INPUTSYSTEM_SUPPORT` | 包内编译宏，由 asmdef 的 versionDefines 决定 |
