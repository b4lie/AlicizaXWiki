# 设备、读取与重绑定

本页覆盖 `UXInput.Watch`、`UXInput.Reader`、`UXInput.Rebind`、`UXInput.Haptics` 的可调用面：设备上下文与 Profile、读取语义与诊断、运行时重绑定的完整流程、震动反馈。启动、门面与模块清单见 [Input 概览](index.md)。

## 适用场景

- 在 `Update` 里轮询动作值、按下状态、边沿或开关。
- 需要玩家在设置页改键，并把结果持久化到本地。
- 需要手柄震动（内置预设或曲线资产）。

## 使用前提

- `UXInput.IsInitialized` 为 `true`，否则四个模块属性都是 `null`。
- 动作路径一律使用完整 `Map/Action`，例如 `Gameplay/Jump`；包内只注册这种完整路径。
- 重绑定期间目标动作被临时 `Disable()`，此时读取该动作得到 `false`。
- 震动只驱动 `Gamepad.current`；没有手柄时 `Play` 直接返回。
- `ReadTrace`、`OnRead`、`OnContinuousRead` 只在 `UNITY_EDITOR` 或 `DEVELOPMENT_BUILD` 下编译。

## 最短接入

1. 缓存模块实例：订阅与退订必须对同一实例进行，例如 `_watch = UXInput.Watch;` 后再 `-=`。
2. 用 `ResolveAction` / `ResolveCompositePart` 一次性解析，把结果存成字段。
3. 每帧用 `InputAction` 或 `InputCompositePart` 重载读取，不要用字符串重载。
4. 边沿与开关用字段形式的 `InputReader.ButtonState`。

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class InputAccess : MonoBehaviour
{
    private InputAction _move;
    private InputAction _jump;
    private InputReader.ButtonState _jumpOnce;

    private void OnEnable()
    {
        if (!UXInput.IsInitialized)
            return;

        _move = UXInput.Reader.ResolveAction("Gameplay/Move");
        _jump = UXInput.Reader.ResolveAction("Gameplay/Jump");
    }

    private void Update()
    {
        if (!UXInput.IsInitialized)
            return;

        Vector2 move = UXInput.Reader.ReadValue<Vector2>(_move);
        if (_jumpOnce.Once(_jump))
            Jump(move);
    }

    private void Jump(Vector2 move) { }
}
```

## 核心用法

### 设备上下文

`Current` 是 `InputWatch.InputContext` 结构体（赋值即复制），包含 `InputType`、`InputProfile`、`DeviceId`、`DeviceName` 和 `IsValid`。

| 成员 | 说明 |
| --- | --- |
| `Current` | 当前上下文；`DeviceName` 取 `displayName`，为空时退回 `name` |
| `CurrentInputType` / `CurrentInputProfile` | 上下文里的两个枚举 |
| `CurrentProfileId` | Profile 的稳定 ID，返回目录里的共享字符串 |
| `IsNavigationInput` / `IsKeyboardMouseInput` / `IsTouchInput` | 类型判断快捷属性 |
| `MouseDeltaThreshold` | 鼠标累计位移阈值，默认 4 px，Setter 下限 0 |
| `BuildContext(InputDevice)` | 构造指定设备的上下文；传 `null` 返回平台默认上下文 |

| 事件 | 参数 | 触发条件 |
| --- | --- | --- |
| `OnContextChanged` | `InputContext` | 上下文四个字段任一变化，或强制刷新 |
| `OnInputTypeChanged` | `InputType` | 仅输入类型变化时 |
| `OnInputProfileChanged` | `InputProfile` | 仅 Profile 变化时 |
| `OnInputActivity` | `InputContext` | 出现有意义的输入活动；同一设备连续输入也会触发 |

有意义的输入按控件类型判定：键盘 `anyKey`、`ButtonControl` 按下、鼠标 `delta` 累计位移达到阈值、摇杆模长 ≥ 0.25、`AxisControl` 绝对值 ≥ 0.5、其余非 `noisy` 控件；`synthetic` 合成控件除 `Keyboard.anyKey` 外忽略。当前设备已经是该鼠标时移动即算活动，从其他设备切过来要累计阈值，避免手柄游玩中鼠标抖动抢回上下文。设备增删时：新增手柄会从键鼠/触屏上下文抢占；当前设备被移除或断开时回退——优先第一个手柄类设备，其次平台默认类型对应的现有设备，最后平台默认上下文（移动端 Touch，其余 KeyboardMouse）。切换设备只发一次 `OnContextChanged`。

### Profile 与 `InputProfileCatalog`

`InputProfileCatalog` 是 Profile 元数据的唯一定义源，稳定 ID 为 `Unknown`、`Touch`、`KeyboardMouse`、`GenericGamepad`、`Xbox`、`PlayStation`、`Switch`、`SteamDeck`、`SteamController`。

| 成员 | 说明 |
| --- | --- |
| `GetId(InputWatch.InputProfile)` | 枚举 → 稳定 ID |
| `TryGet(string id, out InputProfileInfo)` | 按 ID 查元数据，忽略大小写 |
| `TryGet(InputWatch.InputProfile, out InputProfileInfo)` | 按枚举查元数据 |
| `Editable` | 不含 `Unknown`，供编辑器下拉使用 |
| `All` | 全部 Profile，含 `Unknown` |

`InputProfileInfo` 提供 `Profile`、`Id`、`DisplayName`、`FallbackIds`（图标回退链）和 `BindingGroupHints`（绑定评分提示）；后两个是共享只读数组，不要修改。

### `Reader` 读取

| API | 说明 |
| --- | --- |
| `TryResolveAction(path, out InputAction)` / `ResolveAction(path)` | 完整路径解析；未知路径分别返回 `false` / `null` |
| `FindActionMap(name, throwIfNotFound = false)` | 资产不可用且 `throwIfNotFound` 为 `true` 时抛 `InvalidOperationException` |
| `ReadValue<T>(action / path)` | 直接读值；`null` 动作返回 `default` |
| `TryReadValue<T>(action / path, out T)` | 仅在动作处于按下状态时返回值 |
| `IsPressed(action / path / part)` | 按下判断；`null` 参数返回 `false` |
| `ResolveCompositePart(action, partName)` | 组合绑定部件句柄，未知部件返回 `null` |
| `IsCompositePartPressed(action / path, partName)` | 便捷重载，每次调用做一次句柄字典查找 |

字符串重载每次做一次字典查找，高频路径应先把 `InputAction` 存字段、把部件句柄缓存下来。组合部件按名字定位绑定（大小写不敏感），读取时合并同布局的所有设备；目标动作被禁用时立即返回 `false`。句柄在绑定版本变化（绑定覆盖、设备增删）后惰性重建，可以长期持有。

`Reader` 是轮询式 API，本身不提供回调；包内用回调的是 `Watch`（内部 AnyInput 动作）与 `Hotkey`（订阅 action 事件）。

### `ButtonState` 语义

| 成员 | 语义 |
| --- | --- |
| `Once(action)` / `Once(part)` | 本次按下周期的第一帧返回 `true`，持续按住与松开都返回 `false` |
| `Toggle(action)` / `Toggle(part)` | 每次首次按下翻转并缓存，松开保持当前值 |
| `Once(action, partName)` / `Toggle(action, partName)` | 部件名为空等价于动作重载，否则经 `UXInput.Reader` 解析句柄 |
| `Reset()` | 清零边沿与开关状态 |
| `SetToggled(bool)` | 直接设置开关值，不影响边沿 |

状态归调用方所有：两个组件各持有自己的 `ButtonState`，互不影响。它必须声明成字段——属性会返回 struct 副本，边沿状态会丢失。

### `ReadTrace` 诊断

| `ReadTraceKind` | 触发点 |
| --- | --- |
| `Value` | `TryReadValue` 成功读取（连续事件） |
| `Pressed` | `IsPressed(InputAction)` 返回 `true`（连续事件） |
| `CompositePartButton` | `IsPressed(InputCompositePart)` 返回 `true`（连续事件） |
| `Once` / `Toggle` | `ButtonState` 对应重载命中边沿（离散事件） |
| `CompositePartOnce` / `CompositePartToggle` | 组合部件重载命中边沿（离散事件） |
| `Button` | 枚举保留值，包内读取路径不会触发 |

```csharp
#if UNITY_EDITOR || DEVELOPMENT_BUILD
void OnRead(InputReader.ReadTrace trace) =>
    UnityEngine.Debug.Log(trace.Kind + " " + trace.Action.name);
UXInput.Reader.OnRead += OnRead;
#endif
```

`OnRead` 只在离散读取命中时触发；`OnContinuousRead` 在连续读取读到有效按下时触发，可能每帧多次。`ReadTrace` 提供 `Kind`、`Action`、`CompositePartName`、`BoolValue`。发布包没有这两个事件，不要写进业务逻辑。

### 运行时重绑定流程

1. `Begin(action / reference / "Map/Action", request)`：先取消旧操作，再按请求选择绑定索引——组合部分要求 `isPartOfComposite` 且名字匹配，否则跳过 composite 绑定行；然后按 `RequiredControlPath` 前缀或 `BindingGroup` 过滤。找不到可绑定项时记 warning 并返回 `false`。
2. 记录目标动作原启用状态并临时 `Disable()`，创建 `PerformInteractiveRebinding(bindingIndex)`，应用取消键、排除控件、必需路径、绑定分组和 `WaitDelay`，`Start()` 后触发 `OnRebindStarted`。
3. 捕获控件后按 `WaitDelay` 再等一段时间（`OnMatchWaitForAnother`）；期间没有新的匹配输入就采用已捕获的控件。
4. 完成：清理操作并恢复动作启用状态，把结果暂存（`PrepareRebind`）并触发 `OnRebindPrepared`；同路径已被其他绑定（含已暂存项）占用时触发 `OnBindingConflict`；`PrepareOnly` 为 `false` 时立刻走一次 `ConfirmApply(true, true)`；最后触发 `OnRebindEnded(true, change)`。
5. 取消（`Cancel()` 或触发取消键）：清理操作、恢复动作，触发 `OnRebindEnded(false, RebindChange.Empty)`。
6. `ConfirmApply(clearConflicts, save)`：逐条 `ApplyBindingOverride`，`clearConflicts` 为 `true` 时把其他动作上同路径的覆盖清空，推进绑定版本并触发 `OnBindingsChanged`，`save` 为 `true` 时写文件并触发 `OnBindingsSaved`，最后触发 `OnApply(true, 快照)`。

```csharp
using AlicizaX.Input;
using UnityEngine.InputSystem;

public sealed class RebindPanel
{
    public bool Begin(InputAction action) => UXInput.Rebind.Begin(
        action,
        RebindRequest.ForTarget(RebindTarget.Gamepad)
            .WithWaitDelay(0.2f)
            .WithPrepareOnly(true));

    public bool Apply() => UXInput.Rebind.ConfirmApply(clearConflicts: true, save: true);
    public void CancelListening() => UXInput.Rebind.Cancel();
    public void Discard() => UXInput.Rebind.DiscardPrepared();
}
```

`RebindRequest` 是不可变结构体，用 `With*` 派生新值，不会修改共享默认：

| 选项 | `Default`（键鼠） | 说明 |
| --- | --- | --- |
| `BindingGroup` | `"KeyboardMouse"` | 按绑定分组过滤 |
| `RequiredControlPath` | `"<Keyboard>"` | 按路径前缀过滤，优先于分组 |
| `CancelKey` | `"<Keyboard>/escape"` | 取消键 |
| `ExcludedControls` | 鼠标/指针的 `position`、`delta` | 排除的控件路径 |
| `WaitDelay` | `0.1` | 命中后的等待秒数 |
| `PrepareOnly` | `true` | 只暂存，等 `ConfirmApply` |

`RebindRequest.ForGamepad`（等价 `ForTarget(RebindTarget.Gamepad)`）把分组与路径换成 `"Gamepad"` / `"<Gamepad>"`；默认请求只接受键盘。

| 事件 | 参数 | 时机 |
| --- | --- | --- |
| `OnRebindStarted` | — | 操作开始监听后 |
| `OnRebindEnded` | `bool completed, RebindChange change` | 完成或取消；取消时传 `RebindChange.Empty` |
| `OnRebindPrepared` | `RebindChange` | 捕获结果已暂存 |
| `OnBindingConflict` | `RebindChange, IReadOnlyList<RebindChange>` | 暂存路径与已有绑定冲突 |
| `OnBindingsChanged` | — | 覆盖因应用、重置、加载或导入而变化 |
| `OnApply` | `bool applied, IReadOnlyList<RebindChange>` | 暂存被确认或丢弃 |

持久化：绑定覆盖写在 `BindingFilePath`（编辑器 `Library/AlicizaXInput/input_bindings.json`，运行时 `Application.persistentDataPath/input_bindings.json`），不会修改 `.inputactions` 资产。加载在 `Rebind.Initialize` 中进行，先记录资产当前覆盖作为入口状态再读文件；文件损坏时恢复入口覆盖、把坏文件改名为 `.corrupt` 并记 warning，输入层继续可用。保存先写 `.tmp` 再移动到位，IO 失败只记 error，内存中的覆盖保持已应用。`InputService` 销毁时会取消进行中的操作并恢复入口覆盖。

失败与取消：

| 情况 | 行为 |
| --- | --- |
| `Begin(null)`、动作不存在、资产不可用或找不到可绑定控件 | 返回 `false`（资产/控件问题另记 warning） |
| 已有操作时再次 `Begin` | 先取消旧操作；若取消回调里又开启了新操作，本次返回 `false` |
| `Cancel()` 无操作 / `ConfirmApply` 无暂存 | 空操作 / 返回 `false` |
| `DiscardPrepared` 无暂存 | 直接返回，不触发 `OnApply` |
| 完成/取消回调抛异常 | 操作已先清理，`IsRebinding` 为 `false` |

### 震动

| 预设 | 左马达 | 右马达 | 时长（秒） |
| --- | --- | --- | --- |
| `Light` | 0.2 | 0.2 | 0.1 |
| `Medium` | 0.5 | 0.5 | 0.15 |
| `Heavy` | 1.0 | 1.0 | 0.2 |
| `Pulse` | 0.8 | 0.3 | 0.1 |
| `Success` | 0.3 | 0.6 | 0.15 |
| `Error` | 0.8 | 0.2 | 0.25 |
| `Selection` | 0.1 | 0.3 | 0.05 |

```csharp
UXInput.Haptics.Intensity = 0.6f;
UXInput.Haptics.Play(HapticPreset.Selection);
UXInput.Haptics.Play(0.4f, 0.8f, 0.15f);
UXInput.Haptics.Play(hitPattern);   // HapticPattern 资产
UXInput.Haptics.Stop();
```

`Play(float left, float right, float duration)` 立即 `SetMotorSpeeds`；`Play(HapticPattern)` 按归一化时间采样 `LeftMotorCurve` / `RightMotorCurve`，由 `InputSystem.onAfterUpdate` 驱动，只在播放期间注册。两路强度都会再乘 `Intensity`。`Stop()` 立即归零并注销；`Enabled = false` 会先停止；`Reset()` 恢复 `Intensity = 1` 与启用状态。没有 `Gamepad.current` 时播放直接返回，播放中切换设备需要重新 `Play`。失焦或暂停时 `InputService` 会调用 `Haptics.Stop()`。

## 常见配置

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `RebindRequest.Default` | 键盘 | 只接受 `<Keyboard>` 路径 |
| `RebindRequest.ForGamepad` | 手柄 | 只接受 `<Gamepad>` 路径 |
| `Haptics.Intensity` | `1` | 0 到 1，`Clamp01` |
| 绑定覆盖文件 | `Library/AlicizaXInput/input_bindings.json`（编辑器）、`persistentDataPath/input_bindings.json`（运行时） | 只读属性 `Rebind.BindingFilePath` |

## 容易踩的坑

1. **用短动作名读取。** `ResolveAction("Jump")` 返回 `null`，只有 `Gameplay/Jump` 会命中；编辑器下字符串重载还会触发 `Debug.Assert`。
2. **拿 `TryReadValue` 读轴。** 它只在动作按下时返回 `true`；摇杆静止时读值用 `ReadValue<T>`。
3. **把 `ButtonState` 写成局部变量或属性。** 每次调用都是新状态，`Once` 会一直返回 `true` 或一直 `false`。
4. **只调 `Begin` 就以为绑定已改。** 默认 `PrepareOnly` 为 `true`，必须 `ConfirmApply`；而 `ConfirmApply` 默认还会清掉其他动作上同路径的覆盖。
5. **把绑定覆盖当工程资产。** 覆盖写在 `Library` 或 `persistentDataPath`，清缓存或换机就丢；需要随账号走时用 `ExportBindingsJson` / `ImportBindingsJson` 自己搬。

## API 速查

| 类型 | 关键成员 |
| --- | --- |
| `InputWatch` | `Current`、`CurrentInputType`、`CurrentInputProfile`、`CurrentProfileId`、`MouseDeltaThreshold`、`IsNavigationInput`、`IsTouchInput`、`IsKeyboardMouseInput`、`BuildContext`、四个事件 |
| `InputWatch.InputContext` | `InputType`、`InputProfile`、`DeviceId`、`DeviceName`、`IsValid`、`KeyboardMouse(device)`、`Touch(device)` |
| `InputProfileCatalog` | `GetId`、`TryGet(id)`、`TryGet(profile)`、`Editable`、`All` |
| `InputReader` | `TryResolveAction`、`ResolveAction`、`FindActionMap`、`ReadValue<T>`、`TryReadValue<T>`、`IsPressed`、`ResolveCompositePart`、`IsCompositePartPressed` |
| `InputReader.ButtonState` | `Once`、`Toggle`、`Reset`、`SetToggled` |
| `InputCompositePart` | `Action`、`PartName`、`IsPressed()` |
| `InputRebind` | `Begin`、`Cancel`、`ConfirmApply`、`DiscardPrepared`、`Reset`、`ResetAllBindings`、`ResetToDefault`、`ExportBindingsJson`、`ImportBindingsJson`、`GetDisplayString` |
| `InputRebind` 状态 / `RebindChange` | `IsInitialized`、`IsRebinding`、`HasPreparedRebinds`、`HasSavedBindings`、`BindingFilePath`；`RebindChange` 公开 `Action`、`OverridePath`、`IsValid` |
| `RebindRequest` | `Default`、`ForGamepad`、`ForTarget`、`WithCompositePart`、`WithCancelKey`、`WithExcludedControls`、`WithWaitDelay`、`WithBindingGroup`、`WithRequiredControlPath`、`WithPrepareOnly` |
| `InputHaptics` | `Intensity`、`Enabled`、`IsPlaying`、`Play(preset / pattern / left,right,duration)`、`Stop`、`Reset` |
