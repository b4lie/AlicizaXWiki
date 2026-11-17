# 常见错误排查

按“你看到的报错”查表，再按处理步骤操作。所有报错文本都来自当前脚本与插件源码。

## 适用场景

- `build.bat` / `build.command` 中断。
- `gen-all` 失败或生成的代码不完整。
- Unity 里编译报缺类型，或运行时读不到配置。

## 构建工具链（build）

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| `No compatible .NET SDK is available to this script.` | PATH 里没有 8.0+ 的 SDK，或只装了 Runtime | 安装 .NET 8 SDK，重开终端；`dotnet --list-sdks` 应能看到 `8.x` |
| `.NET 8 Runtime is missing. The .NET 8 SDK includes it.` | 有 SDK 但缺 `Microsoft.NETCore.App 8.x` | 安装 .NET 8 SDK（自带运行时） |
| `"…\Config\Luban" is not a clean checkout of the official Luban repository.` | `Config/Luban` 被手工改过、origin 不对，或是符号链接 | 确认没有本地改动；无用时删除整个 `Config/Luban` 后重跑 |
| `…\Config\Tools.backup already exists.` | 上次构建留下回滚点 | 确认新工具可用后删除 `Tools.backup`；如果新工具坏了，先把它移回 `Tools` |
| `Could not resolve the latest Luban release.` | 网络无法访问 GitHub Releases（Windows 用 PowerShell 请求，macOS 用 `curl`） | 配置代理，确认能打开 `github.com/focus-creative-games/luban/releases/latest` |
| `Luban source build failed.` | `dotnet build` 失败，常见为 SDK 版本不匹配或插件编译错误 | 先在本机直接执行脚本打印的 `dotnet build` 命令看完整错误；改 `Extensions/` 后尤其要检查插件 |
| `Build output is missing …Luban.dll` / `…Templates/cs-bin/bean.sbn` | 构建产物不完整 | 重跑 `build`；持续失败时检查磁盘与杀毒软件拦截 |
| `Staged Luban validation failed.` | 表结构、数据或多语言 key 校验不通过 | 看上一行 Luban 的具体报错；常见是缺列、重复 key、`#ref` 指向不存在的表 |
| `Staged Luban output differs from generated files.` | 仓库里的生成物不是新工具的输出 | 见下方“生成物与工具不一致” |
| `Could not back up existing Tools.` | `Tools` 被占用（Unity/杀软/资源管理器） | 关闭占用进程后重试 |
| `Installed Tools failed verification. Restoring the previous build...` | 新工具已装入，但 `gen-all --check` 失败 | 自动回滚；检查是否同时改了 `config.ini` 输出目录或插件 |

### 生成物与工具不一致

`build` 的第 7 步会用新工具生成到暂存区，再和仓库里的生成物逐字节比较。改过表还没生成、或升级了 Luban 模板时必然不一致。恢复流程：

```bat
cd /d D:\UnityProject\AlicizaXTemplate\Config

:: 1. 上一次 build 失败后，暂存区还在（build 开头会清空它）
::    先用暂存工具把生成物补齐
dotnet Tools.build\Luban.dll -t client --conf luban.conf -c cs-bin -d bin ^
  -p alicizax-localization --customTemplateDir Defines\ClientTemplate --strict ^
  -x outputSaver=alicizax-stage -x alicizax.config=config.ini ^
  -x l10n.provider=alicizax -x alicizax.check=true

:: 2. 有差异时去掉 check 参数正式生成，再重跑 build
build.bat
```

> 若 `Tools.build` 已被清掉，先用当前的 `Tools` 跑 `gen-all` 补齐；仍然失败说明新旧模板确实不兼容，需要人工确认生成物差异后再决定是否提交。

## 日常生成（gen-all）

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| `Luban localization extension is missing. Run build.bat.` | `Tools/Luban.AlicizaX.Localization.dll` 不存在 | 跑 `build.bat` 重建工具链 |
| `Usage: gen-all.bat [--check]` | 传了不认识的参数 | 只支持无参数或 `--check` |
| `Generated files are out of date:` + 文件列表 | 生成物与 Excel 不一致 | 去掉 `--check` 正式生成，提交变更 |
| `Output collision: <路径>` | 代码、普通表、多语言三路输出落到同一路径（通常是 `code_out` / `table_data_out` 配成了同一个目录） | 改 `config.ini`，两个目录必须不同 |
| `Missing ##var row: <xlsx>` | 多语言 Excel 缺 `##var` 表头行 | 在 `Texts` 工作表首行补 `##var` + `key` + 各语言列 |
| `Missing <列名> column: <xlsx>` | `config.ini` 的 `languages` 里有语言在 `Texts` 表中没有对应列（或缺 `key` 列） | 补齐列名，注意大小写完全一致 |
| `Missing Texts sheet: <xlsx>` | 多语言工作簿里没有名为 `Texts` 的工作表 | 重命名工作表或改用模板 |
| `Duplicate key 'xxx': a.xlsx:3 / b.xlsx:8` | 两个文件或同一文件出现重复 key | 按提示的两处位置去重 |
| `Localization languages must be nonempty and unique.` | `languages` 有空项或重复 | 修正 `config.ini` |
| `Missing [paths] code_out in config.ini.` | `config.ini` 缺必填键 | 补 `[paths]` 的 `code_out` / `table_data_out` |
| `Invalid or repeated config section at …` / `Invalid or repeated config key at …` | `config.ini` 有重复段/重复键或格式错误 | 按行号修正；注释用 `;` 或 `#` |
| `The alicizax-localization pipeline requires -x outputSaver=alicizax-stage.` | 手动执行命令时漏了 `outputSaver` | 按 [Luban 生成与脚本](generate.md) 的完整命令执行 |
| `Luban validation failed; generated files were not changed.` | `--strict` 下有校验告警 | 看 Luban 输出的具体告警；插件保证此时不写任何文件 |

## Unity 侧

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| 编译报 `The type or namespace name 'IConfigService' could not be found` | 生成代码没进 `GameProto` 程序集，或 `Generate/` 为空 | 确认 `code_out` 指向 `Scripts/Hotfix/GameProto/Config/Generate`，重新 `gen-all`，等 Unity 刷新 |
| 编译报 `ExternalTypeUtil 不存在` | `Config/Extension/ExternalTypeUtil.cs` 被删或移入命名空间 | 该类型必须在全局命名空间，名字与 `builtin.xml` mapper 一致 |
| 运行时 `Service IConfigService is not registered.` | 没注册就 `Require` | 在 `HotfixEntry` 里先 `AppServices.Register<IConfigService>(new ConfigService())` |
| 运行时 `Config service is not loaded.` | 先调了 `SetLanguageAsync` | 先 `await Initialize(loader)` |
| 运行时 `Missing config asset: tables_tbitem` | YooAsset 地址不对，或 `bytes/ordinary` 没被收集 | 地址用去扩展名的文件名；检查 `Configs` 收集器路径 |
| 运行时 `Missing config asset: Game_English` | 多语言包没生成/没进收集器 | 检查 `config.ini` 的 `languages` 与 `bytes/localization` 目录 |
| 反序列化抛 `SerializationException` 或 `table mode=one, but size != 1` | 代码与数据版本不一致（只更新了其中一侧） | 重新全量 `gen-all`，确保代码与 `.bytes` 同批产出 |
| 文案一直显示 key | 没调 `SetLanguageAsync`、语言包缺该 key，或字段不是 `text` 类型 | 确认字段 `##type` 为 `text`；确认语言包包含该 key |

## 手动恢复

```bat
:: 丢弃本地生成物，重新生成
cd /d D:\UnityProject\AlicizaXTemplate
git checkout -- Client/Assets/Scripts/Hotfix/GameProto/Config/Generate
git checkout -- Client/Assets/Bundles/Configs/bytes
cd Config
gen-all.bat
```

- 恢复 `Tools`：把 `Tools.backup` 改回 `Tools`，删除 `Tools.build`。
- 恢复 Luban 源码目录：整个 `Config/Luban` 可以随时删除，下一次 `build` 会重新克隆。

## 容易踩的坑

1. **在 Unity 打开时跑 `build`**：`Tools` 目录可能被占用，导致备份/替换失败；先关编辑器或至少停止播放。
2. **只提交 Excel 不提交生成物**：CI 的 `gen-all --check` 会失败；代码、`.bytes`、`.meta` 要一起提交。
3. **用 `git checkout` 恢复 `Generate/` 后忘记重新生成**：仓库里的生成物可能是旧版本，恢复后仍要 `gen-all` 对齐 Excel。
4. **报错只看最后一行**：Luban 的真实原因通常在 `[ERROR]` 之前的校验输出里。

## 相关页面

- 目录与职责：[Config 配置管线](index.md)
- 表结构写法：[表结构与 Excel 约定](tables.md)
- 脚本与参数：[Luban 生成与脚本](generate.md)
- 运行时装配：[生成物与运行时衔接](outputs.md)
