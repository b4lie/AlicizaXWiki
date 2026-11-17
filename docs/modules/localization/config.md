# 词库配置与生成

这一页讲编辑器侧怎么把词库建起来：`LocalizationSettings` 的语言配置、Localization 窗口的建表/录入/校验、CSV 与代码生成，以及构建前的源资产校验。Config/Excel 链路如何产出 `Game_{locale}.bytes` 也在这里交代边界，具体生成规则指向 Config 模块。

## 适用场景

- 新增或调整项目支持的语言、默认语言。
- 新建词库、批量录入或修改各语言译文。
- 把 Key 导出给翻译，回收 CSV 后按差异导入。
- 生成强类型常量访问器，供 UI 代码调用。
- 排查词库校验错误、构建期源资产报错，或确认 `Game_{locale}.bytes` 由谁生成。

## 使用前提

- `Assets/Resources/Localization/Settings.asset` 已存在（不存在时设置页提供「创建设置」按钮，默认建一份 `zh-Hans`）。
- 父 Table 与语言子资产只在编辑器存在；运行时读的是 `.loclang` 导入出的独立资产。
- 编辑、导入、生成常量都会写盘并触发 AssetDatabase 刷新，操作前先提交或备份。
- 只有走 Config 链路时才需要 `Config` 目录的 Luban 工具（`build.bat` 生成 `Tools`，`gen-all` 产出 `Game_{locale}.bytes`）。

## 最短接入

1. 打开设置：菜单 `AlicizaX/Localization/Settings`，或双击 `Settings.asset`（两个入口都打开 `Project/AlicizaX/Localization`）。点「+ 添加语言」，从 `CultureInfo` 列表多选；特殊语言用「添加自定义语言」。默认语言在「启动语言」里选，必须属于已添加语言。

2. 创建词库：菜单 `AlicizaX/Localization/Create Table` 或 Project 右键 `Create/AlicizaX/Localization/Table`。填 `TableId`、`运行入口目录`（`.loclang` 的落盘目录）、可选的 `常量代码输出`，点「创建」。

3. 录入 Key 与译文：在「新 Key」输入后点「新增」，在网格里按语言列填文本；长文本、换行和多参数内容用底部「文本详情」编辑。空单元格是「未翻译」，显式空串是「空文本」，两者不同。

4. 交付翻译：`导出 CSV` → 翻译 → `导入 CSV`，先看差异预览（新增 Key、更新、移除翻译、无变化单元格），确认后一次 Undo 可整体撤销。

5. 生成常量：填好 `常量代码输出` 后，编辑提交会自动重新生成；也可以随时点 `生成代码`。生成失败的错误会显示在窗口顶部，带「定位」按钮。

6. 生成配置侧多语言数据（可选，属于 Config 链路）：确认 `Config/config.ini`：

```ini
[localization]
path=Excels/Localization
languages=ChineseSimplified,English,Japanese

[paths]
table_data_out=../Client/Assets/Bundles/Configs/bytes/ordinary
```

然后在 `Config` 目录运行 `.\gen-all.bat`（macOS 用 `./gen-all.command`），产出 `Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes`。生成规则见[Luban 生成与脚本](../config/generate.md)。

## LocalizationSettings 字段

序列化字段（`Runtime/Catalog/LocalizationSettings.cs`）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `Languages` | `LanguageDefinition[]` | 语言定义数组，元素为 `Id`、`LocaleCode`、`DisplayName` |
| `DefaultLocale` | `string` | 默认语言，取某个 `Id` |
| `UseSystemLanguage` | `bool` | 没有有效指定语言或玩家记录时，是否用系统语言匹配 |

字段语义与设置界面行为：

| 字段 | 设置界面 | 约束 |
| --- | --- | --- |
| `Id` | 「资源 Id」 | 用于语言子资产命名、`.loclang` 文件名与 Luban 语言别名；已有语言子资产使用的 `Id` 会被锁定，必须先处理 Table |
| `LocaleCode` | 「语言代码」，只读 | 标准语言代码（`zh-Hans`、`en`、`en-GB`），添加时写入，用于匹配系统语言 |
| `DisplayName` | 「显示名称」 | 可改，供语言下拉显示 |
| `DefaultLocale` | 「默认语言」下拉 | 必须是已添加语言；改名或删除语言时自动跟随 |
| `UseSystemLanguage` | 「跟随系统语言」 | 关闭时不做系统匹配 |

`Validate()` 的失败条件（抛 `FormatException`）：语言列表为空、默认语言不在列表、`Id` 为空或重复、非空 `LocaleCode` 重复（忽略大小写）。

启动语言的选择顺序是「有效 `initialLanguage` → 有效玩家记录 → 系统语言（开启时）→ 默认语言」，编辑器在未显式传语言时还会叠加预览语言；完整语义见[运行时加载与切换](runtime.md)。

## Localization 窗口

菜单 `AlicizaX/Localization/Tables`（窗口标题 `Localization`）。顶部两个页签，右上「语言设置 ↗」跳到 Project Settings。

### 词库编辑

| 区域 | 能做什么 |
| --- | --- |
| 左栏 | 列出项目内全部 Table（悬停显示词条数），底部「+ 新建词库」 |
| 标题行 | 当前 `TableId`、词条数/语言数、「定位资产」 |
| 工具行 | 搜索（匹配 Key 或任一语言译文）、「只看缺失」、`导入 CSV`、`导出 CSV`、`生成代码` |
| 词库设置与语言资产 | `运行入口目录`、`运行入口命名`、`常量代码输出`；逐语言一行，未创建显示「创建」，已创建显示「定位入口」 |
| 网格 | Key 列 + 各语言列；单元格延迟提交，空值显示 `未翻译`、空串显示 `空文本`，语言列未建资产时表头带 `· 未创建` |
| Key 操作 | 「新增」「复制」（生成 `{key}_Copy` 并复制各语言值）「删除」 |
| 文本详情 | 改 Key 名、TextArea 编辑长文本、「设为未翻译」「设为空文本」「应用文本」 |

校验错误显示在窗口顶部：普通错误带「定位」，点开后选中对应 Table、Key、语言并滚动到单元格；跨表格式参数冲突额外有「对照」，跳到被对照的单元格。设置类错误直接打开 Project Settings。

### 叠加预览

用于在没有运行游戏时确认多层词库的最终文本：

- 非播放模式：语言下拉（写入 `EditorPrefs`，不动设置资产）、可拖动排序的「模拟加入顺序（越靠后优先）」、「+ 添加 Table」，右侧「最终文本与来源」列出 Key、胜出 Table、译文，多来源时显示叠加链。
- 播放模式：改成显示实际加载顺序与 `LocalizationKit.LoadedTables`。
- 点击结果行会跳回词库编辑并选中该 Key 的语言单元格。

## Table 资产与 .loclang

父表 `LocalizationTableCollection` 的字段：

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `TableId` | 新建时填写 | 词库标识，须与 `AddTableAsync` 的 `tableId`、运行时资产 `TableId` 一致 |
| `Settings` | 新建时取全局设置 | 指向 `LocalizationSettings` |
| `Keys` | 空 | Key 列表，窗口顺序即此顺序 |
| `Tables` | 空 | 各语言 `LocalizationLanguageTable`，作为同一 `.asset` 的子资产保存 |
| `AssetDirectory` | `Assets/Bundles/Configs/sciptableObject` | 运行入口目录 |
| `AddressTemplate` | `{table}_{locale}` | 运行入口命名模板 |
| `CodeOutput` | 空 | 常量代码输出路径，空表示不生成 |

语言子资产字段只有 `TableId`、`Locale`、`Entries`（每项 `Key` + `Value`）；`Read()` 会拒绝空 Key 与重复 Key。

`.loclang` 只是入口文件，内容是父表 GUID 与语言：

```json
{
  "TableGuid": "bd0488ec1d5d4349a57f86ba0267d1d3",
  "Locale": "English"
}
```

`LocalizationLanguageImporter`（`ScriptedImporter`，扩展名 `loclang`）在导入时按 GUID 找到父表，取该语言的子资产，做一次 `Read()` 校验，再生成一份独立的 `LocalizationLanguageTable` 运行资产（资产名取入口文件名）。入口路径 = `AssetDirectory` + `/` + `AddressTemplate` 替换 `{table}`、`{locale}` + `.loclang`。每次提交编辑都会 `SyncEntries`：缺文件就创建、命名变了就移动、语言被删就删除，并拦截路径冲突和「路径已被别的表占用」。编辑父表后入口资产会自动重导入，因为导入器把父表登记成了依赖。

## CSV 格式与列约定

第一行是表头 `Key,<语言 Id>,...`，语言列顺序取 `LocalizationSettings.Locales`：

```csv
Key,ChineseSimplified,English,Japanese
Bootstrap.Update.Ready,更新完成,Update complete,更新が完了しました
UI.Shop.Title,商店,Shop,ショップ
```

单元格约定：

| 写法 | 含义 |
| --- | --- |
| 空单元格 | 未翻译（内部 `null`），运行时回退为 Key |
| 显式空串 | 文件里写作 6 个双引号 `""""""`（CSV 引号包裹的两个双引号），预览显示 `〈空文本〉` |
| 含逗号/引号/换行 | 按标准 CSV 规则加引号并转义内部引号 |

导入规则：

- 导出是 UTF-8 带 BOM；导入接受带或不带 BOM 的严格 UTF-8，非法字节直接报错。
- 表头首列必须是 `Key`；语言列必须是设置里已有的语言，否则报「Unknown CSV language」；重复列、重复 Key、列数不齐都拒绝。
- 只改 CSV 里出现的 Key 和语言列，未出现的保持原样，因此可以只交付部分语言。
- 解析或校验失败会给出「CSV 第 N 行，第 M 列」的物理行列号，并带上 Key 与语言，便于定位。
- 预览与导入是同一份计划：预览后如果源资产被改动，确认导入会抛「词库内容已改变，请刷新差异后再导入」。
- 一次导入是一个 Undo 组，撤销会连带撤销新建的语言子资产与 `.loclang`。

## 代码生成

`LocalizationCodeGenerator` 生成的是「Key 常量 + 取文案访问器」，不是数据集：

```csharp
using AlicizaX.Localization.Runtime;

public static class LocalizationKey
{
    public static class UI
    {
        public const string HOME_BUTTONSHOP_Raw = "UI.Home.ButtonShop";
        public static string HOME_BUTTONSHOP => LocalizationKit.Get(HOME_BUTTONSHOP_Raw);

        public const string SHOP_CURRENCY_Raw = "UI.Shop.Currency";
        public static string SHOP_CURRENCY<T0>(T0 arg0) => LocalizationKit.Format(SHOP_CURRENCY_Raw, arg0);
    }
}
```

命名与生成规则：

| 规则 | 说明 |
| --- | --- |
| 外层类名 | `CodeOutput` 的文件名（不含扩展名），必须是合法 C# 标识符且路径以 `.cs` 结尾 |
| 嵌套类 | Key 第一个 `.` 之前的前缀；无 `.` 的 Key 归入 `Common` |
| 成员名 | 去掉前缀后把 `.` 换成 `_` 并转大写，再加 `_Raw` |
| 无参数 | 生成 `public static string NAME => LocalizationKit.Get(NAME_Raw);` |
| 有参数 | 按占位符个数生成 `Format<T0..Tn>`，参数个数来自该 Key 在所有表、所有语言里的格式签名 |
| 冲突 | 外层类名、嵌套类名、成员名冲突都会抛 `LocalizationValidationException`，并定位到具体 Key |
| 重写 | 内容未变化时不写文件；写入后若在 `Assets` 下会自动 `ImportAsset` |

`Validate()` 除生成前检查外，还负责跨表校验：重复 `TableId`、`CodeOutput` 与其他表冲突、语言子资产的 `TableId`/`Locale`/Key 不合法、Key 未在父表登记，以及同一个 Key 在各语言、各表之间的格式参数必须一致（空文本与未翻译不参与比较）。格式错误的文案（例如 `{0` 未闭合）会直接报「Invalid format」。

## 构建校验

`LocalizationBuildGuard` 实现 `IPreprocessBuildWithReport`（`callbackOrder = int.MaxValue`）与 `IProcessSceneWithReport`：

- 构建前扫描所有 `/Resources/` 下的资产和 `PlayerSettings` 预加载资产，任何依赖命中父表就抛 `BuildFailedException`，并同时打印依赖路径与引用来源。
- 每个场景处理阶段检查场景资产依赖，以及 `EditorUtility.CollectDependencies` 收集到的对象，防止前置回调新引入的引用漏网。
- 「编辑源资产」的判定是「路径主资产类型为 `LocalizationTableCollection`」，子资产与父表同路径，因此引用语言子资产同样会被拦下。

YooAsset 收集侧用项目的 `LocalizationAssetIgnoreRule`：先按同一判定排除父表，再对剩余资产执行 `ValidateAssets`，确保运行资产不反向依赖编辑源。新 Package 也应选这个忽略规则。

## 与 Config/Excel/Luban 的生成衔接

Config 链路与 Localization 链路互相独立，只靠语言 Id 对齐。要点：

- `Config/config.ini` 的 `[localization] languages` 同时决定工作簿里要读哪些语言列，以及 `Game_{locale}.bytes` 的文件名后缀；这些值必须等于 `LocalizationSettings` 的 `Id`。
- 输出目录不是单独配置的，而是 `[paths] table_data_out` 的兄弟目录 `localization`，本工程即 `Client/Assets/Bundles/Configs/bytes/localization`。
- 文案工作簿放在 `[localization] path`（本工程 `Config/Excels/Localization`），`Texts` 工作表用 `##var` 行声明 `key` 与语言列，`##type` 行声明类型，`##` 行写注释；数据行首列留空。
- 二进制布局为「条数 + 每条（Key、是否有值、值）」，按 Key 排序；空单元格写成「无值」，运行时 `ConfigLocalization.Get` 回退 Key；单元格写两个双引号表示显式空串。
- 生成由 `gen-all` 调用 Luban 的 `alicizax-localization` 管线完成；写入模式会删除不再属于任何语言的旧 `Game_*.bytes`，`--check` 只比较不写盘。

命令行参数、`build.bat` 与工作表完整约定见 [Luban 生成与脚本](../config/generate.md)；生成物如何被 `ConfigService` 消费见[生成物与运行时衔接](../config/outputs.md)；生成失败排查见[常见错误排查](../config/troubleshooting.md)。

## 常见配置

| 项目 | 本工程取值 |
| --- | --- |
| 设置资产 | `Assets/Resources/Localization/Settings.asset`（路径写死在 `LocalizationSettings.AssetPath`） |
| 语言与代码 | `ChineseSimplified`/`zh-Hans`、`English`/`en`、`Japanese`/`ja` |
| 父表位置 | `Client/Assets/Editor/Localization/StartupTable.asset`、`CommonTable.asset` |
| 启动词库入口 | `Client/Assets/Resources/Localization/StartupTable_{locale}.loclang` |
| 热更词库入口 | `Client/Assets/Bundles/Configs/sciptableObject/CommonTable_{locale}.loclang` |
| 常量输出 | `Assets/Scripts/Startup/UpdateLocalizationKey.cs`、`Assets/Scripts/Hotfix/GameLogic/LocalizationKey.cs` |
| 配置多语言输出 | `Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes` |
| 生成命令 | `Config/gen-all.bat`、`gen-all.bat --check` |

## 容易踩的坑

1. 语言 `Id` 一旦被某个 Table 的语言子资产使用就会被锁定；直接改 `Id` 会同时影响 `.loclang` 文件名、`Game_{locale}.bytes` 后缀和 `config.ini` 的语言列，必须先调整词库再改。
2. 新增语言后旧词库不会自动生成该语言子资产，网格表头显示 `· 未创建`，此时不会产出对应 `.loclang`；点「创建」或用 CSV 该语言列导入才会建。
3. `CodeOutput` 不能和其他表重复（同路径或同文件名都不行），类名/嵌套类名/成员名冲突会直接失败；Key 前缀决定嵌套类名，非法字符与关键字会被替换成下划线或加前缀，最好一开始就用字母开头的 ASCII 前缀。
4. CSV 空单元格和显式空串语义相反：空单元格是未翻译（回退 Key），`""` 是空文本（显示空）；预览确认后源被改动，导入会被拒绝而不是覆盖。
5. 不要手改 `.loclang`：它只记录父表 GUID 与语言，改 GUID 会让导入失败；要搬目录请改 Table 的 `运行入口目录`，让 `SyncEntries` 移动文件并保留 GUID。
6. 父表放错位置（例如放进 `Resources`、被 Prefab 或场景引用）会在构建或 YooAsset 收集时报错；编辑源应留在 `Assets/Editor` 这类仅编辑器目录。

## API 速查

| API | 用途 |
| --- | --- |
| `LocalizationTableEditing.FindAll()` | 列出项目内全部父表 |
| `LocalizationTableEditing.EnsureLanguage(table, locale)` | 按语言创建/取得子资产 |
| `LocalizationTableEditing.SetValue / Value / AddKey / RenameKey / DeleteKey` | 编辑源数据 |
| `LocalizationTableEditing.Commit(table)` | 保存、同步 `.loclang`、触发代码生成 |
| `LocalizationCsv.Export / Read / Preview / Import` | CSV 导出、解析、差异预览、导入 |
| `LocalizationCodeGenerator.Validate / Generate` | 校验词库与生成常量 |
| `LocalizationLanguageImporter.EntryPath / ValidateEntryPath / SyncEntries` | 入口路径与入口同步 |
| `LocalizationBuildGuard.IsSourceAsset / ValidateAssets` | 识别编辑源资产、构建前校验依赖 |
