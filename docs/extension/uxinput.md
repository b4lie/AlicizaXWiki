# 设备、读取与重绑定

本页覆盖 `UXInput.Watch`、`Reader`、`Rebind` 和 `Haptics`。启动方式见 [Input 概览](../modules/input/index.md)。

所有类型都在 `AlicizaX.Input` 命名空间：

```csharp
using AlicizaX.Input;
```

## 设备监听

`UXInput.Watch` 缓存当前输入上下文，并在设备变化时发事件：

```csharp
public sealed class InputDeviceView : MonoBehaviour
{
    private void OnEnable()
    {
        UXInput.Watch.OnInputTypeChanged += OnInputTypeChanged;
        Refresh(UXInput.Watch.CurrentInputType);
    }

    private void OnDisable()
    {
        UXInput.Watch.OnInputTypeChanged -= OnInputTypeChanged;
    }

    private void OnInputTypeChanged(InputWatch.InputType inputType)
    {
        Refresh(inputType);
    }

    private void Refresh(InputWatch.InputType inputType)
    {
        // Gamepad / KeyboardMouse / Touch / Unknown
    }
}
```

常用成员：

| API | 说明 |
| --- | --- |
| `Current` | 当前 `InputContext` |
| `CurrentInputType` | `Unknown`、`Touch`、`KeyboardMouse`、`Gamepad` |
| `CurrentInputProfile` | `Touch`、`KeyboardMouse`、`Xbox`、`PlayStation` 等 |
| `CurrentProfileId` | 当前 Profile 的稳定字符串 ID |
| `MouseDeltaThreshold` | 鼠标位移超过多少像素才切回键鼠；默认 4 |
| `IsNavigationInput` | 当前是否是手柄 |
| `BuildContext(InputDevice)` | 构建指定设备的上下文 |

事件：`OnContextChanged`、`OnInputActivity`、`OnInputTypeChanged`、`OnInputProfileChanged`。

`InputContext` 只包含 `InputType`、`InputProfile`、`DeviceId`、`DeviceName`。

订阅时要缓存当前实例；输入层重建后 `UXInput.Watch` 是新的对象，旧订阅随旧实例一起失效。

## 读取输入

动作路径使用完整的 `Map/Action`：

```csharp
InputAction moveAction = UXInput.Reader.ResolveAction("Gameplay/Move");
Vector2 move = UXInput.Reader.ReadValue<Vector2>(moveAction);

// TryReadValue 只在动作处于按下状态时返回 true。
if (UXInput.Reader.TryReadValue<Vector2>("Gameplay/Aim", out Vector2 aim))
{
    // 有值。
}

bool submit = UXInput.Reader.IsPressed("UI/Submit");
```

按下与组合部分：

```csharp
bool sprint = UXInput.Reader.IsPressed("Gameplay/Sprint");

InputCompositePart up =
    UXInput.Reader.ResolveCompositePart(moveAction, "Up");

if (up != null && up.IsPressed())
{
    // 组合绑定的 Up 部分按下。
}
```

边沿与开关由业务持有 `InputReader.ButtonState`：

```csharp
private InputReader.ButtonState _jumpState;
private InputReader.ButtonState _inventoryState;

private void Update()
{
    if (_jumpState.Once(UXInput.Reader.ResolveAction("Gameplay/Jump")))
    {
        Jump();
    }

    bool open = _inventoryState.Toggle(UXInput.Reader.ResolveAction("UI/Inventory"));
    SetInventoryVisible(open);
}
```

`ButtonState` 会自动记录按下/松开，业务只需要保存字段。不要把它声明成局部变量，否则每次都是新状态。

诊断事件 `OnRead` / `OnContinuousRead` 只用于调试，不在高频逻辑里订阅。

## 运行时重绑定

```csharp
using AlicizaX.Input;

public void StartRebind()
{
    bool started = UXInput.Rebind.Begin(
        "UI/Submit",
        RebindRequest.ForGamepad);
}

public void Confirm()
{
    bool applied = UXInput.Rebind.ConfirmApply(clearConflicts: true, save: true);
}

public void Cancel()
{
    UXInput.Rebind.Cancel();
    UXInput.Rebind.DiscardPrepared();
}
```

常用操作：

| 操作 | API |
| --- | --- |
| 开始重绑定 | `Begin(action / name / reference, RebindRequest)` |
| 取消监听 | `Cancel()` |
| 确认暂存 | `ConfirmApply(bool clearConflicts = true, bool save = true)` |
| 丢弃暂存 | `DiscardPrepared()` |
| 重置单个动作 | `Reset(action / name, save: true)` |
| 重置全部 | `ResetAllBindings()` |
| 恢复默认 | `ResetToDefault()` |
| 导出/导入 JSON | `ExportBindingsJson()` / `ImportBindingsJson(json)` |
| 显示当前绑定 | `GetDisplayString(action / name, request)` |

`RebindRequest` 可指定目标设备、组合部分、取消键、排除控制、等待时间、绑定分组和 `PrepareOnly`：

```csharp
RebindRequest request = RebindRequest.ForGamepad
    .WithCompositePart("Up")
    .WithWaitDelay(0.2f)
    .WithPrepareOnly(true);
```

重绑定先进入暂存区，确认后才应用和保存。绑定覆盖文件的默认位置：编辑器是工程 `Library` 目录（不进版本库），运行时是 `Application.persistentDataPath/input_bindings.json`。

事件：`OnRebindStarted`、`OnRebindEnded`、`OnRebindPrepared`、`OnBindingConflict`、`OnBindingsChanged`、`OnApply`、`OnBindingsLoaded`、`OnBindingsSaved`。UI 刷新统一监听 `OnBindingsChanged`。

## 手柄震动

```csharp
UXInput.Haptics.Intensity = 0.8f;
UXInput.Haptics.Play(HapticPreset.Selection);
UXInput.Haptics.Play(HapticPreset.Success);
UXInput.Haptics.Play(leftMotor: 0.4f, rightMotor: 0.8f, duration: 0.15f);

UXInput.Haptics.Stop();
```

自定义模式使用 `HapticPattern` 资产：

```text
Assets > Create > AlicizaX > Input > Haptic Pattern
```

```csharp
[SerializeField] private HapticPattern pattern;

UXInput.Haptics.Play(pattern);
```

`Enabled = false` 可整体关闭震动；没有当前手柄时播放会直接跳过。

## 注意事项

1. 只通过 `UXInput.Watch`、`UXInput.Reader` 等访问模块，不要在业务里 `new InputReader`。
2. `ResolveAction` 在找不到动作时返回 `null`；`ReadValue` / `IsPressed` 对 `null` 动作返回默认值。
3. 动作资产由 `InputService` 持有并启用，业务不要单独 `Disable` 整份资产。
4. 重绑定结果先暂存，`ConfirmApply` 后才生效；不要只调用 `Begin` 就认为绑定已修改。
5. 输入层重建后，旧模块实例的订阅和引用都要重新获取。
