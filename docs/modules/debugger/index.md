# Debugger 调试器

`com.alicizax.unity.debugger`（程序集 `AlicizaX.Debugger`）是框架的可选运行时调试包：一个基于 UI Toolkit 的面板，把 Unity 日志、运行时信息、Profiler 采样和一套反射式命令控制台收进同一个窗口。它不参与业务逻辑，只在开发期提供观察和操作入口，生产包可以通过激活模式整体关闭。

源码位置：

- `Client/Packages/com.alicizax.unity.debugger/Runtime/Debugger`：`DebuggerComponent`、`DebuggerService`、窗口树、内置窗口
- `Client/Packages/com.alicizax.unity.debugger/Runtime/Console`：`AlicizaX.Console` 命令系统
- `Client/Packages/com.alicizax.unity.debugger/Editor`：Inspector 与命令配置编辑器
- `Client/Packages/com.alicizax.unity.debugger/Debugger.prefab`：只挂载 `DebuggerComponent` 的接入 prefab

## 适用场景

- 在编辑器或真机上查看 `Debug.Log` / Warning / Error，按级别过滤，查看选中日志的堆栈。
- 查看运行时环境：设备系统、屏幕、图形 API、输入设备、场景、时间、质量等级。
- 采样内存占用，观察引用池、音频、计时器的运行时状态。
- 通过命令控制台改时间缩放、分辨率、场景，或对场景对象执行传送、销毁、加组件等操作。
- 需要把自己的调试页接进同一套侧边栏，或打开右上角的 Stats 性能悬浮窗。

## 使用前提

1. Unity 2022.3 及以上（见 `package.json` 的 `unity` 字段），且工程启用了运行时 UI Toolkit（`UIDocument`、`PanelSettings`）。
2. 场景中的框架根节点已经创建服务容器。`DebuggerComponent.Awake` 会先判断 `AppServices.IsAvailable`，为 `false` 时输出 `Debugger requires an initialized service registry. Component disabled.` 并直接把组件 `enabled = false`，不会创建任何 UI。
3. `DebuggerComponent` 会从容器取 `IDebuggerService`，取不到时自己 `AppServices.Register<IDebuggerService>(new DebuggerService())`，因此不需要额外注册服务。
4. 场景里只放一个 `DebuggerComponent`（类上有 `[DisallowMultipleComponent]`）。
5. `com.unity.inputsystem` 是可选依赖。未安装时命令面板的按键走旧 `Input`；`Active Input Handling` 选 `Input System Package (New)` 时必须安装，否则热键不响应。

## 最短接入

1. 先确认框架根节点（模板里的 `Entry`）存在，服务容器可用。
2. 把 `Client/Packages/com.alicizax.unity.debugger/Debugger.prefab` 拖进启动场景。根对象名为 `Debugger`，只挂 `DebuggerComponent`。模板工程的 `Client/Assets/Scenes/Main.unity` 就是这种挂法：`Debugger` 是 `Debugger.prefab` 的根级实例，**不在 `Entry.prefab` 里**（`Entry.prefab` 只包含 UI、Memory Pool、Entry、Resources、Audio、GameObjectPool、Scene、Timer）。
3. 没有用 prefab 时，在场景里给任意激活物体添加 `AlicizaX.Debugger.DebuggerComponent`（菜单 `Game Framework/Debugger`），或用代码挂载：

```csharp
using AlicizaX.Debugger;
using UnityEngine;

public sealed class DebuggerBootstrap : MonoBehaviour
{
    private void Start()
    {
        // 场景里已经放了 Debugger.prefab 时不重复创建。
        if (DebuggerComponent.Instance != null)
        {
            return;
        }

        new GameObject("Debugger").AddComponent<DebuggerComponent>();
    }
}
```

4. 运行后默认只显示浮动入口（`FPS:60` 形式的按钮），双击打开完整窗口，再在侧边栏选页。用代码直接打开某一页：

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
if (debugger != null)
{
    debugger.ActiveWindow = true;             // 等价于 IDebuggerService.ActiveWindow = true
    debugger.ShowFullWindow = true;           // 显示完整窗口
    debugger.SelectDebuggerWindow("Console"); // 切到日志页
}
```

## 程序集与宏依赖

| 程序集 | 平台 | 依赖与说明 |
| --- | --- | --- |
| `AlicizaX.Debugger` | 全平台 | 引用 `AlicizaX.Framework.Runtime`、`ZString`、`Unity.InputSystem`；`autoReferenced`，开启 `allowUnsafeCode` |
| `AlicizaX.Debugger.Editor` | 仅 Editor | `DebuggerComponentInspector`、`ModifierKeyComboField`、`Editor/Console` 编辑器工具 |

| 宏 | 来源 | 作用 |
| --- | --- | --- |
| `INPUTSYSTEM_SUPPORT` | asmdef `versionDefines`（`com.unity.inputsystem` ≥ 1.0.0） | `InputKeys` 走 `UnityEngine.InputSystem` 分支；未定义时回退旧 `Input`，`Both` 模式仍用旧 `Input` |
| `AlicizaXConsole_DISABLED` | 手动定义 | 整体关闭 `Extras` 下的内置命令 |
| `AlicizaXConsole_DISABLE_BUILTIN_ALL` | 手动定义 | 同上，内置命令不参与编译 |
| `AlicizaXConsole_DISABLE_BUILTIN_EXTRA` | 手动定义 | 同上，并让命令扫描规则跳过 `AlicizaX.Console.Extra` |
| `UNITY_2020_2_OR_NEWER` | Unity 版本 | Stats 悬浮窗用 `ProfilerRecorder` 读取三角面 / DrawCall 等数据 |

## 内置窗口一览

窗口路径用 `/` 分组。下表是源码中真实注册的路径与实现类。

| 路径 | 实现类 | 内容 | 注册方式 |
| --- | --- | --- | --- |
| `Console` | `ConsoleWindow` | Unity 日志、过滤、堆栈 | 显式 |
| `Command` | `CommandWindow` | 命令控制台 | 显式 |
| `Information/Input` | `InputInformationWindow` | 鼠标、键盘、触摸、输入设备 | 显式 |
| `Information/Other/Time` | `TimeInformationWindow` | `timeScale`、各类时间与帧计数 | 显式 |
| `Information/Environment` | `EnvironmentInformationWindow` | 应用、Unity 版本、平台、语言 | `[DebuggerWindow]` 自动 |
| `Information/Graphics` | `GraphicsInformationWindow` | 显卡、图形 API、Shader Level | `[DebuggerWindow]` 自动 |
| `Information/Screen` | `ScreenInformationWindow` | 分辨率、DPI、方向、安全区 | `[DebuggerWindow]` 自动 |
| `Information/System` | `SystemInformationWindow` | 设备、系统、处理器、内存、电池 | `[DebuggerWindow]` 自动 |
| `Information/Other/Scene` | `SceneInformationWindow` | 场景数量与活动场景 | `[DebuggerWindow]` 自动 |
| `Information/Other/Quality` | `QualityInformationWindow` | 质量等级与渲染设置 | `[DebuggerWindow]` 自动 |
| `Profiler/Summary` | `ProfilerInformationWindow` | Profiler 开关、Mono 堆、内存统计 | 显式 |
| `Profiler/Memory/Summary` | `RuntimeMemorySummaryWindow` | 按类型汇总的内存采样 | 显式 |
| `Profiler/Memory/All` | `RuntimeMemoryInformationWindow<Object>` | 全对象内存采样 | 显式 |
| `Profiler/Memory/Texture` | `RuntimeMemoryInformationWindow<Texture>` | 纹理内存采样 | 显式 |
| `Profiler/Memory/Mesh` | `RuntimeMemoryInformationWindow<Mesh>` | Mesh 内存采样 | 显式 |
| `Profiler/Memory/Material` | `RuntimeMemoryInformationWindow<Material>` | Material 内存采样 | 显式 |
| `Profiler/Memory/Shader` | `RuntimeMemoryInformationWindow<Shader>` | Shader 内存采样 | 显式 |
| `Profiler/Memory/AnimationClip` | `RuntimeMemoryInformationWindow<AnimationClip>` | 动画片段内存采样 | 显式 |
| `Profiler/Memory/AudioClip` | `RuntimeMemoryInformationWindow<AudioClip>` | 音频片段内存采样 | 显式 |
| `Profiler/Memory/Font` | `RuntimeMemoryInformationWindow<Font>` | 字体内存采样 | 显式 |
| `Profiler/Memory/TextAsset` | `RuntimeMemoryInformationWindow<TextAsset>` | 文本资源内存采样 | 显式 |
| `Profiler/Memory/ScriptableObject` | `RuntimeMemoryInformationWindow<ScriptableObject>` | 配置资源内存采样 | 显式 |
| `Profiler/Reference Pool` | `ReferencePoolInformationWindow` | 框架 `MemoryPool` 各池统计 | 显式 |
| `Profiler/Audio` | `AudioInformationWindow` | 音频分组与 agent | 显式 |
| `Profiler/Timer` | `TimerInformationWindow` | 计时器池与计时器 | 显式 |
| `Other/Settings` | `SettingsWindow` | 吸附开关、窗口宽高、缩放与预设 | 显式 |

不注册为窗口、只服务于界面的嵌套类型：`StatsOverlay`（性能悬浮窗）、`LogNode`（日志节点，公开嵌套类型）、`FpsCounter`（浮动入口的 FPS 统计）、`ScrollableDebuggerWindowBase` / `PollingDebuggerWindowBase`（滚动式与轮询式窗口基类）、`DebuggerTheme`（配色常量）。

`Profiler/Memory/*` 页需要先点 `Take Sample` 才显示对象数量与内存占用。

## 激活模式

`Active Window` 字段是 `DebuggerActiveWindowType`：

| 值 | 含义 | `Start` 时的结果 |
| --- | --- | --- |
| `AlwaysOpen` | 总是打开 | `ActiveWindow = true` |
| `OnlyOpenWhenDevelopment` | 仅在开发模式打开 | `ActiveWindow = Debug.isDebugBuild` |
| `OnlyOpenInEditor` | 仅在编辑器中打开 | `ActiveWindow = Application.isEditor` |
| `AlwaysClose` | 总是关闭 | `ActiveWindow = false` |

`ApplyActiveMode()` 只在 `Start` 执行一次。运行期改 `DebuggerComponent.ActiveWindow` 会直接写 `IDebuggerService.ActiveWindow`，同时同步组件 `enabled` 和窗口可见性，可以随时切回 `true`。

## 与 IDebuggerService 的关系

- `IDebuggerService : IService` 只有六个成员：`ActiveWindow`、`DebuggerWindowRoot`、`RegisterDebuggerWindow`、`UnregisterDebuggerWindow`、`GetDebuggerWindow`、`SelectDebuggerWindow`。
- `DebuggerComponent.ActiveWindow` 是 `IDebuggerService.ActiveWindow` 的转发；窗口树的唯一数据源是 `DebuggerWindowRoot`（类型 `IDebuggerWindowGroup`），路径查找、选中和侧边栏渲染共用它。
- 需要通过服务访问时用 `AppServices.Require<IDebuggerService>()`；服务由 `DebuggerComponent.Awake` 注册，所以必须已经执行过 `Awake`。容器机制见 [Service 模块](../service/index.md)。
- 换成自定义 `IDebuggerService` 实现时，`[DebuggerWindow]` 自动扫描会跳过并输出 `Debugger: auto-registration skipped, IDebuggerService is not the built-in implementation.`；内置窗口仍按接口注册。
- 容器销毁时 `DebuggerService.OnDispose` 会把整棵窗口树 `Shutdown()`。

## 关闭与重建

| 操作 | 行为 |
| --- | --- |
| `ActiveWindow = false` | 服务标志置 `false`，组件 `enabled = false`（`Update` 停止），浮动入口与窗口全部隐藏；组件、窗口树、日志缓存保留 |
| `ShowFullWindow = false` | 只收起完整窗口，回到浮动入口（标题栏 `-` 按钮同一路径） |
| 销毁组件 | `OnDestroy` 会先 `OnLeave` 当前窗口，再遍历窗口树逐条 `UnregisterDebuggerWindow`，然后释放 `StatsOverlay`，销毁自建的 `UIDocument`、`PanelSettings`、`PanelTextSettings`、系统字体和 `EventSystem`，最后清空 `DebuggerComponent.Instance` |

注销会一并 `Shutdown` 窗口，所以同一服务容器内重建组件不会再撞 `Debugger window has been registered.`。`IDebuggerService` 本身留在容器里，`ActiveWindow = false` 只是静音。

## 常见配置

| Inspector | 序列化字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Active Window | `m_ActiveWindow` | `AlwaysOpen` | 激活模式 |
| Panel Settings | `m_PanelSettings` | 包内 `Runtime/Debugger/Resources/DebuggerPanelSettings.asset` | 运行时会克隆一份，排序 `short.MaxValue - 64` |
| Font | `m_Font` | 空 | 回退字体；模板 `Main.unity` 覆盖为 `Assets/Art/Font/CHUNBAI.ttf` |
| Enable Snap | `m_EnableFloatingToggleSnap` | `true` | 浮动入口贴边吸附 |
| Window Opacity | `m_WindowOpacity` | `1` | 取值范围 `0.2`–`1` |
| Auto Register Assemblies | `m_AutoRegisterAssemblyNames` | 空数组 | 额外扫描 `[DebuggerWindow]` 的程序集名 |
| Command 区域 | `m_CommandWindow.*` | 见[命令控制台](command.md) | 提交键、补全、历史等 |

未在 Inspector 显示但会序列化的字段：`m_ShowFullWindow`（`Start` 会强制置 `false`）、`m_FontAsset`（对应 `CustomFontAsset`）、`m_ConsoleWindow`（`m_MaxLine`、过滤开关、日志配色）。

## 容易踩的坑

1. 服务容器没起来时组件在 `Awake` 就禁用，界面上完全看不到 Debugger；先确认框架根节点的初始化顺序。
2. `Start()` 会强制 `ShowFullWindow = false`，在 prefab 里勾选展开不会生效，需要展开就在业务启动代码里设置。
3. 组件会自己创建 `EventSystem`（场景没有时）和 `UIDocument`，排序固定为 `32703`；场景里有多套 UI 时注意输入抢占与排序。
4. 生产包只把 `Active Window` 调成 `AlwaysClose` 还不够，要确认运行时代码没有把 `ActiveWindow` 设回 `true`。
5. `GetRecentLogs` 返回的是 `MemoryPool` 池化的 `LogNode`，只能当帧读取，不能跨帧保存。
6. 只有 `Console` 窗口采集 `Debug.Log`；`Command` 窗口不接管 Unity 日志。

## 相关页面

| 页面 | 内容 |
| --- | --- |
| [启动、窗口树与日志](usage.md) | 挂载细节、Inspector 字段、浮动入口与全屏窗口、窗口树与自动注册、日志与 Stats 悬浮窗 |
| [命令控制台](command.md) | 命令扫描、参数解析、内置命令、宏、程序内调用 |
| [API 速查](api.md) | 公开类型与成员速查表 |
| [快速开始](../../quickstart.md) | 工程接入流程 |
