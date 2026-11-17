# HotkeyComponent 快捷键

`HotkeyComponent` 把一个 Input System 动作注册到最近 `UIHolderObjectBase` 的热键作用域。按键发生时，模块按「按下瞬间的最上层 Holder」沿层级链分发，命中的组件执行 `TriggerHotkey()`，再把提交转给同物体的 `ISubmitHandler` 和 `IHotkeySubmitListener`。模块入口与启动顺序见 [Input 概览](index.md)。

## 适用场景

- 手柄 B / 键盘 Esc 关闭当前窗口，且只对最上层窗口生效。
- 弹窗、页签内快捷键：被遮挡时不响应，关掉后自动恢复。
- 列表项或自定义控件需要热键提交，但不想继承 `Button`。
- 同一节点上多个快捷键（各自动作、各自 Press Type）。

## 使用前提

- 工程安装 `com.unity.inputsystem`。`AlicizaX.Input` 的 asmdef 在版本 ≥ 1.0.0 时定义 `INPUTSYSTEM_SUPPORT`；没有该宏时整包类型不存在。
- 输入层已初始化（场景有 `InputService` 且 `Actions` 已填），否则 `UXInput.Hotkey` 为 `null`，组件不会注册。
- 组件必须在 `UIHolderObjectBase` 之下。编辑器用 `GetComponentInParent<UIHolderObjectBase>(true)` 自动写 `Holder`；找不到 Holder 时 Inspector 直接报「不会在运行时注册」。
- `Input Action` 必填，且动作必须存在于当前 `InputActionAsset`（按 action id 解析）。注册不要求动作已启用，但只有启用后按键才会触发。
- `TriggerHotkey()` 需要 `EventSystem.current`，没有 EventSystem 时提交被跳过。
- 可选依赖：UGUI（`ISubmitHandler`、`EventSystem`）与框架的 `UIHolderObjectBase`；本包不依赖 PrimeTween。

## 最短接入

1. 在窗口或按钮节点 `Add Component` 搜索 `Hotkey Component`（该组件没有 `AddComponentMenu` 菜单项），节点放在 Window / Widget 的 Holder 层级内。
2. Inspector 里把 `Input Action` 指向动作引用，例如 `UI/Back`；`Press Type` 默认 `Performed`。
3. 同物体挂 `Button`，或挂任何实现 `ISubmitHandler` 的组件；`Submit Target` 由编辑器扫描后写回，Inspector 中只读显示。
4. 由动作所有者启用 map，例如 `InputDefine.UI.Enable = true`（生成方式见 [InputGlyph 输入图标](input-glyph.md)）或 `UXInput.Reader.FindActionMap("UI").Enable()`。
5. 不需要手写注册：`OnEnable` 注册到当前作用域，`OnDisable` 注销。

```csharp
using AlicizaX.Input;
using UnityEngine;

public sealed class HotkeyMapBoot : MonoBehaviour
{
    private void Start()
    {
        if (!UXInput.IsInitialized)
            return;

        // 热键只观察动作：动作没启用时注册成功，但按键不会触发。
        UXInput.Reader.FindActionMap("UI")?.Enable();
    }
}
```

## 核心用法

### 触发链与优先级

按下瞬间，模块通过 `IUIService.TryGetTopVisibleHolder` 取最上层可见 Holder（带 `HotkeyPassThrough` 的跳过），再找最深的 scope 作为 leaf，然后沿 Holder 链向上派发：命中第一个存在该 action 与 pressType 可用注册的 scope 就停止；已销毁或未启用的 trigger 不算命中，继续向父级冒泡。同一 Holder 内后注册（后 `OnEnable` / `SetActive`）者优先；父 scope 只在子 scope 没有可用注册时收到。

跨 scope 排序与导航共用 `InputLayerOrder`：Canvas `sortingOrder` → 显示序号（后显示优先）→ 层级深度，细节见 [NavigationScope 导航](navigation.md)。

### Press Type

| 值 | 触发时机 | 分发目标 |
| --- | --- | --- |
| `Started` | 按下 | 捕获本次按下的 Holder 与 leaf scope，然后派发 |
| `Performed` | 动作触发 | 只发给 `Started` 时捕获的目标；按下时没有目标就丢弃整次 |
| `Canceled` | 松开或取消 | 发给同一目标，随后清空按下目标 |

同一个动作可以被多个组件以不同 Press Type 注册，模块按 action 共享订阅并分别计数；换按法不需要改动作资产。

### 提交目标与旁路监听者

`HotkeyComponent.TriggerHotkey()` 先对 `Submit Target` 调 `ISubmitHandler.OnSubmit`，再遍历监听者列表调 `IHotkeySubmitListener.OnHotkeySubmit`。同一个组件既当提交目标又当监听者时只会收到一次；目标或监听者失活时跳过。

`Submit Target` 与监听者列表只由编辑器期 `ResolveSubmitTargets()` 扫描写回（`Reset` / `OnValidate`），运行时不再扫描，所以运行时 `AddComponent<HotkeyComponent>()` 的组件没有提交目标。

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.EventSystems;

public sealed class HotkeyHint : MonoBehaviour, IHotkeySubmitListener
{
    public void OnHotkeySubmit(BaseEventData eventData)
    {
        // 不替代 ISubmitHandler，只做额外通知；eventData.used 可能已被提交目标置位。
    }
}
```

### 运行期换绑与改键

`HotkeyAction` 的 setter 在播放中且组件启用时重新注册；新配置注册失败（动作解析不到、Holder 丢失、PassThrough 动作）会回滚为旧引用，旧注册保留。改键不需要换组件：注册保存的是 `InputAction` 实例，binding override 和 `UXInput.Rebind` 生效后按键自动跟着变，做法见 [设备、读取与重绑定](uxinput.md)。

```csharp
using AlicizaX.Input;
using UnityEngine.InputSystem;

public static class HotkeySwitch
{
    // 播放中切换到另一个动作；失败时 hotkey.HotkeyAction 仍是旧值。
    public static void Swap(HotkeyComponent hotkey, InputActionReference next)
    {
        hotkey.HotkeyAction = next;
    }
}
```

### 作用域生命周期与 HotkeyPassThrough

Holder 的 `OnShowEvent` 会刷新作用域并更新显示序号，`OnCloseEvent` 清掉该 Holder 的按下目标，`OnDestroyEvent` 注销该 Holder 作用域里的全部注册。可用性要求 Holder 节点在激活层级中，且 `Holder.IsEnabled`（视图逻辑状态）为 `true`，所以窗口淡出期间快捷键是否响应由窗口状态决定，参见 [窗口](../ui/window.md)。

`HotkeyPassThrough` 挂在 Holder 上，让该 Holder 不作为热键焦点目标，通常给全屏遮罩、加载层或转场层使用：按键会落到下层窗口，而不是被遮罩吃掉。

## 常见配置

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `_holder`（Holder） | 父级自动获取 | 只读；为空时不注册 |
| `_hotkeyAction`（Input Action） | 空 | 必填；按 action id 在当前资产解析 |
| `_hotkeyPressType`（Press Type） | `Performed` | `Started` / `Performed` / `Canceled` |
| `_submitTarget`（Submit Target） | 编辑器扫描 | 只读；实现 `ISubmitHandler`，运行时不再扫描 |
| `_listenerComponents` | 编辑器扫描 | `IHotkeySubmitListener[]`，不替代提交目标 |
| `HotkeyPassThrough` | 无 | 挂在 Holder 上，使其不作为热键焦点目标 |

## 容易踩的坑

1. **节点不在 Holder 下。** 不会注册，Inspector 报 "No UIHolderObjectBase was found in parents. This hotkey will not register at runtime."。
2. **动作没启用。** 注册成功但按键无反应；注册与动作启用是两件事，排错时先看 map 是否 `Enable()`。
3. **运行时 `AddComponent<HotkeyComponent>()`。** 运行时不再扫描提交目标，组件拿不到 `Submit Target`，热键会进 `TriggerHotkey()` 后直接返回。
4. **用 PassThrough 动作。** 注册会被直接拒绝；非 Button 动作会在编辑器 / 开发包给警告，Value 动作按每次值变化派发。
5. **输入层重建后不补注册。** 新会话里组件必须重新 `OnEnable`（切 `SetActive`、重开窗口）才会注册，旧注册随 shutdown 清空。
6. **以为热键是全局的。** 分发目标是按下瞬间的最上层 Holder，背景窗口的快捷键不会响应；遮罩层要挂 `HotkeyPassThrough` 才不挡按键。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `HotkeyComponentBase.TriggerHotkey()` | 抽象方法，实现具体触发行为 |
| `HotkeyComponentBase.HotkeyAction` | get / set；运行期换绑，失败回滚 |
| `HotkeyComponentBase.HotkeyPressType` | 只读，注册时的 Press Type |
| `HotkeyComponentBase.HotkeyHolder` | 只读，注册使用的 Holder |
| `IHotkeySubmitListener.OnHotkeySubmit(BaseEventData)` | 旁路提交通知 |
| `HotkeyPassThrough` | Holder 焦点穿透标记 |
| `EHotkeyPressType` | `Started` / `Performed` / `Canceled` |
| `UXInput.Hotkey` | 热键模块实例；未初始化时 `null`，注册由组件驱动 |
