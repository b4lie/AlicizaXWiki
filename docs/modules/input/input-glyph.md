# InputGlyph 输入图标

`InputGlyph` 把「当前输入 Profile + 动作的当前绑定」解析成 Sprite 或 TMP `<sprite>` 标签；图标内容存在 `InputGlyphDatabase` 资源里，`InputGlyphImage` / `InputGlyphText` 两个组件负责把结果显示到 UI。设备切换或通过 `UXInput.Rebind` 改键之后，组件自动刷新，业务不需要手动驱动。模块清单与启动顺序见 [Input 概览](index.md)。

## 适用场景

- 提示文案里嵌按键图标，例如「跳跃 {0}」「返回 {0}」。
- 同一功能在键鼠、Xbox、PlayStation、Switch 上显示不同图标。
- 列表项、按钮需要在运行时动态指定动作，而不是在预制体里写死引用。

## 使用前提

- 工程安装 `com.unity.inputsystem`。`AlicizaX.Input` 的 asmdef 在版本 ≥ 1.0.0 时定义 `INPUTSYSTEM_SUPPORT`；没有该宏时整包类型不存在，报的是编译错误。
- 输入层已初始化（场景有 `InputService` 且 `Actions` 已填），否则 `UXInput.Glyph` 为 `null`。图标组件在输入层就绪前启用会一直不刷新，也没有补接回调。
- 图标数据库可空：`InputService` 的 `Glyph Database` 留空也能启动，但运行时必须 `UXInput.Glyph.SetDatabase(...)`，否则 `TryGetSprite` 直接返回 `false`。
- 文本图标需要 TextMeshPro，并且 TMP Sprite Asset 里要有同名 sprite（默认取 `Sprite.name`，可在 Advanced 里用 `tmpSpriteName` 覆盖）。输入包运行时 asmdef 已引用 TMP；本包不依赖 PrimeTween。

## 最短接入

1. 创建数据库资源：`Assets/Create/AlicizaX/Input/Glyph Database`，或菜单 `AlicizaX/Extension/Input/Create Input Glyph Database`。
2. 打开 `AlicizaX/Extension/Input/Input Glyph Database`，左侧选 Profile，点工具栏 `+` 添加一条 glyph，拖入 Sprite。
3. 在该 glyph 卡片的 Bindings 区点 `+`，用 Input Control Picker 选控制路径，例如 `<Gamepad>/buttonSouth`。数据库会把它归一化成 glyph key（`gamepad/buttonsouth`）。
4. 把数据库挂到 `InputService` 的 `Glyph Database`，或运行时调用 `UXInput.Glyph.SetDatabase(database)`。
5. 在图标节点挂 `UI/Input Glyph Image`（同物体有 `Image` 时自动填 `targetImage`）或 `UI/Input Glyph Text`（自动填 `targetText`），`Reference Mode` 保持 `ActionReference` 并指定 `Jump`。
6. 文本控件把文案写成 `跳跃 {0}`：`{0}` 会被替换成 `<sprite name="...">`，没有 `{0}` 时文案保持原样。

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.UI;

public sealed class JumpIconProbe : MonoBehaviour
{
    [SerializeField] private InputActionReference jump;
    [SerializeField] private Image icon;

    private void OnEnable()
    {
        if (!UXInput.IsInitialized || jump == null)
            return;

        // 按当前 Profile 选绑定：手柄取 gamepad 图标，键鼠取 keyboard 图标。
        // 返回 true 表示拿到了可显示的图（可能是占位图），未配置占位图时为 false。
        if (UXInput.Glyph.TryGetSprite(jump, null, out Sprite sprite))
            icon.sprite = sprite;
    }
}
```

## 核心用法

### 两个渲染组件

`InputGlyphImage` 只改 `Image.sprite`（动作解析不到时清空），`InputGlyphText` 只替换文案里第一个 `{0}`（解析不到时恢复模板原文）。两者都在 `OnEnable` 接入当前输入会话，并订阅 `InputWatch.OnInputProfileChanged` 与 `InputRebind.OnBindingsChanged`，设备切换和改键后自动刷新。

`InputGlyphText` 的模板为空时直接显示 token；`{1}` 等其它占位符原样保留，不会抛异常（只认第一个 `{0}`）。

### 运行时换动作与组合部件

`SetAction(InputAction action, string compositePartName = null)` 覆盖 Inspector 的数据源并立即刷新，`ClearRuntimeAction()` 还原。组合绑定传部件名，例如 `Move` 的 `"Up"`：

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class GlyphBinder : MonoBehaviour
{
    [SerializeField] private InputGlyphImage glyphImage;
    [SerializeField] private InputAction move;

    private void Start()
    {
        if (!UXInput.IsInitialized || glyphImage == null)
            return;

        glyphImage.SetAction(move, "Up");   // 未接入会话时会先接入再刷新
        // glyphImage.ClearRuntimeAction(); // 还原为 Inspector 配置的引用
    }
}
```

如果动作本身没有 composite 部件，Inspector 里不会出现 `Composite Part` 下拉，手动写入的部件名会被忽略。

### 平台事件

`profileEvents` 每条规则是 `profileId` + `onMatched` + `onNotMatched`，Profile 变化（以及组件首次接入）时按当前 Profile 触发一轮，匹配用 `OrdinalIgnoreCase`。它适合「按 X 确认 / 按 Enter 确认」这类整块提示的切换；`profileId` 取 `InputProfileCatalog` 的稳定 ID（`KeyboardMouse`、`GenericGamepad`、`Xbox` 等）。组件没有公开的当前 Profile 属性，业务侧整块切换时直接监听 `InputWatch`：

```csharp
using AlicizaX.Input;
using UnityEngine;

public sealed class ConfirmHintSwitcher : MonoBehaviour
{
    private InputWatch _watch;

    private void OnEnable()
    {
        if (!UXInput.IsInitialized)
            return;

        _watch = UXInput.Watch;                       // 缓存并订阅同一实例
        _watch.OnInputTypeChanged += OnInputTypeChanged;
        OnInputTypeChanged(_watch.CurrentInputType);  // 首次进入同步一次
    }

    private void OnDisable()
    {
        if (_watch != null)
            _watch.OnInputTypeChanged -= OnInputTypeChanged;

        _watch = null;
    }

    private void OnInputTypeChanged(InputWatch.InputType inputType)
    {
        bool gamepad = inputType == InputWatch.InputType.Gamepad;
        // 这里切换手柄提示与键鼠提示节点
    }
}
```

`profileEvents` 只能通过 Inspector 配置；它是序列化数组，运行时新建 `DeviceProfileEvent` 不会写回组件。

### 代码查询与显示名

`TryGetTag` 的 `tag` 是 `<sprite name="SouthButton">`，`displayFallback` 是可读文本（如 `Space`）；`GetDisplayName` 直接给可读名；`GetBindingControlPath` 给当前选中的 `effectivePath`。这些查询按 Profile 与绑定版本缓存，稳态不分配，细节见 [性能与分配](performance.md)。

```csharp
string path = UXInput.Glyph.GetBindingControlPath(jump);   // 例如 <Gamepad>/buttonSouth
if (UXInput.Glyph.TryGetTag(jump, null, out string tag, out _))
    prompt.text = "跳跃 " + tag;                           // 有图标时用 TMP 标签
```

### 生成输入定义

菜单 `AlicizaX/Extension/Input/Generate Input Define`，或选中 `.inputactions` 后在 Assets 右键 `Assets/AlicizaX/Input/Generate Input Define`。生成一个静态类（默认 `InputDefine`、输出 `Assets/Generated/InputDefine.cs`），每个 map 一个嵌套类，提供 `MapName`、`Map`、`Enable`、`ActionPath`、`ActionAction` 和值访问器：Button 动作生成 `bool` 按下态，Value 动作生成 `ReadValue<T>`。

```csharp
// 生成代码只做无状态读取；边沿和开关请用调用方自己的 InputReader.ButtonState。
InputDefine.UI.Enable = true;      // 等价于 FindActionMap("UI").Enable()
bool back = InputDefine.UI.Back;   // Button 动作：当前是否按下
```

## 常见配置

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `placeholderSprite`（数据库） | 空 | Sprite 查询未命中时的回退图；文本查询不使用占位图 |
| `profiles[].id` | 由 `InputProfileCatalog` 同步 | 稳定 ID：`KeyboardMouse`、`Xbox`、`PlayStation`、`Switch` 等；`Touch` 会被同步删除 |
| `profiles[].fallbackProfileIds` | Xbox/PlayStation/Switch/Steam 系列各有回退 | 构建缓存时展平为查找表，查询只做一次字典命中 |
| `entries[].controlPaths` | 空 | 写 Input System 控制路径，glyph key 由路径归一化得到 |
| `entries[].sprite` | 空 | `InputGlyphImage` 显示的图；TMP 标签默认取它的 `name` |
| `entries[].tmpSpriteName` | 空 | Advanced 下；空时用 `Sprite.name`，要与 TMP Sprite Asset 对齐 |
| `actionSourceMode` | `ActionReference` | 另一个选项是 `HotkeyTrigger`，从同物体的 `HotkeyComponentBase` 取动作 |
| `actionReference` / `hotkeyTrigger` | 空 | 按模式二选一；`HotkeyTrigger` 会自动查同物体组件 |
| `targetImage` / `targetText` | 同物体 `Image` / `TMP_Text` | 为空时 `RefreshGlyph` 直接返回 |

## 容易踩的坑

1. **把 `TryGetSprite` 的 `true` 当成命中。** 没有明确条目时会回退 `placeholderSprite` 并仍返回 `true`。要判断命中用 `TryGetTag`（`false` + 可读文本），或直接查数据库 `TryGetSprite(profileId, glyphKey, out _)`。
2. **文案里没写 `{0}`。** token 不会插入，文本保持原文；写 `{1}` 也不会替换。编辑器只对「含 `{` 但没有 `{0}`」发一条警告。
3. **运行时改 `TMP_Text.text` 当模板。** 模板只在组件第一次 `OnEnable` 缓存，之后刷新会用旧模板覆盖新文案。模板应在设计期固定，动态内容交给 `{0}` 之外的拼接。
4. **在输入层就绪前启用 UI。** 组件不会补接入：`OnEnable` 时 `UXInput.IsInitialized` 为 `false` 就直接返回。等输入层起来后重新启用节点，或调用 `SetAction` 触发接入。
5. **手工加 `Touch` Profile。** 数据库窗口的 `SyncFixedProfiles` 会删掉 Touch；触屏图标只能落到占位图或可读文本。
6. **工程里放多个 `InputGlyphDatabase`。** 菜单打开搜索结果第一个并弹重复提示；真正生效的是 `InputService` 上挂的那个，或运行时 `SetDatabase` 传的那个。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `InputGlyph.SetDatabase(InputGlyphDatabase)` | 换数据库并清空查询缓存 |
| `InputGlyph.TryGetSprite(InputAction, string, out Sprite)` | 按动作当前绑定取图；未命中回退占位图 |
| `InputGlyph.TryGetTag(InputAction, string, out string, out string)` | `true` 给 TMP 标签，`false` 给可读文本 |
| `InputGlyph.GetDisplayName(InputAction, string)` | 当前绑定的显示名 |
| `InputGlyph.GetBindingControlPath(InputAction, string)` | 当前选中绑定的 `effectivePath` |
| `InputGlyph.GetGlyphKeyFromControlPath(string)` | 控制路径归一化成 glyph key |
| `InputGlyphBehaviourBase.SetAction` / `ClearRuntimeAction` | 运行时覆盖 / 还原数据源 |
| `InputGlyphDatabase.Placeholder` | 数据库占位图 |
