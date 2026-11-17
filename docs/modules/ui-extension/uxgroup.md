# UXGroup

`UXGroup` 是 `UXToggle` 的单选分组容器：组内同一时刻只允许一个 Toggle 处于开启状态，并负责默认项、全关和前后切换。它挂在**分组的父节点**上，Toggle 挂在子节点。

## 适用场景

- 页签、分段选择器、设置面板里的单选组。
- 需要“默认选中第一个”“允许全部关闭”“上一项/下一项”这类分组行为。
- 用代码动态增删 Toggle，而不是只在 Inspector 里摆固定几个。

## 使用前提

- 已安装 `com.alicizax.unity.ui.extension`。
- 分组父节点挂 `UXGroup`（`AddComponentMenu` 为 `UI/UXGroup`，同一节点只能有一个）。
- 子节点挂 `UXToggle`；`UXToggle` 的 `group` 由 `UXGroup` 在初始化时自动回填，不需要手拖。
- 需要交互时节点上要有可接收点击的 `Graphic` 与 `GraphicRaycaster`。

## 最短接入

1. 建一个空节点，挂 `UXGroup`。
2. 在它下面建若干 Toggle 子节点，每个挂 `UXToggle`。
3. 把 Toggle 拖进 `UXGroup` 的 `Toggles` 数组（也可以留空，运行时用 `RegisterToggle` 注册）。
4. 需要默认选中项时设置 `Default Toggle`；不设置且 `Allow Switch Off` 为 false 时，组会自动选中第一个可用的 Toggle。
5. 监听每个 Toggle 的 `onValueChanged`，或在业务层读 `GetFirstActiveToggle()`。

```csharp
using UnityEngine;
using UnityEngine.UI;

public sealed class SettingsTabs : MonoBehaviour
{
    [SerializeField] private UXGroup _group;

    private void Start()
    {
        // 运行时切换默认项：会在下一次校验时生效
        _group.defaultToggle = _group.GetFirstActiveToggle();

        // 代码切换上一项 / 下一项
        _group.Next();
        _group.Previous();
    }

    public void CloseAll()
    {
        // 参数为 false 时不触发 Toggle 回调
        _group.SetAllTogglesOff(sendCallback: false);
    }
}
```

## 核心用法

### 运行时注册与注销

动态创建的 Toggle 必须在初始化后注册，销毁前注销：

```csharp
UXToggle toggle = Instantiate(_togglePrefab, _group.transform);
toggle.group = _group;          // 也可由 UXGroup 自动回填
_group.RegisterToggle(toggle);

// 销毁前
_group.UnregisterToggle(toggle);
Destroy(toggle.gameObject);
```

- `RegisterToggle` 对 `null` 和重复注册是安全的空操作。
- `UnregisterToggle` 只移除引用并清理默认/当前项，**不会**去选中其他 Toggle（销毁顺序不确定，避免触发回调与补间）。
- 若注册时 Toggle 已经是 `isOn`，会立即走一次单选同步。

### 查询与切换

| 成员 | 说明 |
| :--- | :--- |
| `AnyTogglesOn()` | 组内是否有选中项（会先校验状态） |
| `GetFirstActiveToggle()` | 第一个 `isOn` 的 Toggle，没有返回 `null` |
| `ContainsToggle(toggle)` | 是否已注册 |
| `NotifyToggleOn(toggle, sendCallback = true)` | 把某个 Toggle 设为当前项，并关掉其他项 |
| `SetAllTogglesOff(sendCallback = true)` | 全部关闭；期间临时允许“无选中” |
| `Next()` / `Previous()` | 循环选择下一个/上一个可交互且激活的 Toggle |
| `EnsureValidState()` | 手动触发一次整组校验（默认项、单选、Toggle 归属） |

### 状态校验时机

- `Start` 时校验一次，此时所有子 Toggle 的 `OnEnable` 已完成。
- `OnEnable`、`allowSwitchOff` / `defaultToggle` 赋值、注册/注销都会把组标记为脏，下一次 `EnsureValidState` 重新计算。
- 校验包含三步：同步每个 Toggle 的 `group` → 解析默认项 → 保证单选（不允许全关时至少选中一个）。

## 常见配置

| Inspector 字段 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `Allow Switch Off` | false | 允许组内没有任何选中项 |
| `Toggles` | 空 | 初始 Toggle 列表，只在首次初始化时读取一次 |
| `Default Toggle` | 空 | 默认选中项；为空且不允许全关时取第一个 |

`Toggles` 只是**初始来源**：初始化后所有增删都以运行时列表为准，运行期改这个数组不会自动同步。

## 容易踩的坑

1. **把 `UXGroup` 挂在 Toggle 自己身上**：分组要覆盖多个 Toggle，必须挂在共同父节点。
2. **`Allow Switch Off` 为 false 却手动全关**：会被下一次校验纠正回“至少一个选中”；确实需要全关时用 `SetAllTogglesOff`（内部临时放行）或打开该开关。
3. **运行时改了 `Toggles` 数组**：不会生效，用 `RegisterToggle` / `UnregisterToggle`。
4. **在 `Awake` 里读 `GetFirstActiveToggle`**：此时子 Toggle 可能还没 `OnEnable`，结果为空；放到 `Start` 之后或先调 `EnsureValidState()`。
5. **默认项被销毁**：`UnregisterToggle` 会清掉 `defaultToggle` 引用，下一次校验按规则重新选。

## API 速查

| 成员 | 类型 | 说明 |
| :--- | :--- | :--- |
| `allowSwitchOff` | 属性 | 是否允许无选中项 |
| `defaultToggle` | 属性 | 默认选中项 |
| `RegisterToggle(UXToggle)` / `UnregisterToggle(UXToggle)` / `ContainsToggle(UXToggle)` | 方法 | 运行时注册表 |
| `NotifyToggleOn(UXToggle, bool)` | 方法 | 设为当前项 |
| `AnyTogglesOn()` / `GetFirstActiveToggle()` | 方法 | 查询 |
| `SetAllTogglesOff(bool)` | 方法 | 全部关闭 |
| `Next()` / `Previous()` | 方法 | 循环切换 |
| `EnsureValidState()` | 方法 | 手动校验 |

相关：[UXToggle](uxtoggle.md)｜[UXSelectable 基础](uxselectable.md)
