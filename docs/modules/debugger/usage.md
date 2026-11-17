# 启动、窗口树与日志

本页讲 `DebuggerComponent` 的挂载与 Inspector 字段、浮动入口和全屏窗口的操作方式、窗口路径树与自动注册规则，以及日志采集和 Stats 悬浮窗。命令控制台单独见[命令控制台](command.md)。

## 适用场景

- 第一次把 Debugger 接进场景，需要知道组件挂在哪、字段怎么填。
- 需要在运行时用代码开关面板、切换页面、调整布局。
- 需要把自己的调试页接进侧边栏，或批量读取最近的日志做上报。
- 排查“浮动按钮点不开”“窗口不出现”“日志只有 100 条”这类现象。

## 使用前提

1. 框架服务容器已初始化（否则 `DebuggerComponent.Awake` 会禁用组件）。
2. 场景中只有一个 `DebuggerComponent`；它自己负责创建 `UIDocument`、`PanelSettings`，场景没有 `EventSystem` 时会补一个。
3. 工程已包含 UI Toolkit 运行时模块。

## 最短接入

1. 把 `Client/Packages/com.alicizax.unity.debugger/Debugger.prefab` 放进启动场景（模板的 `Client/Assets/Scenes/Main.unity` 就是这个做法）。
2. 检查 Inspector 的 `Active Window`：开发期保持 `AlwaysOpen`。
3. 运行后**双击**屏幕上的 `FPS:xx` 按钮打开窗口，侧边栏点 `Console`。
4. 用代码完成同样的动作：

```csharp
using AlicizaX.Debugger;
using UnityEngine;

public sealed class DebuggerOpenPanel : MonoBehaviour
{
    private void Start()
    {
        DebuggerComponent debugger = DebuggerComponent.Instance;
        if (debugger == null)
        {
            return;
        }
        debugger.ShowFullWindow = true;
        debugger.SelectDebuggerWindow("Console");
    }
}
```

`SelectDebuggerWindow` 会顺带把 `ShowFullWindow` 置为 `true`；路径不存在时返回 `false`，不抛异常。

## Inspector 字段

| Inspector | 序列化字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| Active Window | `m_ActiveWindow` | `DebuggerActiveWindowType` | `AlwaysOpen` | 激活模式 |
| Panel Settings | `m_PanelSettings` | `PanelSettings` | 包内 `DebuggerPanelSettings` | 运行时克隆；排序 `short.MaxValue - 64`，`ConstantPixelSize`，`scale = 1` |
| Font | `m_Font` | `Font` | 空 | 回退字体；模板场景覆盖为 `CHUNBAI.ttf` |
| Enable Snap | `m_EnableFloatingToggleSnap` | `bool` | `true` | 浮动入口贴边吸附 |
| Window Opacity | `m_WindowOpacity` | `float` | `1` | 范围 `0.2`–`1` |
| Auto Register Assemblies | `m_AutoRegisterAssemblyNames` | `string[]` | 空 | 额外扫描 `[DebuggerWindow]` 的程序集名 |
| Command 区域 | `m_CommandWindow.*` | — | — | 见[命令控制台](command.md) |

会序列化但不在 Inspector 显示的字段：`m_ShowFullWindow`（`Start` 强制 `false`）、`m_FontAsset`（对应 `CustomFontAsset`）、`m_ConsoleWindow`（日志条数、过滤、配色）。窗口尺寸与缩放不在 Inspector，放在运行时 `Other/Settings` 页。

## 浮动入口与全屏窗口

浮动入口：

- 默认位置 `(24, 24)`，尺寸固定 `180 × 56`，位置被限制在屏幕内（按分辨率缩放换算）。
- 文本是 `FPS:{0:F0}` 形式的实时帧率；颜色按当前最高日志级别取，顺序是 `Exception` → `Error` → `Warning` → `Log`。
- **双击**打开完整窗口：两次点击间隔不超过 `0.3` 秒、位移不超过 `24`；单击不打开。
- 拖动超过 `10` 个单位才算拖拽；拖着走时位置实时更新，松手后 `0.35` 秒内不会把这次操作判成双击。
- 开启吸附时，松手后用 `SmoothDamp`（平滑时间 `0.08` 秒）贴到最近的屏幕边。

全屏窗口：

- 默认矩形 `(24, 96)`、尺寸 `1320 × 760`、缩放 `1.2`；最小尺寸 `420 × 320`。
- 标题栏左侧显示当前窗口路径（没有选中时显示 `ALICIZAX DEBUGGER`）；右侧 `R` 恢复默认布局，`-` 收起窗口回到浮动入口。
- 拖动标题栏移动窗口；拖动右下角手柄改尺寸。
- 侧边栏宽 `266`，按路径分层显示；每级按名称序数排序，深度 0 的分组默认展开，选中项自动滚动到可见位置。

状态会写入 `PlayerPrefs`：浮动入口位置 `Debugger.Icon.X` / `Debugger.Icon.Y`，吸附 `Debugger.Icon.Snap`，窗口位置与尺寸 `Debugger.Window.X` / `Y` / `Width` / `Height`，缩放 `Debugger.Window.Scale`，日志开关 `Debugger.Console.LockScroll` / `InfoFilter` / `WarningFilter` / `ErrorFilter` / `FatalFilter`。

## 窗口树

窗口按 `/` 分段的路径注册，路径树由 `IDebuggerService.DebuggerWindowRoot`（`IDebuggerWindowGroup`）持有。侧边栏、`GetDebuggerWindow`、`SelectDebuggerWindow` 都走同一棵树。

- `DebuggerWindowCount` 是当前层子节点数，`SelectedIndex` / `SelectedWindow` 是当前选中的子节点，`GetDebuggerWindowNames()` 返回当前层名称。
- 空路径或 `null` 窗口会被拒绝；路径里的空段会被去掉，`A//B` 等价于 `A/B`。
- 中间分组自动创建；注销后分组为空会自动移除。
- 同一路径重复注册抛 `GameFrameworkException("Debugger window has been registered.")`。
- 侧边栏点击分组只会展开/折叠；用 `SelectDebuggerWindow` 直接选中分组路径不会产生内容页（分组 `CreateView()` 返回 `null`）。
- 组件按窗口缓存 `CreateView()` 的结果；改缩放或屏幕分辨率变化会重建整棵视觉树，此时才重新调用 `CreateView()`。

## 自动注册

给窗口类型加 `[DebuggerWindow]`，`DebuggerComponent.Start` 会在内置窗口注册完后扫描并自动注册：

```csharp
using AlicizaX.Debugger;
using UnityEngine;
using UnityEngine.UIElements;

[DebuggerWindow("Gameplay/Player")]
public sealed class PlayerDebugWindow : IDebuggerWindow
{
    private Label _label;

    public void Initialize(params object[] args) { }
    public void Shutdown() { }
    public void OnEnter() { Refresh(); }
    public void OnLeave() { }
    public void OnUpdate(float elapseSeconds, float realElapseSeconds) { Refresh(); }
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

扫描规则（`DebuggerWindowScanner`）：

- 始终扫描本包程序集 `AlicizaX.Debugger`；外部程序集必须把 asmdef 名填进 `Auto Register Assemblies`（按短名或完整名匹配，找不到输出 warning）。
- 只看 `class`：抽象类、开放泛型、未实现 `IDebuggerWindow` 的类型、实现 `IDebuggerWindowGroup` 的类型都跳过；`DebuggerComponent` 的嵌套类型没有标签也跳过。
- 路径取标签的 `Path`（去掉首尾空白）；为空时回退 `Auto/<类名>`。同一路径重复只保留第一个，并输出 `Debugger: duplicate auto-register path ...`。
- 用 `Activator.CreateInstance(type, nonPublic: true)` 实例化，所以窗口需要无参构造（非 public 也可以），`Initialize` 收到空参数。
- 手动注册先执行，撞路径时保留手动注册的一方，自动扫描只输出 warning。
- IL2CPP 下只被反射引用的类型会被裁剪：扩展程序集需要自己加 `link.xml`（`<assembly fullname="Your.Gameplay" preserve="all" />`）或 `[Preserve]`；包内 `Runtime/Debugger/link.xml` 已保留 `AlicizaX.Debugger`。

## 手动注册自定义窗口

需要 `Initialize` 参数、或窗口实例由外部持有时，用手动注册：

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
if (debugger == null)
{
    return;
}

// 第三个参数会原样传给窗口的 Initialize，例如传一个被观察的对象。
debugger.RegisterDebuggerWindow("Gameplay/Player", new PlayerDebugWindow());

// 注销会调用窗口的 Shutdown()。
debugger.UnregisterDebuggerWindow("Gameplay/Player");
```

## 日志采集

`Console` 窗口（`ConsoleWindow`）负责采集日志：

- 注册时订阅 `Application.logMessageReceived`，因此组件注册窗口之前的日志不会进缓存。
- `LogType.Assert` 统一按 `Error` 处理；`LogType.Exception` 计入 Fatal。
- 缓存队列最多 `m_MaxLine` 条（默认 `100`），超出时丢掉最旧一条并归还 `MemoryPool`。
- 顶部显示 `Info / Warning / Error / Fatal` 计数与当前显示条数；过滤开关 `Lock`、`Info`、`Warn`、`Error`、`Fatal` 写入 `PlayerPrefs`，列表刷新被节流到 10 Hz。
- 选中一条日志后，下方显示「正文 + 空行 + 堆栈」；`Copy Selected` 复制同样内容到系统剪贴板；`Clear` 释放全部缓存节点。
- 单行文本由 `LogNode.DisplayText` 生成，格式是 `[HH:mm:ss.fff][帧号] 正文`（本地时间，换行压成空格）并缓存。

```csharp
using System.Collections.Generic;
using AlicizaX.Debugger;

private readonly List<DebuggerComponent.LogNode> _logs = new List<DebuggerComponent.LogNode>(128);

public void DumpRecentLogs()
{
    DebuggerComponent debugger = DebuggerComponent.Instance;
    if (debugger == null)
    {
        return;
    }

    debugger.GetRecentLogs(_logs, 20); // 不传 count 时取全部缓存
    for (int i = 0; i < _logs.Count; i++)
    {
        DebuggerComponent.LogNode log = _logs[i];
        // LogTime / LogFrameCount / LogType / LogMessage / StackTrack / DisplayText
    }
}
```

`count <= 0` 只会输出 `Count is invalid.` 并保持列表不变。返回的节点是池化对象，只在当前帧有效，不要跨帧保存，也不要手动释放。

## Stats 悬浮窗

- 默认关闭，显示在屏幕右上角，`0.25` 秒刷新一次。
- 内容：FPS 与帧耗时、三角面、Batches、DrawCall、SetPass，以及 Mono 堆、总分配 / 保留内存和显存。
- 三角面 / DrawCall 等依赖 `ProfilerRecorder`（`UNITY_2020_2_OR_NEWER`），取不到的项按不可用处理。
- 悬挂在 UIDocument 根节点上，和完整窗口的开关互不影响。

```csharp
using AlicizaX.Debugger;

DebuggerComponent debugger = DebuggerComponent.Instance;
debugger.SetStatsOverlayVisible(true);
bool visible = debugger.IsStatsOverlayVisible;
```

也可以在命令控制台执行 `profile stats true` / `profile stats false`。

## 字体与缩放

- 字体解析顺序：`CustomFontAsset`（有效 `FontAsset`）→ `CustomFont`（`m_Font`）→ 平台系统字体 → Unity 内置 `Arial`。系统字体候选：Windows `Segoe UI`、`Microsoft YaHei UI`、`Microsoft YaHei`、`Arial`、`Tahoma`；macOS / iOS `PingFang SC`、`Hiragino Sans GB`、`Helvetica Neue`、`Helvetica`、`Arial`；Android `Roboto`、`Noto Sans CJK SC`、`Droid Sans Fallback`、`Arial`。
- 字体写到 UIDocument 根元素的 `unityFontDefinition`，所有面板统一生效。
- `WindowScale` 最小 `0.5`；`Other/Settings` 页的滑杆上限 `4`，另有 `1.0x` / `1.5x` / `2.0x` 预设。
- 改缩放会延迟 `0.15` 秒重建整棵视觉树（拖动滑杆时合并成一次）；屏幕分辨率变化也会触发重建。
- 所有尺寸按分辨率缩放 `min(Screen.width / 1920, Screen.height / 1080)` 烘焙；`WindowOpacity` 范围 `0.2`–`1`，浮动入口透明度是 `clamp(opacity + 0.1, 0.35, 1)`。

```csharp
using AlicizaX.Debugger;
using UnityEngine;

DebuggerComponent debugger = DebuggerComponent.Instance;
debugger.IconRect = new Rect(24f, 24f, 180f, 56f); // 宽高固定，只有位置生效
debugger.WindowRect = new Rect(24f, 96f, 1320f, 760f); // 最小 420 × 320
debugger.WindowScale = 1.5f;
debugger.WindowOpacity = 0.9f;
debugger.EnableFloatingToggleSnap = true;
// debugger.ResetLayout(); // 恢复默认布局并写回 PlayerPrefs
```

## 常见配置

| 需求 | 做法 |
| --- | --- |
| 生产包关闭面板 | `Active Window` 设为 `OnlyOpenWhenDevelopment` / `OnlyOpenInEditor` / `AlwaysClose` |
| 换字体 | Inspector 的 `Font`，或运行期 `CustomFont` / `CustomFontAsset` |
| 窗口太小或位置错乱 | `Other/Settings` 页改宽高、标题栏 `R`，或调 `ResetLayout()` |
| 减少面板存在感 | `WindowOpacity` 调低，或把 `ActiveWindow` 置 `false` |
| 接入自己的调试页 | 窗口类加 `[DebuggerWindow("Group/Name")]`，并把 asmdef 名填进 `Auto Register Assemblies` |
| 需要自定义日志消费 | 用 `GetRecentLogs` 读取，不要自己再订阅 `logMessageReceived` |

卸载与销毁：

- `DebuggerComponent` 不调用 `DontDestroyOnLoad`，跨场景要自己搬运或用常驻场景。
- 销毁组件会自动注销全部窗口并 `Shutdown`；自定义窗口持有原生资源时在 `Shutdown()` 里释放。销毁后 `DebuggerComponent.Instance` 为 `null`，同一服务容器内重新挂载可以再次注册。

## 容易踩的坑

1. 浮动入口要**双击**（`0.3` 秒内、位移不超过 `24`），单击不会打开窗口。
2. 缩放变更延迟 `0.15` 秒重建，拖动滑杆期间不要依赖当帧的布局数值。
3. `CreateView()` 结果按窗口缓存，数据变化要写在 `OnUpdate` / `OnEnter` 里改控件，不会自动重建。
4. 自动注册的窗口必须有无参构造，`Initialize` 会收到空参数；外部程序集必须加进 `Auto Register Assemblies`。
5. 同一个路径手动注册两次会抛 `GameFrameworkException`，而自动扫描撞路径只输出 warning。
6. 日志默认只留 `100` 条，且 `GetRecentLogs` 返回池化节点——不要跨帧保存，也不要手动归还。

## 相关页面

- [Debugger 调试器](index.md)：组成、激活模式与服务关系
- [命令控制台](command.md)：命令扫描、快捷键与内置命令
- [API 速查](api.md)：公开类型与成员
