# 运行时加载

运行时由 `LocalizationKit` 统一管理：启动时加载第一份词库，热更资源可用后追加其余词库，切换语言时重新加载所有已注册来源并通知 UI。

## 初始化

在启动流程的异步方法中先加载更新界面需要的词库：

```csharp
using AlicizaX.Localization.Runtime;
using UnityEngine;

TextAsset options = Resources.Load<TextAsset>("Localization/RuntimeOptions");
string json = options.text;
Resources.UnloadAsset(options);

await LocalizationKit.InitializeAsync(
    json,
    new ResourcesCatalogSource("Localization/Update_{locale}"));
```

`ResourcesCatalogSource` 的地址相对于 `Resources` 目录，并省略 `.bytes` 后缀；构造参数必须包含 `{locale}`，加载时替换为当前语言。

`InitializeAsync` 的行为：

1. 解析 `RuntimeOptions.json` 的 `Locales` 和 `DefaultLocale`，非法时抛 `FormatException`。
2. 选择启动语言。
3. 用第一份词库创建 `LocalizationProvider`。
4. 发布一次 `Changed` 事件。

已经初始化或正在初始化时再次调用会抛 `InvalidOperationException`。初始化失败会清理中间状态，可以重试。

## 启动语言

编辑器：

1. 玩家保存的语言（`PlayerPrefs`）。
2. 编辑器语言选择（`EditorPrefs`）。
3. 项目默认语言。

Player：

1. 玩家保存且仍受支持的语言。
2. 本次构建语言。
3. 项目默认语言。

构建语言由 BuildCLI 的 `UCL_PARAM_LANGUAGE` 指定，必须出现在 `RuntimeOptions.json` 的 `Locales` 中。BuildCLI 会把语言写入 `StreamingAssets/AlicizaX/Localization/build-locale.txt`，并添加 `ALICIZAX_BUILD_LOCALE_FILE` 宏；运行时优先读取这个文件。没有指定构建语言时，首次启动使用项目默认语言。

## 追加词库

热更资源和普通配置可用后，追加其余词库：

```csharp
await LocalizationKit.AddCatalogAsync(
    new ResourceCatalogSource("Game_{locale}"));
```

`ResourceCatalogSource` 使用框架的 `GameApp.Resource` 异步加载 `TextAsset` 租约；加载失败会抛异常。

追加规则：

- 新词库与已有词库按 Key 合并，重复 Key 抛 `InvalidOperationException`。
- 成功后 `Revision` 增加，并发布 `Changed`。
- 未初始化或正在加载时返回 `false`。
- 来源会保留，切换语言时重新加载所有来源。

## 切换语言

```csharp
bool changed = await LocalizationKit.SetLanguageAsync("English");
```

- 成功后重新加载所有已注册来源，更新 `CurrentLanguage` 和 `Revision`，写入 `PlayerPrefs`，并发布 `Changed`。
- 不支持的语言抛 `ArgumentException`。
- 未初始化或正在加载时返回 `false`。
- 目标语言与当前语言相同时直接返回 `true`，不重新加载、不发布事件。

## 取文案

```csharp
string title = LocalizationKit.Get("SHOP_TITLE");
string message = LocalizationKit.Format("BUY_QUANTITY", count);
string detail = LocalizationKit.Format("SHOP_SIZE", width, height);
```

`Get` 在 Key 缺失或未初始化时返回 Key 本身。`Format` 使用 `ZString` 的格式化，最多支持 16 个参数；超过 16 个抛 `FormatException`。

其他状态：

| API | 说明 |
| --- | --- |
| `LocalizationKit.CurrentLanguage` | 当前语言；未初始化为空字符串 |
| `LocalizationKit.Revision` | 词库或语言变化的版本号；未初始化为 `0` |
| `LocalizationKit.SupportedLanguages` | 支持的语言列表；未初始化为空列表 |
| `LocalizationKit.Changed` | 语言或词库变化事件，参数为当前语言 |
| `LocalizationKit.SubscribeChanged(Action<string>)` | 订阅事件并返回可释放的 `IDisposable` |

`SubscribeChanged` 在回调为空时抛 `ArgumentNullException`。事件处理器内部异常会被捕获并记录，不会中断其他订阅者。

```csharp
private IDisposable _subscription;

private void OnEnable()
{
    _subscription = LocalizationKit.SubscribeChanged(OnLanguageChanged);
}

private void OnDisable()
{
    _subscription?.Dispose();
    _subscription = null;
}
```

`LocalizationKit` 没有公开的 `Shutdown`；静态状态在 `SubsystemRegistration` 时清理。不要在业务里假设可以反复初始化和关闭。

## 词库来源

内置两种来源：

| 来源 | 用途 |
| --- | --- |
| `ResourcesCatalogSource` | 从 `Resources` 加载，适合更新界面等不需要热更的词库 |
| `ResourceCatalogSource` | 通过框架 `GameApp.Resource` 加载，适合热更包内词库 |

使用其他资源系统时实现 `ILocalizationCatalogSource`：

```csharp
using Cysharp.Threading.Tasks;
using AlicizaX.Localization.Runtime;

public sealed class CustomCatalogSource : ILocalizationCatalogSource
{
    public UniTask<byte[]> LoadAsync(string locale)
    {
        // 从 Addressables / AssetBundle / 服务器读取对应语言的 Luban 词库。
        byte[] bytes = LoadCatalogBytes(locale);
        return UniTask.FromResult(bytes);
    }

    private static byte[] LoadCatalogBytes(string locale)
    {
        // 由项目实现。
        throw new System.NotImplementedException();
    }
}
```

来源必须按传入的 `locale` 返回对应词库。缺少词库时抛 `InvalidOperationException`，不要返回空数组。

## 配置服务接入

普通 Luban `ConfigService` 生成代码提供两个回调：翻译函数和语言变化订阅。热更入口在配置初始化后接入：

```csharp
await LocalizationKit.AddCatalogAsync(new ResourceCatalogSource("Game_{locale}"));

ConfigService = AppServices.Register<IConfigService>(new ConfigService());
await ConfigService.Initialize(new ConfigBytesLoader());
ConfigService.SetLocalizationChange(LocalizationKit.Get, LocalizationKit.SubscribeChanged);
```

`SetLocalizationChange` 会立即刷新一次配置里的翻译字段，并在语言变化时再次刷新；服务销毁时释放订阅。多语言包本身不依赖项目的 `Game.Config`。

## 错误与边界

| 场景 | 结果 |
| --- | --- |
| `RuntimeOptions.json` 缺少语言或默认语言非法 | `FormatException` |
| 语言列表为空、重复或默认语言不在列表中 | `FormatException` |
| 重复初始化或并发初始化 | `InvalidOperationException` |
| 初始化时词库加载失败 | 异常向上抛，状态可重试 |
| 未初始化时 `Get` | 返回 Key |
| 未初始化时 `AddCatalogAsync` / `SetLanguageAsync` | 返回 `false` |
| 未初始化时 `Format` | 返回 Key（不格式化） |
| 切换语言时词库加载失败 | 异常向上抛，保持原语言 |
| 跨词库重复 Key | `InvalidOperationException` |
| `Format` 参数超过 16 个 | `FormatException` |

## 注意事项

1. 启动阶段必须在更新界面显示前完成第一份词库加载。
2. `RuntimeOptions.json` 只用于语言列表和编辑器预览，运行时要加载哪些词库由启动代码显式决定。
3. 追加词库只在热更资源可用后调用；`Game_{locale}` 由 `ResourceCatalogSource` 从框架资源服务加载。
4. 切换语言会重新加载全部来源；来源的 `LoadAsync` 必须能按任意受支持语言返回词库。
5. 语言变化订阅要配对释放，窗口关闭或对象销毁时调用 `Dispose`。
