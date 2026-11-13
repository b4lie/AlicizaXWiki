# HotkeyComponent 快捷键

`HotkeyComponent` 用 `InputActionReference` 触发 UI 上的提交行为。它只观察动作，不启用/禁用 InputAction；动作由 `InputService` 统一启用。

源码位置：

- `Client/Packages/com.alicizax.unity.input/Runtime/Hotkey`
- Editor：`Client/Packages/com.alicizax.unity.input/Editor/Hotkey`

## 组件结构

| 类型 | 作用 |
| --- | --- |
| `HotkeyComponentBase` | 热键组件基类：Holder、Action、Press Type、注册/注销 |
| `HotkeyComponent` | 默认 Submit 适配器：把热键转成 `ISubmitHandler.OnSubmit`，并通知旁路监听者 |
| `IHotkeySubmitListener` | 同物体上额外的热键提交监听接口 |
| `HotkeyPassThrough` | 标记组件：让某个 Holder 不参与热键焦点竞争 |
| `InputHotkey` | 热键注册、焦点链和分发模块 |

## 基础用法

在按钮上挂 `UXButton` 和 `HotkeyComponent`：

```text
BtnConfirm (UXButton + HotkeyComponent)
├─ Submit Target: BtnConfirm 上的 UXButton
├─ Holder: 自动找到 ConfirmWindow 的 UIHolderObjectBase
├─ Input Action: UI/Submit
└─ Press Type: Performed
```

按下 `UI/Submit` 时，`HotkeyComponent` 调用目标对象的 `ISubmitHandler.OnSubmit`，再触发 `UXButton.onClick`。

Inspector：

| 字段 | 说明 |
| --- | --- |
| `Submit Target` | 接收提交的 `ISubmitHandler`；必须有 |
| `Holder` | 所属 `UIHolderObjectBase`，自动向上查找 |
| `Input Action` | 热键动作 |
| `Press Type` | `Started`、`Performed`、`Canceled`，默认 `Performed` |

同一个 GameObject 可以挂多个 `HotkeyComponent`，每个组件有自己的 Action 和提交目标。

## 旁路监听

除了 `ISubmitHandler`，同物体实现 `IHotkeySubmitListener` 的组件也会收到热键提交：

```csharp
using AlicizaX.Input;
using UnityEngine.EventSystems;

public sealed class SubmitSound : MonoBehaviour, IHotkeySubmitListener
{
    public void OnHotkeySubmit(BaseEventData eventData)
    {
        // 只旁路监听，不替代 OnSubmit。
    }
}
```

监听者由 `HotkeyComponent` 在编辑器期解析并序列化，运行时不再扫描组件。

## 自定义热键

只做业务动作、不需要提交到控件时，继承 `HotkeyComponentBase`：

```csharp
using AlicizaX.Input;

public sealed class CloseWindowHotkey : HotkeyComponentBase
{
    public override void TriggerHotkey()
    {
        // 关闭窗口或执行其它动作。
    }
}
```

在 Inspector 配置 `Input Action` 和 `Press Type` 即可。`OnEnable` 注册，`OnDisable` 注销；子类覆盖时必须调用 `base`。

## 焦点链

热键不是所有窗口同时响应：

1. 取 UI 栈最上层且未挂 `HotkeyPassThrough` 的 Holder。
2. 在该 Holder 及其子 Holder 中找叶子作用域。
3. 按 Canvas 排序、显示顺序和层级深度选择作用域。
4. 从叶子 Holder 向上分发，直到命中注册的热键。

`HotkeyPassThrough` 用于 Loading、Toast、纯展示浮层：它不抢热键焦点，下层继续响应。

```text
AWindow 有 Cancel 热键
打开 BWindow
```

| BWindow | Cancel 结果 |
| --- | --- |
| 没挂 `HotkeyPassThrough`，也没有热键 | 焦点在 B，A 不响应 |
| 挂了 `HotkeyPassThrough` | 跳过 B，热键打到 A |
| 自己有 Cancel 热键 | B 响应 |

不要依赖“B 没配热键就自动透传”，需要透传时主动挂 `HotkeyPassThrough`。

## 注册规则

- 同一个 Holder 的同一个 action/pressType 允许多个组件注册，后注册（后启用）的优先。
- 不同 Holder 的作用域按 Canvas 排序、显示序号、层级深度选择。
- 热键注册在 `OnEnable`，禁用或节点失活会自动注销；重新启用会补注册。
- 热键动作必须由 `InputService` 加载的同一份 `InputActionAsset` 提供；找不到动作时注册失败。
- 动作不要求是 Button 类型，但非 Button 动作在 Development 下会输出提示；`PassThrough` 动作没有 `Started` 阶段，注册会直接失败。

## API 速查

| API | 说明 |
| --- | --- |
| `HotkeyComponentBase.HotkeyAction` | 读取或切换热键动作；注册失败会回滚 |
| `HotkeyComponentBase.HotkeyPressType` | 当前触发阶段 |
| `HotkeyComponentBase.HotkeyHolder` | 所属 Holder |
| `HotkeyComponentBase.TriggerHotkey()` | 热键触发回调，业务重写 |
| `IHotkeySubmitListener.OnHotkeySubmit(BaseEventData)` | 旁路提交监听 |

## 注意事项

1. 热键组件必须位于 `UIHolderObjectBase` 的子树内，否则没有所属作用域。
2. `HotkeyComponent` 的 `Submit Target` 必须实现 `ISubmitHandler`；`UXButton`、`UXToggle` 满足。
3. 目标对象或 Holder 未激活时不会注册；重新启用后自动注册。
4. 不要手动调用 `RegisterHotkey` / `UnregisterHotkey`，它们由生命周期维护。
5. 同一窗口多个控件使用同一 action 时，由启用顺序决定优先级；需要确定性时只保留一个启用组件。
