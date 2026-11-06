# 2.x → 3.x 迁移

3.0 把输入模块从静态工具类拆成了 `UXInput` 门面 + 五个普通实例，并移除了旧的静态入口和 `ownerId` 读取 API。本页对应当前包版本 `3.0.1`。

迁移前先确认：

- 场景中只有一个 `InputActionProvider`，它负责 `UXInput.Initialize` / `UXInput.Shutdown`。
- 业务访问实例仍通过 `UXInput.Watch`、`UXInput.Reader`、`UXInput.Rebind`、`UXInput.Glyph`、`UXInput.Haptics`；声明类型时使用 `InputWatch`、`InputReader`、`InputRebind`、`InputGlyph`、`InputHaptics`。
- 业务自己的边沿、开关状态改为持有 `InputReader.ButtonState` 字段。

## 2.x API 迁移

| 变更 | 迁移方式 |
| --- | --- |
| `Reader.Read*Once(Object owner, ...)` / `Read*Toggle(...)` / `ResetToggled*` 移除 | 声明 `InputReader.ButtonState` 字段，调用 `Once(action)` / `Toggle(action)` / `Reset()` |
| `ReadPressedOnce(ulong ownerId, ...)` 等公开 ownerId 重载移除 | 状态归调用方所有，不再使用实例 ID |
| `Rebind.BeginRebind` / `BeginCompositePartRebind` 重命名 | `Rebind.Begin(action, RebindRequest...)` |
| `Rebind.ResetBinding` / `ResetActionBindings` / `ResetCompositePartBinding` 合并 | `Rebind.Reset(action)`（全部）或 `Rebind.Reset(action, request)`（单条） |
| `Rebind.GetBindingDisplayString` 重命名 | `Rebind.GetDisplayString` |
| `RebindOptions` 改为公开的 `RebindRequest` | `RebindRequest.Default.WithCompositePart("Up")` |
| `RebindTarget.Joystick` / `Any` 移除 | 只保留 `KeyboardMouse` 与 `Gamepad` |
| `Haptics.SetIntensity` 移除 | 使用 `Haptics.Intensity` 属性 |
| `Watch.InputContext` 精简为 6 个字段 | 移除 `VendorId` / `ProductId` / `Layout` / `InterfaceName` / `Manufacturer` / `Product` |
| `Reader.ReadTrace` 移除 `ActionName` / `OwnerId` / `OwnerKey` | 使用 `Action` 与 `CompositePartName` |
| `InputDefineGenerator` 不再生成 `Once/Toggle/Reset` 辅助方法 | 生成类只暴露无状态访问器，边沿状态由业务持有 `ButtonState` |

## 实例化 API 迁移

| 原调用 | 当前调用 |
| --- | --- |
| `UXInput.Watch.InputType` / `InputProfile` / `InputContext` / `InputDeviceType` | `InputWatch.InputType` / `InputProfile` / `InputContext` / `InputDeviceType` |
| `UXInput.Reader.ButtonState` / `ReadTrace` / `ReadTraceKind` | `InputReader.ButtonState` / `ReadTrace` / `ReadTraceKind` |
| `InputActionProvider.ResolveAction` / `TryResolveAction` / `FindActionMap` | `UXInput.Reader.ResolveAction` / `TryResolveAction` / `FindActionMap` |
| 各模块独立启停、`InputStatics`、`ResetForPlayMode` | 仅通过 `UXInput.Initialize(actions, glyphDatabase)` / `UXInput.Shutdown()` 统一启停 |

日常调用仍为 `UXInput.Watch.xxx`、`UXInput.Reader.xxx` 等，但类型名使用 `InputWatch`、`InputReader`。动作资产和路径查询表由 `Reader` 持有，`ButtonState` 的读取和诊断事件通过当前 Reader 实例处理。

## 迁移后的行为变化

- `Reader` 不再提供 `ReadButton` / `ReadPressed` / `ReadCompositePartButton`，对应方法改为 `IsButtonPressed` / `IsPressed` / `IsCompositePartPressed`。
- `Rebind.CancelRebinding` 改为 `Rebind.Cancel()`。
- `InputActionProvider` 不再暴露动作解析方法；动作查询统一走 `UXInput.Reader`。
- 重绑定结果仍先进入暂存区，`ConfirmApply` 后才应用并保存；`OnBindingsChanged` 是刷新 UI 的统一事件。
- 已保存的绑定覆盖会保留，未保存的临时改绑不会跨启动继承。

## 相关页面

- [Input 概览](index.md)：启动、关闭和所有权规则
- [设备、读取与重绑定](../../extension/uxinput.md)：迁移后的完整用法
- [性能与分配](performance.md)：`ButtonState` 和 Reader 的 0 B 热路径
