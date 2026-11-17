# NavigationScope 导航

`NavigationScope` 标记一块「焦点域」，`InputNavigation`（`UXInput.Navigation`）负责在设备切换、域显示或关闭、控件可用性变化时维护 `EventSystem` 的选中对象，并按层级决定哪个域可以拿焦点。一个域绑定一个 `UIHolderObjectBase`，域内控件来自编辑器烘焙或运行时注册。模块清单见 [Input 概览](index.md)。

## 适用场景

- 手柄打开窗口后自动选中默认按钮；键鼠默认不补选，需要时打开 `KeyboardRequireSelection`。
- 弹窗或 Widget 覆盖时挡住下层焦点，关闭后恢复。
- 关窗重开时记住上一次选中的控件。
- 列表、动态生成的控件加入和退出导航。

## 使用前提

- 工程安装 `com.unity.inputsystem`。`AlicizaX.Input` 的 asmdef 在版本 ≥ 1.0.0 时定义 `INPUTSYSTEM_SUPPORT`；没有该宏时整包类型不存在。
- 输入层已初始化（场景有 `InputService` 且 `Actions` 已填），否则 `NavigationScope.OnEnable` 找不到模块，作用域不会注册。
- 场景里有 `EventSystem` 与 UI 输入模块（新输入系统下是 `InputSystemUIInputModule`），Navigate / Submit / Cancel 动作已启用。
- 每个域必须绑定 `Holder`：Window / Widget 绑自身 Holder，普通物体绑所属父级 UI 的 Holder。无 Holder 的域既不拿焦点也不挡下层。
- 要让 `Refresh` 收集到并参与方向导航，`Selectable` 的 `Navigation.mode` 不能是 `None`；归属判定要求最近的 `NavigationScope` 是这个域（用 `GetComponentInParent`）。
- 补选策略：手柄默认自动补选（`GamepadRequireSelection = true`），键鼠默认不补选（`KeyboardRequireSelection = false`）。
- 可选依赖：UGUI `Selectable`、框架 `UIHolderObjectBase`。ui.extension 的 RecyclerView 导航要求 `UXNAVIGATION_SUPPORT`（其 asmdef 在 `com.alicizax.unity.input` ≥ 1.0.0 时定义）；本包不依赖 PrimeTween。

## 最短接入

1. 在窗口或面板根节点 Add Component → `UI/Navigation Scope`。
2. Inspector 会自动补 `Holder`；在 `Selectables` 区点 `Refresh` 收集本域内可导航控件，再用 `Default` 下拉指定默认焦点。
3. 保持 `Block` 开启（`_blockLowerScopes` 默认 `true`）：本域存活时压制下层。
4. 需要恢复上次焦点时打开 `Remember`（`_rememberLastSelection`，默认 `false`）。
5. 动态控件在启用时注册、销毁前注销。

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.UI;

public sealed class DynamicNavItem : MonoBehaviour
{
    [SerializeField] private NavigationScope scope;
    [SerializeField] private Selectable selectable;

    private void OnEnable()
    {
        if (scope != null && selectable != null)
            scope.RegisterSelectable(selectable, true);   // true：跨关闭/遮挡仍可记忆
    }

    private void OnDisable()
    {
        if (scope != null && selectable != null)
            scope.UnregisterSelectable(selectable);       // 必须在销毁前注销
    }
}
```

## 核心用法

### 选中优先级与记忆

选中优先级从高到低是：上次选中（仍可用）→ `Default Selectable` → 域内第一个可用控件。「可用」指 `Selectable.IsActive()` 且 `IsInteractable()`。

`Remember` 与注册时的 `rememberable` 分工不同：同一域内切换设备始终保留最近选中；只有跨越「关闭 / 被遮挡」边界时才看 `Remember` 与控件的 `rememberable`。烘焙控件默认 `rememberable = true`，运行时 `RegisterSelectable` 默认 `false`。

### 遮挡与压制

`BlockLowerScopes`（默认 `true`）表示本域存活时压制所有更低优先级的域。压制时域内控件的 `navigation.mode` 被写成 `None`，释放时逐个写回进入压制前的原值，不会破坏你配置的方向导航。

- 存活条件：`scope.isActiveAndEnabled && Holder != null && Holder.IsEnabled`；除了节点处于激活层级，还要求 Holder 的视图逻辑状态为可用。
- 状态查询：`NavigationSuppressed`；变化通知：`NavigationSuppressionChanged`，只在值真正变化时触发。
- 空域（没有任何可用控件）也会挡住下层，但自己拿不到焦点。

### 补选策略

`SetRequireSelection(bool gamepad, bool keyboard)` 一次改两个开关；`GamepadRequireSelection` / `KeyboardRequireSelection` 单独设置。关闭补选不会清空已有的合法焦点，只影响「没有选中时是否主动补一个」。

```csharp
using AlicizaX.Input;

// 设置页这类键鼠也要默认焦点的界面
UXInput.Navigation.SetRequireSelection(true, true);

// 纯手柄游戏：只让手柄补选
UXInput.Navigation.GamepadRequireSelection = true;
UXInput.Navigation.KeyboardRequireSelection = false;
```

键鼠补选关闭时，选中未归属任何导航域的对象导航层不接管；只有选中属于某个域、该域被压制或控件失效时才清理。

### 排序与嵌套

嵌套子域优先于它的祖先域；其余按 `InputLayerOrder` 比较：Canvas `sortingOrder` → 显示序号（域从不可用变为可用时更新）→ 层级深度。没有 Canvas 的域排在所有 Canvas 之下。

运行时修改 Canvas `sortingOrder`、或改非当前域的 `interactable`，每帧快路径发现不了，需要调用 `NavigationScope.NotifySelectableStateChanged()` 主动刷新。

### 与 RecyclerView 导航集成

`RecyclerViewNavigationController` 继承 `Selectable`，是列表在 EventSystem 中的唯一选中对象：`OnEnable` 时 `RegisterSelectable(this, true)` 到父级 `NavigationScope`，`OnDisable` 注销，`navigationScope` 字段在编辑器里自动找最近的父级域。列表内虚拟焦点移动不产生 EventSystem 的 `Select`，焦点音效由控制器补发 `InputFeedbackCue.FocusEnter`。这是 `UXNAVIGATION_SUPPORT` 的消费方，列表侧配置见 [RecyclerView 导航](../ui-extension/recyclerview/navigation.md)。

## 常见配置

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `_defaultSelectable`（Default） | 空 | 默认焦点；必须在域内控件列表中 |
| `_holder`（Holder） | 自动取父级 | 必填；无 Holder 的域不存活 |
| `_bakedSelectables`（Selectables） | 空数组 | 点 `Refresh` 自动收集；烘焙项不可注销 |
| `_rememberLastSelection`（Remember） | `false` | 跨关闭 / 遮挡恢复上次选中 |
| `_blockLowerScopes`（Block） | `true` | 本域存活时压制下层 |
| `GamepadRequireSelection` | `true` | 手柄无选中时自动补选 |
| `KeyboardRequireSelection` | `false` | 键鼠无选中时自动补选 |

## 容易踩的坑

1. **没有 Holder，或 Holder 的视图没打开。** 域一直 `NavigationSuppressed`，既不拿焦点也不挡下层；普通面板记得绑父级窗口的 Holder。
2. **控件 `Navigation.mode = None`。** `Refresh` 收集时会被跳过，运行时也不会被补选；需要显式方向导航就用 `Explicit`。
3. **运行时改 Canvas 排序或非当前域的 `interactable`。** 快路径检测不到，必须 `NotifySelectableStateChanged()`。
4. **动态控件忘记注册或注销。** 没注册导航看不见；销毁前没 `UnregisterSelectable` 会在编辑器触发断言 "Call UnregisterSelectable before destroying it."。
5. **以为键鼠也自动补选。** 默认 `KeyboardRequireSelection = false`，键鼠只在当前选中失效或被遮挡时清理，不会凭空选一个。
6. **同一个节点或嵌套层级里挂多个域。** 归属只看最近的 `GetComponentInParent<NavigationScope>`，子节点只属于最近的那个域；父域不会重复管理，`Refresh` 也不会把子节点的控件收集进来。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `NavigationScope.RegisterSelectable(Selectable, bool)` | 注册运行时控件；重复或不属于本域时返回 `false` |
| `NavigationScope.UnregisterSelectable(Selectable)` | 注销；烘焙控件返回 `false` |
| `NavigationScope.DefaultSelectable` | get / set；set 会请求刷新 |
| `NavigationScope.NavigationSuppressed` | 当前是否被压制 |
| `NavigationScope.NavigationSuppressionChanged` | 压制状态变化事件，参数为新状态 |
| `NavigationScope.RememberLastSelection` / `BlockLowerScopes` | 只读配置值 |
| `NavigationScope.NotifySelectableStateChanged()` | 手动请求一次全量刷新 |
| `NavigationScope.RuntimeSelectableCount` | 运行时注册的控件数量 |
| `InputNavigation.SetRequireSelection(bool, bool)` | 一次设置手柄 / 键鼠补选 |
| `InputNavigation.GamepadRequireSelection` / `KeyboardRequireSelection` | 单设备补选开关 |
| `UXInput.Navigation` | 导航模块实例；未初始化时 `null` |
