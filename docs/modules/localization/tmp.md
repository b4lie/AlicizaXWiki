# TMP 绑定与预览

`LocalizedText` 把 TMP 文本绑定到本地化 Key，在语言切换、词库追加和组件重新启用时刷新。它只在项目提供 TextMeshPro 或 UGUI 2.0+ 时编译。

## 添加组件

给带 `TMP_Text` 的对象添加：

```text
Add Component > Localization > Localized Text
```

Inspector 字段：

| 字段 | 说明 |
| --- | --- |
| `Target` | 要写入的 `TMP_Text`；为空时在 `Awake` 自动取同节点组件 |
| `Language Key` | Key 选择器，支持搜索；选中后显示预览 |
| `Arguments` | 序列化的字符串参数，用于 `{0}`、`{1}` 等占位符 |

## 编辑器预览

`LocalizedText` 的自定义 Inspector 会：

- 用 `Language Key` 下拉搜索 `RuntimeOptions.json` 中的所有 Key。
- 在 Key 下方显示当前语言的预览文本。
- 修改 Key 或 Arguments 后立即刷新 Scene 中的预览。
- 提供 `Refresh Preview` 按钮，清空缓存后重新读取词库。

编辑器顶部的语言菜单可以切换预览语言，选择写入 `EditorPrefs`，不会修改 `RuntimeOptions.json` 或项目默认语言。Play Mode 下菜单项不可用。

预览只作用于 Scene 中的对象，不会写入 TMP 的序列化文本。Prefab 资源、Project 窗口里的对象不会显示预览。

## 运行时使用

`LocalizedText` 在 `OnEnable` 订阅 `LocalizationKit.Changed` 并立即刷新；`OnDisable` 退订。重新启用时会再次刷新，因此关闭期间发生的语言切换不会丢失。

```csharp
using AlicizaX.Localization.UI;
using UnityEngine;

public sealed class ShopTitle : MonoBehaviour
{
    [SerializeField] private LocalizedText _title;

    public void SetCount(int count)
    {
        _title.SetContent("SHOP_GOODS_COUNT", count);
    }
}
```

| API | 说明 |
| --- | --- |
| `Key` | 当前绑定的 Key，只读 |
| `SetKey(string key)` | 修改 Key 并刷新 |
| `SetArguments(params object[] arguments)` | 设置运行时格式化参数并刷新 |
| `SetContent(string key, params object[] arguments)` | 同时设置 Key 和参数并刷新 |
| `Refresh()` | 主动重新取文案 |

`SetArguments` 和 `SetContent` 的参数优先于 Inspector 里的 `Arguments`。它们不会写回序列化字段，适合运行时动态数据。

格式化规则：

- 文案里的 `{0}`、`{1}` 等占位符由 `LocalizationFormatter` 处理。
- 最多支持 16 个参数；超过会抛 `FormatException`。
- 没有 `{0}` 的文案在传入参数时仍会交给格式化器，多余参数不会报错。
- Key 缺失时显示 Key 本身；不需要参数时直接调用 `SetText`，不会重复设置相同文本。

## 编辑器如何找到词库

包内编辑器工具按文件名在 `Assets` 下查找 `RuntimeOptions.json`：

- 必须恰好存在一份；多份会抛 `InvalidOperationException`。
- 读取 `Locales`、`DefaultLocale` 和 `Catalogs`。
- `Catalogs` 中的路径相对于 `RuntimeOptions.json` 所在目录，`{locale}` 替换为当前预览语言。
- 不存在的词库文件会被跳过；重复 Key 会报错。
- 如果 `RuntimeOptions.json` 不存在，预览为空，但包和 `LocalizedText` 仍可正常编译。

编辑器不读取 `config.ini`，也不依赖生成目录结构；调整输出位置后只需要保证 `RuntimeOptions.json` 的 `Catalogs` 正确。

## 注意事项

1. 只安装包、尚未生成 `RuntimeOptions.json` 时，`LocalizedText` 的 Key 会显示 Key 本身。
2. `LocalizedText` 依赖 `TEXTMESHPRO_SUPPORT`；项目没有 TextMeshPro / UGUI 2.0+ 时不会生成该组件。
3. 预览语言是编辑器状态，不会改变玩家保存的语言，也不会改变项目默认语言。
4. 预览只刷新当前 Scene 的对象；Prefab 资源需要打开对应 Prefab 查看。
5. 运行时动态文案用 `SetKey` / `SetArguments` / `SetContent`，不要直接改 TMP 的 `text`，否则会绕过语言切换刷新。
6. `LocalizedText` 只在 `Application.isPlaying` 时订阅和刷新，编辑期由自定义 Inspector 负责预览。
