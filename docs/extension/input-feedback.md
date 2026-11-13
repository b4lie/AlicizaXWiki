# Input Feedback 交互反馈

`InputFeedback` 根据当前输入设备播放 UI 交互音效，支持全局主题和单个控件的主题覆盖。它属于 `com.alicizax.unity.input`，不依赖 UI extension 的旧音效组件。

源码位置：

- `Client/Packages/com.alicizax.unity.input/Runtime/Feedback`
- Editor：`Client/Packages/com.alicizax.unity.input/Editor/Feedback`

## 启动与主题

输入层初始化后设置全局主题：

```csharp
using AlicizaX.Input;
using UnityEngine;

// HotfixEntry / 启动流程中，输入层初始化之后。
if (UXInput.Feedback != null)
{
    UXInput.Feedback.Profile = Resources.Load<InputFeedbackProfile>("UX/InputFeedbackProfile");
}
```

`UXInput.Feedback` 由 `InputService` 创建；输入层未初始化时为 `null`。输入层重建后需要重新设置主题。

主题是 `InputFeedbackProfile` ScriptableObject，菜单：

```text
Assets > Create > AlicizaX > Input > Feedback Profile
```

每条规则包含：

| 字段 | 说明 |
| --- | --- |
| `Cue` | `PointerEnter`、`Press` 或 `FocusEnter` |
| `Devices` | `Touch`、`KeyboardMouse`、`Gamepad` 的掩码；可用 `All` |
| `Clip` | 播放的 `AudioClip`；为空则静音 |
| `AllowProgrammaticFocus` | 仅对 `FocusEnter` 生效；导航层程序化选中时是否仍播放 |

匹配规则是“提示类型 + 当前设备”的第一条命中。

## 给控件加音效

在控件同一个 GameObject 上添加：

```text
Add Component > UI > Input Feedback Observer
```

`InputFeedbackObserver` 监听同一对象上的 UGUI 交互：

- `PointerEnter`：鼠标进入。
- `FocusEnter`：EventSystem 选中，且不是指针引起。
- `Press`：左键点击、Submit、热键提交。
- 控件不可交互或未激活时不播放。

Inspector：

| 字段 | 说明 |
| --- | --- |
| `Mode` | `Global` 使用全局主题；`Custom` 使用 `Profile` |
| `Profile` | `Custom` 模式的主题；为空时静音，不回退全局主题 |

通常直接在 Inspector 配置 `InputFeedbackObserver`，不需要代码创建。运行时切换全局主题：

```csharp
if (UXInput.Feedback != null)
{
    UXInput.Feedback.Profile = Resources.Load<InputFeedbackProfile>("UX/InputFeedbackProfile_SciFi");
}
```

## 手动播放

没有对应 EventSystem 事件的交互（例如列表内虚拟焦点移动）由业务主动调用：

```csharp
using AlicizaX.Input;

public static class UiFeedback
{
    public static void PlayPress()
    {
        UXInput.Feedback?.Play(InputFeedbackCue.Press);
    }

    public static void PlayFocus(InputFeedbackProfile custom)
    {
        UXInput.Feedback?.Play(InputFeedbackCue.FocusEnter, custom);
    }
}
```

`Play(cue, profile)` 的 `profile` 为 `null` 时直接静音，不会回退到全局主题。

## 设备与程序化焦点

- 设备来自 `UXInput.Watch.CurrentInputType`：触摸、手柄、键鼠。
- 手柄/键盘导航程序化补选时，`FocusEnter` 默认不播放；需要播放时给对应规则勾选 `AllowProgrammaticFocus`。
- 鼠标点击引起的 `OnSelect` 不重复发 `FocusEnter`。
- 没有注册 `IAudioService` 时跳过播放，不报错。

## 注意事项

1. `InputFeedbackObserver` 必须和 `Selectable` 挂在同一个 GameObject 上。
2. `Custom` 模式留空 `Profile` 是静音，不是回退全局。
3. 输入层重建后，`UXInput.Feedback` 是新的实例；全局主题需要重新赋值。
4. 主题规则的 `Devices` 是位掩码，同一个 Cue 可以配置多条设备规则。
5. `AllowProgrammaticFocus` 只对 `FocusEnter` 有意义；其它 Cue 忽略该字段。
