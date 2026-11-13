# UXComponent 扩展组件

`UXComponent` 是 UI extension 的 UGUI 增强控件集合，重点解决状态过渡、Toggle 分组、渐变图片、拖拽和虚拟列表。快捷键、导航、输入图标和交互反馈已经拆到可选的 `com.alicizax.unity.input`。

源码位置：

- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/UXComponent`
- 编辑器：`Client/Packages/com.alicizax.unity.ui.extension/Editor/UX`

## 控件入口

| 文档 | 内容 |
| --- | --- |
| [UXButton](uxbutton.md) | 点击、Submit、子节点过渡、外部状态 |
| [UXToggle](uxtoggle.md) | Toggle、`UXGroup` 分组、默认选中 |
| [UXImage](uximage.md) | 渐变、镜像、描边、阴影、运行时图集 |
| [UXDraggable](uxdraggable.md) | 拖拽事件转发 |
| [RecyclerView](recyclerview/index.md) | 虚拟列表、模板与数据适配 |

## UXSelectable 基础

`UXButton` 和 `UXToggle` 都继承 `UXSelectable`。除了 UGUI `Selectable` 的 `Transition`、`Interactable`、`Navigation`，它还提供“外部状态”：

```csharp
using UnityEngine.UI;

public void SetHighlighted(UXSelectable selectable)
{
    selectable.SetExternalState(UXSelectionState.Highlighted);
}

public void ClearHighlight(UXSelectable selectable)
{
    selectable.ClearExternalState();
}
```

外部状态适合：

- 控件本身不是 EventSystem 当前选中对象，但需要显示高亮。
- 列表项、热键焦点、引导层等由业务驱动视觉状态。
- 临时按下脉冲：`PulseExternalState(UXSelectionState.Pressed, restoreState)`。

`UXSelectionState`：`Normal`、`Highlighted`、`Pressed`、`Selected`、`Disabled`。

## 子节点过渡

`UXSelectable` 可以让图标、文本、背景等子节点跟随控件状态变色或换图：

1. 在控件 Inspector 的 `Child Transitions` 添加子 `Graphic`。
2. 每个子项单独设置 `Transition`、`Colors` 或 `Sprite State`。
3. 子项的过渡类型彼此独立，可以混用颜色和图片；根节点自己的 `Target Graphic` 仍由根控件 `Transition` 决定。

```text
BtnBag (UXButton, Transition = ColorTint)
├── Bg    (Image, 根 Target Graphic)
├── Icon  (Image, Child Transitions)
└── Label (TextMeshProUGUI, Child Transitions)
```

## 依赖与宏

| 宏 | 来源 | 作用 |
| --- | --- | --- |
| `TEXTMESHPRO_SUPPORT` | 安装 TextMeshPro / UGUI 2.0+ | `UXButton` 创建菜单、TMP 相关组件 |
| `INPUTSYSTEM_SUPPORT` | 安装 `com.unity.inputsystem` | input 包内的输入代码 |
| `UXNAVIGATION_SUPPORT` | 安装 `com.alicizax.unity.input` | RecyclerView 导航接入 |

宏由 asmdef 自动生成，不需要在 Player Settings 手动添加。只使用基础 UX 控件时不需要安装 input 包。

## 注意事项

1. `UXButton`、`UXToggle` 等类型在 `UnityEngine.UI` 命名空间，和 UGUI 控件同一套使用方式。
2. `UXButton` 不继承 Unity `Button`，但 `onClick` 类型保持一致。
3. 不要在控件上再挂旧的 UI 音效组件；交互音效统一用 input 包的 `InputFeedbackObserver` 或 `UXInput.Feedback`，见 [Input Feedback](input-feedback.md)。
4. `UXSelectable` 的外部状态会覆盖 Selectable 自己推导的状态；清除后恢复推导状态。
