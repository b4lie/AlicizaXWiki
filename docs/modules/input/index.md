# Input 输入

`com.alicizax.unity.input` 基于 Unity Input System，提供设备监听、读取、重绑定、输入图标、UI 导航、热键和交互反馈。当前文档对应包版本 `4.0.0`。

源码位置：

| 内容 | 路径 |
| --- | --- |
| 输入入口与模块 | `Client/Packages/com.alicizax.unity.input/Runtime` |
| 输入、重绑定、图标、震动 | `Runtime/Input` |
| 图标数据库与组件 | `Runtime/InputGlyph` |
| UI 导航 | `Runtime/Navigation` |
| 热键 | `Runtime/Hotkey` |
| 交互反馈 | `Runtime/Feedback` |

## 启动

在常驻节点挂载：

```text
Add Component > Input > Input Service
```

Inspector 配置：

| 字段 | 说明 |
| --- | --- |
| `Actions` | 全局 `InputActionAsset` |
| `Glyph Database` | 可选的 `InputGlyphDatabase` |

也可以在启动代码中创建：

```csharp
using AlicizaX.Input;
using UnityEngine.InputSystem;

public static class InputBootstrap
{
    public static void Setup(InputActionAsset actions, InputGlyphDatabase glyphs)
    {
        InputService.Initialize(actions, glyphs);
    }
}
```

`InputService` 带 `DefaultExecutionOrder(-30000)`，会早于普通业务脚本初始化。它按顺序创建 `Watch`、`Reader`、`Rebind`、`Glyph`、`Haptics`、`Navigation`、`Hotkey`、`Feedback`，然后初始化 Watch、Rebind、Reader、Navigation。

只允许一个 `InputService`。`Actions` 为空时记录错误，输入不会初始化。

## 生命周期

`InputService.Awake` 调用 `UXInput.Initialize`，`OnDestroy` 调用 `UXInput.Shutdown`：

```text
Hotkey -> Navigation -> Haptics -> Rebind -> Reader -> Glyph -> Watch
```

- 关闭后 `UXInput` 的模块属性全部变成 `null`；组件订阅时必须缓存自己订阅的实例，并对该实例退订。
- 应用失焦或暂停时，`InputService` 会清空热键按下目标并停止震动；不会关闭整个输入层。
- `SubsystemRegistration` 会重置静态输入状态；下一次仍由 `InputService` 启动。
- 输入层重建后，`Watch`、`Reader`、`Glyph` 等实例都是新的；反馈主题等业务绑定需要重新设置。

## 模块

| 模块 | 用途 |
| --- | --- |
| `UXInput.Watch` | 当前设备、输入类型、Profile 和上下文事件 |
| `UXInput.Reader` | 动作查询、值读取、按下状态、组合部分、`ButtonState` |
| `UXInput.Rebind` | 运行时重绑定、冲突处理、绑定覆盖文件 |
| `UXInput.Glyph` | 控制路径到图标、TMP 标签和显示名 |
| `UXInput.Haptics` | 手柄马达震动 |
| `UXInput.Navigation` | UI 焦点域、导航补选和压制 |
| `UXInput.Hotkey` | 热键注册和按 Holder 焦点链分发 |
| `UXInput.Feedback` | 按设备和交互提示播放 UI 音效 |

业务只通过 `UXInput` 静态属性访问模块，不直接 new 这些类型。

## 快速示例

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class MoveExample
{
    public void Tick()
    {
        InputAction moveAction = UXInput.Reader.ResolveAction("Gameplay/Move");
        Vector2 move = UXInput.Reader.ReadValue<Vector2>(moveAction);

        if (UXInput.Reader.IsPressed("UI/Submit"))
        {
            // Submit 按下。
        }
    }
}
```

需要 UI 导航、热键或输入图标时：

1. 安装 `com.alicizax.unity.ui.extension`（导航/热键依赖 Holder）。
2. 在窗口或面板上挂 `NavigationScope`。
3. 在按钮上挂 `HotkeyComponent` 或 `InputFeedbackObserver`。
4. 输入图标使用 `InputGlyphImage` / `InputGlyphText`。

## 页面导航

| 页面 | 内容 |
| --- | --- |
| [设备、读取与重绑定](../../extension/uxinput.md) | Watch、Reader、Rebind、Haptics 的使用 |
| [输入图标](../../extension/input-glyph.md) | 图标数据库、`InputGlyphImage` / `InputGlyphText` |
| [快捷键](../../extension/hotkey-component.md) | `HotkeyComponent`、Holder 焦点链、`IHotkeySubmitListener` |
| [UI 导航](../../extension/navigation.md) | `NavigationScope`、焦点域、动态控件注册 |
| [交互反馈](../../extension/input-feedback.md) | `InputFeedbackProfile`、`InputFeedbackObserver`、程序化焦点 |
| [性能与分配](performance.md) | 更新、分配和缓存边界 |

## 注意事项

1. 安装 `com.unity.inputsystem` 并在 Player Settings 启用 Input System 后端。
2. 输入层只允许一个 `InputService`；不要同时手动初始化并挂组件。
3. 输入包依赖 UI extension 的 Holder 来实现导航和热键；纯读输入可以只用 `InputService` + `UXInput.Reader`。
4. 输入图标数据库需要在 `InputService` 上配置，也可以运行时调用 `UXInput.Glyph.SetDatabase` 替换。
5. 交互反馈依赖框架 `IAudioService`；没有音频服务时自动跳过播放。
