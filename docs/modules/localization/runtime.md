# 运行时加载与切换

`LocalizationKit` 是唯一的运行时入口：它按当前语言把多个 `ILocalizationTableSource` 加载成词库、合并成一张 Key 表，并在切换语言时按注册顺序整体重载、通知订阅者。这一页给出真实 API、来源路径模板、失败返回值，以及与 `ConfigService` 的联动顺序。

## 适用场景

- 启动期只有 Resources 可用时先加载更新界面词库。
- 热更资源就绪后追加常规 UI、日志词库，或用 `ReloadTableAsync` 替换 DLC/热更数据。
- 游戏内切换语言，并让 TMP、窗口、配置表文案一起刷新。
- 自己实现词库来源（远端、加密包、其他资源系统）。

## 使用前提

- `LocalizationSettings` 资产可加载，语言列表非空且默认语言合法。
- 启动词库已放在 `Resources` 下，或热更词库已打进 YooAsset Package。
- 引用 `AlicizaX.Localization.Runtime`；`AssetBundleTableSource` 还需要框架资源服务 `GameApp.Resource` 与已初始化的 YooAsset Package。
- 取文案的代码在初始化完成之后运行；UI 代码通常还引用 `AlicizaX.Localization.UI`（`LocalizedText`）。

## 最短接入

1. 启动期初始化（`ProcedureEntry.cs` 的真实写法）：

```csharp
using AlicizaX.Localization.Runtime;
using UnityEngine;

LocalizationKit.Initialize(Resources.Load<LocalizationSettings>("Localization/Settings"));
```

2. 加载第一层词库，并立刻检查返回值（以下片段都在异步方法中）：

```csharp
if (!await LocalizationKit.AddTableAsync("StartupTable", new ResourcesTableSource("Localization/StartupTable_{locale}")))
    throw new InvalidOperationException("Startup localization could not be loaded.");
```

3. 热更资源就绪后追加第二层词库：

```csharp
if (!await LocalizationKit.AddTableAsync("CommonTable", new AssetBundleTableSource("CommonTable_{locale}")))
    throw new InvalidOperationException("Common localization could not be loaded.");
```

4. 让配置表跟着词库走：先按当前语言刷新一次配置，再把配置服务注册成语言加载器（`HotfixEntry.cs` 的真实顺序）：

```csharp
ConfigService = AppServices.Register<IConfigService>(new ConfigService());
await ConfigService.Initialize(new ConfigBytesLoader());
await ConfigService.SetLanguageAsync(LocalizationKit.CurrentLanguage);
ConfigService.SetLanguageBinding(LocalizationKit.RegisterLanguageLoader(ConfigService.SetLanguageAsync));
```

5. 取文案：优先用生成的常量访问器，临时文案用 `Get` / `Format`：

```csharp
string title = LocalizationKey.UI.SHOP_TITLE;                     // 生成的访问器
string price = LocalizationKey.UI.SHOP_CURRENCY(priceText);       // 带参数的访问器
string raw = LocalizationKit.Get("UI.Home.NoticeTitle");          // 直接取
string msg = LocalizationKit.Format("UI.Buy.TotalPrice", total);  // 直接格式化
```

访问器类名取 `CodeOutput` 的文件名（本工程是 `LocalizationKey` 与 `UpdateLocalizationKey`），生成规则见[词库配置与生成](config.md)。

## 核心用法

### 初始化与语言选择

```csharp
LocalizationKit.Initialize(LocalizationSettings settings, string initialLanguage = null);
```

- 重复调用抛 `InvalidOperationException`（"Localization is already initialized."），`settings` 为 `null` 抛 `ArgumentNullException`，`Validate()` 失败抛 `FormatException`。
- 启动语言优先级：有效 `initialLanguage` → 有效玩家记录（`PlayerPrefs` 键 `AlicizaX.Localization.Language`）→ 系统语言（`UseSystemLanguage` 开启时，从 `CultureInfo.CurrentUICulture` 开始沿父级匹配）→ `DefaultLocale`。
- 编辑器中若 `initialLanguage` 为 `null`，且 `EditorPrefs` 的预览语言受支持，则用预览语言覆盖启动选择。
- `Initialize` 本身不发布 `Changed`，也不加载词库；词库要靠 `AddTableAsync` 显式加入。

初始化前后的静态状态：

| 成员 | 初始化前 | 初始化后 |
| --- | --- | --- |
| `CurrentLanguage` | `string.Empty` | 当前语言 Id |
| `Revision` | `0` | 每次成功加载/重载/切换递增 |
| `SupportedLanguages` | 空列表 | 设置里的语言 Id 列表 |
| `LoadedTables` | 空列表 | 已加载的单语言词库，按加入顺序 |
| `Settings` | `null` | 传入的设置资产 |

### 分层加载与覆盖

```csharp
UniTask<bool> AddTableAsync(string tableId, ILocalizationTableSource source);
```

- 未初始化返回 `false`；加载中再次调用返回 `false`（不排队）。
- `tableId` 为空、重复注册、`source` 为 `null`，或来源返回的资产 `TableId`/`Locale` 与请求不一致，都会抛异常。
- 成功后按加入顺序重新合并：后加入的 Table 覆盖同名 Key；`null` 值不写入；`Revision` 递增，并以 `persistLanguage: false` 发布 `Changed`。
- 加载期间如果 `LocalizationKit` 被重置（域重载、退出播放），本次加载返回 `false`，临时资产被释放。

热替换：

```csharp
await LocalizationKit.ReloadTableAsync("CommonTable");                              // 用原来的来源重载
await LocalizationKit.ReloadTableAsync("CommonTable", new AssetBundleTableSource("CommonTableV2_{locale}"));
```

`ReloadTableAsync` 保持该 Table 原来的优先级位置：新数据替换旧的覆盖值，被删掉的 Key 会回落到更早的 Table；替换来源会被保留，后续切语言用新来源加载。表未注册抛 `ArgumentException`；新来源加载失败时保留旧数据和旧来源。

### 切换语言

```csharp
bool changed = await LocalizationKit.SetLanguageAsync("English");
```

一次成功切换的顺序是固定的：

1. 校验语言是否受支持（不支持抛 `ArgumentException`）；与当前语言相同直接返回 `true`，不重载、不发事件。
2. 按注册顺序为**每个**已注册 Table 加载目标语言，结果先暂存（不会让 `Get` 看到半套数据）。
3. 合并暂存结果，在提交前调用已注册的语言加载器（例如 `ConfigService.SetLanguageAsync`）。
4. 提交：替换全部 Table、更新 `CurrentLanguage`、`Revision` 递增、释放旧语言资产。
5. 发布一次 `Changed`，回调参数是新语言；同时写入 `PlayerPrefs`，编辑器下还写入 `EditorPrefs` 预览语言。

失败与并发：

| 场景 | 行为 |
| --- | --- |
| 未初始化 | 返回 `false` |
| 正在加载/正在切换 | 返回 `false` |
| 不支持的语言 | 抛 `ArgumentException` |
| 词库或语言加载器抛异常 | 异常向上抛，旧语言、旧词库、旧配置保持不变，不发 `Changed` |
| 订阅者回调抛异常 | 被捕获并 `Debug.LogException`，不影响其他订阅者 |

### 取文案与格式化

| API | 行为 |
| --- | --- |
| `Get(string key)` | Key 缺失、值为 `null` 或未初始化时返回 Key 本身 |
| `Format(string key, params object[] args)` | 未初始化返回 Key；`args` 为空返回原文；最多 16 个参数，超过抛 `FormatException` |
| `Format<T0>` / `Format<T0,T1>` / `Format<T0,T1,T2>` | 泛型重载，走 `ZString.Format`，未初始化返回 Key |
| `LocalizationFormatter.Format(format, args)` | 直接格式化一段文本，同样支持到 16 个参数 |

缺失 Key 不会跨语言回退，也不会回退到默认语言：当前语言没加载到就显示 Key。

### 订阅变化

```csharp
private IDisposable _subscription;

private void OnEnable() => _subscription = LocalizationKit.SubscribeChanged(OnLanguageChanged);
private void OnDisable() { _subscription?.Dispose(); _subscription = null; }
private void OnLanguageChanged(string locale) { /* 重新取文案 */ }
```

`Changed` 是 `Action<string>` 事件，`SubscribeChanged` 返回可释放订阅；回调为 `null` 抛 `ArgumentNullException`。追加词库、重载词库、切换语言都会触发，参数是当前语言。窗口类可以像 `UIBuyAlertWindow` 那样在 `OnInitialize` / `OnDestroy` 里直接 `+=` / `-=`，需要一次性释放时用 `SubscribeChanged`。

### 词库来源

内置两种来源，模板字符串都必须含 `{locale}`（构造时校验，否则抛 `ArgumentException`）：

| 来源 | 地址模板 | 加载方式 | 缺失时 |
| --- | --- | --- | --- |
| `ResourcesTableSource` | `Localization/StartupTable_{locale}` | `Resources.LoadAsync<LocalizationLanguageTable>(地址)` | 抛 `InvalidOperationException("Missing localization table: …")` |
| `AssetBundleTableSource` | `CommonTable_{locale}`，可带包名 | `GameApp.Resource.LoadLeaseAsync<LocalizationLanguageTable>(地址, packageName: 包名)` | 释放租约后抛 `InvalidOperationException("Missing localization table: 包名/地址")` |

- Resources 地址相对 `Resources` 目录、省略扩展名；同一资产被多次加载时按引用计数释放，最后一个 `Dispose` 才 `Resources.UnloadAsset`。
- AssetBundle 地址就是打包收集给出的资源地址（本工程等于 `.loclang` 的文件名），包名为 `null` 时使用资源服务的默认包（`DefaultPackageName`）；来源持有租约，释放时归还。
- 自定义来源只需实现接口，并保证返回资产的 `TableId`/`Locale` 与注册参数一致、`Dispose` 能真正释放资源：

```csharp
public interface ILocalizationTableSource
{
    UniTask<LoadedLocalizationTable> LoadAsync(string locale);
}
```

### 与 ConfigService 联动

`ConfigService.SetLanguageAsync(locale)` 通过 `IConfigBytesLoader.LoadLocalizationAsync` 读取 `Game_{locale}.bytes`，把配置表里的文案列（本工程是 `TbItem.Name` / `Desc`）刷成新语言。联动方式是把方法本身注册成语言加载器：

```csharp
ConfigService.SetLanguageBinding(LocalizationKit.RegisterLanguageLoader(ConfigService.SetLanguageAsync));
```

- 加载器在 `SetLanguageAsync` 提交前执行，所以 `Changed` 回调里配置数据和词库都已是新语言；集成测试正是这样断言 `TbItem.DataList[0].Name` 与 `LoadedTables.Locale`。
- 必须在注册前手动调一次 `SetLanguageAsync(LocalizationKit.CurrentLanguage)`：初始化与追加词库都不触发加载器，只有「切换语言」才触发。
- `SetLanguageBinding` 会先释放上一次的绑定；`ConfigService` 销毁时释放当前绑定。加载器同一时刻只能注册一个，重复注册抛 `InvalidOperationException`。
- 配置侧的数据格式与生成链见[生成物与运行时衔接](../config/outputs.md)，`Game_{locale}.bytes` 的来源见[Luban 生成与脚本](../config/generate.md)。

### 退出与重入

- `LocalizationKit` 没有公开的 `Shutdown`：静态状态在 `RuntimeInitializeLoadType.SubsystemRegistration` 时重置（释放 Provider、清空 `Settings` 与 `Changed`），所以只能在「重新进入运行」后再次初始化。
- 不能在业务里反复 `Initialize`；需要换设置就重启运行流程。
- 重置期间仍在进行的 `AddTableAsync` / `SetLanguageAsync` 会返回 `false`，已经加载到的临时词库会被释放，不会写进新会话。
- 订阅与语言加载器都是 `IDisposable`：窗口销毁时退订，配置服务销毁时释放绑定，避免静态事件残留旧对象。
- 切语言过程中失败不需要手动回滚，Provider 只在成功后提交；UI 侧根据返回值或 `CurrentLanguage` 复位下拉框即可，`UIHomeWindow` 就是这么做的。

## 常见配置

| 项目 | 值 |
| --- | --- |
| 启动词库地址 | `Localization/StartupTable_{locale}`（Resources） |
| 热更词库地址 | `CommonTable_{locale}`，入口目录 `Assets/Bundles/Configs/sciptableObject` |
| 默认 Package | `AssetBundleTableSource` 不传包名时使用资源服务的默认包 |
| 玩家语言记录 | `PlayerPrefs`：`AlicizaX.Localization.Language` |
| 编辑器预览语言 | `EditorPrefs`：`AlicizaX.Localization.Editor.Language` |
| 语言索引互转 | `LanguageTypes.StringToIndex(locale)`（缺失返回 `-1`）、`LanguageTypes.IndexToString(index)` |
| 配置语言文件 | `Client/Assets/Bundles/Configs/bytes/localization/Game_{locale}.bytes` |

## 容易踩的坑

1. 地址与 `.loclang` 文件名（或 Resources 路径）不一致会直接抛 `Missing localization table`；`{locale}` 替换用的是语言 `Id`，不是 `LocaleCode`。
2. `AddTableAsync` / `SetLanguageAsync` 不排队：加载中再调用返回 `false`。并发触发两次切换时第二次直接失败，界面会停在中途，调用方要按返回值回滚 UI。
3. 初始化时传非法 `initialLanguage` 不会报错，会静默回退到记录/默认语言；而 `SetLanguageAsync` 传不支持的语言会抛 `ArgumentException`。判断实际语言请读 `CurrentLanguage`。
4. 语言加载器同一时刻只能有一个，注册前必须已初始化且不在加载中；忘记 `Dispose` 后再注册会抛 `InvalidOperationException`。
5. 切换失败不会发 `Changed`，旧语言与旧数据都保留；不要假设失败后 UI 已刷新。
6. `AssetBundleTableSource` 依赖资源服务与 Package 已初始化；在 YooAsset 就绪前调用会失败（抛异常或拿到无效租约），启动期应先用 `ResourcesTableSource`。

## API 速查

| 成员 | 签名 | 说明 |
| --- | --- | --- |
| `Initialize` | `void Initialize(LocalizationSettings settings, string initialLanguage = null)` | 只允许一次；选择启动语言并创建 Provider |
| `AddTableAsync` | `UniTask<bool> AddTableAsync(string tableId, ILocalizationTableSource source)` | 追加一层词库，后加入优先 |
| `ReloadTableAsync` | `UniTask<bool> ReloadTableAsync(string tableId)`、`UniTask<bool> ReloadTableAsync(string tableId, ILocalizationTableSource replacementSource)` | 原位替换一层词库 |
| `SetLanguageAsync` | `UniTask<bool> SetLanguageAsync(string locale)` | 重载全部词库、调用加载器、发布事件 |
| `Get` | `string Get(string key)` | 取当前语言文案，缺失返回 Key |
| `Format` | `string Format(string key, params object[] arguments)` 及 1–3 个泛型重载 | 取文案并格式化 |
| `RegisterLanguageLoader` | `IDisposable RegisterLanguageLoader(Func<string, UniTask> loader)` | 注册唯一的语言加载器 |
| `SubscribeChanged` | `IDisposable SubscribeChanged(Action<string> callback)` | 订阅语言变化 |
| `Changed` | `event Action<string>` | 直接事件订阅方式 |
| `CurrentLanguage` / `Revision` / `SupportedLanguages` / `LoadedTables` / `Settings` | 只读属性 | 当前状态快照 |
