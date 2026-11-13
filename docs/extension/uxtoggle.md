# UXToggle 和 UXGroup

`UXToggle` 类似 Unity `Toggle`，通过 `isOn` 和 `onValueChanged` 控制状态；`UXGroup` 管理一组选项，支持默认选中、禁止全部关闭和顺序切换。

源码位置：

- `Client/Packages/com.alicizax.unity.ui.extension/Runtime/UXComponent/Group`

## 基础用法

```csharp
using AlicizaX.UI.Runtime;
using Game.UI;
using UnityEngine.UI;

[Window(UILayer.UI)]
public sealed class SettingWindow : UIWindow<ui_SettingWindow>
{
    protected override void OnInitialize()
    {
        baseui.TglMusic.onValueChanged.AddListener(OnMusicChanged);
        baseui.TglSfx.onValueChanged.AddListener(OnSfxChanged);
    }

    private void OnMusicChanged(bool isOn)
    {
        GameApp.Audio.SetCategoryEnable(AlicizaX.Audio.Runtime.AudioType.Music, isOn);
    }

    private void OnSfxChanged(bool isOn)
    {
        GameApp.Audio.SetCategoryEnable(AlicizaX.Audio.Runtime.AudioType.UISound, isOn);
    }
}
```

不触发回调地同步状态：

```csharp
baseui.TglMusic.SetIsOnWithoutNotify(true);
```

`UXToggle` 的选中状态会强制视觉状态为 `Selected`，所以子节点状态、图标和文字可以统一表现。

Inspector 关键字段：

| 字段 | 说明 |
| --- | --- |
| `Graphic` | 选中时显示的标记图，例如勾选图、页签底线 |
| `Group` | 所属 `UXGroup`；有分组时自动互斥 |
| `Is On` | 当前是否选中 |

## UXGroup 分组

```csharp
UXGroup group = baseui.TabGroup;

group.allowSwitchOff = false;   // 必须有一项选中
group.Next();                   // 选中下一项，跳过不可用项
group.Previous();               // 选中上一项
group.SetAllTogglesOff(sendCallback: false);
```

Inspector 字段：

| 字段 | 说明 |
| --- | --- |
| `Allow Switch Off` | 是否允许组内没有选中项 |
| `Default Toggle` | 没有选中项时默认恢复的 Toggle |
| `Toggles` | 分组管理的 Toggle 列表 |

编辑器辅助按钮：

| 按钮 | 作用 |
| --- | --- |
| `Collect Children` | 收集当前节点子树下的 `UXToggle` |
| `Clean Nulls` | 清理数组中的空引用 |
| `Sort By Hierarchy` | 按层级顺序排序，影响 `Next` / `Previous` |

## 页签示例

```csharp
[Window(UILayer.UI)]
public sealed class RoleWindow : UIWindow<ui_RoleWindow>
{
    private UXGroup _tabGroup;

    protected override void OnInitialize()
    {
        _tabGroup = baseui.TabGroup;
        _tabGroup.allowSwitchOff = false;

        baseui.TglInfo.onValueChanged.AddListener(isOn =>
        {
            if (isOn) SwitchTab(0);
        });
        baseui.TglSkill.onValueChanged.AddListener(isOn =>
        {
            if (isOn) SwitchTab(1);
        });

        baseui.TglInfo.SetIsOnWithoutNotify(true);
        SwitchTab(0);
    }

    private void SwitchTab(int index)
    {
        baseui.PageInfo.SetActive(index == 0);
        baseui.PageSkill.SetActive(index == 1);
    }

    public void SelectNext()
    {
        _tabGroup.Next();
    }
}
```

## API 速查

| API | 说明 |
| --- | --- |
| `UXToggle.isOn` | 读取或设置选中状态 |
| `UXToggle.onValueChanged` | 状态变化事件 |
| `UXToggle.SetIsOnWithoutNotify(bool)` | 不触发回调地设置状态 |
| `UXGroup.allowSwitchOff` | 是否允许组内无选中项 |
| `UXGroup.defaultToggle` | 默认选中的 Toggle |
| `UXGroup.Next()` / `Previous()` | 选中下一项/上一项，跳过不可用项 |
| `UXGroup.GetFirstActiveToggle()` | 获取第一个选中项 |
| `UXGroup.SetAllTogglesOff(bool sendCallback = true)` | 关闭全部 |
| `UXGroup.RegisterToggle` / `UnregisterToggle` | 运行时增删分组成员 |

## 注意事项

1. `SetIsOnWithoutNotify` 适合初始化或同步服务端状态，不触发业务回调。
2. `Next` / `Previous` 会跳过不可交互或未激活的 Toggle。
3. 分组必须有一项选中时，取消最后一项会被拒绝，不会触发 `onValueChanged`。
4. `Sort By Hierarchy` 只影响 `Next` / `Previous` 的顺序，不改变视觉层级。
5. `UXToggle` 的交互音效由 input 包的 `InputFeedbackObserver` 处理，不在本组件里配置。
