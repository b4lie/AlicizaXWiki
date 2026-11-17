# UI Extension 扩展组件

`com.alicizax.unity.ui.extension`（4.0.2，Unity 2022.3）是 uGUI 之上的一层控件扩展：`UXSelectable` 提供可外部驱动的交互状态和「子节点过渡」，`UXButton`、`UXToggle`、`UXGroup` 是它的具体控件，再往上是 RecyclerView 列表体系。它只管控件长什么样、怎么响应，不管窗口生命周期、资源加载、输入读取和音效。

源码位置：

| 内容 | 路径 |
| --- | --- |
| 控件运行时 | `Client/Packages/com.alicizax.unity.ui.extension/Runtime`（`Selectable`、`Button`、`Group` 子目录） |
| 列表运行时 | `Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView` |
| 编辑器（Inspector / 创建菜单） | `Client/Packages/com.alicizax.unity.ui.extension/Editor` |
| 交互测试 | `Client/Packages/com.alicizax.unity.ui.extension/Tests/PlayMode/UXControlInteractionPlayModeTests.cs` |
| 真实使用 | `Client/Assets/Scripts/Hotfix/GameLogic/UI`、`Client/Assets/Bundles/UI/Window` |

## 适用场景

- 一个按钮需要底板、图标、文字等多个图形一起做 Hover / Press 反馈。
- 焦点由业务决定：列表虚拟焦点、引导高亮要在控件没有被 EventSystem 选中时也显示高亮或按下态。
- 页签式单选：一组里永远只有一个选中，且不允许全部关掉。
- 需要一个按钮的回调与 uGUI `Button` 完全一致，但视觉更强。
- 不适合：窗口的打开关闭、层级与缓存（交给 [UI 界面](../ui/index.md)）；输入读取、导航焦点与点击音效（交给 Input 输入）。

## 使用前提

控件的运行时不依赖框架服务：PlayMode 测试就是在裸 `GameObject` 上 `AddComponent` 验证点击的；`baseui` 字段绑定和 `ui_*` 生成才依赖框架 UI 的 Holder。包内 asmdef 对这些包是硬引用，见下表。

| 依赖 | 角色 | 用途 | 缺失后果 |
| --- | --- | --- | --- |
| `com.unity.ugui` 2.0.0 | 必需 | `Selectable`、`Graphic`、`Button.ButtonClickedEvent` | 整个包无法编译 |
| 框架 UI（Holder + 生成流程） | 模板内必需 | `Btn` / `Tog` 前缀生成 `ui_*` 字段 | 控件仍可手动挂，只是没有 `baseui` 字段 |
| TextMeshPro 3.0.9 | 本工程已装，宏可选 | 创建菜单 `UXButton` 用 TMP 按钮模板，文本用 `TextMeshProUGUI` | 菜单项不参与编译；asmdef 硬引用 `Unity.TextMeshPro`，卸包前先改引用 |
| PrimeTween 1.4.0 | 本工程已装，宏可选 | RecyclerView 滚动条平滑滚动 | 滚动条缓动退化为直接赋值；asmdef 硬引用 `PrimeTween.Runtime` |
| `com.alicizax.unity.input` | 本工程已装，宏可选 | RecyclerView 列表导航控制器 | 导航代码不参与编译；asmdef 硬引用 `AlicizaX.Input` |

包以 embedded package 形式放在 `Client/Packages/com.alicizax.unity.ui.extension`，不需要在 `Client/Packages/manifest.json` 里额外声明。场景里必须有 `EventSystem`：模板的 `UIRoot.prefab` 已自带（`InputSystemUIInputModule`）。

编译宏由 `Runtime/AlicizaX.UI.Extension.asmdef` 的 `versionDefines` 自动产生，不需要手填：

| 宏 | 触发条件 | 实际作用范围 |
| --- | --- | --- |
| `TEXTMESHPRO_SUPPORT` | `com.unity.ugui` ≥ 2.0.0，或存在 `com.unity.textmeshpro` | 创建菜单 `GameObject/UI/UXButton` |
| `PRIMETWEEN_SUPPORT` | `com.kyrylokuzyk.primetween` ≥ 1.2.1 | `ScrollbarEx` 的平滑滚动 |
| `INPUTSYSTEM_SUPPORT` | `com.unity.inputsystem` ≥ 1.0.0 | 与 `UXNAVIGATION_SUPPORT` 组合，决定 RecyclerView 导航代码是否编译 |
| `UXNAVIGATION_SUPPORT` | `com.alicizax.unity.input` ≥ 1.0.0 | RecyclerView 导航控制器、`NavigationViewHolder`、`UGList` |

宏只是包内代码开关：`Runtime/AlicizaX.UI.Extension.asmdef` 与 `Editor/AlicizaX.UI.Extension.Editor.asmdef` 对上面几个包都是硬引用，本工程这些包都已安装；要在新工程裁剪依赖，得同时改 asmdef 的 `references`。`UXSelectable`、`UXButton`、`UXToggle`、`UXGroup` 的代码本身不含任何宏分支。

## 最短接入

1. 在窗口预制体里创建控件：Hierarchy 右键 `GameObject/UI/UXButton`（本工程是 Unity 2022.3；Unity 6000.3 及以上版本为 `GameObject/UI (Canvas)/UXButton`，下同）。
2. 按绑定格式给节点改名，例如 `Btn@Confirm`、`Tog@Recommend`、`TogGroup@Tabs`。前缀与 `@` 的含义见[窗口声明与生命周期](../ui/lifecycle.md)。
3. 选中窗口预制体，执行菜单 `GameObject/UI生成/绑定`，生成 Holder 与 `ui_*` 字段。
4. 在窗口逻辑类里用 `baseui` 拿控件（模板真实写法，摘自 `Client/Assets/Scripts/Hotfix/GameLogic/UI/Shop/UIShopWindow.cs`）：

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
            baseui.BtnClose.onClick.AddListener(OnBtnCloseClick);
            baseui.TogRecommend.onValueChanged.AddListener(OnTogRecommendChanged);
        }

        private void OnBtnCloseClick()
        {
            CloseSelf();
        }

        private void OnTogRecommendChanged(bool isOn)
        {
            if (isOn)
            {
                // 只有被选中时才切分类
            }
        }
    }
}
```

5. 没有预制体、需要代码创建时，直接 `AddComponent` 并补 `targetGraphic`：

```csharp
var go = new GameObject("Btn", typeof(RectTransform));
go.transform.SetParent(parent, false);
var image = go.AddComponent<Image>();
var button = go.AddComponent<UXButton>();
button.targetGraphic = image;   // 不设的话主过渡不生效
```

## 核心用法

### 创建菜单

菜单项全部由 `Editor/Inspector/UXCreateHelper.cs` 注册：

| 菜单（Unity 2022.3） | Unity 6000.3+ | 产物与行为 |
| --- | --- | --- |
| `GameObject/UI/UXButton` | `GameObject/UI (Canvas)/UXButton` | 先走 TMP 的 `AddButton`，再删掉 `Button` 换成 `UXButton`；包在 `TEXTMESHPRO_SUPPORT` 里 |
| `GameObject/UI/UXToggle` | `GameObject/UI (Canvas)/UXToggle` | 先建标准 Toggle，把 `Toggle` 换成 `UXToggle`，回填 `Graphic`（`Background/Checkmark`）与 `Target Graphic`（`Background`），保留原 `isOn` |
| `GameObject/UI/UXImage` | `GameObject/UI (Canvas)/UXImage` | 把 `Image` 换成 `UXImage` |
| `GameObject/UI/UXScrollView` | `GameObject/UI (Canvas)/UXScrollView` | 在选中节点下实例化并解包 RecyclerView 的 `ScrollView.prefab`；未选中时取当前 Prefab Stage 根 |
| `GameObject/UI/UXTemplateWindow` | `GameObject/UI (Canvas)/UXTemplateWindow` | 在选中节点下实例化并解包 `UITemplateWindow.prefab`；必须先选中父节点，否则直接返回 |

### 控件清单

| 控件 | 来源 | 用途 | 文档 |
| --- | --- | --- | --- |
| `UXSelectable` | 无创建菜单，作为基类 | 交互状态、外部状态、子节点过渡、导航查找 | [UXSelectable 基础](uxselectable.md) |
| `UXButton` | 创建菜单 / Add Component | 点击、Submit、点击脉冲 | [UXButton](uxbutton.md) |
| `UXToggle` | 创建菜单 / Add Component | 开关、页签单选 | [UXToggle](uxtoggle.md) |
| `UXGroup` | Add Component（`UI/UXGroup`） | `UXToggle` 的单选组 | [UXToggle](uxtoggle.md) 的组章节 |
| `UXImage` | 创建菜单 / Add Component | 渐变、描边、阴影 `Image` | 本组另有页面 |
| `UXDraggable` | Add Component | 拖拽事件转发 | 本组另有页面 |
| `RecyclerView` | `UXScrollView` 菜单 | 虚拟列表 | [RecyclerView 概览](recyclerview/index.md) |

`UXGroup` 不是按钮的替代品：它只管一组 `UXToggle` 的互斥、默认项和全关策略，本身不接收点击。第一次接工程先从[快速开始](../../quickstart.md)确认工程能跑起来。

### 与框架 UI 的分工

| 职责 | 谁负责 |
| --- | --- |
| 窗口声明、打开关闭、层级排序、缓存 | [UI 界面](../ui/index.md)，细节见[打开、关闭与查询](../ui/window.md) |
| Holder 与 `ui_*` 字段生成 | [窗口声明与生命周期](../ui/lifecycle.md) |
| 控件外观与交互状态 | 本包（`UXSelectable` 及其子类） |
| 输入读取、导航焦点、点击音效 | [Input 输入](../input/index.md) |
| 列表与列表项复用 | [RecyclerView 概览](recyclerview/index.md) |

## 常见配置

| 配置 | 位置 | 说明 |
| --- | --- | --- |
| `Btn` / `Tog` / `TogGroup` 前缀 | `Client/ProjectSettings/UIGenerateConfiguration.asset`（UISetting 的 Element Mapping） | 决定生成字段的类型；`TogGroup` 必须排在 `Tog` 之前 |
| `Main Transition` | 控件 Inspector 的 `Image` 页 | 本体 Graphic 的 ColorTint / SpriteSwap / Animation |
| `Child Transitions` | 同上 | 子节点一起过渡，只支持 ColorTint / SpriteSwap |
| `Allow Switch Off` | `UXGroup` Inspector | 默认不勾，组内必须有一项选中 |
| `Default Toggle` | `UXGroup` Inspector | 没有选中项时补这一项 |
| 创建菜单路径 | `Editor/Inspector/UXCreateHelper.cs` | Unity 6000.3 起前缀变为 `GameObject/UI (Canvas)/` |
| 编译宏 | `Runtime/AlicizaX.UI.Extension.asmdef` | 由 `versionDefines` 自动定义 |

## 容易踩的坑

1. **同一个节点挂 `Button` 又挂 `UXButton`。** 两个组件都实现 `IPointerClickHandler`，一次点击触发两份 `onClick`。创建菜单已经删掉了 `Button`，手动加组件时不要补回来。
2. **只加 `UXButton` 不设 `Target Graphic`。** 主过渡完全不生效，Inspector 会提示 `You must have a Graphic target in order to use a color transition.`。
3. **以为扩展组件自带音效。** 包里没有音效组件：点击音效由 Input 包的 `InputFeedbackObserver` 负责，且必须和控件挂在同一个 GameObject 上，见[交互反馈](../input/input-feedback.md)。
4. **在 Unity 6 里找不到 `GameObject/UI/UXButton`。** 菜单前缀变成了 `GameObject/UI (Canvas)/`。
5. **没有 TextMeshPro 却说菜单丢了。** `UXButton` 菜单项包在 `TEXTMESHPRO_SUPPORT` 里，没定义时只有它消失，其他控件不受影响。
6. **把控件当窗口用。** 扩展包没有窗口、层级与缓存能力，这些都在框架 UI。
7. **让 `Tog` 规则抢先匹配 `TogGroup`。** 生成器按 Element Mapping 的列表顺序取第一个前缀命中，`TogGroup@Tabs` 会被生成成 `UXToggle`。
8. **在普通场景里不选中任何节点就执行 `UXScrollView`。** 没有落点时会取当前 Prefab Stage 根，不在 Prefab Stage 里就取不到。

## API 速查

| 类型 | 关键成员 | 说明 |
| --- | --- | --- |
| `UXSelectable` | `SetExternalState`、`ClearExternalState`、`PulseExternalState`、`HasExternalState`、`ExternalState` | 外部驱动的视觉状态 |
| `UXSelectable` | `UXSelectionState`、`TransitionData` | 状态枚举与子过渡数据 |
| `UXButton` | `onClick`、`PlayClickPulse()` | 点击事件与手动脉冲 |
| `UXToggle` | `isOn`、`SetIsOnWithoutNotify(bool)`、`onValueChanged`、`group`、`graphic` | 开关值、无回调赋值、事件与外观 |
| `UXGroup` | `allowSwitchOff`、`defaultToggle`、`NotifyToggleOn`、`SetAllTogglesOff`、`Next`、`Previous` | 单选组 |
| `RecyclerView` | 见 [RecyclerView 概览](recyclerview/index.md) | 列表 |
