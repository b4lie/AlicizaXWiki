# Debugger 调试器

`com.alicizax.unity.debugger` 是框架的可选运行时调试扩展包：一个基于 UI Toolkit 的调试面板，加上一套反射式命令控制台。它不参与业务逻辑，只在开发期提供观察和操作入口；生产包可以通过激活模式整体关闭。

当前文档对应包版本 `2.1.5`。

源码位置：

- `Client/Packages/com.alicizax.unity.debugger/Runtime/Debugger`：`DebuggerComponent`、`IDebuggerService`、窗口树和内置窗口
- `Client/Packages/com.alicizax.unity.debugger/Runtime/Console`：`AlicizaX.Console` 命令系统
- `Client/Packages/com.alicizax.unity.debugger/Editor`：Inspector 界面
- `Client/Packages/com.alicizax.unity.debugger/Debugger.prefab`：只挂载 `DebuggerComponent` 的推荐接入 prefab

## 组成

| 类型 | 职责 |
| --- | --- |
| `DebuggerComponent` | 场景中的调试器组件，持有 UI、浮动入口、窗口树和布局状态。`DebuggerComponent.Instance` 是全局入口。 |
| `IDebuggerService` | 调试器服务，管理激活状态和窗口树。`DebuggerComponent` 在 `Awake` 中注册或复用 `AppServices` 里的实例。 |
| `IDebuggerWindow` | 单个调试窗口。每个侧边栏选项卡对应一个实现，`CreateView()` 返回 UI Toolkit 的 `VisualElement`。 |
| `AlicizaX.Console` | 命令扫描、解析、补全、历史和执行；UI 作为 `Command` 窗口挂进 Debugger。 |

窗口路径用 `/` 分组。例如 `Profiler/Memory/Texture` 在侧边栏显示为 `Profiler` → `Memory` → `Texture`，同一路径也可以直接传给 `SelectDebuggerWindow`。

内置窗口都以 `DebuggerComponent` 的嵌套类实现，按文件拆分在 `Runtime/Debugger/Windows` 下。

## 功能一览

| 功能 | 入口 | 说明 |
| --- | --- | --- |
| 浮动入口 | 屏幕上的 `FPS:xx` 按钮 | 双击打开完整窗口，拖动移动位置；文字颜色跟随当前最高日志级别变化。 |
| 日志 | `Console` | 捕获 `Debug.Log` / Warning / Error，支持过滤、锁定滚动、堆栈查看、复制和清空。 |
| 命令 | `Command` | `[Command]` 命令控制台，支持补全、历史、异步 action、宏和外部脚本。 |
| 运行时信息 | `Information/*` | 系统、环境、屏幕、图形、输入、场景、时间、质量。 |
| Profiler | `Profiler/*` | 内存采样、引用池、音频和计时器。 |
| 性能悬浮窗 | `profile stats true` | 右上角半透明面板，显示 FPS、三角面、DrawCall 和内存。 |
| 布局 | 拖拽 / `Other/Settings` | 窗口位置、尺寸、缩放和吸附状态写入 `PlayerPrefs`。 |

## 快速接入

1. 安装 `com.alicizax.unity.debugger`。
2. 把 `Packages/com.alicizax.unity.debugger/Debugger.prefab` 拖入启动场景。prefab 只挂载 `DebuggerComponent`。
3. 确认启动场景里有框架根节点（`RootModule` 等模块组件），让 `DebuggerComponent` 能注册到 `AppServices`。
4. 运行后默认显示浮动 `FPS` 按钮；双击按钮打开面板，在侧边栏选择 `Console`、`Command` 或其他页。

也可以手动创建 `GameObject` 并添加 `AlicizaX.Debugger.DebuggerComponent`。

`DebuggerComponent` 会在运行时创建或复用 `EventSystem`，并创建自己的 `UIDocument` / `PanelSettings`：

- Inspector 指定了 `Panel Settings` 时先克隆该配置。
- 否则尝试 `Resources.Load<PanelSettings>("DebuggerPanelSettings")`，即工程任意 `Resources` 目录下的同名资源。
- 仍找不到时创建默认 `PanelSettings`，字体按平台从系统字体回退。

完整窗口不会随场景启动自动出现：`Start()` 会强制 `ShowFullWindow = false`，只保留浮动入口。

## 内置窗口

### 日志与命令

| 路径 | 内容 |
| --- | --- |
| `Console` | Unity 日志查看器 |
| `Command` | 命令控制台 |

### 运行时信息

| 路径 | 内容 |
| --- | --- |
| `Information/System` | 设备、操作系统、处理器、内存、电池和传感器支持 |
| `Information/Environment` | 应用信息、Unity 版本、平台、语言、构建 ID、网络与后台加载 |
| `Information/Screen` | 分辨率、DPI、方向、全屏模式、安全区、刘海和可用分辨率 |
| `Information/Graphics` | 显卡、图形 API、Shader Level、渲染管线和色彩空间 |
| `Information/Input` | 鼠标、键盘、触摸、IME 与设备传感器 |
| `Information/Other/Scene` | 场景数量、活动场景及其句柄和构建索引 |
| `Information/Other/Time` | `timeScale`、各类时间、帧计数和捕获设置 |
| `Information/Other/Quality` | 质量等级、渲染设置和阴影设置 |

### Profiler

| 路径 | 内容 |
| --- | --- |
| `Profiler/Summary` | Profiler 开关、Mono 堆、总分配 / 保留内存等统计 |
| `Profiler/Memory/Summary` | 按类型统计的运行时内存采样 |
| `Profiler/Memory/All` | 全对象内存采样 |
| `Profiler/Memory/Texture` | 纹理内存采样 |
| `Profiler/Memory/Mesh` | Mesh 内存采样 |
| `Profiler/Memory/Material` | Material 内存采样 |
| `Profiler/Memory/Shader` | Shader 内存采样 |
| `Profiler/Memory/AnimationClip` | AnimationClip 内存采样 |
| `Profiler/Memory/AudioClip` | AudioClip 内存采样 |
| `Profiler/Memory/Font` | Font 内存采样 |
| `Profiler/Memory/TextAsset` | TextAsset 内存采样 |
| `Profiler/Memory/ScriptableObject` | ScriptableObject 内存采样 |
| `Profiler/Reference Pool` | 框架 `MemoryPool` 各池的空闲 / 借出 / 创建统计 |
| `Profiler/Audio` | 音频分类、活跃 agent 和 clip 缓存 |
| `Profiler/Timer` | 框架计时器池与计时器采样 |

`Profiler/Memory/*` 页面需要先点击 `Take Sample`，采样后才会显示对象数量和内存占用。

### 设置

| 路径 | 内容 |
| --- | --- |
| `Other/Settings` | 悬浮入口吸附开关、窗口宽高、窗口缩放和缩放预设 |

### 框架模块页面

`Profiler/Reference Pool` 直接读取 `MemoryPool` 的静态数据；`Profiler/Audio`、`Profiler/Timer` 依赖框架对应服务，服务不存在时页面显示空数据或不可用提示。

## 相关页面

| 页面 | 内容 |
| --- | --- |
| [使用方式](usage.md) | 接入、激活模式、面板操作、日志、悬浮窗、自定义窗口和布局 |
| [命令控制台](command.md) | 快捷键、命令扫描、参数、宏、外部脚本和内置命令 |
| [API 速查](api.md) | Debugger 与 Console 的常用 API 和注意事项 |
