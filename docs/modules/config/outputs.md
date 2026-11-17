# 生成物与运行时衔接

`gen-all` 的产物分三处：生成代码、普通表数据、多语言数据。这一页说明它们长什么样、怎么被 YooAsset 打进包、以及运行时如何装配 `IConfigService`。

## 适用场景

- 第一次接配置服务，不知道 `IConfigService` 从哪来、谁来 `new`。
- 想知道生成代码里哪些能改、哪些一改就被覆盖。
- 排查“表生成了但运行时读不到”“切换语言后配置文案没变”。

## 输出目录

| 产物 | 目录 | 来源 |
| :--- | :--- | :--- |
| 生成代码 | `Client/Assets/Scripts/Hotfix/GameProto/Config/Generate` | `config.ini` 的 `code_out` |
| 普通表数据 | `Client/Assets/Bundles/Configs/bytes/ordinary` | `config.ini` 的 `table_data_out` |
| 多语言数据 | `Client/Assets/Bundles/Configs/bytes/localization` | `table_data_out` 同级的 `localization/`，由插件推导 |

生成代码目录：

```text
Config/Generate
├─ ConfigService.cs        # IConfigBytesLoader / IConfigService / ConfigService
├─ AudioType.cs            # builtin.xml 里的枚举
├─ color.cs  vec2.cs  vec3.cs  vec4.cs
├─ EShopCategory.cs
├─ item
│  ├─ EItemType.cs
│  └─ EQuality.cs
└─ Tables
   ├─ TbAudio.cs  TbItem.cs  Shop.cs  Global.cs     # 表对象
   └─ AudioConfig.cs  ItemConfig.cs  ShopConfig.cs  GlobalConfig.cs  # 记录
```

普通表数据：

```text
Bundles/Configs/bytes/ordinary
├─ tables_tbaudio.bytes
├─ tables_tbitem.bytes
├─ tables_shop.bytes
└─ tables_global.bytes
```

多语言数据：

```text
Bundles/Configs/bytes/localization
├─ Game_ChineseSimplified.bytes
├─ Game_English.bytes
└─ Game_Japanese.bytes
```

## 哪些是生成的，哪些是手写的

| 路径 | 归属 | 说明 |
| :--- | :--- | :--- |
| `Config/Generate/**` | 生成 | 整体覆盖，**不要手改** |
| `Config/ConfigLocalization.cs` | 手写 | 解析多语言字节并回填 `text` 字段 |
| `Config/Extension/ExternalTypeUtil.cs` | 手写 | `builtin.xml` mapper 引用的转换函数，必须在全局命名空间 |
| `Config/GameProto.asmdef` | 手写 | 程序集定义，`autoReferenced: false` |

要扩展生成类型，用两种方式：

1. **partial 钩子**：生成表类里留了 `partial void PostInit();` / `partial void PostResolveRef();`，在生成目录外写 `partial class TbItem` 实现它们。
2. **partial 类**：生成的 bean 与表都是 `partial`，可以在 `Config/Extension/` 里加计算属性或辅助方法。

`ConfigService` 本身**不是** partial，不要试图用 partial 扩展它；需要额外能力时组合一个服务，或在 `IConfigService` 外层封装。

## 生成代码的结构

`ConfigService.cs` 里同时生成三个类型：

```csharp
namespace Game.Config
{
    public interface IConfigBytesLoader
    {
        UniTask<ByteBuf> LoadCommonAsync(string file);
        UniTask<ByteBuf> LoadLocalizationAsync(string locale);
    }

    public interface IConfigService : IService
    {
        Tables.TbAudio TbAudio { get; }
        Tables.Global Global { get; }
        Tables.TbItem TbItem { get; }
        Tables.Shop Shop { get; }
        UniTask Initialize(IConfigBytesLoader loader);
        UniTask SetLanguageAsync(string locale);
        void SetLanguageBinding(IDisposable binding);
    }

    public class ConfigService : ServiceBase, IConfigService { ... }
}
```

要点：

- `ConfigService` 继承框架的 `ServiceBase`，所以注册后受服务容器管理，`OnDispose` 会释放语言订阅（见 [Service 服务容器](../service/index.md)）。
- 属性名来自 `__tables__.xlsx` 的 `full_name` 末段。
- 表对象延迟加载：`Initialize` 里先 `new` 每张表并并行 `LoadAsync`，再逐个 `ResolveRef`。
- `Initialize` 是幂等的：第二次调用直接返回，不会重新加载。
- 表类提供 `GetOrDefault` / `Get` / `DataList` / `DataMap`；单例表只有 `Data`。

## YooAsset 如何收集配置

`Client/Assets/YooAsset/AssetBundleCollectorSetting.asset` 里的 `Configs` 组：

| CollectPath | PackRule | AddressRule |
| :--- | :--- | :--- |
| `Assets/Bundles/Configs/bytes/ordinary` | `PackCollector` | `AddressByFileName` |
| `Assets/Bundles/Configs/bytes/localization` | `PackSeparately` | `AddressByFileName` |

`AddressByFileName` 表示**资源地址就是去掉扩展名的文件名**，所以代码里写：

```csharp
await loader.LoadCommonAsync("tables_tbitem");      // tables_tbitem.bytes
await loader.LoadLocalizationAsync("English");      // Game_English.bytes 的英文部分
```

多语言包用 `PackSeparately`，每个语言的 `Game_*.bytes` 是独立包，切语言时只需加载对应包。

## 运行时装配

真实装配代码在 `Client/Assets/Scripts/Hotfix/GameLogic/HotfixEntry.cs`：

```csharp
using AlicizaX;
using AlicizaX.Localization.Runtime;
using Game.Config;
using Luban;

ConfigService = AppServices.Register<IConfigService>(new ConfigService());
await ConfigService.Initialize(new ConfigBytesLoader());
await ConfigService.SetLanguageAsync(LocalizationKit.CurrentLanguage);
ConfigService.SetLanguageBinding(
    LocalizationKit.RegisterLanguageLoader(ConfigService.SetLanguageAsync));
```

`IConfigBytesLoader` 由业务实现，把 YooAsset 的 `TextAsset` 包成 Luban 的 `ByteBuf`：

```csharp
private sealed class ConfigBytesLoader : IConfigBytesLoader
{
    public UniTask<ByteBuf> LoadLocalizationAsync(string locale) =>
        LoadCommonAsync("Game_" + locale);

    public async UniTask<ByteBuf> LoadCommonAsync(string file)
    {
        using var asset = await GameApp.Resource.LoadLeaseAsync<TextAsset>(file);
        if (!asset.IsValid)
            throw new InvalidOperationException($"Missing config asset: {file}");
        return ByteBuf.Wrap(asset.Asset.bytes);
    }
}
```

### 初始化顺序

1. `ResourceComponent` 已注册 `IResourceService`（见 [Resources 资源](../resource/index.md)）。
2. `LocalizationKit` 已初始化，并能提供 `CurrentLanguage`（见 [Localization 本地化](../localization/index.md)）。
3. 注册 `IConfigService` → `Initialize(loader)` → `SetLanguageAsync(当前语言)`。
4. 绑定语言加载器，之后切语言会自动刷新配置文案。
5. 业务服务（如 `FakePlayerDataService`）通过 `AppServices.Require<IConfigService>()` 取表。

顺序错了会得到 `Service IConfigService is not registered.` 或 `Config service is not loaded.`。

## 多语言如何回填

`text` 字段在生成 bean 里是两段式：

```csharp
public readonly string NameKey;            // Excel 里的 key
public string Name { get; internal set; }  // 当前语言文案
```

`SetLanguageAsync` 的流程：

1. `ConfigLocalization.Read(await _loader.LoadLocalizationAsync(locale))` 解出 `key → 文案` 字典。
2. 遍历所有含 `text` 字段的表，执行 `row.Name = ConfigLocalization.Get(values, row.NameKey)`。
3. 缺失的 key 回退为 key 本身；`""` 表示显式空字符串。

语言切换由 `LocalizationKit.RegisterLanguageLoader` 触发，返回的 `IDisposable` 交给 `SetLanguageBinding` 保存；服务销毁时自动解除。

## 校验生成物是否最新

```bat
cd /d D:\UnityProject\AlicizaXTemplate\Config
gen-all.bat --check
```

- 通过：`Checked N generated files.`
- 失败：列出所有有差异的文件并以退出码 1 结束，适合放进 CI。

生成物未提交或与 Excel 不一致时，`build.bat` 也会在这一步失败，见 [常见错误排查](troubleshooting.md)。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 换资源地址风格 | 改 YooAsset 收集器的 `AddressRule`，同步改 `IConfigBytesLoader` 的拼接方式 |
| 配置表很多、启动慢 | 按业务拆分资源包或改为按需 `Initialize`，不要改生成代码 |
| 想手动控制加载时机 | 保留生成的表类，自己 `new TbXxx(() => ...)` 并调用 `LoadAsync`（`internal`，需要同程序集） |
| 新增 Unity 类型映射 | 在 `Defines/builtin.xml` 加 mapper，并在 `ExternalTypeUtil` 补转换方法 |

## 容易踩的坑

1. **生成代码没有 `.meta`**：在 Unity 外生成的新文件首次进入编辑器时才补 `.meta`，记得一起提交，否则队友拉下来会重新生成 GUID。
2. **地址写错**：`AddressByFileName` 只保留文件名，写 `Configs/bytes/ordinary/tables_tbitem` 会找不到资源。
3. **先 `SetLanguageAsync` 后 `Initialize`**：抛 `Config service is not loaded.`
4. **重复 `Initialize`**：第二次直接返回，换了 loader 也不会生效；需要重载请销毁服务后重新注册。
5. **改生成文件**：本地能跑，下次 `gen-all` 覆盖；`--check` 也会失败。
6. **多语言包没进构建**：`bytes/localization` 不在收集器里或没打标签，运行时报 `Missing config asset: Game_English`。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `IConfigService.Initialize(IConfigBytesLoader)` | 并行加载所有表并解析 `#ref` |
| `IConfigService.SetLanguageAsync(string)` | 加载指定语言并回填 `text` 字段 |
| `IConfigService.SetLanguageBinding(IDisposable)` | 保存语言切换订阅，销毁时释放 |
| `<表>.GetOrDefault(key)` / `Get(key)` / `DataMap` / `DataList` | `map` / `list` 表查询 |
| `<单例表>.Data` | `singleton` 表数据 |
| `<记录>.<字段>Key` / `<记录>.<字段>` | 多语言 key 与当前值 |
| `ConfigLocalization.Read` / `Get` | 手写辅助类，解析多语言字节 |

下一步：出问题时查 [常见错误排查](troubleshooting.md)。
