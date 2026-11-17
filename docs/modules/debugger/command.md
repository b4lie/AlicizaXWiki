# 命令控制台

`Command` 是 Debugger 侧边栏里路径为 `"Command"` 的命令控制台，背后是 `AlicizaX.Console`：`AlicizaXConsoleProcessor` 反射扫描 `[Command]` 生成命令表，`AlicizaXConsoleController` 负责输入、历史、补全和异步任务，`CommandWindow` 只是它的一套 UI Toolkit 视图。它和 `Console` 页不是一回事——`Command` 不接管 `Debug.Log`。

## 适用场景

- 不改代码就想改运行时状态：时间缩放、分辨率、场景、对象位置。
- 给自己的系统加 GM / 调试命令，让策划和测试直接敲命令。
- 脚本化地连跑一串命令（外部命令文件）。
- 在没有 UI 的服务器、测试或工具代码里执行同一条命令。

## 使用前提

1. 命令所在程序集必须进入命令表扫描范围。默认只有 `AlicizaX.Debugger`（`AlicizaXConsoleProcessor.DefaultCommandAssemblyName`）。
2. UI 版依赖 `DebuggerComponent` 的 `Command` 窗口；纯逻辑调用不依赖 UI。
3. 键盘快捷键走 `AlicizaX.Console.InputKeys`：`INPUTSYSTEM_SUPPORT && !ENABLE_LEGACY_INPUT_MANAGER` 时用 Input System，否则回退旧 `Input`。

## 最短接入

1. 打开命令窗口（侧边栏点 `Command`，或用代码）：

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
debugger.ShowFullWindow = true;
debugger.SelectDebuggerWindow("Command");
```

2. 输入 `help` 看基础说明，`commands` 列出全部命令，`man <命令>` 看单个命令的手册。
3. 写一个命令，并把所在程序集名填进 Inspector 的 `Command Assemblies`：

```csharp
using AlicizaX.Console;

public static class GameDebugCommands
{
    [Command("give-gold", "增加金币")]
    public static string GiveGold(int amount)
    {
        // PlayerModel.Gold += amount;
        return $"gold += {amount}";
    }
}
```

```text
give-gold 100
```

## 控制台界面与快捷键

界面从上到下是：工具栏（`Clear` + 右侧 job 计数）、输出区（富文本、可滚动）、补全弹窗、输入行（占位提示 + ghost text + 自绘光标）和 `Submit` 按钮。

| 操作 | 默认键 | 序列化字段 |
| --- | --- | --- |
| 提交命令 | `Return` | `m_SubmitCommandKey` |
| 下一条建议 | `Tab` | `m_SelectNextSuggestionKey` |
| 上一条建议 | `Shift + Tab` | `m_SelectPreviousSuggestionKey` |
| 上一条历史 | `UpArrow` | `m_HistoryPreviousKey` |
| 下一条历史 | `DownArrow` | `m_HistoryNextKey` |
| 取消正在执行的 action | `Ctrl + C` | `m_CancelActionsKey` |

历史键只在 `m_StoreCommandHistory` 为 `true` 时生效，补全键只在 `m_EnableAutocomplete` 为 `true` 时生效；组合键要求 Ctrl / Alt / Shift 状态与配置完全一致。`Ctrl + C` 取消的是当前 action 序列，不会取消已经启动的 `Task`。

## 命令扫描与注册

`[Command]` 可用于方法、属性和字段，允许同一成员挂多个别名：

```csharp
[Command]                                       // 别名取成员名
[Command("spawn-enemy")]                        // 指定别名
[Command("spawn-boss", "生成 Boss")]             // 别名 + 描述
[Command("heal", MonoTargetType.All)]           // 指定目标类型
```

规则：

- 别名不能包含空格、`(`、`)`、`{`、`}`、`[`、`]`、`<`、`>`，否则构造 attribute 时抛 `ArgumentException` 并 `Debug.LogError`。
- 扫描类型自身声明的成员，含 `private` / `internal`、静态与实例。
- 带 `[Command]` 的属性按 `CanRead` / `CanWrite` 生成读取和写入命令；带 `[Command]` 的字段生成读取命令，非 `readonly` / 非 `const` 字段再生成写入命令；委托字段也可以作为命令。
- 带默认参数的方法会展开成多个重载：`Foo(int a, int b = 1)` 同时提供 `foo a` 和 `foo a b`。命令表的键是 `名字(参数个数)`；同名同参数个数的只保留第一个，其余输出 warning。
- 非静态命令要求声明类型派生自 `MonoBehaviour`，除非 `MonoTarget` 是 `Registry`。
- 参数类型必须被 `AlicizaXConsoleParser` 支持，否则命令被跳过并输出 warning。
- 平台判断是 `SupportedPlatforms.HasFlag(Application.platform.ToPlatform())`，置位的平台才可用。

实例命令按 `MonoTargetType` 找目标：

| 值 | 行为 |
| --- | --- |
| `Single` / `All` | 场景中第一个 / 所有激活实例 |
| `SingleInactive` / `AllInactive` | 同上，但包含未激活对象 |
| `Registry` | `AlicizaXConsoleRegistry` 里注册的实例；普通类只能用这一种 |
| `Singleton` | 不存在就自动创建并复用 |
| `Argument` / `ArgumentMulti` | 第一个参数就是目标实例 / 目标实例数组 |

```csharp
using AlicizaX.Console;
using AlicizaX.Debugger;
using UnityEngine;

public sealed class PlayerController : MonoBehaviour
{
    [Command("damage-player", "扣除玩家生命", MonoTargetType.Single)]
    private void Damage(int value) { /* Hp -= value; */ }
}

// 普通类走 Registry：先用 AlicizaXConsoleRegistry.RegisterObject(_player) 注册。
```

其它 attribute：

| Attribute | 作用域 | 作用 |
| --- | --- | --- |
| `CommandPlatformAttribute(Platform)` | 方法 / 属性 / 字段 | 覆盖 `[Command]` 里的平台限制 |
| `CommandPrefixAttribute(prefix)` | 类 / 结构体 / 程序集 | 给该类（含嵌套与子类）所有命令加前缀；实际命令名是「程序集前缀 + 外层类型前缀 + 内层类型前缀 + 别名」，没有分隔符 |
| `CommandDescriptionAttribute(desc)` | 方法 / 属性 / 字段 | 补命令说明；`[Command]` 自带说明时以 `[Command]` 为准 |
| `CommandParameterDescriptionAttribute(desc)` | 参数 | 补参数说明，供 `man` 使用 |
| `AlicizaXConsoleIgnoreAttribute` | 程序集 / 类 | 扫描时跳过 |

扫描规则集（`AlicizaXConsoleScanRuleset`）会跳过：程序集名以 `System`、`Unity`、`Microsoft`、`Mono.`、`mscorlib`、`NSubstitute`、`JetBrains`、`nunit.`、`GeNa.` 开头的程序集，名为 `mcs`、`AssetStoreTools`、`Facepunch.Steamworks` 的程序集，带 `[AlicizaXConsoleIgnore]` 的对象，以及编译器生成的类型。

## 参数格式与类型解析

命令按空格拆分参数，但 `""`、`()`、`[]`、`{}`、`<>` 里的空格和分隔符会保留。

| 类型 | 写法示例 |
| --- | --- |
| `string` | `"hello world"` |
| `int` / `float` / `bool` | `10`、`1.5`、`true` |
| `Vector2` / `Vector3` / `Vector4` / `Quaternion` | `1,2`、`1,2,3`、`1,2,3,4`、`0,0,0,1` |
| `Color` | `red`、`0xFF0000` |
| 数组 / 集合 | `[1,2,3]` |
| `GameObject` / `Component` | 对象名 |
| `Type` | `int`、`UnityEngine.Vector3`、`List<int>` |
| 可空 / 元组 | `10?`、`(1,2)` |

泛型命令写成 `名字<T>`，例如 `some-generic-command<UnityEngine.Vector3> 1,2,3`。类型解析默认带 `System`、`System.Collections`、`System.Collections.Generic`、`UnityEngine`、`UnityEngine.UI`、`AlicizaX.Debugger`、`AlicizaX.Console`，运行期可以增删：

```text
all-namespaces
use-namespace Game.Runtime
remove-namespace Game.Runtime
reset-namespaces
```

## 内置命令

| 命令 | 说明 |
| --- | --- |
| `help` / `man <命令>` / `manual <命令>` | 基础帮助 / 命令手册 |
| `commands` / `all-commands` / `user-commands` | 列出全部命令 / 只列非本包命令 |
| `command-count` | 已加载命令数量 |
| `clear` | 清空命令输出 |
| `verbose-errors` / `verbose-logging` / `logging-level` / `max-logs` | 读取或设置控制台选项 |
| `qc-script-extern <文件>` | 逐行执行外部命令文件（WebGL 不支持） |
| `quit` | 退出应用；Editor 与 WebGL 不提供 |
| `max-fps` / `vsync` / `msaa` / `time-scale` | 帧率 / 垂直同步 / MSAA / 时间缩放 |
| `fullscreen` / `screen-dpi` / `screen-orientation` | 全屏 / DPI / 屏幕方向（后者仅移动平台） |
| `current-resolution` / `set-resolution <x> <y> [fullscreen]` | 当前分辨率 / 设置分辨率 |
| `supported-resolutions` / `capture-screenshot <文件名> [放大倍数]` | 支持的分辨率（WebGL 不提供）/ 保存 PNG |
| `profile stats <true\|false>` | 开关 Stats 悬浮窗 |
| `load-scene` / `load-scene-index` / `unload-scene` / `unload-scene-index` | 按名称或索引加载 / 卸载场景 |
| `all-scenes` / `loaded-scenes` / `active-scene` / `set-active-scene` | 场景查询与切换 |
| `get-scene-hierarchy` / `get-object-info` | 场景层级 / 对象详情 |
| `instantiate` / `destroy` / `destroy-component` / `add-component` | 实例化 / 销毁 / 组件增删 |
| `teleport` / `teleport-relative` / `rotate` / `set-active` / `set-parent` / `send-message` | 对象操作 |
| `call-static <类型> <方法> [参数...]` | 反射调用静态方法或属性 |
| `call-instance <类型> <方法> <MonoTargetType> [参数...]` | 反射调用实例方法或属性 |
| `register-object<T>` / `deregister-object<T>` / `clear-registry<T>` / `display-registry<T>` | Registry 维护 |
| `#define` / `remove-macro` / `clear-macros` / `all-macros` / `dump-macros` / `load-macros` | 宏维护 |
| `all-namespaces` / `use-namespace` / `remove-namespace` / `reset-namespaces` | 类型解析命名空间表 |

`Extras` 目录下的内置命令（应用、画面、场景、时间、对象、`profile`）整段受编译宏控制：定义 `AlicizaXConsole_DISABLED`、`AlicizaXConsole_DISABLE_BUILTIN_ALL` 或 `AlicizaXConsole_DISABLE_BUILTIN_EXTRA` 中任意一个，这些命令就不会编进包。

## 宏与预处理

输入先经 `AlicizaXConsolePreprocessor.Process`：以 `#define` 开头的输入不展开（方便定义嵌套宏），其它输入交给 `AlicizaXConsoleMacros.ExpandMacros`。

- `#name` 展开为宏内容，按宏名长度从长到短匹配；最多展开 `1000` 次，超出抛异常（通常意味着递归宏）。
- 宏名不能含空白或 `#`，不能叫 `define`；展开内容不能含换行，也不能包含 `#名字` 自身。
- 进入 Play 时会清空宏表。

```text
#define p Player
teleport #p 0,10,0
all-macros
remove-macro p
dump-macros "C:/Temp/macros.txt"
load-macros "C:/Temp/macros.txt"
```

## 程序内调用

```csharp
using AlicizaX.Console;

// 1. 显式生成命令表（推荐，避免 InvokeCommand 退化为扫描全部已加载程序集）。
AlicizaXConsoleProcessor.GenerateCommandTableFromAssemblyNames(
    new[] { "AlicizaX.Debugger", "Game.Runtime" },
    deployThread: true,
    forceReload: true);

// 2. 不依赖 UI 直接执行，返回命令结果。
object result = AlicizaXConsoleProcessor.InvokeCommand("time-scale 0.5");

// 3. 通过活动控制台写输出 / 清屏 / 跑脚本。
IAlicizaXConsole console = AlicizaXConsoleRouter.ActiveConsole;
console?.LogToConsole("route message");
console?.ClearConsole();
await console.InvokeExternalCommandsAsync("C:/Temp/cmds.txt");
```

`AlicizaXConsoleRouter.ActiveConsole` 在 `Command` 窗口初始化之前为 `null`。`InvokeCommand` 发现命令表还没生成时会调用无参 `GenerateCommandTable()`，也就是扫描全部已加载程序集；已有表则直接复用，不会按新程序集名单重扫。

## 控制台外壳与视图接口

`IAlicizaXConsole` 由六个小接口组成，自定义外壳时可以实现，或直接用 `AlicizaXConsoleController`：

| 接口 | 成员 |
| --- | --- |
| `IAlicizaXConsoleState` | `IsActive` |
| `IAlicizaXConsoleSettings` | `VerboseErrors`、`VerboseLogging`、`LoggingLevel`、`MaxStoredLogs` |
| `IAlicizaXConsoleResponse` | `BeginResponse(Action<string>, ResponseConfig)` |
| `IAlicizaXConsoleSerialization` | `Serialize(object)` |
| `IAlicizaXConsoleOutput` | `ClearConsole()`、`LogToConsole(string, bool = true)`、`RemoveLogTrace()` |
| `IAlicizaXConsoleCommandSource` | `InvokeExternalCommandsAsync(string filePath)` |

`ICommandConsoleView` 是控制台对视图的全部要求：`InputValue`、`InputEnabled`、`IsInputFocused`、`SetPlaceholder`、`SetLogText`、`SetGhostText`、`SetPopupText`、`SetJobCounter`、`ScrollToLatest`、`FocusInput`。

```csharp
using AlicizaX.Console;

var controller = new AlicizaXConsoleController(new AlicizaXConsoleConfig
{
    CommandAssemblyNames = new[] { "AlicizaX.Debugger", "Game.Runtime" },
});

controller.Initialize();
controller.AttachView(myView); // myView 实现 ICommandConsoleView
AlicizaXConsoleRouter.Register(controller);
controller.Tick();             // 每帧调用：异步任务、action 序列、日志刷入视图
```

## 扩展命令的步骤

1. 新建运行时 asmdef，引用 `AlicizaX.Debugger`（`AlicizaX.Console` 命名空间在此程序集）。
2. 写 `public static` 命令类并加 `[Command]`；需要实例命令就放在 `MonoBehaviour`，或走 `MonoTargetType.Registry`。
3. 把 asmdef 名加到 `Command Assemblies`，或启动时调用 `GenerateCommandTableFromAssemblyNames(..., forceReload: true)`。
4. 进 Play 后用 `commands` 确认命令出现，用 `man <命令>` 检查签名与参数说明。
5. 发布 IL2CPP 包前确认命令类未被裁剪（命令是反射调用的，必要时加 `[Preserve]` 或 `link.xml`）。

## 常见配置

| Inspector | 序列化字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Command Assemblies | `m_CommandAssemblyNames` | `AlicizaX.Debugger` | 生成命令表时扫描的程序集 |
| Autocomplete / Suggestion Popup | `m_EnableAutocomplete` / `m_ShowPopupDisplay` | `true` / `true` | 补全建议 / 建议弹窗 |
| Command History | `m_StoreCommandHistory` | `true` | 上下键历史 |
| Max Stored Logs | `m_MaxStoredLogs` | `1024` | 命令输出保留条数 |
| Submit Key | `m_SubmitCommandKey` | `Return` | 提交键 |
| Next / Prev Suggestion | `m_SelectNextSuggestionKey` / `m_SelectPreviousSuggestionKey` | `Tab` / `Shift + Tab` | 建议切换 |
| Previous / Next History | `m_HistoryPreviousKey` / `m_HistoryNextKey` | `UpArrow` / `DownArrow` | 历史切换 |
| Cancel Actions | `m_CancelActionsKey` | `Ctrl + C` | 取消 action 序列 |

只序列化、不在 Inspector 显示的项：`m_LoggingLevel`、`m_VerboseErrors`、`m_VerboseLogging`、`m_AutoScroll`、`m_ShowCurrentJobs`、`m_BlockOnAsync`、`m_MaxLogSize`、`m_SuggestionDisplayOrder`、`m_MaxSuggestionDisplaySize`、`m_UseFuzzySearch`、`m_CaseSensitiveSearch`、`m_CollapseSuggestionOverloads`、`m_CommandHistorySize`、`m_StoreDuplicateCommands`、`m_StoreAdjacentDuplicateCommands`、`m_ShowInitLogs`。

## 容易踩的坑

1. 命令表默认只扫 `AlicizaX.Debugger`；自定义 asmdef 没写进 `Command Assemblies`，命令就永远找不到。
2. 命令表生成一次后缓存，运行期新加载的程序集需要 `forceReload: true` 才会进表。
3. 别名含禁用字符会在 attribute 构造时抛 `ArgumentException`，异常点看起来像类型初始化失败。
4. 非静态命令必须是 `MonoBehaviour`，普通类只能用 `MonoTargetType.Registry`，否则命令被跳过并 warning。
5. `Command` 页不采集 `Debug.Log`；要对照 Unity 日志请用 `Console` 页。
6. `Task` 命令只体现在 job 计数，`Ctrl + C` 不会取消已经启动的 `Task`。

## 相关页面

- [Debugger 调试器](index.md)：组成、激活模式与服务关系
- [启动、窗口树与日志](usage.md)：挂载、布局、日志与 Stats 悬浮窗
- [API 速查](api.md)：命令相关公开类型与成员
