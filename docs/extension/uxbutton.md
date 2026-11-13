# UXButton

`UXButton` 继承 `UXSelectable`，用法接近 Unity `Button`，通过 `onClick` 注册点击事件，同时支持子节点状态过渡和外部视觉状态。它不负责音效；交互音效由 input 包的 `InputFeedbackObserver` 处理。

源码位置：

- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/UXComponent/Button/UXButton.cs`
- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/UXComponent/Selectable/UXSelectable.cs`

## 基础用法

```csharp
using AlicizaX.UI.Runtime;
using Game.UI;

[Window(UILayer.UI)]
public sealed class LoginWindow : UIWindow<ui_LoginWindow>
{
    protected override void OnInitialize()
    {
        baseui.BtnLogin.onClick.AddListener(OnLoginClick);
        baseui.BtnClose.onClick.AddListener(() => CloseSelf());
    }

    private void OnLoginClick()
    {
        // 登录逻辑。
    }
}
```

启用/禁用按钮：

```csharp
baseui.BtnConfirm.interactable = CanDeleteItem(_targetId);
```

点击和 Submit 都会触发 `onClick`。按钮不可交互或节点未激活时不会触发。

## 子节点状态

`UXButton` 继承 `UXSelectable` 的 `Child Transitions`。适合按钮内有多层图标、文字和背景的情况：

```text
BtnBag (UXButton, Transition = ColorTint)
├── Bg    (Image, 根 Target Graphic)
├── Icon  (Image, Child Transitions)
└── Label (TextMeshProUGUI, Child Transitions)
```

每个子项单独配置过渡类型：

- `Transition = ColorTint`：配置 `Colors` 的各状态颜色。
- `Transition = SpriteSwap`：配置 `Sprite State` 的各状态图片。

子项之间可以混用颜色和图片过渡；根节点自己的 `Target Graphic` 仍由根控件 `Transition` 控制。

## 外部视觉状态

按钮不是 EventSystem 当前选中对象时，也可以由业务驱动高亮或按下：

```csharp
using UnityEngine.UI;

public void SetHot(UXButton button, bool hot)
{
    if (hot)
    {
        button.SetExternalState(UXSelectionState.Highlighted);
    }
    else
    {
        button.ClearExternalState();
    }
}
```

API：

| API | 作用 |
| --- | --- |
| `SetExternalState(UXSelectionState state, bool instant = false)` | 强制视觉状态，覆盖 Selectable 推导状态 |
| `ClearExternalState(bool instant = false)` | 清除外部状态，恢复推导状态 |
| `PulseExternalState(UXSelectionState pulseState, UXSelectionState restoreState, float duration = -1f)` | 播放一次按下/高亮脉冲，结束后恢复 |
| `PlayClickPulse()` | 只播放点击视觉脉冲，不触发 `onClick` |

`UXSelectionState`：`Normal`、`Highlighted`、`Pressed`、`Selected`、`Disabled`。

外部状态激活时，左键点击不会触发 `onClick`，点击业务由驱动外部状态的逻辑处理；`OnSubmit` 仍会正常提交。列表项等需要“外部焦点视觉 + 自己处理点击”的场景应使用 `InteractiveViewHolder`，不要直接给 `UXButton` 设置外部状态后再依赖它的 `onClick`。

## 导航行为

`UXButton` 沿用 UGUI `Navigation`。当 `Navigation.Mode` 是 `Explicit` 且显式目标不可交互时，方向查找会返回 `null`，避免焦点跳到禁用按钮；`Horizontal` / `Vertical` / `Automatic` 仍按 UGUI 规则查找。

需要手柄/键盘焦点管理时，给父窗口挂 `NavigationScope`，见 [NavigationScope](navigation.md)。

## 与 Unity Button 的差异

1. 只响应左键点击，右键不会触发 `onClick`。
2. `OnSubmit` 会先播放点击脉冲，再触发 `onClick`，键盘/手柄/热键可以复用同一套业务。
3. `onClick` 类型仍是 `Button.ButtonClickedEvent`，`AddListener` / `RemoveListener` 写法不变。
4. 按钮禁用或节点未激活时，点击和 Submit 都不触发。
5. 没有内置音效字段；需要 hover/click 音时加 `InputFeedbackObserver`，见 [Input Feedback](input-feedback.md)。

## 注意事项

1. 不要把点击逻辑写在 `OnPointerClick` 里再调用 `onClick`，会重复触发。
2. `SetExternalState` 会覆盖 Selectable 状态，清除后再回到指针/选中推导状态。
3. 子节点过渡只应包含需要跟随状态的 `Graphic`，不要包含独立动画的装饰节点。
4. 需要“按下音效 + 视觉脉冲”时，加 `InputFeedbackObserver`，见 [Input Feedback](input-feedback.md)，不要给 `UXButton` 挂旧 clip 组件。
