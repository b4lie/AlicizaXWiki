# 命令控制台（Command）

`Command` 是 Debugger 侧边栏的一个窗口，路径为 `"Command"`。它和日志窗口 `Console` 不是同一个功能：

- `Command` 用于输入和执行 `[Command]` 命令、查看命令输出。
- `Console` 只展示 Unity 日志。

命令控制台不会拦截 `Debug.Log`，Unity 日志请到 `Console` 页查看。

## 打开

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
if (debugger != null)
{
    debugger.ShowFullWindow = true;
    debugger.SelectDebuggerWindow("Command");
}
```

`SelectDebuggerWindow` 返回 `false` 表示路径不存在或切换失败。也可以直接在侧边栏点击 `Command`。

## 界面

- 顶部工具栏：`Clear` 清空命令输出；右侧显示当前 job 计数。
- 中间：命令输出、建议弹窗。
- 底部：输入框和 `Submit` 按钮。
- 输入时显示补全建议和 ghost text；`Tab` / `Shift + Tab` 切换建议。

### 快捷键

| 操作 | 默认键 | Inspector 字段 |
| --- | --- | --- |
| 提交命令 | `Return` | `Submit Key` |
| 选择下一条建议 | `Tab` | `Next Suggestion` |
| 选择上一条建议 | `Shift + Tab` | `Prev Suggestion` |
| 上一条历史命令 | `UpArrow` | `Previous History` |
| 下一条历史命令 | `DownArrow` | `Next History` |
| 取消正在执行的 action | `Ctrl + C` | `Cancel Actions` |

快捷键都在 `DebuggerComponent` Inspector 的 `Command` 区域修改，不写死在代码里。

### 配置

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `Command Assemblies` | `AlicizaX.Debugger` | 生成命令表时扫描的程序集 |
| `Autocomplete` | 开 | 补全建议和 ghost text |
| `Suggestion Popup` | 开 | 输入时弹出建议列表 |
| `Command History` | 开 | 记录上下键历史 |
| `Max Stored Logs` | `1024` | 命令控制台输出保留条目数 |

## 命令扫描

`AlicizaXConsoleProcessor` 默认只扫描程序集 `AlicizaX.Debugger`。自定义命令写在其他 asmdef 中时，有两种方式让它们进入命令表。

在 Inspector 中加程序集名：

```text
AlicizaX.Debugger
Game.Runtime
Game.DebugCommands
```

或在代码中手动生成：

```csharp
using AlicizaX.Console;

AlicizaXConsoleProcessor.GenerateCommandTableFromAssemblyNames(
    new[] { "AlicizaX.Debugger", "Game.Runtime" },
    deployThread: true,
    forceReload: true);
```

参数：

| 参数 | 说明 |
| --- | --- |
| `deployThread` | 使用第二个线程生成命令表 |
| `forceReload` | 清空并重新生成命令表 |

WebGL 不支持后台线程生成命令表，传 `deployThread: true` 时会退回主线程。命令表生成一次后会缓存，新增程序集需要 `forceReload: true` 或重新进入 Play。

## 添加命令

### 静态方法

```csharp
using AlicizaX.Console;
using UnityEngine;

public static class GameDebugCommands
{
    [Command("give-gold", "增加金币")]
    public static string GiveGold(int amount)
    {
        // PlayerModel.Gold += amount;
        return $"Give gold: {amount}";
    }

    [Command("teleport-player", "传送玩家")]
    public static void TeleportPlayer(Vector3 position)
    {
        Debug.Log($"Teleport to {position}");
    }
}
```

控制台输入：

```text
give-gold 100
teleport-player 1,2,3
```

方法返回值会写入命令输出；`void` 方法没有额外输出。

### 描述参数

`[CommandDescription]` 和 `[CommandParameterDescription]` 用于 `man` 命令生成手册：

```csharp
using AlicizaX.Console;

public static class MatchCommands
{
    [Command("set-round")]
    [CommandDescription("设置当前回合数")]
    public static void SetRound(
        [CommandParameterDescription("目标回合，从 1 开始")] int round)
    {
        // ...
    }
}
```

### 字段和属性

字段、属性也可以加 `[Command]`。字段自动生成读取命令；非 `readonly` 字段还会生成写入命令。属性按 `CanRead` / `CanWrite` 生成读取和写入命令：

```csharp
using AlicizaX.Console;

public static class DebugSettings
{
    [Command("god-mode", "读取或设置无敌模式")]
    public static bool GodMode;

    [Command("spawn-rate", "读取或设置刷怪倍率")]
    public static float SpawnRate { get; set; } = 1f;
}
```

控制台输入：

```text
god-mode
god-mode true
spawn-rate
spawn-rate 2.5
```

### MonoBehaviour 命令

非静态 `MonoBehaviour` 命令会按 `MonoTargetType` 查找目标实例：

```csharp
using AlicizaX.Console;
using UnityEngine;

public sealed class PlayerController : MonoBehaviour
{
    [Command("damage-player", "扣除玩家生命", MonoTargetType.Single)]
    public void Damage(int value)
    {
        // Hp -= value;
    }

    [Command("heal-all-players", "治疗所有玩家", MonoTargetType.All)]
    public void Heal(int value)
    {
        // Hp += value;
    }
}
```

| 类型 | 行为 |
| --- | --- |
| `Single` | 查找场景中第一个激活的目标组件 |
| `All` | 对场景中所有激活的目标组件各调用一次 |
| `Registry` | 使用 `AlicizaXConsoleRegistry` 注册的实例；非 `MonoBehaviour` 实例只支持这种方式 |
| `Singleton` | 目标不存在时自动创建并复用单例实例 |
| `SingleInactive` | 查找第一个目标组件，包含未激活对象 |
| `AllInactive` | 查找所有目标组件，包含未激活对象 |
| `Argument` | 第一个命令参数指定目标实例 |
| `ArgumentMulti` | 第一个命令参数指定目标实例数组 |

把普通对象注册到 Registry：

```csharp
using AlicizaX.Console;

public sealed class DebugCommandOwner
{
    [Command("reload-config", MonoTargetType.Registry)]
    public void ReloadConfig()
    {
        // ...
    }
}

public static class DebugCommandBootstrap
{
    private static readonly DebugCommandOwner Owner = new();

    public static void Register()
    {
        AlicizaXConsoleRegistry.RegisterObject(Owner);
    }

    public static void Deregister()
    {
        AlicizaXConsoleRegistry.DeregisterObject(Owner);
    }
}
```

### 平台限制

命令可以加 `[CommandPlatform]` 限制可用平台：

```csharp
[Command("quit")]
[CommandPlatform(Platform.AllPlatforms ^ (Platform.EditorPlatforms | Platform.WebGLPlayer))]
private static void Quit()
{
    Application.Quit();
}
```

内置的 `quit`、`screen-orientation`、`supported-resolutions`、`qc-script-extern`、`dump-macros`、`load-macros` 都带平台限制。

## 参数格式

命令按空格拆分参数，但会保留 `""`、`()`、`[]`、`{}` 和 `<>` 中的空格与分隔符。

| 类型 | 示例 |
| --- | --- |
| `string` | `"hello world"` |
| `int` / `float` / `bool` | `10`、`1.5`、`true` |
| `Vector2` / `Vector3` / `Vector4` | `1,2`、`1,2,3`、`1,2,3,4` |
| `Vector2Int` / `Vector3Int` | `1,2`、`1,2,3` |
| `Quaternion` | `0,0,0,1` |
| `Color` | `red`、`0xFF0000` |
| 数组 | `[1,2,3]` |
| `GameObject` | 对象名 |
| `Component` | 组件所在对象名 |
| `Type` | `int`、`Vector3`、`List<int>`、完整类型名 |

泛型命令使用 `<T>`：

```text
some-generic-command<int> 1
some-generic-command<UnityEngine.Vector3> 1,2,3
```

类型解析的默认命名空间：

- `System`
- `System.Collections`
- `System.Collections.Generic`
- `UnityEngine`
- `UnityEngine.UI`
- `AlicizaX.Debugger`
- `AlicizaX.Console`

运行时管理命名空间：

```text
all-namespaces
use-namespace Game.Runtime
remove-namespace Game.Runtime
reset-namespaces
```

## 宏

宏以 `#` 引用。除 `#define` 本身外，输入命令会先经过宏展开。

```text
#define p Player
all-macros
remove-macro p
clear-macros
dump-macros "C:/Temp/console-macros.txt"
load-macros "C:/Temp/console-macros.txt"
```

定义后 `#p` 会替换为宏内容。宏名不能包含空白、换行和 `#`，也不能命名为 `define`；宏展开不能引用自身。

`dump-macros` 和 `load-macros` 依赖文件系统，WebGL 不支持。

## 外部命令脚本

脚本每行一条命令：

```text
help
commands
max-fps 60
time-scale 1
```

运行：

```text
qc-script-extern "C:/Temp/debug-commands.txt"
```

WebGL 不支持外部文件命令。

## 内置命令

### 控制台与命令表

| 命令 | 说明 |
| --- | --- |
| `help` | 显示基础帮助 |
| `man <command>` / `manual <command>` | 查看指定命令的手册 |
| `commands` / `all-commands` | 列出所有命令 |
| `user-commands` | 列出用户命令 |
| `command-count` | 查看已加载命令数量 |
| `clear` | 清空命令控制台输出 |
| `verbose-errors` | 读取或设置详细错误输出 |
| `verbose-logging` | 读取或设置详细日志阈值 |
| `logging-level` | 读取或设置日志拦截阈值 |
| `max-logs` | 读取或设置最大存储日志数 |

### Registry、宏与命名空间

| 命令 | 说明 |
| --- | --- |
| `register-object<T>` | 注册对象到 Registry |
| `deregister-object<T>` | 从 Registry 移除对象 |
| `display-registry<T>` | 显示 Registry 内容 |
| `clear-registry<T>` | 清空 Registry |
| `#define` | 定义宏 |
| `all-macros` | 列出宏 |
| `remove-macro` | 移除宏 |
| `clear-macros` | 清空宏 |
| `dump-macros` / `load-macros` | 导出 / 导入宏 |
| `all-namespaces` | 查看类型解析命名空间 |
| `use-namespace` | 添加类型解析命名空间 |
| `remove-namespace` | 移除类型解析命名空间 |
| `reset-namespaces` | 重置类型解析命名空间 |

### 运行时工具

| 命令 | 说明 |
| --- | --- |
| `quit` | 退出应用；Editor 和 WebGL 不提供 |
| `max-fps` | 设置 `Application.targetFrameRate`，`-1` 不限制 |
| `vsync` | 开关垂直同步 |
| `msaa` | 设置 MSAA 采样数 |
| `time-scale` | 设置 `Time.timeScale` |
| `profile stats <bool>` | 开关 Stats 悬浮窗 |
| `capture-screenshot <filename> [superSize]` | 按 `ScreenCapture` 规则保存 PNG 截图 |

### 场景与屏幕

| 命令 | 说明 |
| --- | --- |
| `active-scene` | 查看当前活动场景 |
| `all-scenes` | 列出构建中的场景 |
| `loaded-scenes` | 列出已加载场景 |
| `load-scene` / `load-scene-index` | 按名称 / 索引加载场景 |
| `unload-scene` / `unload-scene-index` | 按名称 / 索引卸载场景 |
| `set-active-scene` | 设置活动场景 |
| `current-resolution` | 查看当前分辨率 |
| `set-resolution` | 设置分辨率 |
| `supported-resolutions` | 列出支持的分辨率；WebGL 不提供 |
| `fullscreen` | 开关全屏 |
| `screen-dpi` | 查看屏幕 DPI |
| `screen-orientation` | 设置屏幕方向；仅移动平台 |

### 对象操作

| 命令 | 说明 |
| --- | --- |
| `get-scene-hierarchy` | 查看场景层级 |
| `get-object-info` | 查看对象信息 |
| `set-active` | 设置对象启用状态 |
| `teleport` / `teleport-relative` | 设置对象位置 / 按偏移移动 |
| `rotate` | 旋转对象 |
| `set-parent` | 设置父节点 |
| `add-component` / `destroy-component` | 添加 / 销毁组件 |
| `instantiate` | 实例化对象 |
| `destroy` | 销毁对象 |
| `send-message` | 向对象发送消息 |
| `call-static` / `call-instance` | 通过反射调用静态 / 实例方法 |

## 程序调用

```csharp
using AlicizaX.Console;
using AlicizaX.Debugger;

// 打开 Debugger 并切到 Command 选项卡
DebuggerComponent.Instance.ShowFullWindow = true;
DebuggerComponent.Instance.SelectDebuggerWindow("Command");

// 通过活动控制台写输出 / 清屏
AlicizaXConsoleRouter.ActiveConsole?.LogToConsole("route message");
AlicizaXConsoleRouter.ActiveConsole?.ClearConsole();

// 不依赖 UI，直接执行命令
object result = AlicizaXConsoleProcessor.InvokeCommand("command-count");
```

`AlicizaXConsoleRouter.ActiveConsole` 在 `DebuggerComponent` 初始化 `Command` 窗口之前为 `null`。正常接入后（`Start` 注册内置窗口时）它已经可用；只执行命令、不需要 UI 时直接用 `AlicizaXConsoleProcessor`。

## 注意事项

1. `Command` 不会拦截 `Debug.Log`；日志统一在 `Console` 页查看。
2. 自定义命令所在 asmdef 必须加入 `Command Assemblies`，否则命令表里没有这些命令。
3. 命令别名和 `[CommandPrefix]` 不能包含空格、圆括号、方括号、花括号和尖括号。
4. 命令重载按“命令名 + 参数数量”区分，同名同参数数量只会注册一个。
5. 带默认参数的方法会生成多个可调用签名。例如 `Foo(int a, int b = 1)` 同时提供 `foo a` 和 `foo a b`。
6. 静态命令直接调用；非静态命令必须使用 `MonoTargetType`，普通对象只能走 `Registry`。
7. 返回 `Task` 的命令会作为 job 显示在工具栏；`Ctrl + C` 取消的是正在执行的 action 序列，不会取消已启动的 `Task`。
8. WebGL 不支持后台线程生成命令表、外部文件命令和宏文件导入导出。
