# 表结构与 Excel 约定

Luban 只认固定表头。这一页讲清 `Config/Excels` 里三类工作簿的写法：表清单、数据表、枚举与 bean。

## 适用场景

- 第一次给项目加配置表，不确定 `##var` / `##type` / `##group` 各写什么。
- 需要给字段加多语言、加外键引用、加结构体或枚举。
- 生成时报“缺列 / 重复 key / schema 错误”，需要对照表头排查。

## 三个 schema 工作簿

| 文件 | 作用 | 关键列 |
| :--- | :--- | :--- |
| `Excels/__tables__.xlsx` | 声明有哪些表、数据从哪来 | `full_name`、`value_type`、`input`、`index`、`mode`、`group` |
| `Excels/__beans__.xlsx` | 声明可复用的结构体 | `full_name`、`parent`、`valueType`、`sep`、`*fields` |
| `Excels/__enums__.xlsx` | 声明枚举 | `full_name`、`flags`、`unique`、`*items` |

三者都由 `luban.conf` 的 `schemaFiles` 引入：

```json
"schemaFiles": [
  {"fileName": "Defines/builtin.xml", "type": ""},
  {"fileName": "Excels/__tables__.xlsx", "type": "table"},
  {"fileName": "Excels/__beans__.xlsx", "type": "bean"},
  {"fileName": "Excels/__enums__.xlsx", "type": "enum"}
]
```

## 表清单：`__tables__.xlsx`

当前的表：

| `full_name` | `value_type` | `input` | `mode` | `group` | 生成属性 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Tables.TbAudio` | `AudioConfig` | `.\Tables\Audio_audio.xlsx` | 默认（map） | `c` | `IConfigService.TbAudio` |
| `Tables.Global` | `GlobalConfig` | `.\Tables\Global_global.xlsx` | `singleton` | `c` | `IConfigService.Global` |
| `Tables.TbItem` | `ItemConfig` | `.\Tables\Item_item.xlsx` | 默认（map） | `c` | `IConfigService.TbItem` |
| `Tables.Shop` | `ShopConfig` | `.\Tables\Shop_shop.xlsx` | 默认（map） | — | `IConfigService.Shop` |

列含义：

| 列 | 必填 | 说明 |
| :--- | :--- | :--- |
| `full_name` | 是 | `模块.表名`。最后一段决定生成属性名（`Tables.TbItem` → `TbItem`） |
| `value_type` | 是 | 记录类型名，对应生成的 bean 类（`ItemConfig`） |
| `read_schema_from_file` | 否 | `true` 表示从数据表表头读取字段定义（当前全部为 `true`） |
| `input` | 是 | 数据文件，相对 `Config/Excels` |
| `index` | 否 | 索引字段；留空时取 `value_type` 的第一个字段。多字段索引用 `,` 分隔 |
| `mode` | 否 | `one` / `map` / `list`；留空默认 `map`。单例表写 `singleton` |
| `group` | 否 | `c` / `s` / `e` 或多个用 `,` 分隔；为空表示所有目标都导出 |
| `comment` | 否 | 表注释，会写进生成代码的 `<summary>` |
| `output` | 否 | 输出文件名，默认 `<module>_<name>.<suffix>` |

`group` 与 `luban.conf` 的 `targets` 对应：`client` 只要 `c`，`server` 只要 `s`，`all` 要 `c,s,e`。当前 `gen-all` 只跑客户端目标，所以 `Shop` 表没写 `group` 也会导出，而只标 `s` 的表不会。

## 数据表：`Excels/Tables/*.xlsx`

一个数据表就是一张工作表，前 5 行是表头：

```text
##var    | id  | name | icon | type          | quality
##var    |     |      |      |               |            ← 多级表头行，当前表留空
##type   | int | text | string | item.EItemType | item.EQuality
##group  |     | c    | c    | c             | c
##       | 道具ID | 名称 | 图标 | 道具类型 | 道具品质
         | 9000 | item_1_name | Item_1000 | OTHER | WHITE
```

- `##var`：字段名。生成代码里按 Luban 命名风格转成 PascalCase（`default_icon` → `DefaultIcon`）。
- `##type`：字段类型，见下表。
- `##group`：该字段对哪些导出目标可见；留空表示所有目标。
- `##`：注释行，会写进生成代码的 `<summary>`。
- 从第 6 行起是数据，空行会被跳过。

### 常用字段类型

| 写法 | 生成类型 | 说明 |
| :--- | :--- | :--- |
| `int` / `float` / `bool` / `string` | 同名 C# 类型 | 基础类型 |
| `text` | `string` + `string` Key | 多语言文案，见下文 |
| `item.EItemType` | 枚举 | `full_name` 里定义的枚举 |
| `(list#sep=;),string` | `List<string>` | 用 `;` 分隔的列表 |
| `int#ref=Tables.TbItem` | `int` + `Tables.ItemConfig` 引用 | 外键，见下文 |
| `color` | `UnityEngine.Color` | 由 `builtin.xml` 的 mapper 映射，见 [Luban 生成与脚本](generate.md) |

### `text`：多语言文案

`##type` 写 `text` 的字段会生成两个成员：

```csharp
// 以 Item_item.xlsx 的 name / desc 为例（生成代码）
public readonly string NameKey;          // Excel 里的原始 key
public string Name { get; internal set; } // 运行时按语言填充后的值
```

- `NameKey` = Excel 单元格里的 key（例如 `item_1_name`）。
- `Name` 在 `IConfigService.SetLanguageAsync` 时被替换成对应语言的文案；语言包缺失时回退为 key 本身。
- 具体字节格式与运行时回填见 [生成物与运行时衔接](outputs.md)。

### `#ref`：外键引用

`Shop_shop.xlsx` 的 `itemId` 写作 `int#ref=Tables.TbItem`，生成：

```csharp
public readonly int ItemId;
public Tables.ItemConfig ItemId_Ref;   // ResolveRef 阶段填充

internal void ResolveRef(ConfigService tables)
{
    ItemId_Ref = tables.TbItem.GetOrDefault(ItemId);
}
```

- 引用在 `ConfigService.Initialize` 的最后一步 `ResolveRef` 统一解析。
- 引用的目标必须是已声明的表（`Tables.TbItem` 对应 `full_name`）。
- 找不到 key 时引用为 `null`，生成阶段不会因为数据缺失而失败。

### 纵向单例表

`Global_global.xlsx` 用的是纵向布局：字段在行上，值在右侧列。

```text
##var#column | ##type | ##comment | ##
volume       | float  | 默认音量   | 1
default_icon | string | 默认 Icon  | 2
commonColorTabs | (list#sep=;),color | | #FFD700;#FFB6C1
```

配合 `__tables__.xlsx` 里 `mode=singleton`，生成 `GlobalConfig` 单条记录，表对象用 `Data` 访问：

```csharp
float volume = configService.Global.Data.Volume;
```

## 枚举：`__enums__.xlsx`

一行一个枚举项，枚举本身的信息只写在第一行：

| `full_name` | `flags` | `unique` | `*items.name` | `*items.value` | `*items.comment` |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `item.EQuality` | `False` | `True` | `WHITE` | `1` | 最差品质 |
|  |  |  | `BLUE` | `2` | 蓝色的 |

- `full_name` 用 `模块.枚举名`，`item.EQuality` 生成到命名空间 `Game.Config.item`。
- `flags=True` 时按位枚举生成，可用 `A|B` 组合。
- 枚举值必须是显式数字，新增项不要改旧值。

## bean：`__beans__.xlsx`

用于把一组字段复合成结构体（例如坐标、区间）。`full_name` 写在第一行，`*fields` 区一行一个字段：

| `full_name` | `*fields.name` | `*fields.type` |
| :--- | :--- | :--- |
| `vec2` | `x` | `float` |
|  | `y` | `float` |

生成的 `vec2` 在 `Game.Config` 命名空间下，再通过 `builtin.xml` 的 mapper 映射成 `UnityEngine.Vector2`。新增结构体时，如果要在表里直接用 Unity 类型，需要同时补 mapper，见 [Luban 生成与脚本](generate.md)。

## 命名与分组规则

| Excel 写法 | 生成结果 |
| :--- | :--- |
| `__tables__.full_name = Tables.TbItem` | `Game.Config.Tables.TbItem`，`IConfigService.TbItem` |
| `__tables__.value_type = ItemConfig` | `Game.Config.Tables.ItemConfig` |
| `__enums__.full_name = item.EQuality` | `Game.Config.item.EQuality` |
| `__beans__.full_name = vec2` | `Game.Config.vec2` |
| 字段 `default_icon` | `DefaultIcon` |
| 字段 `name` + `text` | `NameKey` + `Name` |

命名空间前缀来自 `luban.conf` 的 `topModule`（`Game.Config`），不是手写配置。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 新增字段 | 在数据表表头加一列：`##var` / `##type` / `##group` / `##` 都要写 |
| 新增表 | `__tables__.xlsx` 加行 → 新建 `Tables/<模块>_<名>.xlsx` → `gen-all` |
| 字段只给客户端 | `##group` 写 `c` |
| 字段不进代码注释 | `##` 行留空 |
| 列表分隔符换别的 | `(list#sep=|),int`，分隔符写在 `#sep=` 后 |

## 容易踩的坑

1. **表头缺行**：`##type` 行缺失时 Luban 会把字段当 `string` 或直接报错；`##var` 与 `##type` 的列必须严格对齐。
2. **`full_name` 与文件名不一致**：`input` 指向的文件名不参与命名，改文件名不用改 `full_name`，但改 `full_name` 会改生成类名和属性名。
3. **`text` 字段直接写文案**：`text` 列要写 key（如 `item_1_name`），文案放在 `Excels/Localization` 的 `Texts` 表里；直接写中文会导致切语言时不翻译。
4. **`#ref` 写成短名**：必须写完整表名 `Tables.TbItem`，写 `TbItem` 会找不到表。
5. **改枚举值**：已导出的枚举值被改动会让旧存档/旧数据解析错位，新增项追加在末尾。
6. **`singleton` 表用 `DataList`**：单例表只有 `Data`，没有 `DataList`，遍历写法会编译失败。

## API 速查

| 生成成员 | 出现位置 | 说明 |
| :--- | :--- | :--- |
| `IConfigService.<表名>` | 生成 `ConfigService.cs` | 每张表一个属性 |
| `<表>.DataMap` / `DataList` / `Data` | 生成表类 | `map` 表有 `DataMap`/`DataList`，`singleton` 表只有 `Data` |
| `<表>.GetOrDefault(key)` / `Get(key)` | 生成表类 | `map` 表的按主键查询 |
| `<表>.GetBy<索引字段>(key)` | 生成表类 | 声明了 `index` 时生成 |
| `<字段>_Ref` | 生成 bean | `#ref` 外键解析结果 |
| `<字段>Key` / `<字段>` | 生成 bean | `text` 字段的 key 与当前语言值 |

下一步：配置输出目录与运行时装配见 [生成物与运行时衔接](outputs.md)。
