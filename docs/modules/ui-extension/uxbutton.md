# UXButton

`UXButton` 是 `UXSelectable` 的按钮实现，事件与 uGUI `Button` 对齐（`onClick` 就是 `Button.ButtonClickedEvent`），额外提供点击脉冲 `PlayClickPulse()` 和「外部状态优先」的点击规则。

## 适用场景

- 底板、图标、文字要一起做 Hover / Press 反馈。
- 鼠标点击与键盘、手柄 Submit 走同一个回调，且都要尊重 `interactable`。
- 焦点由业务控制（列表、引导），需要代码强制高亮。
- 不适合：不需要任何视觉反馈的纯点击热区（直接用 uGUI 组件更轻）；点击音效也不在这里，见下文。

## 使用前提

- 类型在 `UnityEngine.UI` 命名空间，`AddComponentMenu` 是 `UI/UXButton`。
- 需要 `EventSystem`（模板 `UIRoot.prefab` 已带 `InputSystemUIInputModule`）。
- 同一个节点不要再挂 uGUI `Button`：两者都实现 `IPointerClickHandler`，一次点击会触发两份 `onClick`。
- 创建菜单 `GameObject/UI/UXButton` 只在 `TEXTMESHPRO_SUPPORT` 下存在；没有 TextMeshPro 时该菜单项不参与编译，手动 Add Component 的 `UXButton` 本身不使用 TMP。
- 生成器把 `Btn` 前缀映射到 `UXButton`，节点叫 `Btn@Confirm` 时字段是 `BtnConfirm`。

## 最短接入

1. 菜单创建：`GameObject/UI/UXButton`（Unity 6000.3 及以上版本是 `GameObject/UI (Canvas)/UXButton`）。
2. 创建出来的节点已带 `Image` 与 TMP 子文本，确认 `Target Graphic` 指向底板 `Image`。
3. 改名成 `Btn@Confirm`，在窗口预制体上执行 `GameObject/UI生成/绑定`。
4. 在逻辑类里订阅 `onClick`：

```csharp
using AlicizaX.UI.Runtime;
using Game.UI;

namespace GameLogic.UI
{
    [Window(UILayer.Popup, -1)]
    public class UIBuyAlertWindow : UIWindow<ui_UIBuyAlertWindow>
    {
        protected override void OnInitialize()
        {
            baseui.BtnConfirm.onClick.AddListener(OnBtnConfirmClick);
            baseui.BtnClose.onClick.AddListener(OnBtnCloseClick);
        }

        private void OnBtnConfirmClick()
        {
            // 业务处理
        }

        private void OnBtnCloseClick()
        {
            CloseSelf();   // CloseSelf 有默认参数，不能直接当 UnityAction 用
        }
    }
}
```

5. 需要纯代码创建时，`new GameObject(..., typeof(RectTransform))` 后 `AddComponent<UXButton>()`，再把底板 `Image` 赋给 `targetGraphic`，否则主过渡不生效。

## 核心用法

### 字段与事件

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `onClick` | `Button.ButtonClickedEvent` | 点击或提交时 `Invoke`，Inspector 在 `Event` 页 |
| `interactable` | `bool`（继承） | 关掉后 `OnPointerClick` / `OnSubmit` 都直接返回 |
| `targetGraphic` | `Graphic`（继承） | 主过渡作用的图形 |
| `transition` | `Selectable.Transition`（继承） | 主过渡类型 |
| `navigation` | `Navigation`（继承） | 方向键与摇杆的目标 |

`onClick` 是 `UnityEvent`，用 `AddListener` / `RemoveListener` 即可；它没有 uGUI `Button` 那种在 Inspector 根节点上的内置事件折叠面板，事件配置在 `UButtonEditor` 注册的 `Event` 页签里。

### 点击链路

| 输入 | 入口 | 行为 |
| --- | --- | --- |
| 鼠标左键 | `OnPointerClick` | 非左键返回；外部状态、未激活、不可交互返回；否则 `Press()` |
| Submit（键盘/手柄） | `OnSubmit` | 未激活、不可交互返回；先 `PlayClickPulse()` 再 `Press()` |

- `Press()` 先打 `UISystemProfilerApi.AddMarker("Button.onClick", this)`，再 `onClick.Invoke()`，Profiler 里能直接看到这次点击。
- 鼠标点击不额外播放脉冲：按下态由 `Selectable` 自己的指针逻辑驱动，再补一次会重复。
- `PlayClickPulse()` 是 `public virtual` 的：没有外部状态时脉冲 `Pressed` 再回到推导状态；有外部状态时回到 `ExternalState`（原本就是 `Pressed` 则回到 `Highlighted`）。
- `UXControlInteractionPlayModeTests.ClickAndSubmitRespectInteractableState` 断言了这条边界：`interactable = false` 后点击、Submit、`PlayClickPulse()` 都不生效。

### 子节点视觉状态

按钮的整体反馈通常不只改底板：把图标、描边、文字分别加进继承来的子节点过渡（[UXSelectable 基础](uxselectable.md)），Hover / Press 时它们会按各自的 `ColorBlock` 一起过渡。

| 场景 | 配置 |
| --- | --- |
| 底板变色、图标不变 | 底板放 `Main Transition`，图标不配置 |
| 底板变色 + 图标同时提亮 | 底板 `Main Transition`，图标一条 `ColorTint` 子过渡 |
| 整套换精灵 | `Main Transition` 用 `SpriteSwap`，要额外换图的子节点各配一条 `SpriteSwap` |

子过渡只支持 `ColorTint` / `SpriteSwap`，`Animation` 只能配在主过渡上。

### 导航（`UXNAVIGATION_SUPPORT`）

- `UXButton` 是标准 `Selectable`，EventSystem 的选中、方向移动、Submit 都走 uGUI 那套；焦点能落到谁身上由 `navigation` 和 `UXSelectable` 覆写的 `FindSelectableOnLeft/Right/Up/Down` 决定。
- 覆写带来的差异：`Navigation.Mode.Explicit` 指定的目标只有 `interactable` 为真才会被采用；否则在 `Horizontal` / `Vertical` 模式下退回按方向几何查找，不会把焦点扔给一个禁用控件。
- `UXNAVIGATION_SUPPORT` 由 asmdef 在 `com.alicizax.unity.input` ≥ 1.0.0 时定义，但 `UXButton` 自己不用这个宏；包里只有 RecyclerView 的导航控制器（含它的编辑器面板）、`NavigationViewHolder`、`UGList` 引用它。
- 谁在什么时候选中按钮、焦点域怎么切换，由 Input 包的 `UXInput.Navigation`（`NavigationScope` 的 `Selectable`、`DefaultSelectable`、`BlockLowerScopes` 等）负责，见 [UI 导航](../input/navigation.md)。

### 与 Unity Button 的差异

| 差异点 | uGUI `Button` | `UXButton` |
| --- | --- | --- |
| 基类 | `Selectable` | `UXSelectable`（多外部状态与子过渡） |
| 外部状态 | 没有 | `HasExternalState` 为真时鼠标点击不发 `onClick` |
| 点击脉冲 | 无 API | `PlayClickPulse()` |
| Submit | 按下态后延时触发 `onClick` | 先脉冲再立即 `onClick.Invoke()` |
| 子节点过渡 | 不支持 | `Child Transitions` |
| 事件面板 | Inspector 根节点 | `Event` 页签 |

### 运行时禁用与换图

```csharp
public sealed class UXButtonStateDemo : MonoBehaviour
{
    [SerializeField] private UXButton mBtnConfirm;

    public void SetEnabled(bool enabled) => mBtnConfirm.interactable = enabled;   // 只挡交互

    public void SwapBackSprite(Sprite sprite)
    {
        // 主过渡是 SpriteSwap 时状态机会写 overrideSprite，业务改 sprite 才不会被覆盖
        if (mBtnConfirm.targetGraphic is Image back)
            back.sprite = sprite;
    }
}
```

换文字走子节点的 TMP 组件，模板里的写法是 `button.GetComponentInChildren<TextMeshProUGUI>(true)`，见 `Client/Assets/Scripts/Hotfix/GameLogic/UI/RecyclerViewTest/UIRecyclerViewExampleWindow.cs`。

## 常见配置

| 配置 | 位置 | 说明 |
| --- | --- | --- |
| `Transition` = `ColorTint` | `Image` 页 | 最常用；`Colors` 里 `Disabled Color` 决定禁用外观 |
| `Transition` = `SpriteSwap` | `Image` 页 | 要求 `Target Graphic` 是 `Image` |
| `Transition` = `Animation` | `Image` 页 | 只能配主过渡，可 `Auto Generate Animation` |
| `Target Graphic` | `Image` 页 | 为空时主过渡不生效 |
| `Child Transitions` | `Image` 页 | 图标、描边的同步反馈 |
| `Navigation` | `Image` 页 | 方向导航，`Visualize` 可看连线 |
| `On Click()` | `Event` 页 | 编辑器里挂回调 |

## 容易踩的坑

1. **`Button` 和 `UXButton` 同时存在。** 一次点击两份 `onClick`。创建菜单会删掉 `Button`，手动加组件时注意。
2. **外部状态开着时点鼠标没反应。** `OnPointerClick` 见到 `HasExternalState` 直接返回，这是有意为之：点击业务由上层处理。但 `OnSubmit` 仍会发 `onClick`，两条输入路径并不一致。
3. **拿 `PlayClickPulse()` 当提示动画。** 不激活或不可交互时它什么都不做。
4. **在 `UXButton` 上找音效。** 没有音效代码；给同一节点挂 `InputFeedbackObserver`（`UI/Input Feedback Observer`），见[交互反馈](../input/input-feedback.md)。
5. **Sprite Swap 下业务改 `overrideSprite`。** 状态切换会覆盖它，改 `sprite`。
6. **以为 `interactable = false` 会自动灰。** 它只挡住交互，视觉由 `Colors.DisabledColor` 或子过渡提供。
7. **Unity 6 里菜单找不到。** 前缀是 `GameObject/UI (Canvas)/`。
8. **没有 TMP 就没有创建菜单。** 手动 Add Component 仍然可用。

## API 速查

| 成员 | 签名 | 说明 |
| --- | --- | --- |
| `onClick` | `Button.ButtonClickedEvent` | 点击与提交事件 |
| `PlayClickPulse()` | `public virtual void` | 手动播放按下脉冲 |
| `OnPointerClick` | `public virtual void(PointerEventData)` | 左键点击入口，外部状态时不发事件 |
| `OnSubmit` | `public virtual void(BaseEventData)` | Submit 入口，先脉冲后事件 |
| `interactable` | `bool` | 继承自 `Selectable` |
| `targetGraphic` | `Graphic` | 继承自 `Selectable` |
| `transition` | `Selectable.Transition` | 继承自 `Selectable` |
| `navigation` | `Navigation` | 继承自 `Selectable` |
| `SetExternalState` / `ClearExternalState` | `UXSelectable` | 外部状态 |
