# InputGlyph 输入图标

`InputGlyph` 把 Input System 的动作绑定解析成 Sprite、TMP Sprite 标签或可读文本，用在按键提示、设置页和操作引导里。

源码位置：

- 运行时：`Client/Packages/com.alicizax.unity.input/Runtime/InputGlyph`
- 查询入口：`Runtime/Input/InputGlyph.cs`
- 编辑器：`Editor/InputGlyph`

## 启动

在 `InputService` 上配置图标数据库：

```text
Input Service
├─ Actions: 项目的 InputActionAsset
└─ Glyph Database: 项目的 InputGlyphDatabase
```

`InputService` 初始化时会把数据库注入 `UXInput.Glyph`，不需要额外挂 Provider。

数据库入口：

```text
AlicizaX > Extension > Input > Input Glyph Database
AlicizaX > Extension > Input > Create Input Glyph Database
```

## 数据库

`InputGlyphDatabase` 按输入 Profile 保存控制路径到图标/TMP 名称的映射。常用 Profile：

- `KeyboardMouse`
- `Xbox`
- `PlayStation`
- `Switch`
- `Touch`

每个 Profile 可以配置兜底 Profile。当前 Profile 找不到图标时，按 `FallbackProfileIds` 顺序继续查。

同一个手柄物理按键在不同平台有不同图标，所以同一批控制路径通常在每个 Profile 里各配一份 Sprite，而不是复制控制路径。

## 使用图标组件

在 UI 上添加：

```text
Add Component > UI > Input Glyph Image
Add Component > UI > Input Glyph Text
```

两者都由 `InputGlyphBehaviourBase` 驱动，Inspector 主要配置：

| 字段 | 说明 |
| --- | --- |
| `Action Source Mode` | `ActionReference` 用固定动作；`HotkeyTrigger` 跟随热键组件的动作 |
| `Action Reference` | 动作引用 |
| `Hotkey Trigger` | 同物体的 `HotkeyComponentBase` |
| `Composite Part Name` | 组合绑定的部分，例如 `Up`、`Left` |
| `Profile Events` | 当前 Profile 匹配/不匹配时触发 UnityEvent |

运行时切换动作：

```csharp
using AlicizaX.Input;
using UnityEngine.InputSystem;

public sealed class GlyphDemo : MonoBehaviour
{
    [SerializeField] private InputGlyphImage icon;
    [SerializeField] private InputGlyphText label;

    public void ShowMove(InputAction moveAction)
    {
        icon.SetAction(moveAction, "Up");
        label.SetAction(moveAction, "Up");
    }

    public void Clear()
    {
        icon.ClearRuntimeAction();
        label.ClearRuntimeAction();
    }
}
```

`SetAction(null)` 会清空显示；`ClearRuntimeAction()` 恢复 Inspector 配置的数据源。

## 直接查询

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class GlyphQuery
{
    public void Print(InputAction action)
    {
        if (UXInput.Glyph.TryGetSprite(action, "Up", out Sprite sprite))
        {
            // 显示 sprite。
        }

        if (UXInput.Glyph.TryGetTag(action, "Up", out string tag, out string fallback))
        {
            // TMP: <sprite name="..."> 或 fallback 文本。
        }

        string name = UXInput.Glyph.GetDisplayName(action);
    }
}
```

控制路径也可以直接查：

```csharp
UXInput.Glyph.TryGetSprite("<Gamepad>/buttonSouth", out Sprite sprite);
UXInput.Glyph.TryGetTag("<Keyboard>/space", out string tag, out string fallback);
```

## 刷新时机

- Profile 切换：`UXInput.Watch.OnInputProfileChanged`。
- 绑定变化：`UXInput.Rebind.OnBindingsChanged`。
- `InputGlyphImage` / `InputGlyphText` 已经自动监听这两类事件，不需要业务每帧刷新。

## 注意事项

1. 图标数据库不自动 `Resources.Load`，必须由 `InputService` 注入，或运行时 `UXInput.Glyph.SetDatabase(...)`。
2. 数据库更换后，旧图标缓存会失效；已经显示的组件需要触发刷新。
3. 组合绑定要传 `CompositePartName`，普通动作传 `null`。
4. Sprite 查询找不到明确图标时会回退到数据库的 `Placeholder`；TMP 标签查询返回 `false` 并给出可显示的文本回退。
5. 手柄控制路径表达的是物理位置，不同平台的按钮文字差异应放在不同 Profile 的 Sprite 上。
