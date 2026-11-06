# 配置与词库生成

本地化数据由项目的 Luban 链条生成：`Config/config.ini` 决定语言、工作簿扫描范围和输出路径，`gen-all` 一次产出常量访问器、词库 `.bytes` 和编辑器预览用的 `RuntimeOptions.json`。

## config.ini

`[localization]` 段：

| 字段 | 说明 |
| --- | --- |
| `languages` | 支持的语言列表，逗号分隔，例如 `ChineseSimplified,English,Japanese` |
| `default_locale` | 项目默认语言，必须出现在 `languages` 中 |
| `runtime_options_out` | `RuntimeOptions.json` 输出路径，通常指向 `Assets/Resources/Localization` |
| `exclude_constant_prefixes` | 按 Key 首段排除常量访问器；数据和校验仍保留这些 Key |

所有路径相对于 `Config` 目录。当前工程的配置：

```ini
[localization]
languages=ChineseSimplified,English,Japanese
default_locale=ChineseSimplified
runtime_options_out=../Client/Assets/Resources/Localization/RuntimeOptions.json
exclude_constant_prefixes=

[constant.dir.main]
path=Excels/Localization
prefix=LocalizationConst
data_out=../Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes
code_out=../Client/Assets/Scripts/Hotfix/GameLogic/LocalizationKey.cs

[constant.file.update]
path=Excels/Localization/LocalizationConst_Update.xlsx
data_out=../Client/Assets/Resources/Localization/Update_{locale}.bytes
code_out=../Client/Assets/Scripts/Startup/UpdateLocalizationKey.cs

[ordinary.dir.main]
path=Excels/Localization
prefix=Localization_
data_out=../Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes
```

## 工作簿格式

文案工作簿使用 `Texts` 工作表，前两行分别以 `##var`、`##type` 开头，列名包含 `key` 和配置中的各语言。仓库示例在 `Config/Excels/Localization`。

规则：

- 不要添加 `layer` 列；多语言表没有层级。
- 同一个工作簿的 Key 必须唯一。
- 翻译单元格留空表示缺失，运行时返回原始 Key。
- 翻译单元格写入 `""` 表示空字符串，不是缺失。
- 生成时校验格式参数，以及普通 Luban 表中 `text` 字段引用的 Key。

## 输出规则

### 常量表

`[constant.dir.main]` 扫描 `Excels/Localization` 当前层、文件名以 `LocalizationConst` 开头的 `.xlsx`。

| 字段 | 说明 |
| --- | --- |
| `path` | 扫描目录，相对于 `Config` |
| `prefix` | 文件名前缀 |
| `data_out` | 词库输出路径，必须包含 `{locale}` 并以 `.bytes` 结尾 |
| `code_out` | 常量访问器输出路径，必须是 `.cs`；类名取文件名 |

新增 `LocalizationConst_Shop.xlsx` 会自动纳入目录规则并共用 `Game_{locale}` 输出。需要为某张表单独指定输出时，添加 `[constant.file.shop]`，填 `path`、`data_out`、`code_out`；`file` 规则优先于目录规则。

`[constant.file.update]` 是更新界面的单表规则，输出 `Update_{locale}.bytes` 到 `Resources`，让启动流程在热更资源可用前也能加载。对应访问器是 `UpdateLocalizationKey.cs`。

### 普通映射表

`[ordinary.dir.main]` 只导出词库，不生成常量访问器，适合只按 Key 取文案、不需要强类型属性的表。当前工程用 `Localization_` 前缀，输出与常量表相同的 `Game_{locale}`，相同目标会合并词库。

`exclude_constant_prefixes` 按 Key 首段跳过常量访问器生成，但数据和 Key 校验仍保留。

## 生成与检查

首次构建或更新 Luban 工具时，在 `Config` 目录运行 `build.bat`（Windows）或 `build.command`（macOS）。需要 .NET 8 SDK、.NET 8 Runtime 和 Git；macOS 还需要 curl。成功后工具在 `Config/Tools`。

生成普通表和多语言词库：

```text
Windows: .\gen-all.bat
macOS:   ./gen-all.command
```

提交或构建前检查生成文件是否与表格一致：

```text
Windows: .\gen-all.bat --check
macOS:   ./gen-all.command --check
```

Unity 编辑器里使用工具栏 **Tools** 下拉的：

- `Gen-Config`：等价于运行 `gen-all`。
- `Gen-Check Generated Files`：等价于运行 `gen-all --check`。

构建前由 BuildCLI 自动执行 `Gen-Check Generated Files`；`--check` 只比较结果，不修改生成文件。

## RuntimeOptions.json

`gen-all` 把语言列表、默认语言和编辑器预览词库路径写入 `runtime_options_out`：

```json
{
  "Version": 1,
  "Locales": ["ChineseSimplified", "English", "Japanese"],
  "DefaultLocale": "ChineseSimplified",
  "Catalogs": [
    "../../Bundles/Configs/bytes/localization/Game_{locale}.bytes",
    "Update_{locale}.bytes"
  ]
}
```

字段：

| 字段 | 运行时 | 编辑器预览 |
| --- | --- | --- |
| `Locales` / `DefaultLocale` | 解析语言列表和默认语言 | 解析语言列表和默认语言 |
| `Catalogs` | 不使用，运行时由启动代码加载 | 相对 `RuntimeOptions.json` 所在目录解析，用于预览 |
| `Version` | 不使用 | 必须为 `1` |

编辑器按文件名在 `Assets` 下查找 `RuntimeOptions.json`，只能有一份；存在多份会报错。升级旧项目时删除原来的 `AlicizaXLocalizationPreview.json`。调整输出位置后，还要同步启动代码的 `Resources.Load` 路径和 BuildCLI 的构建语言校验路径。

## 注意事项

1. `data_out` 缺少 `{locale}` 或不是 `.bytes` 时生成失败。
2. 相同 `data_out` 会合并词库；跨表重复 Key 会在生成或运行时被拦截。
3. `runtime_options_out` 只影响运行时语言列表和编辑器预览，不决定运行时要加载哪些词库。
4. 构建前先跑一次 `--check`，避免提交的生成文件与表格不一致。
5. 生成文件的访问器类名取文件名，改名会改变 `LocalizationKey` 的引用。
