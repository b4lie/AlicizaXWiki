# UXToggle

`UXToggle` 是包里的开关与页签控件，继承 `UXSelectable`。它用 `UXGroup` 做互斥，用 `graphic` 的透明度表现开关，值变化通过 `onValueChanged`（`ToggleEvent`）抛给业务。

## 适用场景

- 商店、背包这类页签：同一时间只有一个选中，且不允许全部关掉。
- 单个复选开关：允许关掉，也不需要组。
- 需要代码批量刷新 `isOn` 但不想触发回调。
- 不适合：连续数值（用 `Slider`）；`UXToggle` 只认 `UXGroup`，不参与 Unity `ToggleGroup`。

## 使用前提

- 类型在 `UnityEngine.UI` 命名空间，`AddComponentMenu` 是 `UI/UXToggle`，并带 `RequireComponent(typeof(RectTransform))`。
- 创建菜单 `GameObject/UI/UXToggle`（Unity 6000.3 及以上为 `GameObject/UI (Canvas)/UXToggle`）先建 Unity 标准 Toggle 结构，再把 `Toggle` 换成 `UXToggle`，并回填 `Graphic` 与 `Target Graphic`。
- 需要 `EventSystem`；同一个节点不要再挂 uGUI `Toggle`。
- 生成器映射：`Tog` → `UXToggle`，`TogGroup` → `UXGroup`。生成器按规则列表顺序取第一个前缀命中，所以 `TogGroup` 规则必须排在 `Tog` 之前，否则 `TogGroup@Tabs` 会被生成成 `UXToggle`。

## 最短接入

1. 创建 `UXToggle` 节点，确认 `Graphic` 指向 `Background/Checkmark`，`Target Graphic` 指向 `Background`。
2. 需要单选：在父节点 Add Component → `UI/UXGroup`，点 Inspector 里的 `Collect Children`，再选 `Default Toggle`，`Allow Switch Off` 保持不勾。
3. 改名 `Tog@Recommend` 并生成绑定。
4. 订阅 `onValueChanged`（模板 `UIShopWindow` 的真实写法）：

```csharp
using AlicizaX.UI.Runtime;
using Game.UI;

namespace GameLogic.UI
{
    [Window(UILayer.UI, 3)]
    public class UIShopWindow : UIWindow<ui_UIShopWindow>
    {
        protected override void OnInitialize()
        {
            baseui.TogRecommend.onValueChanged.AddListener(OnTogRecommendChanged);
            baseui.TogRecommend.isOn = true;
        }

        private void OnTogRecommendChanged(bool isOn)
        {
            if (isOn)
            {
                // 切到推荐分类
            }
        }
    }
}
```

## 核心用法

### 字段

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `isOn` | `bool` | 当前值，赋值会走完整链路（含回调） |
| `group` | `UXGroup` | 所属单选组，赋值会注册或注销并刷新视觉 |
| `graphic` | `Graphic` | 靠 alpha 表现开关的图形，为空时没有开关视觉 |
| `onValueChanged` | `ToggleEvent` | 值变化事件，参数是切换后的值 |

`graphic` 与 `group` 的 setter 都会调 `RefreshVisual()`，所以运行时换引用会立刻刷新外观。

### ToggleEvent

`ToggleEvent` 就是 `[Serializable] public class ToggleEvent : UnityEvent<bool>`，参数是切换后的值。

- 只在值真正变化且 `sendCallback` 为真时触发一次。
- 组内兄弟被关掉时是否回调，跟随本次调用的 `sendCallback`：`isOn = true` 会带回调，`SetIsOnWithoutNotify(true)` 不会。

### isOn 的完整链路

`isOn` 的 setter 最终走 `Set(value, sendCallback)`：

1. 值相同，直接返回，没有回调也没有过渡。
2. 单选组不允许全关、且没有别的选中项时，请求被静默拒绝：值不变、不回调、不做过渡。
3. 调 `OnBeforeValueChanged(value)`。
4. 写入 `m_IsOn`；组处于激活状态且新值为真时，调 `group.NotifyToggleOn(this, sendCallback)` 关掉兄弟。
5. `sendCallback` 为真时打 `UISystemProfilerApi` marker 并 `onValueChanged.Invoke(m_IsOn)`。
6. `PlayEffect` 把 `graphic` 的 alpha 在 1 / 0 间 `CrossFadeAlpha`（0.1 秒；非 Play Mode 直接 `canvasRenderer.SetAlpha`），接着 `DoStateTransition` 刷状态（打开按 `Selected`，关闭按推导状态），最后调 `OnAfterValueChanged(value)`。

`SetIsOnWithoutNotify(bool)` 只是把第 5 步换成不发事件，视觉和组内互斥照常执行。

### 视觉状态

- `isOn = true`：状态强制 `Selected`，`graphic` alpha 到 1。
- `isOn = false`：状态用推导值，`graphic` alpha 到 0。
- 关闭 + `interactable = false`：`Disabled` 被映射成 `Normal`，看不到灰化。

因为打开时状态被固定成 `Selected`，主过渡的 `Highlighted` / `Pressed` 在打开状态下不会出现；要 Hover 效果得用继承来的[子节点过渡](uxselectable.md)给子节点配。`UXToggle` 还实现了 `ICanvasElement`，编辑期在 `CanvasUpdate.Prelayout` 与 `OnValidate` 时刷新视觉。

### 与 UXGroup 的关系

`UXToggle` 只认 `UXGroup`，不能和 Unity `ToggleGroup` 混用。

| 成员 | 说明 |
| --- | --- |
| `allowSwitchOff` | 默认 `false`，组内必须有一项选中 |
| `defaultToggle` | 默认项；在注册之后才解析，允许全关时可以为空 |
| `RegisterToggle` / `UnregisterToggle` / `ContainsToggle` | 成员维护，`OnEnable` 注册、`OnDisable` 与 `OnDestroy` 注销 |
| `NotifyToggleOn(toggle, sendCallback = true)` | 单选核心：关掉其他项 |
| `Next()` / `Previous()` | 选相邻的可交互项，越界回绕，跳过未激活与不可交互的项 |
| `SetAllTogglesOff(bool sendCallback = true)` | 全部关掉，内部临时放行「不允许全关」的检查 |

组在 `Start` 里做一次校验：没有选中项且不允许全关时，会把 `defaultToggle`（没配则第一个）设成 `isOn = true`，这一步会走 setter，因此可能触发一次 `onValueChanged`。

### 代码监听与批量刷新

```csharp
using UnityEngine;

public sealed class UXToggleDemo : MonoBehaviour
{
    [SerializeField] private UXToggle mTogA;
    [SerializeField] private UXToggle mTogB;

    private void OnEnable() => mTogA.onValueChanged.AddListener(OnAChanged);

    private void OnDisable() => mTogA.onValueChanged.RemoveListener(OnAChanged);

    public void ResetAll()
    {
        mTogA.SetIsOnWithoutNotify(false);   // 刷新视觉，但不触发 onValueChanged
        mTogB.SetIsOnWithoutNotify(false);
    }

    private void OnAChanged(bool isOn) => Debug.Log(isOn ? "A on" : "A off");
}
```

## 常见配置

| 配置 | 位置 | 说明 |
| --- | --- | --- |
| `Is On` | `Image` 页（`UXToggleEditor` 追加） | 编辑期直接改；不允许全关时改 false 会警告并回滚 |
| `Graphic` | `Image` 页 | 开关视觉，alpha 0 / 1 |
| `UXGroup` | `Image` 页 | 拖入所属组 |
| `On Value Changed` | `Event` 页 | bool 参数事件 |
| `Allow Switch Off` | `UXGroup` Inspector | 默认关闭 |
| `Default Toggle` | `UXGroup` Inspector | 不允许全关时必填，未指定时自动取第一个 |
| `Collect Children` / `Clean Nulls` / `Sort By Hierarchy` | `UXGroup` Inspector | 批量维护成员列表 |

## 容易踩的坑

1. **想关掉组内最后一个。** 请求被静默拒绝，没有回调也没有动画；要允许全关就勾 `Allow Switch Off`。
2. **`isOn = true` 没触发回调。** 值本来就是 `true` 时直接返回，去重是设计行为。
3. **以为 `SetIsOnWithoutNotify` 连视觉都不刷。** 它照常改值、照常刷 alpha 和状态，只是不发 `onValueChanged`。
4. **组在 `Start` 补默认选中时收到一次回调。** 那是 `isOn = true` 引起的；初始化里要么提前订阅并容忍首帧回调，要么显式赋值前先判断。
5. **`graphic` 留空。** 点击有回调，但看不见开关变化。
6. **和 uGUI `Toggle` / `ToggleGroup` 混用。** 两个组件都响应点击会切两次；`ToggleGroup` 对 `UXToggle` 无效。
7. **打开状态下等 Hover 反馈。** 状态被固定成 `Selected`，主过渡的 `Highlighted` 不会出现。
8. **关闭 + 不可交互时想看到灰化。** `ResolveVisualState` 把 `Disabled` 映射成 `Normal`，主过渡和子过渡都拿不到 `Disabled` 配色；这类提示要业务自己改色或换图。

## API 速查

| 成员 | 签名 | 说明 |
| --- | --- | --- |
| `isOn` | `bool` | 读写当前值，写会走完整链路 |
| `SetIsOnWithoutNotify` | `void(bool)` | 改值不回调 |
| `group` | `UXGroup` | 所属组 |
| `graphic` | `Graphic` | 开关视觉图形 |
| `onValueChanged` | `ToggleEvent` | `UnityEvent<bool>` |
| `group.NotifyToggleOn` | `void(UXToggle, bool)` | 组内单选 |
| `group.SetAllTogglesOff` | `void(bool = true)` | 全部关掉 |
| `group.Next` / `group.Previous` | `void()` | 相邻项切换 |
| `Set` | `protected virtual void(bool, bool)` | 子类可覆写的赋值入口 |
| `OnBeforeValueChanged` / `OnAfterValueChanged` | `protected virtual void(bool)` | 值变化前后钩子 |
