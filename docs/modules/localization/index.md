# Localization 本地化

`com.alicizax.unity.localization`（1.0.2）用一份 `LocalizationSettings` 描述项目支持的语言，用按语言导入的 `.loclang` 运行资产承载词库，由 `LocalizationKit` 在启动期与热更期分层加载。它只负责「Key → 当前语言文本」和 TMP 绑定，不含机器翻译、字体/字库切换，也不生成配置表。

## 适用场景

- 启动和更新界面在热更资源就绪前就要显示文案：启动期加载 Resources 层词库。
- 进入热更逻辑后追加常规 UI、日志文案：运行期加载 AssetBundle 层词库。
- 游戏内提供语言下拉，切换后所有已加载文案立即刷新。
- 文案带占位符（`{0}`、`{1}`），由代码以强类型常量访问器调用。

不属于这个包的能力：

| 需求 | 归属 |
| --- | --- |
| 机器翻译、在线翻译、翻译平台同步 | 项目外部流程 |
| 按语言切换字体、字库、图集 | 项目资源方案 / TMP |
| 配置表结构、Excel 到 Luban 生成物 | [Config 配置管线](../config/index.md) |
| 按语言加载不同 UI 预制体 | [Resource 资源模块](../resource/index.md) |

## 使用前提

- Unity 2022.3 及以上；包已通过 `Client/Packages/manifest.json` 以本地路径引入。
- 设置资产固定在 `Assets/Resources/Localization/Settings.asset`（`LocalizationSettings.AssetPath`），启动期用 `Resources.Load<LocalizationSettings>("Localization/Settings")` 读取。
- 词库编辑源资产 `LocalizationTableCollection` 与其语言子资产只存在于编辑器；运行期读的是 `.loclang` 导入出的独立单语言资产。
- 运行期 AssetBundle 层依赖框架资源服务：`GameApp.Resource` 可用、YooAsset Package 已初始化。
- 项目安装了 `com.unity.ugui 2.0.0` 与 `com.unity.textmeshpro 3.0.9`，`TEXTMESHPRO_SUPPORT` 因此生效（UGUI 2.0+ 或 TextMeshPro 任一存在即可）；两者都没有时 `LocalizedText` 不参与编译。
- 代码要引用 `AlicizaX.Localization.Runtime`；用 `LocalizedText` 时还要引用 `AlicizaX.Localization.UI`（该程序集 `autoReferenced` 为 `false`）。

源码位置（按修改目标查）：

| 内容 | 路径 |
| --- | --- |
| 运行时门面与词库合并 | `Client/Packages/com.alicizax.unity.localization/Runtime/Core` |
| 设置资产与单语言词库资产 | `Client/Packages/com.alicizax.unity.localization/Runtime/Catalog` |
| 词库来源 | `Client/Packages/com.alicizax.unity.localization/Runtime/Sources` |
| TMP 绑定 | `Client/Packages/com.alicizax.unity.localization/Runtime/UI` |
| 编辑窗口、CSV、代码生成、构建校验 | `Client/Packages/com.alicizax.unity.localization/Editor` |

## 最短接入

1. 打开语言设置：菜单 `AlicizaX/Localization/Settings`，或双击 `Settings.asset`。添加语言、指定默认语言，见[词库配置与生成](config.md)。

2. 创建词库：`AlicizaX/Localization/Create Table`（或 Project 右键 `Create/AlicizaX/Localization/Table`），填 `TableId` 与运行入口目录，然后在窗口里新增 Key、逐语言填译文。

3. 把 Key 交给翻译：窗口 `导出 CSV` → 回收后 `导入 CSV`，先看差异预览再确认；需要强类型调用时填 `常量代码输出`，窗口会生成访问器（本工程产出 `LocalizationKey.cs` 这类文件）。

4. 启动期初始化并加载第一层词库（真实入口 `Client/Assets/Scripts/Startup/ProcedureEntry.cs`）：

```csharp
using System;
using AlicizaX.Localization.Runtime;
using UnityEngine;

LocalizationKit.Initialize(Resources.Load<LocalizationSettings>("Localization/Settings"));
if (!await LocalizationKit.AddTableAsync("StartupTable", new ResourcesTableSource("Localization/StartupTable_{locale}")))
    throw new InvalidOperationException("Startup localization could not be loaded.");
```

5. 热更资源可用后追加第二层词库，并把语言切换交给配置服务（真实入口 `Client/Assets/Scripts/Hotfix/GameLogic/HotfixEntry.cs`）：

```csharp
if (!await LocalizationKit.AddTableAsync("CommonTable", new AssetBundleTableSource("CommonTable_{locale}")))
    throw new InvalidOperationException("Common localization could not be loaded.");

ConfigService = AppServices.Register<IConfigService>(new ConfigService());
await ConfigService.Initialize(new ConfigBytesLoader());
await ConfigService.SetLanguageAsync(LocalizationKit.CurrentLanguage);
ConfigService.SetLanguageBinding(LocalizationKit.RegisterLanguageLoader(ConfigService.SetLanguageAsync));
```

到这里 UI 有两条取文案路线：代码里用生成的常量访问器（`LocalizationKey.UI.HOME_BUTTONSHOP`）或直接 `LocalizationKit.Get(key)`；TMP 文本挂 `LocalizedText` 并在 Inspector 选 Key，见 [TMP 绑定与编辑器预览](tmp.md)。

## 核心用法

### 三层结构

```text
LocalizationSettings（Assets/Resources/Localization/Settings.asset）
  Languages: Id / LocaleCode / DisplayName
  DefaultLocale / UseSystemLanguage
        │ 决定支持语言与启动语言
        ▼
LocalizationKit.Initialize(settings)
        │
        ├─ AddTableAsync("StartupTable", ResourcesTableSource("Localization/StartupTable_{locale}"))
        └─ AddTableAsync("CommonTable",  AssetBundleTableSource("CommonTable_{locale}"))
                │ 每个来源按当前语言加载一份 .loclang（LocalizationLanguageTable）
                ▼
        Dictionary<string,string>（后加入的 Table 覆盖同名 Key）
                ▼
LocalizationKit.Get / Format ──▶ LocalizedText、常量访问器、UI 代码
```

- 设置资产：语言列表、默认语言、是否跟随系统语言；`Validate()` 拒绝空语言、重复 `Id`、重复 `LocaleCode`、默认语言不在列表。
- `.loclang`：只有 `TableGuid` 和 `Locale` 两个字段的 JSON 入口，由 `LocalizationLanguageImporter` 导入成独立的 `LocalizationLanguageTable`（`TableId`、`Locale`、`Entries`）。它不引用父 Table，也不持有其他语言。
- 分层词库：启动期从 Resources 加载，热更后从 AssetBundle 追加；同一个 Key 以最后加入的 Table 为准，缺失 Key 返回 Key 本身，不做跨语言回退。

### 主要类型

| 类型 | 职责 |
| --- | --- |
| `LocalizationKit` | 静态门面：初始化、加载/重载词库、切语言、取文案、格式化、变化事件 |
| `LocalizationProvider` | 运行时实例：当前语言、已加载 Table 的顺序与合并结果、加载中互斥 |
| `LocalizationSettings` | 语言列表、默认语言、启动语言选择 |
| `LocalizationLanguageTable` | 单语言词库资产，`Read()` 产出 Key 字典 |
| `ResourcesTableSource` / `AssetBundleTableSource` | 两种内置词库来源，模板必须含 `{locale}` |
| `LocalizedText` | TMP 文本绑定，语言切换时自动刷新 |
| `LanguageTypes` | 语言列表与索引互转，供下拉框使用 |

运行期 API、切换语义和失败边界见[运行时加载与切换](runtime.md)。

### 与 Config/Luban 的衔接

项目里有两条独立的本地化链路，只共享「语言 Id」：

- Localization 链路：编辑源 Table → `.loclang` → `LocalizationKit`，管 UI 与日志文案。
- Config 链路：`Config/Excels/Localization/*.xlsx` → `gen-all` → `Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes`，管配置表里的文案列（如 `TbItem.Name` / `Desc`）。

衔接点只有一个：`Config/config.ini` 的 `[localization] languages` 里的语言名，必须同时是 `LocalizationSettings` 的语言 `Id` 和 `.loclang` / `Game_{locale}.bytes` 的文件名后缀；运行时把 `ConfigService.SetLanguageAsync` 注册成语言加载器，切换语言时配置数据与词库一起换。生成细节、Excel 约定与排查见 [Config 概览](../config/index.md)、[表结构与 Excel 约定](../config/tables.md)、[Luban 生成与脚本](../config/generate.md)、[生成物与运行时衔接](../config/outputs.md)、[常见错误排查](../config/troubleshooting.md)，本文不重复。

## 四页导航

| 页面 | 内容 |
| --- | --- |
| [词库配置与生成](config.md) | `LocalizationSettings` 字段、Localization 窗口、CSV 约定、代码生成、构建校验、`Game_{locale}.bytes` 生成 |
| [运行时加载与切换](runtime.md) | `LocalizationKit` API、来源路径模板、切语言顺序与失败处理、Config 联动、重入注意 |
| [TMP 绑定与编辑器预览](tmp.md) | `LocalizedText` 字段与刷新、Key 选择器、编辑器预览词库来源、`TEXTMESHPRO_SUPPORT` |
| 本页 | 架构、能力边界、5 步接入、与 Config 的分工 |

## 常见配置

本工程的真实取值：

| 项目 | 值 |
| --- | --- |
| 设置资产 | `Client/Assets/Resources/Localization/Settings.asset` |
| 语言 | `ChineseSimplified`(`zh-Hans`)、`English`(`en`)、`Japanese`(`ja`) |
| 默认语言 / 跟随系统 | `ChineseSimplified` / 关闭 |
| 启动词库 | `Client/Assets/Editor/Localization/StartupTable.asset`，入口目录 `Assets/Resources/Localization`，地址 `StartupTable_{locale}` |
| 热更词库 | `Client/Assets/Editor/Localization/CommonTable.asset`，入口目录 `Assets/Bundles/Configs/sciptableObject`，地址 `CommonTable_{locale}` |
| 生成的常量 | `Client/Assets/Scripts/Startup/UpdateLocalizationKey.cs`、`Client/Assets/Scripts/Hotfix/GameLogic/LocalizationKey.cs` |
| 玩家语言记录 | `PlayerPrefs` 键 `AlicizaX.Localization.Language` |
| 编辑器预览语言 | `EditorPrefs` 键 `AlicizaX.Localization.Editor.Language` |

词库编辑源 Assets 放在 `Client/Assets/Editor/Localization` 这类仅在编辑器参与编译/收集的目录，避免被误打进运行时包。

## 容易踩的坑

1. `Initialize` 只能成功一次，重复调用抛 `InvalidOperationException`；静态状态只在 `SubsystemRegistration` 时清理，业务代码不要假设可以反复初始化。
2. 父 Table（`LocalizationTableCollection`）不能放进 `Resources`，也不能被场景、Prefab 或其他运行资产引用——`LocalizationBuildGuard` 会在构建和 YooAsset 收集时报错。
3. 缺失 Key、未加载词库、未初始化时 `Get` 都返回 Key 本身，不会回退到其他语言；看到界面上出现英文 Key 时先查词库是否加载。
4. `.loclang` 路径由 Table 的 `AssetDirectory` + `AddressTemplate` 推导，改这两项会移动入口文件；手改 `.loclang` 里的 `TableGuid` 会破坏导入。
5. `LocalizedText` 只在 `TEXTMESHPRO_SUPPORT` 生效时存在；项目降级 UGUI/TMP 后该组件会消失，场景里留下 Missing Script。
6. 语言 `Id` 同时决定 `.loclang` 文件名、`Game_{locale}.bytes` 后缀和 `config.ini` 里的语言列，改名要三处同步。
