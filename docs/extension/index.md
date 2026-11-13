# UI Extension 扩展包

`com.alicizax.unity.ui.extension` 提供一组基于 UGUI 的增强控件和虚拟列表。当前文档对应包版本 `4.0.0`。

它不负责窗口生命周期；窗口、Holder 和层级仍由 [UI 界面](../modules/ui/index.md) 管理。快捷键、导航、输入图标和交互反馈已经拆到 `com.alicizax.unity.input`，见 [Input 输入](../modules/input/index.md)。

## 组件

| 控件/模块 | 用途 |
| --- | --- |
| [UXButton](uxbutton.md) | 增强按钮，保留 `onClick`，支持子节点状态和外部视觉状态 |
| [UXToggle](uxtoggle.md) | 增强 Toggle、`UXGroup` 单选分组 |
| [UXImage](uximage.md) | 渐变、镜像、描边、阴影和运行时图集 |
| [UXDraggable](uxdraggable.md) | 把 UGUI 拖拽事件转成 UnityEvent |
| [RecyclerView](recyclerview/index.md) | 虚拟滚动列表、多模板、分组、循环和导航 |

## 快速开始

1. 安装 `com.alicizax.unity.ui.extension`，并确认项目已接入 UI 模块。
2. 在 Hierarchy 右键创建控件：

```text
GameObject/UI/UXButton
GameObject/UI/UXToggle
GameObject/UI/UXImage
GameObject/UI/UXScrollView
GameObject/UI/UXTemplateWindow
```

3. 在窗口逻辑里通过生成的 Holder 字段访问控件：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;
using UnityEngine.UI;

[Window(UILayer.UI)]
public sealed class ShopWindow : UIWindow<ui_ShopWindow>
{
    protected override void OnInitialize()
    {
        baseui.BtnBuy.onClick.AddListener(OnBuy);
        baseui.TglAutoBuy.onValueChanged.AddListener(OnAutoBuyChanged);
    }

    private void OnBuy()
    {
        GameApp.UI.ShowUI<BuyConfirmWindow>().Forget();
    }

    private void OnAutoBuyChanged(bool isOn)
    {
        // ...
    }
}
```

## 命名空间

| 类型 | 命名空间 |
| --- | --- |
| `UXButton`、`UXToggle`、`UXGroup`、`UXImage`、`UXDraggable` | `UnityEngine.UI` |
| `RecyclerView`、`UGList`、`UGMixedList`、`UGGroupList`、`UGLoopList`、`ViewHolder` | `AlicizaX.UI` |
| 输入、快捷键、导航、反馈 | `AlicizaX.Input`（输入扩展包） |

## 可选输入扩展

需要输入相关能力时，再安装 `com.alicizax.unity.input`：

```text
Client/Packages/com.alicizax.unity.input
```

- UI 导航：[NavigationScope](navigation.md)
- 快捷键：[HotkeyComponent](hotkey-component.md)
- 输入图标：[InputGlyph](input-glyph.md)
- 交互音效：[Input Feedback](input-feedback.md)

UI extension 的 asmdef 会根据是否安装 input 包生成 `UXNAVIGATION_SUPPORT`；不需要手加宏。

## RecyclerView 快速创建

菜单 `GameObject/UI/UXScrollView` 会从包内模板创建一个带 `RecyclerView` 结构的滚动列表。然后在窗口里创建业务列表：

```csharp
using AlicizaX.UI;

private UGList<BagItemData> _items;

protected override void OnInitialize()
{
    _items = UGListCreateHelper.Create<BagItemData>(baseui.ScrollViewItems);
    _items.SetData(CreateItems());
}
```

`RecyclerView` 的 `Content`、`Templates`、`Adapter` 和 `LayoutManager` 在运行期不能更换；业务通过 `UGList` 等封装更新数据。

## 与 UI 模块配合

1. 先按 [UI 模块](../modules/ui/index.md) 搭好窗口、Holder 和生成流程。
2. 在 Prefab 中使用 `UXButton`、`UXToggle`、`UXImage`、`RecyclerView`。
3. 在窗口逻辑里通过 `baseui` 访问生成字段。
4. 需要手柄/键盘导航时再加 `NavigationScope` 和 `RecyclerViewNavigationController`。

## 注意事项

1. `UXButton` 不继承 Unity `Button`，但 `onClick` 仍然是 `Button.ButtonClickedEvent`，用法基本一致。
2. `RecyclerView` 的业务入口是 `UGList` 系列，不要直接改 `Adapter` 的内部列表。
3. `UXImage` 的 `FlipMode.Horziontal` 是源码里的实际拼写。
4. 输入图标、快捷键、导航和反馈都在 `com.alicizax.unity.input`，不要在本包内找旧组件。
