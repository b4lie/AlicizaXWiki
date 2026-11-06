# Localization 本地化

`com.alicizax.unity.localization` 是基于 Luban 的本地化扩展包：运行时 `LocalizationKit`、TMP 的 `LocalizedText` 绑定和编辑器预览工具。TMP 是可选依赖，项目提供 TextMeshPro 或 UGUI 2.0+ 后才会编译 `LocalizedText` 和对应编辑器。

当前文档对应包版本 `1.0.1`。

源码位置：

| 内容 | 路径 |
| --- | --- |
| 运行时门面与 Provider | `Client/Packages/com.alicizax.unity.localization/Runtime/Core` |
| Luban 词库读取 | `Client/Packages/com.alicizax.unity.localization/Runtime/Catalog` |
| 词库来源 | `Client/Packages/com.alicizax.unity.localization/Runtime/Sources` |
| TMP 绑定 | `Client/Packages/com.alicizax.unity.localization/Runtime/UI` |
| 编辑器预览与 Key 选择 | `Client/Packages/com.alicizax.unity.localization/Editor` |

## 数据流

```text
Config/Excels/Localization/*.xlsx
        -> Config/config.ini
        -> gen-all.bat / gen-all.command
        -> Client/Assets/Resources/Localization/RuntimeOptions.json
        -> Game_{locale}.bytes / Update_{locale}.bytes
        -> LocalizationKit.InitializeAsync / AddCatalogAsync
        -> Dictionary<string, string>
        -> LocalizationKit.Get / LocalizedText
```

`RuntimeOptions.json` 只保存语言列表、默认语言和编辑器预览用的词库相对路径；运行时实际加载哪些词库由启动代码显式传入的 `ILocalizationCatalogSource` 决定。

## 组成

| 类型 | 职责 |
| --- | --- |
| `LocalizationKit` | 静态门面：初始化、追加词库、切换语言、取文案、格式化、语言变化事件 |
| `LocalizationProvider` | 运行时实例：持有当前语言、词库字典和来源列表 |
| `LocalizationSettings` | 解析 `RuntimeOptions.json` 的语言列表与默认语言 |
| `LocalizationCatalog` | 用 Luban `ByteBuf` 读取 `.bytes`，产出 `Dictionary<string, string>` |
| `ResourcesCatalogSource` | 从 `Resources` 同步/异步加载词库 |
| `ResourceCatalogSource` | 通过框架 `GameApp.Resource` 异步加载词库 |
| `LocalizedText` | 把 TMP 文本绑定到 Key，支持参数和语言切换刷新 |
| `LanguageTypes` | 语言列表与索引互转 |

## 快速接入

1. 在 `Config/config.ini` 的 `[localization]` 配置 `languages`、`default_locale` 和 `runtime_options_out`。
2. 在 `Config/Excels/Localization` 准备文案工作簿，运行 `gen-all` 生成词库和 `RuntimeOptions.json`。
3. 启动流程加载 `Resources/Localization/RuntimeOptions`，调用 `LocalizationKit.InitializeAsync`。
4. 热更资源和普通配置可用后，追加 `Game_{locale}` 词库并接入配置服务。
5. UI 文本使用 `LocalizedText` 绑定 Key，或在代码里调用 `LocalizationKit.Get` / `Format`。

详细步骤见：

| 页面 | 内容 |
| --- | --- |
| [配置与词库生成](config.md) | `config.ini`、工作簿格式、生成规则、`RuntimeOptions.json` |
| [运行时加载](runtime.md) | 初始化、语言选择、追加词库、BuildCLI 构建语言、配置服务接入 |
| [TMP 绑定与预览](tmp.md) | `LocalizedText`、Inspector Key 选择、编辑器语言预览 |

## 项目约定

本工程保留外部 `Config` 目录和普通 Luban 链：

- 多语言源表在 `Config/Excels/Localization`，导出规则合并在 `Config/config.ini`。
- 更新界面文案输出到 `Client/Assets/Resources/Localization/Update_{locale}.bytes`，启动流程在更新界面前加载。
- 普通常量表和映射表输出到 `Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes`，热更资源可用后由 `HotfixEntry` 追加。
- 常量访问器分别是 `Client/Assets/Scripts/Startup/UpdateLocalizationKey.cs` 和 `Client/Assets/Scripts/Hotfix/GameLogic/LocalizationKey.cs`。
- 语言列表、默认语言和编辑器预览词库路径共用一份 `RuntimeOptions.json`。

## 注意事项

1. 多语言包本身不依赖项目的 `Game.Config`；普通 Luban `ConfigService` 通过 `SetLocalizationChange` 接入翻译和语言变化订阅。
2. 只安装包、尚未生成 `RuntimeOptions.json` 时仍然可以编译；已有 Key 会显示 Key 本身。
3. 同一个工程只保留一份 `RuntimeOptions.json`，编辑器按文件名在 `Assets` 下查找；存在多份会直接报错。
4. 文案单元格留空表示缺失，运行时返回原始 Key；写入 `""` 表示空字符串。
5. 所有表共用全局 Key 唯一性校验；跨词库重复 Key 会在运行时初始化或追加时抛异常。
