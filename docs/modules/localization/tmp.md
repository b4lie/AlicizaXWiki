# TMP 绑定与编辑器预览

`LocalizedText` 把场景里的 TMP 文本绑定到一个 Key，在语言切换、词库追加和组件重新启用时自动刷新；编辑模式下它不写资产，而是用词库数据把预览字符画到 TMP 上。这一页讲组件字段、运行时刷新、预览的取词来源和 `TEXTMESHPRO_SUPPORT` 约束。

## 适用场景

- 固定文案的按钮、标题、提示，由策划/美术在 Prefab 上直接选 Key。
- 文案带少量占位参数，参数值在运行时才知道。
- 想在不开游戏的情况下检查「当前语言 + 多层词库」下的实际文本。

不适用：需要频繁整段拼接、富文本按片段高亮、或文案完全由数据驱动的列表项——这些通常直接用 `LocalizationKit.Get/Format` 在代码里赋值。

## 使用前提

- 项目提供 TextMeshPro 或 UGUI 2.0+，`TEXTMESHPRO_SUPPORT` 生效（本工程是 `com.unity.ugui 2.0.0` + `com.unity.textmeshpro 3.0.9`）。
- 引用 `AlicizaX.Localization.UI` 程序集；它 `autoReferenced` 为 `false`，业务 asmdef 必须显式加引用，否则代码里看不到 `LocalizedText`。
- 组件所在节点上有 `TMP_Text`（`TextMeshProUGUI` 或 `TextMeshPro`），或者手动指定 `Target`。
- 编辑器预览还需要词库已建好，并在 Localization 窗口的「叠加预览」里加入了 Table（见[词库配置与生成](config.md)）。

## 最短接入

1. 选中带 `TMP_Text` 的节点，`Add Component` → `Localization/Localized Text`（组件菜单名来自 `[AddComponentMenu("Localization/Localized Text")]`）。

2. Inspector 的 `Target` 留空即可，`Awake` 会自动取同节点的 `TMP_Text`；节点上有多套文本时才需要手动指定。

3. 点 `Language Key` 下拉，搜索 Key 或译文选中；Key 下方会显示当前预览语言的文本。带占位符的 Key 在 `Arguments` 里按顺序填参数。

4. 打开 Localization 窗口的「叠加预览」，点「+ 添加 Table」把要参与预览的词库加进去，拖动顺序与运行时一致（越靠后优先）。不加入 Table 时预览只会显示 Key 本身。

5. 运行时不需要额外代码：语言切换或追加词库后组件自己刷新。需要在代码里换 Key/参数时：

```csharp
using AlicizaX.Localization.UI;
using UnityEngine;

public sealed class ShopTitle : MonoBehaviour
{
    [SerializeField] private LocalizedText _title;

    public void SetTotal(int total)
    {
        _title.SetContent("UI.Buy.TotalPrice", total);   // Key 与参数一起换
    }

    public void UpdatePrice(int total, string currency)
    {
        _title.SetArguments(total, currency);            // 只换参数
    }

    public void ShowTitle()
    {
        _title.SetKey("UI.Shop.Title");                  // 只换 Key
    }
}
```

## 核心用法

### Inspector 字段

| 序列化字段 | Inspector 名称 | 类型 | 说明 |
| --- | --- | --- | --- |
| `_target` | `Target` | `TMP_Text` | 为空时 `Awake` 取同节点组件；预览时同样回退到 `GetComponent<TMP_Text>()` |
| `_key` | `Language Key` | `string` | 绑定 Key；由下拉搜索选择，也可手填 |
| `_arguments` | `Arguments` | `string[]` | 序列化参数，按顺序替换 `{0}`、`{1}` |

`_key` 为空时组件完全不动 TMP 文本，节点上原来的占位文字会保留，便于在没接好 Key 前继续排版。

### 运行时刷新

| 时机 | 行为 |
| --- | --- |
| `Awake` | `_target` 为空则取同节点 `TMP_Text` |
| `OnEnable` | 仅在 `Application.isPlaying` 时订阅 `LocalizationKit.Changed` 并立即 `Refresh()` |
| `OnDisable` | 退订；关闭期间发生的语言切换会在重新启用时补上 |
| 语言/词库变化 | `Changed` 回调里 `Refresh()` |
| `SetKey` / `SetArguments` / `SetContent` | 更新字段后立即 `Refresh()` |

`Refresh()` 的取词与写入：

1. 不满足「正在播放 + 组件激活 + Key 非空 + Target 非空」任一条件时直接返回，不做任何事。
2. `format = LocalizationKit.Get(_key)`；Key 缺失时得到的就是 Key。
3. 参数取 `_runtimeArguments ?? _arguments`；没有参数时用 `SetText(format)`，并用内部 `_lastText` 跳过重复赋值。
4. 有参数时用 `ZString` 拼装后 `SetCharArray` 写入，避免产生中间字符串。

`SetArguments` / `SetContent` 的 `object[]` 只存在组件实例上，不写回序列化字段；下次打开 Prefab 仍只有 Inspector 里填的字符串参数。

### 编辑器 Key 选择器

自定义 Inspector `LocalizedTextEditor`（`[CanEditMultipleObjects]`）提供：

- `Language Key` 是一个下拉按钮，点开 `LocalizationKeyPopup`：按 Key 或当前预览译文搜索，支持上下键与回车选择，`Clear` 清空绑定；列表项下方显示该 Key 的词库来源与预览值。
- Key 下方 `HelpBox` 显示当前预览语言的文本（播放模式下改成 `LocalizationKit.Get` 的结果）。
- `Arguments` 数组按序列化字段编辑。
- `Refresh Preview` 按钮清空词库缓存后重刷所有选中对象。
- 修改 Key、Arguments、Target 后立即刷新当前选择；播放模式下走 `Refresh()`，编辑模式下走预览。

### 编辑器预览如何找词库

预览不读 `config.ini`，也不走运行时 Provider，而是用 `LocalizationRefreshHelper` 的缓存：

| 数据 | 来源 |
| --- | --- |
| 预览语言 | `EditorPrefs` 键 `AlicizaX.Localization.Editor.Language`；未设置或已不支持时用 `DefaultLocale` |
| Key 全集 | 项目中所有父表 `Keys` 的并集（弹窗搜索用） |
| 词库值 | 仅来自「叠加预览」里加入的 Table（列表顺序即叠加顺序，越靠后优先），按预览语言取对应子资产 |
| 预览文本 | 有值时用译文，无值时回退 Key |

刷新时机：项目资产变化、Undo、层级变化、以及缓存被显式失效时，编辑器会在 `delayCall` 里重刷当前打开场景中的 `LocalizedText` 并重绘视图。播放模式下预览循环被跳过，界面显示运行时真实值。

编辑器工具栏还有一个 `AlicizaX/Localization` 语言下拉（Unity 6.3+ 是 `MainToolbar` 元素，更早版本在右侧工具栏），用来切换预览语言，写入同一个 `EditorPrefs`；播放模式下它的菜单项被禁用。

### 预览怎么写进 TMP

预览只处理「非持久化对象」：场景或 Prefab 编辑模式下的实例会显示预览，Project 窗口里的 Prefab 资产本体不会被改。写入时用 TMP 的 `SetCharArray` 画字符，然后通过反射把原始 `m_text` 恢复、把 `m_IsTextBackingStringDirty` 置回 `false`，所以预览不会把资产改脏，也不会改序列化文本。TMP 的内部字段名变了（换大版本）会抛 `MissingFieldException`——本工程 TMP 3.0.9 的字段是 `m_text` 与 `m_IsTextBackingStringDirty`。

### 与 LocalizationSettings 的关系

- 运行时当前语言由 `LocalizationKit.Initialize` 按 `LocalizationSettings` 选出，`LocalizedText` 只消费 `LocalizationKit` 的结果。
- 预览语言独立于运行时语言与默认语言：它只存在 `EditorPrefs`，不改设置资产，也不写 `PlayerPrefs`。
- 预览语言的可选值就是设置里的语言 `Id` 列表；设置变更后工具栏下拉会自动刷新，不需要重启编辑器。
- 播放模式下切换语言会写入 `EditorPrefs`，所以退出播放后编辑器预览语言会跟着变成运行时选过的语言。

## 常见配置

| 项目 | 值 |
| --- | --- |
| 组件菜单 | `Localization/Localized Text` |
| 程序集 | `AlicizaX.Localization.UI`（需显式引用） |
| 宏 | `TEXTMESHPRO_SUPPORT`（UGUI ≥ 2.0.0 或安装 TextMeshPro 时自动定义） |
| 组件字段 | `_target`、`_key`、`_arguments` |
| 预览语言 | `EditorPrefs`：`AlicizaX.Localization.Editor.Language` |
| 参与预览的词库 | SessionState 键 `AlicizaX.Localization.PreviewTables`，由「叠加预览」维护 |
| 运行时语言记录 | `PlayerPrefs`：`AlicizaX.Localization.Language` |
| 本工程实例 | `UIHomeWindow.prefab`（`_key: UI.Home.ButtonShop`）、`UIShopWindow.prefab`（`UI.Shop.Title` 等） |

## 容易踩的坑

1. Key 必须与词库逐字符一致（区分大小写、不做 Trim）：词库校验会拒绝前后空白的 Key，手填时多了空格就永远显示 Key。生成的常量名会把 `.` 换成 `_` 并转大写（`UI.Home.ButtonShop` → `UI.HOME_BUTTONSHOP`），别把它直接粘回 `Language Key`。
2. 初始化与词库加载早于/晚于组件启用都会出现「界面显示 Key」：`Initialize` 不发布 `Changed`，只有 `AddTableAsync` / `ReloadTableAsync` 成功或切换语言才触发刷新。启动场景里先加词库、再显示界面最稳；已显示的组件会在词库加入后自己补刷。
3. `_arguments` 是 `string[]`：`{0:F2}`、`{0:N0}` 这类格式说明符对字符串参数无效，会被忽略并原样输出，需要数字格式时用 `SetArguments(数量, 价格)` 传真实数值对象；参数个数少于占位符最大索引时，运行时会抛 `FormatException`，编辑器预览则退回显示原文。
4. 业务程序集没引用 `AlicizaX.Localization.UI` 时，代码里根本看不到 `LocalizedText`；如果之后又关掉 UGUI/TMP 导致宏失效，场景里的组件会变成 Missing Script。
5. 预览在没把 Table 加进「叠加预览」时只显示 Key，看起来像「没翻译」；另外预览只作用于场景实例，Prefab 资产本体的文本不会变。
6. 预览依赖 TMP 私有字段反射，升级 TMP 大版本后可能抛 `MissingFieldException`；此时仍可运行，只是编辑器预览失效。

## API 速查

| 成员 | 签名 | 说明 |
| --- | --- | --- |
| `Key` | `string Key { get; }` | 当前绑定的 Key |
| `SetKey` | `void SetKey(string key)` | 改 Key 并刷新；传 `null` 视为空 Key |
| `SetArguments` | `void SetArguments(params object[] arguments)` | 设置实例级参数并刷新，优先于 Inspector 的 `_arguments` |
| `SetContent` | `void SetContent(string key, params object[] arguments)` | 同时设置 Key 与参数并刷新 |
| `Refresh` | `void Refresh()` | 主动重取；播放、激活、Key、Target 缺一不可 |
