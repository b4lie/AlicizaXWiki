# Input 输入

`com.alicizax.unity.input` 是框架的可选输入扩展包，基于 Unity Input System，提供设备监听、动作读取、运行时重绑定、输入图标、快捷键路由和 UI 导航焦点管理。

当前文档对应包版本 `3.0.1`。

源码位置：

| 内容 | 路径 |
| --- | --- |
| 输入门面与五个模块 | `Client/Packages/com.alicizax.unity.input/Runtime/Input` |
| 图标数据库、Provider、图标 UI 组件 | `Client/Packages/com.alicizax.unity.input/Runtime/InputGlyph` |
| 快捷键 | `Client/Packages/com.alicizax.unity.input/Runtime/Hotkey` |
| UI 导航 | `Client/Packages/com.alicizax.unity.input/Runtime/Navigation` |
| 编辑器工具 | `Client/Packages/com.alicizax.unity.input/Editor` |

## 安装前提

1. 安装 `com.unity.inputsystem`，并在 `Project Settings > Player > Active Input Handling` 中选择 Input System 后端（或 `Both`）。
2. 安装 `com.alicizax.unity.input`。包的 asmdef 会检测 `com.unity.inputsystem` 并自动生成 `INPUTSYSTEM_SUPPORT`，不需要手加宏。
3. 场景中准备一个常驻节点，挂载 `InputActionProvider`，指定 `InputActionAsset` 和可选的 `InputGlyphDatabase`。
4. 使用 UI 导航时还需要 `com.alicizax.unity.ui.extension` 和场景中的 `EventSystem`。

`InputActionProvider` 带 `DefaultExecutionOrder(-10000)`，会早于普通业务脚本初始化输入。

## 组成

| 类型 | 职责 |
| --- | --- |
| `InputActionProvider` | 场景入口组件，持有动作资产和图标数据库，调用 `UXInput.Initialize` / `UXInput.Shutdown` |
| `UXInput` | 静态门面，暴露 `Watch`、`Reader`、`Rebind`、`Glyph`、`Haptics` 五个实例 |
| `InputWatch` | 当前设备、输入类型、Profile 和上下文的监听与缓存 |
| `InputReader` | 动作查询、值读取、按住状态、组合按键与 `ButtonState` |
| `InputRebind` | 运行时重绑定、冲突处理、绑定覆盖持久化 |
| `InputGlyph` | 控制路径到图标键、显示名、TMP 标签和 UI Sprite 的查询 |
| `InputHaptics` | 手柄马达反馈 |
| `UXHotkeySystem` / `HotkeyComponentBase` | 按 UI 层级路由热键 |
| `UXNavigationScope` | UI 导航域、焦点与挡板 |
| `InputProfileCatalog` | Profile 稳定 ID、显示名、回退链和绑定组提示 |

## 启动与生命周期

正常接入时不要手动调用 `UXInput.Initialize`，由 `InputActionProvider.Awake` 负责：

```csharp
UXInput.Initialize(actions, glyphDatabase);
```

初始化按顺序完成：

1. 创建 `InputWatch`、`InputReader`、`InputRebind`、`InputGlyph`、`InputHaptics` 五个实例。
2. `Watch.Initialize()`：创建内部任意输入 Action，订阅设备变化，确定初始上下文。
3. `Rebind.Initialize()`：缓存启动前的绑定覆盖，并加载本地 `input_bindings.json`。
4. `Reader.Initialize()`：建立 `Map/Action` 查询表并启用整份 `InputActionAsset`。
5. `UXNavigationSystem.Initialize()`：启动 UI 导航。

`Glyph` 在构造时接收 `InputGlyphDatabase`；`Haptics` 没有单独的初始化步骤。

关闭顺序与初始化相反：

```text
UXNavigationSystem -> UXHotkeySystem -> Haptics -> Reader -> Rebind -> Glyph -> Watch
```

关闭时会停止导航和热键、禁用动作资产、释放交互式重绑定、恢复启动前的绑定覆盖，并清空 Glyph/Watch 的缓存与事件。导航注册表、Scope 索引、激活序号、选择策略和程序化焦点计数一起重置；热键解除动作订阅和 App Hook；模块清空事件、设备、Action、数据库等引用。

### 所有权规则

- 只允许一个 `InputActionProvider`。重复的 Provider 在 `Awake` 输出 warning 后直接返回，不会接管已经启动的输入。
- 禁用 Provider 组件或它所在的 GameObject **不会**关闭输入，也不会断开仍启用的输入组件；必须销毁 Provider 才会 `Shutdown`。
- 直接重复调用 `UXInput.Initialize` 会抛 `InvalidOperationException`；重复调用 `Shutdown` 由门面忽略。
- 绑定加载失败会关闭已经创建的模块，并把异常抛给调用方。
- `SubsystemRegistration` 只调用统一清理入口，初始化仍由新一轮的 Provider `Awake` 完成。
- 支持关闭 Domain Reload；**Scene Reload 必须保持开启**。
- 关闭后门面保留已清理的模块引用，供组件完成事件退订；下一次初始化会整体替换。不要跨运行轮次缓存模块引用。
- 输入模块只禁用和解除引用项目资产，不销毁 `InputActionAsset` 或 `InputGlyphDatabase`。
- 已保存的绑定文件会保留并在下次启动重新加载；未保存的临时改绑不会继承。

## 主要 API

| 模块 | 用途 |
| --- | --- |
| `UXInput.Watch` | 当前设备 / Profile / 上下文，`BuildContext`、`CurrentProfileId` |
| `UXInput.Reader` | 轮询读取动作值、按住状态、组合部分；`InputReader.ButtonState` 由业务持有 |
| `UXInput.Rebind` | 交互式重绑定、冲突处理、绑定覆盖持久化 |
| `UXInput.Glyph` | 控制路径到图标键、显示名、TMP 标签和 UI Sprite |
| `UXInput.Haptics` | 手柄马达反馈 |
| `UXHotkeySystem` / `HotkeyComponent` | 按 UI 层级路由热键 |
| `UXNavigationScope` | UI 导航域焦点、抑制和选中记忆 |
| `InputProfileCatalog` | Profile 稳定 ID、显示名、回退链和绑定组提示的唯一来源 |

`InputReader`、`InputRebind`、`Hotkey` 和 `InputGlyph` 使用同一份动作资产，不复制资产。重绑定应用后更新绑定版本，`Reader` 和 `Glyph` 的缓存据此失效。

## 页面导航

| 页面 | 内容 |
| --- | --- |
| [设备、读取与重绑定](../../extension/uxinput.md) | Watch、Reader、Rebind、Haptics 和 InputVisualizer 的详细用法 |
| [输入图标](../../extension/input-glyph.md) | 图标数据库、InputGlyphImage / InputGlyphText 和查询 API |
| [快捷键](../../extension/hotkey-component.md) | HotkeyComponent、HotkeyPassThrough、焦点与优先级 |
| [UI 导航](../../extension/navigation.md) | UXNavigationScope、Focus / Block、动态控件注册 |
| [性能与分配](performance.md) | 0 B 热路径、实测数据、容量增长和低频操作 |
| [2.x → 3.x 迁移](migration.md) | 模块实例化、Reader / Rebind 重命名和旧入口替换 |

## 注意事项

1. 输入包依赖 Unity Input System；没有安装或没有启用新后端时，输入相关代码会被 `INPUTSYSTEM_SUPPORT` 条件编译排除。
2. `InputActionProvider` 必须早于依赖输入的系统创建；只在常驻节点保留一个。
3. `Reader` 不保存以对象或字符串为键的全局状态。`ButtonState` 必须由业务声明为字段并自行持有。
4. `Glyph` 的数据库由 `InputActionProvider` 注入；不要在业务代码里给每次查询单独传数据库。
5. UI 导航依赖 UI 框架的 Holder 和 `UIComponent.UIShowLayer`，不能脱离 `com.alicizax.unity.ui.extension` 单独使用。
