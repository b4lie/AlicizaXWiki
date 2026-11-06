# API 速查

## DebuggerComponent

| API | 说明 |
| --- | --- |
| `DebuggerComponent.Instance` | 当前调试器组件；场景中没有时返回 `null` |
| `ActiveWindow` | 是否启用调试器，对应 `IDebuggerService.ActiveWindow` |
| `ShowFullWindow` | 是否显示完整窗口；关闭后保留浮动入口 |
| `WindowOpacity` | 窗口透明度，限制在 `0.2` 到 `1` |
| `WindowScale` | 窗口缩放，最小 `0.5` |
| `CustomFont` | 自定义 `Font` |
| `CustomFontAsset` | 自定义 `UnityEngine.TextCore.Text.FontAsset` |
| `EnableFloatingToggleSnap` | 浮动入口是否吸附屏幕边缘 |
| `IconRect` | 浮动入口位置；宽高固定为 `180 × 56` |
| `WindowRect` | 完整窗口位置和尺寸；最小 `420 × 320` |
| `ResetLayout()` | 恢复默认布局并保存 |
| `GetRecentLogs(List<LogNode> results)` | 获取全部缓存的日志节点 |
| `GetRecentLogs(List<LogNode> results, int count)` | 获取最近 `count` 条日志 |
| `RegisterDebuggerWindow(string path, IDebuggerWindow window, params object[] args)` | 注册调试窗口 |
| `UnregisterDebuggerWindow(string path)` | 注销调试窗口，返回是否成功 |
| `GetDebuggerWindow(string path)` | 获取调试窗口 |
| `SelectDebuggerWindow(string path)` | 选中调试窗口，返回是否成功 |
| `Auto Register Assemblies`（`m_AutoRegisterAssemblyNames`） | 额外扫描 `[DebuggerWindow]` 的程序集名；本包程序集始终扫描 |
| `SetStatsOverlayVisible(bool visible)` | 开关 Stats 悬浮窗 |
| `IsStatsOverlayVisible` | Stats 悬浮窗当前是否可见 |

## DebuggerWindowAttribute

| API | 说明 |
| --- | --- |
| `DebuggerWindowAttribute(string path)` | 标记实现 `IDebuggerWindow` 的类型可自动注册，并声明侧栏路径 |
| `Path` | 侧栏路径；为空时回退为 `Auto/<类名>` |

## IDebuggerService

| API | 说明 |
| --- | --- |
| `ActiveWindow` | 调试器窗口是否激活 |
| `DebuggerWindowRoot` | 调试窗口根节点（`IDebuggerWindowGroup`）；路径查找与侧边栏渲染共用这一棵树 |
| `RegisterDebuggerWindow(path, window, args)` | 注册窗口 |
| `UnregisterDebuggerWindow(path)` | 注销窗口 |
| `GetDebuggerWindow(path)` | 获取窗口 |
| `SelectDebuggerWindow(path)` | 选中窗口 |

通过 `AppServices.Require<IDebuggerService>()` 或 `AppServices.TryGet<IDebuggerService>(out ...)` 获取。

## IDebuggerWindow

| 方法 | 说明 |
| --- | --- |
| `Initialize(params object[] args)` | 注册时调用，接收 `RegisterDebuggerWindow` 的参数 |
| `Shutdown()` | 调试器销毁时调用 |
| `OnEnter()` | 切换到该窗口时调用 |
| `OnLeave()` | 离开该窗口时调用 |
| `OnUpdate(float elapseSeconds, float realElapseSeconds)` | 窗口活动时每帧调用 |
| `CreateView()` | 返回 UI Toolkit 的 `VisualElement` |

## 命令路由

| API | 说明 |
| --- | --- |
| `AlicizaXConsoleRouter.ActiveConsole` | 当前活动控制台（`IAlicizaXConsole`）；Debugger 初始化 `Command` 窗口前为 `null` |
| `ActiveConsole.LogToConsole(text, bool newLine = true)` | 写入命令输出 |
| `ActiveConsole.ClearConsole()` | 清空命令输出 |
| `ActiveConsole.InvokeExternalCommandsAsync(path)` | 执行外部命令文件 |
| `IAlicizaXConsole.IsActive` | 控制台是否处于活动状态 |
| `IAlicizaXConsole.VerboseErrors` | 详细错误输出 |
| `IAlicizaXConsole.VerboseLogging` | 详细日志阈值 |
| `IAlicizaXConsole.LoggingLevel` | 日志拦截阈值 |
| `IAlicizaXConsole.MaxStoredLogs` | 输出保留条数 |

## 命令表

| API | 说明 |
| --- | --- |
| `AlicizaXConsoleProcessor.DefaultCommandAssemblyName` | 默认扫描程序集，值为 `AlicizaX.Debugger` |
| `AlicizaXConsoleProcessor.GenerateCommandTable(bool deployThread = false, bool forceReload = false)` | 扫描已加载程序集生成命令表 |
| `AlicizaXConsoleProcessor.GenerateCommandTableFromAssemblyNames(IEnumerable<string> assemblyNames, bool deployThread = false, bool forceReload = false)` | 按程序集名生成命令表 |
| `AlicizaXConsoleProcessor.InvokeCommand(string commandString)` | 不经过 UI 执行命令，返回命令结果 |
| `AlicizaXConsoleProcessor.GetAllCommands()` | 获取所有命令 |
| `AlicizaXConsoleProcessor.GetUniqueCommands()` | 获取按名称去重后的命令 |
| `AlicizaXConsoleProcessor.LoadedCommandCount` | 已加载命令数量 |
| `AlicizaXConsoleProcessor.TableGenerated` | 命令表是否已生成 |

## Registry

| API | 说明 |
| --- | --- |
| `AlicizaXConsoleRegistry.RegisterObject<T>(T obj)` | 注册 `Registry` 命令目标 |
| `AlicizaXConsoleRegistry.DeregisterObject<T>(T obj)` | 注销 `Registry` 命令目标 |
| `AlicizaXConsoleRegistry.GetRegistryContents<T>()` | 枚举已注册实例 |
| `AlicizaXConsoleRegistry.GetRegistrySize<T>()` | 已注册实例数量 |
| `AlicizaXConsoleRegistry.ClearRegistryContents<T>()` | 清空指定类型的注册表 |

## 宏

| API | 说明 |
| --- | --- |
| `AlicizaXConsoleMacros.DefineMacro(name, expansion)` | 定义宏 |
| `AlicizaXConsoleMacros.RemoveMacro(name)` | 移除宏 |
| `AlicizaXConsoleMacros.ClearMacros()` | 清空宏 |
| `AlicizaXConsoleMacros.GetMacros()` | 获取宏表 |
| `AlicizaXConsoleMacros.ExpandMacros(text, maximumExpansions = 1000)` | 展开文本中的宏 |
| `AlicizaXConsoleMacros.DumpMacrosToFile(path)` | 导出宏；WebGL 不可用 |
| `AlicizaXConsoleMacros.LoadMacrosFromFile(path)` | 导入宏；WebGL 不可用 |

## 类型解析

| API | 说明 |
| --- | --- |
| `AlicizaXConsoleParser.ParseType(string typeName)` | 解析类型名 |
| `AlicizaXConsoleParser.AddNamespace(string namespaceName)` | 添加类型解析命名空间 |
| `AlicizaXConsoleParser.RemoveNamespace(string namespaceName)` | 移除类型解析命名空间 |
| `AlicizaXConsoleParser.ResetNamespaceTable()` | 重置命名空间表 |
| `AlicizaXConsoleParser.GetAllNamespaces()` | 获取当前命名空间表 |

## 注意事项

1. 独立命令浮窗 `AlicizaXConsoleUITK` 已移除，命令 UI 统一在 Debugger 的 `Command` 页。
2. `Command` 页不拦截 `Debug.Log`；日志查看使用 `Console` 页。
3. 自定义命令所在 asmdef 必须被命令表扫描，否则 `InvokeCommand` 和 `Command` 页都找不到命令。
4. `LogNode` 来自 `MemoryPool`，`GetRecentLogs` 返回只读引用，不要手动释放或跨帧保存。
5. WebGL 不支持后台线程生成命令表、外部命令文件以及宏文件导入导出。
6. `AlicizaXConsoleRouter.ActiveConsole` 在 `DebuggerComponent` 初始化 `Command` 窗口前为 `null`；只执行命令、不需要 UI 时用 `AlicizaXConsoleProcessor.InvokeCommand`。
7. `[DebuggerWindow]` 自动注册依赖反射发现类型：扩展程序集在 IL2CPP 下必须用 `link.xml` 或 `[Preserve]` 保底，且窗口需要无参构造；路径冲突时保留先注册的一方并打印 warning。
