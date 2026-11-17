# InputFeedback 交互反馈

`InputFeedback` 把三类交互提示（`PointerEnter`、`Press`、`FocusEnter`）按当前输入设备解析成主题里的音效，交给框架 `IAudioService` 以 `AudioType.UISound` 播放。`InputFeedbackObserver` 负责把同一 GameObject 上的 UGUI 交互事件转成提示，业务侧只需要设置全局主题。模块清单见 [Input 概览](index.md)。

## 适用场景

- 悬停、点击、焦点进入时播放 UI 音效。
- 手柄导航时给焦点音效，且不想让自动补选每次都响。
- 同一套 UI 按设备播放不同音效。
- 个别控件用独立主题，而不跟随全局配置。

## 使用前提

- 工程安装 `com.unity.inputsystem`。`AlicizaX.Input` 的 asmdef 在版本 ≥ 1.0.0 时定义 `INPUTSYSTEM_SUPPORT`；没有该宏时整包类型不存在。
- 输入层已初始化（场景有 `InputService` 且 `Actions` 已填），否则 `UXInput.Feedback` 为 `null`。
- 全局主题必须由业务设置，`UXInput.Feedback.Profile` 为 `null` 时全静音。输入层重建后 `Feedback` 是新实例，主题要重新赋值。
- 播放依赖框架的 `IAudioService`（`AppServices` 注册，见 [音频模块](../audio/index.md)）；没有音频服务时播放是空操作，不报错。
- `InputFeedbackObserver` 必须和控件挂在同一个 GameObject 上（`Awake` 里缓存同物体的 `Selectable`）；没有 `Selectable` 时按始终可用处理，但控件本身仍要在 EventSystem 的射线 / 选中体系内。
- 可选依赖：UGUI、框架音频服务；本包不依赖 PrimeTween。

## 最短接入

1. 创建主题资源：`Assets/Create/AlicizaX/Input/Feedback Profile`。
2. 在主题里配置规则：`Cue`（事件）、`Devices`（设备掩码）、`Clip`（音频）。新建资源默认三条：悬停 / 键鼠、焦点 / 手柄、按下 / 全设备。
3. 把主题放到加载路径，工程示例是 `Client/Assets/Resources/UX/InputFeedbackProfile.asset`。
4. 启动流程里设置全局主题。
5. 给按钮挂 `UI/Input Feedback Observer`，`Mode` 保持 `Global`。

```csharp
using AlicizaX.Input;
using UnityEngine;

public static class FeedbackBootstrap
{
    public static void Apply()
    {
        if (!UXInput.IsInitialized)
            return;

        // 全局主题；null 表示静音。输入层重建后需要重新赋值。
        UXInput.Feedback.Profile = Resources.Load<InputFeedbackProfile>("UX/InputFeedbackProfile");
    }
}
```

## 核心用法

### 三种提示与事件来源

| 提示 | Observer 接收的事件 | 生效条件 |
| --- | --- | --- |
| `PointerEnter` | `IPointerEnterHandler.OnPointerEnter` | 控件 `IsActive()` |
| `FocusEnter` | `ISelectHandler.OnSelect` | 事件参数不是 `PointerEventData`（指针点击引起的选中不发） |
| `Press` | 左键 `IPointerClickHandler`、`ISubmitHandler`、`IHotkeySubmitListener` | 控件 `IsActive()` 且 `IsInteractable()` |

没有 `Selectable` 的物体（例如列表项）按始终可用处理，悬停与按下照常发。

### 规则匹配

`InputFeedbackProfile` 按列表顺序取第一条 `Cue` 相等且 `Devices` 含当前设备的规则；没有匹配就静音，不会回退别的规则。`AllowProgrammaticFocus` 只对 `FocusEnter` 有意义：导航层同步设置选中期间（切换导航域、关窗恢复、补选）算程序化焦点，默认不播放，勾上才播。

### 设备映射

`InputType.Touch → InputFeedbackDevice.Touch`、`Gamepad → Gamepad`、`KeyboardMouse` 与 `Unknown` 都映射到 `KeyboardMouse`。设备由 `InputWatch` 跟踪，业务不用手动传设备。

```csharp
using AlicizaX.Input;
using UnityEngine;

// 列表虚拟焦点这类没有 EventSystem Select 的交互，由宿主手动发提示。
public sealed class ListFocusSound : MonoBehaviour
{
    public void OnVirtualFocus()
    {
        UXInput.Feedback?.Play(InputFeedbackCue.FocusEnter);
    }
}
```

### Global 与 Custom 模式

- `Mode = Global`：用全局主题 `UXInput.Feedback.Profile`；Inspector 不显示主题字段，编辑器下 `_profile` 会被 `OnValidate` 清空。
- `Mode = Custom`：用组件自己的 `_profile`；留空则静音，不回退全局主题。
- 需要自定义播放时继承 `InputFeedbackObserver` 并重写 `Play(InputFeedbackCue)`，它是 `virtual`，可以完全绕开音效或接动效。

```csharp
using AlicizaX.Input;
using UnityEngine;

public sealed class ToggleFeedback : InputFeedbackObserver
{
    public override void Play(InputFeedbackCue cue)
    {
        if (cue == InputFeedbackCue.Press)
            base.Play(cue);   // 只关心按下；悬停与焦点保持安静
    }
}
```

### 主题编辑器

主题 Inspector 按规则卡片编辑：`事件`、`设备`、`音频`，`FocusEnter` 规则多一个「允许程序化焦点发声」。同一个 `Cue` 下设备掩码不能重叠，编辑器只列出尚未被占用的设备；出现重叠会提示「运行时采用列表首条」，可点「修复重叠规则」自动裁剪，或用「添加规则」补齐还没覆盖的事件 / 设备组合。

## 常见配置

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `_mode`（Mode） | `Global` | `Global` 用全局主题，`Custom` 用本组件主题 |
| `_profile`（Profile） | 空 | Custom 模式使用；留空静音，不回退全局 |
| `Rule.Cue` | 三条：`PointerEnter` / `FocusEnter` / `Press` | 提示类型 |
| `Rule.Devices` | 键鼠 / 手柄 / 全部 | 设备位掩码，同 Cue 内不可重叠 |
| `Rule.AllowProgrammaticFocus` | `false` | 只对 `FocusEnter` 生效 |
| `Rule.Clip` | 空 | 匹配到也静音 |
| `UXInput.Feedback.Profile` | `null` | 全局主题；`null` 表示静音 |

## 容易踩的坑

1. **只挂 Observer 没设全局主题。** Global 模式读 `UXInput.Feedback.Profile`，为 `null` 就全静音；主题要在启动流程里赋值。
2. **输入层重建后音效消失。** 新会话的 `Feedback` 是新实例，`Profile` 回到 `null`，需要重新设置（工程里由 `HotfixEntry` 在启动时加载）。
3. **切窗口或补选时 `FocusEnter` 一直响。** 程序化焦点默认不播放；要响就在规则的 `AllowProgrammaticFocus` 上勾选。
4. **Custom 主题留空。** 静音且不回退全局主题；想跟随全局就切回 `Global`。
5. **一个物体挂两个 Observer。** `[DisallowMultipleComponent]` 只允许一个，想要多种策略就拆子物体。
6. **规则重叠与交互条件。** 同一 `Cue` 与设备的规则只有列表首条生效；右键点击不发 `Press`，`interactable = false` 时悬停仍响、按下不响。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `InputFeedback.Profile` | 全局主题；`null` 静音 |
| `InputFeedback.Play(InputFeedbackCue)` | 用全局主题播放 |
| `InputFeedback.Play(InputFeedbackCue, InputFeedbackProfile)` | 指定主题播放；`profile` 为 `null` 静音，不回退全局 |
| `InputFeedbackObserver.Play(InputFeedbackCue)` | `virtual`，可重写 |
| `InputFeedbackProfile.Rule` | `Cue` / `Devices` / `AllowProgrammaticFocus` / `Clip` |
| `InputFeedbackCue` | `PointerEnter` / `Press` / `FocusEnter` |
| `InputFeedbackDevice` | `None` / `Touch` / `KeyboardMouse` / `Gamepad` / `All` |
| `InputFeedbackMode` | `Global` / `Custom` |
