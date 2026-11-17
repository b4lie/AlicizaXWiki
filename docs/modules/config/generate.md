# Luban 生成与脚本

这一页讲清 `luban.conf`、`config.ini` 和四个脚本（`build` / `gen-all` 的 Windows 与 macOS 版本）各自做什么，以及“增量生成”的真实含义。

## 适用场景

- 需要改输出目录、加导出目标、加语言。
- 升级 Luban，或插件代码改动后要重建 `Tools`。
- 想知道 `gen-all` 和 `build` 的区别、CI 里该跑哪个。

## 两个配置文件

### `luban.conf`

```json
{
  "groups": [
    {"names":["c"], "default":true},
    {"names":["s"], "default":true},
    {"names":["e"], "default":true}
  ],
  "schemaFiles": [
    {"fileName":"Defines/builtin.xml", "type":""},
    {"fileName":"Excels/__tables__.xlsx", "type":"table"},
    {"fileName":"Excels/__beans__.xlsx", "type":"bean"},
    {"fileName":"Excels/__enums__.xlsx", "type":"enum"}
  ],
  "dataDir": "Excels",
  "targets": [
    {"name":"server", "manager":"TablesLoader", "groups":["s"], "topModule":"GameConfig"},
    {"name":"client", "manager":"ConfigService", "groups":["c"], "topModule":"Game.Config"},
    {"name":"all", "manager":"Tables", "groups":["c","s","e"], "topModule":"cfg"}
  ]
}
```

| 键 | 说明 |
| :--- | :--- |
| `groups` | 定义 `c`（客户端）/`s`（服务端）/`e`（编辑器）三个分组，`default` 表示默认启用 |
| `schemaFiles` | 引入内置类型、表清单、bean、枚举四个 schema |
| `dataDir` | 数据根目录，脚本里的 `input` 相对它解析 |
| `targets[].name` | 生成目标名，命令行 `-t client` 用的就是它 |
| `targets[].manager` | 生成的表管理类名，决定 `IConfigService`/`ConfigService` 的类名 |
| `targets[].groups` | 该目标包含哪些分组，与表/字段的 `group` 求交集 |
| `targets[].topModule` | 生成代码的命名空间前缀（`Game.Config`）与顶层模块名 |

当前只有 `client` 目标参与日常生成，`all` 目标只用于 `build` 脚本的只校验（`validationOnly`）预检。

### `config.ini`

```ini
[paths]
; 路径相对于 Config 目录，普通表结构与输入由 luban.conf 定义。
table_data_out=../Client/Assets/Bundles/Configs/bytes/ordinary
code_out=../Client/Assets/Scripts/Hotfix/GameProto/Config/Generate

[localization]
; 只读取当前目录的普通多语言 Excel；列标识同时作为 Game_{标识}.bytes 的后缀。
path=Excels/Localization
languages=ChineseSimplified,English,Japanese
```

| 段 | 键 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `[paths]` | `table_data_out` | 是 | 普通表 `.bytes` 输出目录；多语言输出目录由它推导：同级的 `localization/` |
| `[paths]` | `code_out` | 是 | 生成代码输出目录 |
| `[localization]` | `path` | 存在该段时必填 | 多语言 Excel 目录（相对 `Config`） |
| `[localization]` | `languages` | 存在该段时必填 | 语言标识列表，同时是 `Texts` 工作表的列名和 `Game_{语言}.bytes` 的后缀 |

- 路径都相对 `Config/` 解析，不要写绝对路径。
- 删掉 `[localization]` 段可以关闭配置表多语言生成，但 `Game_*.bytes` 会全部变成“过期输出”被清理。
- `languages` 必须非空且不能重复，否则插件直接报错。

## 两个脚本

| 脚本 | 何时用 | 做什么 | 耗时 |
| :--- | :--- | :--- | :--- |
| `build.bat` / `build.command` | 首次搭建、升级 Luban、改 `Extensions/` 插件代码 | 拉取最新 Luban → 编译 Luban + 插件 → 预检 → 校验输出 → 原子替换 `Tools/` → 复查 → 删除 Luban 源码 | 分钟级 |
| `gen-all.bat` / `gen-all.command` | 日常改表 | 用 `Tools/` 里已有的 Luban 生成代码与数据，可加 `--check` 只校验 | 秒级到十几秒 |

`build` 依赖 `gen-all --check` 做验收，所以**两个脚本必须同时可用**；只保留其中一个会导致构建流程中断。

### `build` 的执行顺序

1. 检查 .NET 8 SDK（`8.0.x` 以上）与 `Microsoft.NETCore.App 8.x` 运行时；缺一个就提示下载并退出，`Tools/` 不动。
2. 若 `Config/Luban` 已存在，校验它是一个干净的官方仓库检出（origin 必须等于 `https://github.com/focus-creative-games/luban.git`，且无本地改动）。
3. 若 `Tools.backup/` 已存在，直接报错退出，避免覆盖上一次的回滚点。
4. 解析 Luban 最新 Release 版本号，`git clone --depth 1 --branch <tag>` 或 `fetch + switch --detach` 到该版本。
5. 清空 `Tools.build/`，用 `dotnet build` 编译 `Luban.csproj`，并通过 MSBuild 属性把本地插件注入进去：
   - `CustomAfterMicrosoftCommonTargets=Extensions/Luban.AlicizaX.Localization/LubanHost.targets`
   - `AlicizaXLocalizationProject=.../Luban.AlicizaX.Localization.csproj`
   - `LubanCoreProject=Luban/src/Luban.Core/Luban.Core.csproj`
6. 检查产物是否包含 `Luban.dll`、`Luban.runtimeconfig.json`、`Luban.AlicizaX.Localization.dll`、`Templates/cs-bin/bean.sbn`。
7. 用**暂存工具**跑两遍：
   - `-t all` + `-x alicizax.validationOnly=true`：只校验 schema、多语言 key 与数据，不写盘。
   - `-t client` + `-x alicizax.check=true`：完整生成到临时目录，再和仓库现有生成物逐字节比对。
8. 两次都通过才把 `Tools/` 移到 `Tools.backup/`，把 `Tools.build/` 移到 `Tools/`。
9. 用新 `Tools/` 跑一次 `gen-all --check`；失败就把新工具挪回 `Tools.build/` 并恢复备份。
10. 删除 `Tools.backup/` 与 `Config/Luban`，结束。

第 7 步是关键：**仓库里的生成物必须是“用新工具生成的结果”**，否则 `build` 会在安装前失败（详见[常见错误排查](troubleshooting.md)）。

### `gen-all` 的命令行

```bat
dotnet "Tools\Luban.dll" ^
    -t client ^
    --conf "luban.conf" ^
    -c cs-bin ^
    -d bin ^
    -p alicizax-localization ^
    --customTemplateDir "Defines\ClientTemplate" ^
    --strict ^
    -x "outputSaver=alicizax-stage" ^
    -x "alicizax.config=config.ini" ^
    -x "l10n.provider=alicizax" ^
    %CHECK%
```

macOS 版本参数完全一致，只是路径分隔符与续行符不同。

| 参数 | 作用 |
| :--- | :--- |
| `-t client` | 使用 `luban.conf` 中的 `client` 目标 |
| `--conf luban.conf` | 工程配置 |
| `-c cs-bin` | 代码目标：C# + 二进制数据 |
| `-d bin` | 数据目标：二进制 |
| `-p alicizax-localization` | 启用插件注册的生成管线（`[Pipeline("alicizax-localization")]`） |
| `--customTemplateDir Defines\ClientTemplate` | 优先使用项目自定义模板；同名模板覆盖 `Tools/Templates` 里的内置模板 |
| `--strict` | 严格模式：校验告警按错误处理，直接失败 |
| `-x outputSaver=alicizax-stage` | 把输出重定向到临时暂存目录；插件强制要求该值 |
| `-x alicizax.config=config.ini` | 插件读取的 INI 路径 |
| `-x l10n.provider=alicizax` | 使用插件实现的 `[TextProvider("alicizax")]` 校验文案 key |
| `-x alicizax.check=true` | 只比对、不写盘；有差异或以退出码 1 结束 |
| `--check`（脚本参数） | `gen-all` 的语法糖，展开成上面的 `alicizax.check=true` |

`--check` 成功时输出：

```text
Checked 42 generated files.
```

（数字为本次参与比对的生成物总数，随表数量变化。）

失败时输出差异清单并以退出码 1 结束：

```text
Generated files are out of date:
D:\...\ConfigService.cs
D:\...\tables_tbitem.bytes
```

## 全量与增量的真实含义

| 说法 | 实际情况 |
| :--- | :--- |
| “增量生成” | **计算量始终是全量**：Luban 每次都会解析全部 schema 与 Excel；插件没有“只处理改动表”的开关 |
| 增量体现在写盘 | 生成物先落到临时目录，逐个与目标文件比较；内容相同就跳过写入 |
| 增量体现在清理 | 目标目录里存在、但本次没有生成的文件会被删除（`.cs`、`.bytes`，以及多余的 `Game_*.bytes`），连同 `.meta` |
| `build` 是全量 | 每次都清空 `Tools.build/` 重新编译，并重新下载/切换 Luban 源码 |
| 什么时候会变慢 | 表数量和行数增长时解析变慢；`Config/Tools` 重建是分钟级 |

因为清理逻辑同时覆盖代码目录和数据目录，**不要拆开只跑 `-c` 或只跑 `-d`**：没生成的那一侧会被判定为过期并整体删除。只想检查差异时用 `--check` 或 `validationOnly`。

## 常用命令

```bat
:: 日常生成
cd /d D:\UnityProject\AlicizaXTemplate\Config
gen-all.bat

:: 只校验仓库生成物是否最新（CI 用这个）
gen-all.bat --check

:: 重建工具链（升级 Luban / 改插件）
build.bat

:: 无人值守构建，结束不暂停
build.bat --no-pause
```

手动单步执行（排查用，路径按需替换）：

```bat
:: 只做 schema 与多语言校验，不写任何文件
dotnet Tools\Luban.dll -t all --conf luban.conf -p alicizax-localization --strict ^
  -x outputSaver=alicizax-stage -x alicizax.config=config.ini ^
  -x alicizax.validationOnly=true -x l10n.provider=alicizax
```

## 常见配置

| 需求 | 改动 |
| :--- | :--- |
| 换输出目录 | 改 `config.ini` 的 `table_data_out` / `code_out`，全量生成一次；旧目录不会被自动清理，需手动删除或保留 |
| 加一种语言 | `config.ini` 的 `languages` 追加标识 + `Excels/Localization/*.xlsx` 的 `Texts` 加同名列 |
| 关闭配置表多语言 | 删除 `[localization]` 段（会清理已有 `Game_*.bytes`，谨慎） |
| 导出服务端数据 | 需要单独的 `config.ini`（不同输出目录）；当前仓库只配置了客户端输出，同一个 INI 跑 `-t server` 会覆盖客户端目录 |
| 固定 Luban 版本 | 脚本不支持：`build` 始终切到最新 Release |
| 改生成代码风格 | 改 `Defines/ClientTemplate/cs-bin/*.sbn`，属于源码改动，需评审 |

## 容易踩的坑

1. **`build` 前没提交/生成**：第 7 步的 `alicizax.check` 会失败。先用旧工具跑一次 `gen-all` 把生成物补齐，再重建。
2. **改了插件却只跑 `gen-all`**：`gen-all` 用 `Tools/` 里的旧 DLL，插件改动不生效，必须 `build`。
3. **手动删了 `Tools/Templates`**：`build` 会检查 `Templates/cs-bin/bean.sbn`；临时缺文件时直接重跑 `build` 恢复。
4. **`Tools.backup` 残留**：`build` 拒绝在备份存在时继续。确认旧构建无用后删除该目录，或先恢复它。
5. **在 `Config/Luban` 里改代码**：脚本检测到本地改动会拒绝更新/删除；这里只是临时检出，不要在这里改东西。
6. **`-x outputSaver` 写错**：插件会直接抛 `The alicizax-localization pipeline requires -x outputSaver=alicizax-stage.`。

## 命令速查

| 命令 | 结果 |
| :--- | :--- |
| `gen-all` | 生成代码 + 普通表数据 + 多语言数据 |
| `gen-all --check` | 不写盘，校验生成物是否最新 |
| `build` | 重建 `Config/Tools` |
| `build --no-pause` | 同上，失败/成功都不暂停（CI 用） |
| `dotnet Tools/Luban.dll ... -x alicizax.validationOnly=true` | 只校验 schema 与数据 |

输出目录结构与运行时装配见 [生成物与运行时衔接](outputs.md)。
