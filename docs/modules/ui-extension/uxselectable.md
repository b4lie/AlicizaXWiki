# UXSelectable 基础

`UXSelectable` 是包里所有可交互控件的基类，直接继承 uGUI 的 `Selectable`。它在 Unity 的状态机之外补了两件事：一套公开的 `UXSelectionState` 外部状态驱动，和一组「子节点过渡」，让一次 Hover / Press 能同时驱动多个 Graphic。

## 适用场景

- 自己写可交互控件（继承 `UXSelectable`），需要现成的状态机、子节点过渡和导航查找。
- 列表项、引导遮罩需要用代码强制控件显示高亮或按下态，而它并不是 EventSystem 当前选中的对象。
- 一个按钮身上有底板、图标、描边多个图形，必须同步变色或换图。
- 不适合：纯展示（用 `Image` / `UXImage`）；需要窗口或输入来源（交给框架 UI 与 Input 包）。

## 使用前提

- 创建菜单里没有单独的 `UXSelectable` 项，实际使用是 `UXButton`、`UXToggle` 或你自己的子类。手动挂一个裸 `UXSelectable` 没有点击行为，只有状态切换能力。
- 类型在 `UnityEngine.UI` 命名空间（不是 `AlicizaX.UI`），代码里 `using UnityEngine.UI;` 就能拿到。
- Inspector 由 `Editor/UX/Selectable/UXSelectableEditor.cs` 接管：继承 `SelectableEditor`，再用 `TabbedInspector` 加页签；页签选择记在 `EditorPrefs`，key 是 `UnityEngine.UI.UXSelectable.TabbedIndex`。
- 节点上要有 `Graphic`，场景里要有 `EventSystem`（模板 `UIRoot.prefab` 自带），否则收不到指针与选中事件。

## 最短接入

1. 写一个子类，挂在带 `Graphic` 的节点上。
2. 在 Inspector 的 `Image` 页设 `Main Transition` 与 `Target Graphic`，这是控件本体的反馈。
3. 需要多个子图形一起反馈时，给 `Child Transitions` 逐条填目标与配色。
4. 业务里按需调外部状态 API。

```csharp
using UnityEngine;
using UnityEngine.UI;

[AddComponentMenu("UI/UXFocusable")]
public class UXFocusable : UXSelectable
{
    public void ShowFocus()
    {
        // 不依赖 EventSystem 选中，直接显示高亮
        SetExternalState(UXSelectionState.Highlighted);
    }

    public void HideFocus()
    {
        ClearExternalState();
    }
}
```

## 核心用法

### UXSelectionState

| 值 | 对应 `Selectable.SelectionState` | 典型来源 |
| --- | --- | --- |
| `Normal` | `Normal` | 默认 |
| `Highlighted` | `Highlighted` | 指针进入、外部设置 |
| `Pressed` | `Pressed` | 按下、`PlayClickPulse` |
| `Selected` | `Selected` | EventSystem 选中、`UXToggle` 打开 |
| `Disabled` | `Disabled` | `interactable = false` |

两者值一致但类型不同，编译期不能互转；内部用 switch 映射，未知值落到 `Normal`。

### 外部状态

`DoStateTransition` 里只要 `HasExternalState` 为真，传入的 state 就会被替换成 `ExternalState`，所以主过渡和子过渡会一起被覆盖。

| 成员 | 行为 |
| --- | --- |
| `SetExternalState(UXSelectionState state, bool instant = false)` | 强制显示某个状态，`instant` 为真时不过渡 |
| `ClearExternalState(bool instant = false)` | 取消强制并回到推导状态；未强制时直接返回 |
| `PulseExternalState(UXSelectionState pulse, UXSelectionState restore, float duration = -1f)` | 先切到 `pulse`，到时回到 `restore`；`duration < 0` 用 `colors.fadeDuration` |
| `HasExternalState` / `ExternalState` | 是否在强制、当前强制的状态 |

- 控件未激活或 `duration <= 0` 时脉冲会立刻恢复，不排队。
- 待恢复的脉冲共用一个隐藏驱动对象 `[UXSelectablePulseDriver]`（`HideFlags.HideAndDontSave`），队列空了就销毁，所以控件再多也不会多出 per-instance `Update`。
- `OnDisable` / `OnDestroy` 会把控件移出脉冲队列，不再等恢复。

### TransitionData 与子节点过渡

`TransitionData` 是 `[Serializable]` 的普通类，字段正好对应一套 uGUI 过渡配置：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `targetGraphic` | `Graphic` | 这条过渡作用在哪个图形上 |
| `transition` | `Selectable.Transition` | 默认 `ColorTint`；实际只支持 `None` / `ColorTint` / `SpriteSwap` |
| `colors` | `ColorBlock` | 颜色与 `fadeDuration`，每条子过渡独立 |
| `spriteState` | `SpriteState` | `SpriteSwap` 用 |

- `ColorTint`：对该 Graphic 调 `CrossFadeColor(状态色 × colorMultiplier, fadeDuration)`，忽略时间缩放，所以每条子过渡可以有自己的时长。
- `SpriteSwap`：只对 `Image` 生效，写的是 `overrideSprite`；`Normal` 时把 `overrideSprite` 置空。
- `Animation` 不支持：Inspector 会提示并按 `ColorTint` 显示，运行时分支里也没有 `Animation`，配了等于不动。
- `targetGraphic` 为空时整条跳过，不报错。

### Inspector 通用字段

| 页签 | 字段 | 说明 |
| --- | --- | --- |
| `Image` | `Interactable` | 关掉后点击、Submit 都不触发，状态变 `Disabled` |
| `Image` | `Navigation` 与 `Visualize` | `Visualize` 在 Scene 视图画出导航连线 |
| `Image` | `Main Transition` | `Transition` + `Target Graphic` + `Colors` / `Sprite State` / `Animation Triggers`；选 `Animation` 时可以一键 `Auto Generate Animation` |
| `Image` | `Child Transitions` | 可增删排序的 `TransitionData` 列表 |
| `Fields` | 派生类的序列化字段 | 只在用通用 `UXSelectableEditor` 编辑子类时出现；`UXButton` / `UXToggle` 换成自己的 `Event` 页 |

### 继承扩展点

| 成员 | 可见性 | 用途 |
| --- | --- | --- |
| `DoStateTransition(SelectionState, bool)` | `protected override` | 状态切换的统一入口：先算外部状态，再走主过渡与子过渡 |
| `InstantClearState()` | `protected override` | 清空状态时把外部状态、子过渡一起复位 |
| `PulseDerivedState(UXSelectionState, float)` | `protected` | 没有外部状态时短暂切状态再恢复，[UXButton](uxbutton.md) 用它做点击脉冲 |
| `FindSelectableOnLeft/Right/Up/Down()` | `public override` | 方向键与摇杆的目标查找 |

### 与 Unity Selectable 的差异

| 差异点 | `Selectable` | `UXSelectable` |
| --- | --- | --- |
| 状态枚举 | `SelectionState` 只在内部使用 | 公开 `UXSelectionState`，可用 `SetExternalState` 强制 |
| 过渡目标 | 只有 `targetGraphic` 一个 | `targetGraphic` 加多条 `Child Transitions` |
| 子过渡类型 | — | 只支持 `ColorTint` / `SpriteSwap` |
| 重复状态 | 每次都会重放过渡 | Play Mode 下同一状态直接返回 |
| 编辑模式 | 按传入的 `instant` | 一律按 `instant = true` 处理 |
| Explicit 导航 | 直接返回 `selectOnUp` / `selectOnDown` / `selectOnLeft` / `selectOnRight` | 目标 `interactable` 为假时不采用，退回几何查找 |

## 常见配置

| 配置 | 位置 | 说明 |
| --- | --- | --- |
| `Transition` | `Image` 页 `Main Transition` | 本体过渡类型；`None` 时本体不变色，子过渡仍然生效 |
| `Target Graphic` | 同上 | 为空时主过渡不生效 |
| `Colors` / `Fade Duration` | 同上 | 主过渡用，也是 `PulseExternalState` 的默认时长 |
| `Child Transitions` | 同上 | 每条独立目标、类型、颜色、精灵 |
| `Navigation` | 同上 | `Explicit` 指定的四向目标要求 `interactable` |
| `Interactable` | 同上 | 逻辑开关，不负责自动灰化 |

## 容易踩的坑

1. **把 `UXSelectionState` 当 `Selectable.SelectionState` 传。** 两个独立枚举，需要自己映射。
2. **子过渡 `targetGraphic` 留空。** 静默跳过，看起来像「配了没效果」。
3. **给子过渡配 `Animation`。** 运行时没有该分支，什么都不发生；动画只能配主过渡。
4. **子过渡 `SpriteSwap` 指向非 `Image` 的 Graphic。** 同样静默跳过。
5. **外部状态开着时用鼠标点按钮。** `UXButton` 见到 `HasExternalState` 就不发 `onClick`，点击业务要由上层处理。
6. **靠重复 `SetExternalState` 重播过渡。** 相同状态会被去重，需要重播用 `PulseExternalState`。
7. **在编辑模式里等过渡动画。** 编辑器下强制 `instant`，颜色是直接跳变的。
8. **以为 `InstantClearState` 只影响本体。** 它会把子 `ColorTint` 复位成白色、子 `SpriteSwap` 清空 `overrideSprite`。

## API 速查

| 类型 | 成员 | 说明 |
| --- | --- | --- |
| `UXSelectable` | `HasExternalState`、`ExternalState` | 外部状态查询 |
| `UXSelectable` | `SetExternalState(state, instant)` | 强制状态 |
| `UXSelectable` | `ClearExternalState(instant)` | 取消强制 |
| `UXSelectable` | `PulseExternalState(pulse, restore, duration)` | 脉冲后恢复 |
| `UXSelectable` | `PulseDerivedState(pulse, duration)` | 子类用：无外部状态的脉冲 |
| `UXSelectable` | `DoStateTransition`、`InstantClearState` | 子类覆写点 |
| `UXSelectionState` | `Normal`、`Highlighted`、`Pressed`、`Selected`、`Disabled` | 公开状态枚举 |
| `TransitionData` | `targetGraphic`、`transition`、`colors`、`spriteState` | 子过渡数据 |
