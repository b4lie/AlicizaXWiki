# 使用方式

这页说明如何把 Debugger 接进场景、操作面板、读取日志，以及注册自定义调试窗口和调整布局。

## 接入

推荐直接使用包内 prefab：

1. 将 `Packages/com.alicizax.unity.debugger/Debugger.prefab` 放入启动场景。
2. 确认场景中存在框架根节点，并且 `DebuggerComponent` 能注册到 `AppServices`。
3. 运行后默认显示浮动 `FPS` 按钮，双击按钮打开完整调试面板。
4. 在侧边栏选择 `Console` 查看 Unity 日志，选择 `Command` 进入命令控制台。

也可以手动创建 `GameObject` 并添加 `AlicizaX.Debugger.DebuggerComponent`。

`AlicizaX.Debugger` 的 asmdef 引用 `Unity.InputSystem`，工程需要安装并启用 Input System；模板项目已经包含。Demo 场景和业务场景都建议只保留一个 `DebuggerComponent`。

`DebuggerComponent.Awake` 会查找或注册 `IDebuggerService`，然后准备运行时 UI：

- 如果 `AppServices.IsAvailable` 为 `false`（框架服务容器还没创建），会记录错误并禁用组件，不再继续创建 UI。
- 运行时创建一个专用 `EventSystem`；场景中已有 `EventSystem` 时直接复用。
- 创建自己的 `UIDocument` 和 `PanelSettings`，不改动工程里已有的 UI 配置。
- `Panel Settings` 为空时，先读 `Resources/DebuggerPanelSettings`，再回退到运行时默认配置。

`OnDestroy` 会注销全部调试窗口，并销毁自己创建的 `UIDocument`、`PanelSettings`、`EventSystem` 和系统字体。

## 访问服务

```csharp
using AlicizaX;
using AlicizaX.Debugger;

IDebuggerService debugger = AppServices.Require<IDebuggerService>();
debugger.ActiveWindow = true;

DebuggerComponent.Instance.ShowFullWindow = true;
```

`AppServices.Require<IDebuggerService>()` 需要场景中存在 `DebuggerComponent`，并且已经执行过 `Awake`。没有调试器时 `DebuggerComponent.Instance` 为 `null`，调用前先判空。

## 激活模式

`DebuggerComponent` Inspector 的 `Active Window` 对应 `DebuggerActiveWindowType`：

```csharp
public enum DebuggerActiveWindowType : byte
{
    AlwaysOpen = 0,
    OnlyOpenWhenDevelopment,
    OnlyOpenInEditor,
    AlwaysClose,
}
```

| 值 | 行为 |
| --- | --- |
| `AlwaysOpen` | 始终启用调试器 |
| `OnlyOpenWhenDevelopment` | 仅 `Debug.isDebugBuild` 为 `true` 时启用 |
| `OnlyOpenInEditor` | 仅 Unity Editor 中启用 |
| `AlwaysClose` | 始终关闭 |

`Active Window` 为 `false` 时，浮动按钮和完整窗口都不显示；`DebuggerComponent` 仍然存在，`ActiveWindow` 可以随时切回 `true`。

生产包建议设置为 `OnlyOpenWhenDevelopment`、`OnlyOpenInEditor` 或 `AlwaysClose`。

## 浮动入口

- 默认只显示浮动按钮，文字为 `FPS:60` 形式的实时帧率。
- 文字颜色跟随当前最高日志级别：有 Error 变红，有 Warning 变黄；没有异常时使用普通文字色。
- 双击按钮（默认 0.3 秒内两次点击、位移不超过 24 px）打开完整窗口。
- 拖动按钮移动位置；开启 `EnableFloatingToggleSnap`（Inspector 的 `Enable Snap`）时会吸附到屏幕边缘。
- 拖动结束后短时间内不会把拖拽误判成双击。

按钮尺寸固定为 `180 × 56`，位置会被限制在屏幕内。

## 完整窗口

打开后由标题栏、侧边栏和内容区组成：

- 拖动标题栏移动窗口。
- 拖动右下角调整窗口大小，最小 `420 × 320`。
- 点击标题栏 `R`（Reset Layout）恢复默认布局。
- 点击标题栏 `-`（Close）关闭完整窗口，保留浮动入口。
- 侧边栏按路径分组显示窗口，点击末级项切换内容。

如下状态会写入 `PlayerPrefs`：

| 内容 | Key |
| --- | --- |
| 浮动按钮位置 | `Debugger.Icon.X`、`Debugger.Icon.Y` |
| 吸附开关 | `Debugger.Icon.Snap` |
| 窗口位置和尺寸 | `Debugger.Window.X`、`Debugger.Window.Y`、`Debugger.Window.Width`、`Debugger.Window.Height` |
| 窗口缩放 | `Debugger.Window.Scale` |
| 日志过滤与锁定 | `Debugger.Console.LockScroll`、`Debugger.Console.InfoFilter` 等 |

## 布局控制

```csharp
using AlicizaX.Debugger;
using UnityEngine;

public sealed class DebuggerLayoutExample : MonoBehaviour
{
    private void Start()
    {
        DebuggerComponent debugger = DebuggerComponent.Instance;
        if (debugger == null)
        {
            return;
        }

        debugger.IconRect = new Rect(24f, 24f, 180f, 56f);
        debugger.WindowRect = new Rect(24f, 96f, 1320f, 760f);
        debugger.EnableFloatingToggleSnap = true;
        debugger.WindowOpacity = 0.9f;
        debugger.WindowScale = 1.2f;

        // 需要恢复默认布局时调用。
        // debugger.ResetLayout();
    }
}
```

约束：

- `WindowOpacity` 限制在 `0.2` 到 `1`。
- `WindowScale` 最小 `0.5`，没有上限。
- `IconRect` 的宽高会被强制改成默认值，只有位置生效。
- `WindowRect` 的宽高不小于 `420 × 320`，位置会被限制在屏幕范围内。
- `ResetLayout()` 把按钮、窗口和缩放恢复默认值，并立即写入 `PlayerPrefs`。

## 日志面板

`Console` 窗口监听 `Application.logMessageReceived`，缓存最近日志并显示：

- Info / Warning / Error / Fatal 计数。
- 类型过滤开关和锁定滚动。
- 选中日志的堆栈信息。
- `Copy Selected` 复制选中日志和堆栈。
- `Clear` 清空缓存。

日志窗口默认最多保留 100 条，类型过滤和锁定状态会持久化。

读取最近日志：

```csharp
using System.Collections.Generic;
using AlicizaX.Debugger;
using UnityEngine;

public sealed class DebugLogExportExample
{
    private readonly List<DebuggerComponent.LogNode> _logs = new();

    public void DumpRecentLogs()
    {
        DebuggerComponent debugger = DebuggerComponent.Instance;
        if (debugger == null)
        {
            return;
        }

        _logs.Clear();
        debugger.GetRecentLogs(_logs, 50);

        foreach (DebuggerComponent.LogNode log in _logs)
        {
            Debug.Log($"{log.LogType}: {log.LogMessage}");
        }
    }
}
```

`LogNode` 来自框架 `MemoryPool`，`GetRecentLogs` 返回的是只读引用。调用方只能当前帧读取，不得跨帧缓存或手动 `Release`：`Console` 清空或裁剪日志后，节点会归还池并被复用。

`LogNode` 字段：

| 字段 | 含义 |
| --- | --- |
| `LogTime` | 日志时间，UTC |
| `LogFrameCount` | 记录时的 `Time.frameCount` |
| `LogType` | Unity `LogType` |
| `LogMessage` | 日志正文 |
| `StackTrack` | 堆栈信息 |
| `DisplayText` | 列表单行文本，包含本地时间和帧号；首次读取时生成并缓存 |

## Stats 悬浮窗

Stats 悬浮窗默认关闭，显示在屏幕右上角，内容包括：

- FPS 和帧耗时
- 三角面数、Batches、DrawCall、SetPass
- Mono 堆、总分配 / 保留内存、显卡驱动内存

通过命令控制台开启：

```text
profile stats true
profile stats false
```

或直接调用 API：

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
debugger.SetStatsOverlayVisible(true);
bool visible = debugger.IsStatsOverlayVisible;
```

悬浮窗不依赖完整调试窗口，只要求 `DebuggerComponent` 存在。部分 DrawCall / 三角面数据依赖 Unity Profiler Recorder，取不到时对应项显示为不可用。

## 注册自定义窗口

自定义窗口实现 `IDebuggerWindow`，由 `CreateView()` 返回 UI Toolkit 视图：

```csharp
using AlicizaX.Debugger;
using UnityEngine;
using UnityEngine.UIElements;

public sealed class PlayerDebugWindow : IDebuggerWindow
{
    private Label _label;

    public void Initialize(params object[] args)
    {
    }

    public void Shutdown()
    {
    }

    public void OnEnter()
    {
        Refresh();
    }

    public void OnLeave()
    {
    }

    public void OnUpdate(float elapseSeconds, float realElapseSeconds)
    {
        Refresh();
    }

    public VisualElement CreateView()
    {
        VisualElement root = new VisualElement();
        _label = new Label();
        root.Add(_label);
        Refresh();
        return root;
    }

    private void Refresh()
    {
        if (_label != null)
        {
            _label.text = $"Frame: {Time.frameCount}";
        }
    }
}
```

注册和注销：

```csharp
using AlicizaX.Debugger;
using UnityEngine;

public sealed class RegisterPlayerDebugWindow : MonoBehaviour
{
    private void Start()
    {
        DebuggerComponent debugger = DebuggerComponent.Instance;
        if (debugger == null)
        {
            return;
        }

        debugger.RegisterDebuggerWindow(
            "Gameplay/Player",
            new PlayerDebugWindow());
    }

    private void OnDestroy()
    {
        if (DebuggerComponent.Instance != null)
        {
            DebuggerComponent.Instance.UnregisterDebuggerWindow("Gameplay/Player");
        }
    }
}
```

### 自动注册（标签）

给窗口类型加 `[DebuggerWindow]`，调试器启动时会扫描并自动注册，不需要再写注册代码：

```csharp
using AlicizaX.Debugger;

[DebuggerWindow("Gameplay/Player")]
public sealed class PlayerDebugWindow : IDebuggerWindow
{
    // 实现与上面的手动版本相同
}
```

扫描规则：

- 本包程序集始终扫描；其它程序集需要填到 `DebuggerComponent` 的 `Auto Register Assemblies`（序列化字段 `m_AutoRegisterAssemblyNames`）。
- 标签路径为空时回退为 `Auto/<类名>`。
- 额外程序集里没有标签的窗口同样回退到 `Auto/<类名>`；本包中需要自动注册的内置窗口必须显式声明路径，未加标签的内部窗口不会被扫描注册。
- 自动扫描在 `DebuggerComponent.Start` 执行。扫描前已经手动注册的路径优先；扫描到重复路径时保留先注册的一方并输出 warning。扫描之后再手动注册同一路径会抛 `GameFrameworkException`。
- 自动注册的窗口必须有无参构造（非 public 也可以）；需要 `Initialize` 参数或依赖 `[SerializeField]` 配置的窗口继续用手动注册。
- 自动扫描依赖内置 `DebuggerService`；换成自定义 `IDebuggerService` 实现时扫描会跳过并输出 warning。
- IL2CPP 会裁剪“只被反射引用”的类型，扩展程序集必须加 `link.xml`（`<assembly fullname="Your.Gameplay" preserve="all" />`）或 `[Preserve]`，否则发布包里窗口会消失。本包通过 `Runtime/Debugger/link.xml` 保留自身程序集。

### 手动注册

规则：

- 路径用 `/` 分组，会显示为侧边栏树形菜单；侧边栏每一级按名称排序。
- `RegisterDebuggerWindow` 会把额外参数传给窗口的 `Initialize`。
- 路径不能与已有窗口重复，重复注册会抛 `GameFrameworkException`；路径为空或窗口为 `null` 也会抛异常。
- `CreateView()` 在窗口被选中时调用；`OnUpdate` 只在该窗口处于活动状态且调试器启用时调用。
- 注销会移除侧边栏节点；如果该节点下没有其它窗口，空的父分组会一并移除；如果当前正显示该窗口，内容区会被清空。

## 字体与缩放

- `CustomFont` 对应 UI Toolkit 的回退 `Font`，兼容旧字体资源。
- `CustomFontAsset` 使用 `UnityEngine.TextCore.Text.FontAsset`，适合已有 TextCore / TMP 字库资源的项目。
- `WindowScale` 改变整个面板的 UI 尺寸，不影响业务 UI。
- 未指定字体时，按平台从系统字体中查找（Windows 依次尝试 `Segoe UI`、`Microsoft YaHei UI`，macOS 优先 `PingFang SC`，Android 依次尝试 `Roboto`、`Noto Sans CJK SC`），最后回退到 Unity 内置字体。

## 注意事项

1. `DebuggerComponent.Start()` 会强制 `ShowFullWindow = false`，不要在 prefab 里期待面板默认展开；需要展开时在业务启动代码里设 `ShowFullWindow = true`。
2. 生产包关闭调试器时，除了设置 `Active Window`，也要确认没有代码在运行时把 `ActiveWindow` 设回 `true`。
3. 布局和日志过滤会写入 `PlayerPrefs`，排查显示异常时先用 `ResetLayout()` 恢复默认布局。
4. `LogNode` 是池化对象，不要跨帧保存引用。
5. `DebuggerComponent` 依赖框架服务容器；没有框架根节点时 `AppServices.IsAvailable` 为 `false`，组件会在 `Awake` 记录错误并禁用。
