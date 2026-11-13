# NavigationScope 导航

`NavigationScope` 在 Input System 之上管理 UI 焦点域：哪个窗口/面板能接收手柄或键盘焦点、谁挡住下层、关闭进缓存后为什么不能再被方向键选中。

源码位置：

- `Client/Packages/com.alicizax.unity.input/Runtime/Navigation`
- Editor：`Client/Packages/com.alicizax.unity.input/Editor/Navigation`

## 使用前提

1. 安装 `com.alicizax.unity.ui.extension` 和 `com.alicizax.unity.input`。
2. 安装 `com.unity.inputsystem` 并启用 Input System 后端。
3. 场景中有 `EventSystem`。
4. 场景中有 `InputService`，它会初始化 `UXInput.Navigation`。

## 给窗口加导航域

在窗口或面板根节点添加 `NavigationScope`：

```text
Add Component > UI > Navigation Scope
```

Inspector：

| 字段 | 说明 |
| --- | --- |
| `Default Selectable` | 进入该域时默认选中的控件 |
| `Holder` | 所属 `UIHolderObjectBase`；窗口/Widget 绑自身 Holder，普通面板绑父级 UI 的 Holder |
| `Baked Selectables` | 编辑器烘焙的静态控件 |
| `Remember Last Selection` | 重新打开时恢复上次选中 |
| `Block Lower Scopes` | 当前域存活时挡住下层导航 |

`Holder` 必须绑定；没有 Holder 的 Scope 不会存活。点击 Inspector 上的刷新按钮可按层级收集 `Selectable`。Scope 在 `OnEnable` 自动注册，Holder 显示/关闭或销毁时会自动刷新，不需要额外的导航管理器。

## 焦点与遮挡

- `Block Lower Scopes = true` 的 Scope 存活时，会挡住优先级更低的作用域。
- 作用域按 Canvas 排序、显示顺序、层级深度决定优先级；嵌套 Scope 优先于祖先。
- 有可用 `Selectable` 且有优先级的 Scope 才成为焦点。
- Loading 全屏层可以只开 `Block`、不配置可用控件：自己不当焦点，同时挡住下层。
- Toast/飘字可以不挡下层，让下层继续导航。

| 场景 | Block | 结果 |
| --- | --- | --- |
| 普通窗口/弹窗 | 开 | 自己接管焦点，挡住下层 |
| Loading 遮罩 | 开 | 挡住下层；没有可用控件时焦点为空 |
| Toast / 飘字 | 关 | 当自己不存在，下层继续导航 |
| 只展示不抢焦点的浮层 | 按需 | 不配置控件即可不成为焦点 |

## 动态注册控件

虚拟列表、运行时生成的按钮等不在烘焙列表里的控件要手动注册：

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.UI;

public sealed class RuntimeScopeItem : MonoBehaviour
{
    [SerializeField] private NavigationScope scope;
    [SerializeField] private Selectable selectable;

    private void OnEnable()
    {
        scope.RegisterSelectable(selectable, rememberable: true);
    }

    private void OnDisable()
    {
        scope.UnregisterSelectable(selectable);
    }
}
```

`NotifySelectableStateChanged()` 用于 `interactable`、`active` 变化后主动刷新；运行时修改 Canvas 排序也调用它。

## 补选策略

```csharp
UXInput.Navigation.GamepadRequireSelection = true;   // 默认 true
UXInput.Navigation.KeyboardRequireSelection = false; // 默认 false

UXInput.Navigation.SetRequireSelection(gamepad: true, keyboard: false);
```

- 只在当前没有合法焦点时补选；关闭策略不会清掉已有焦点。
- 手柄/键盘只在对应策略开启时自动补选。
- Scope 被上层遮挡时会清掉当前焦点，等上层关闭后按 `Remember Last Selection` 恢复。

## 与 RecyclerView 配合

`RecyclerViewNavigationController` 作为列表唯一的 EventSystem 选中对象，把自己注册进父级 `NavigationScope`。列表项继承 `NavigationViewHolder<TData>` 或 `InteractiveViewHolder<TData>` 参与虚拟焦点，见 [RecyclerView 导航](recyclerview/navigation.md)。

## API 速查

| API | 说明 |
| --- | --- |
| `NavigationScope.RegisterSelectable(Selectable, bool rememberable = false)` | 注册运行时控件 |
| `NavigationScope.UnregisterSelectable(Selectable)` | 注销运行时控件 |
| `NavigationScope.NotifySelectableStateChanged()` | 控件状态变化后刷新 |
| `NavigationScope.DefaultSelectable` | 默认选中 |
| `NavigationScope.RememberLastSelection` | 是否记住上次选中 |
| `NavigationScope.BlockLowerScopes` | 是否挡住下层 |
| `NavigationScope.NavigationSuppressed` | 当前是否被抑制 |
| `NavigationScope.NavigationSuppressionChanged` | 抑制状态变化事件 |
| `InputNavigation.GamepadRequireSelection` | 手柄自动补选，默认 true |
| `InputNavigation.KeyboardRequireSelection` | 键鼠自动补选，默认 false |
| `InputNavigation.SetRequireSelection(bool, bool)` | 一次设置两套策略 |

## 注意事项

1. `NavigationScope` 的 `Holder` 必须绑定，否则 Scope 不会存活。
2. 运行时生成的控件必须 `RegisterSelectable` / `UnregisterSelectable`，否则方向键找不到。
3. `Block` 只影响导航，不拦截鼠标；需要彻底挡住点击时配合 `CanvasGroup` 或 `GraphicRaycaster`。
4. Scope 被遮挡或没有候选控件时会被抑制；不要把它和业务选中状态混在一起。
5. Prefab 结构变化后重新烘焙 `Baked Selectables`。
