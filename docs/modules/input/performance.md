# 性能与分配

输入包的热路径主要是 `Reader` 读取、`Watch` 设备上下文、`Glyph` 查询、`Navigation` 刷新和 `Hotkey` 分发。先按使用方式控制分配，再用 Profiler 验证。

## 读取热路径

- `Reader.ReadValue`、`TryReadValue`、`IsPressed`、`ButtonState.Once/Toggle` 稳态不分配；首次查询动作、组合部分时会建立缓存。
- 组合部分只缓存一次，绑定变化后失效；不要在每帧重复 `ResolveCompositePart` 并丢弃结果，可以缓存 `InputCompositePart`。
- `ButtonState` 必须是字段。作为局部变量每次都会重新开始边沿检测。
- 不要在高频代码里拼接动作路径字符串；缓存 `InputAction` 或使用一次解析后的引用。
- `OnRead` / `OnContinuousRead` 只用于调试，发布包不要订阅。

## 设备与绑定

- `Watch` 按设备缓存 `InputContext`；同设备连续输入不分配，设备变化只失效对应缓存。
- `CurrentInputType`、`CurrentInputProfile` 是属性读取，适合在 UI 刷新时比较，不要每帧触发不必要的重建。
- `Glyph` 的图标键、显示名和绑定索引有缓存；绑定变化时统一失效。批量重绑定完成后刷新一次 UI，不要每改一个动作刷新整页。
- `Rebind` 是低频操作，但 Unity 的 `RebindingOperation`、JSON 和文件写入会产生分配；不要在打开设置页时反复开始/取消。

## 导航

- `NavigationScope` 的结构性变化（注册、注销、Holder 显示/关闭、控件增删）会触发刷新；稳态每帧只做快路径检查。
- 运行时列表项不要每帧 `RegisterSelectable` / `UnregisterSelectable`；在 `OnEnable` / `OnDisable` 成对处理。
- 修改 `interactable`、激活状态或 Canvas 排序后调用 `NotifySelectableStateChanged()`，不要依赖每帧自动发现。
- 一个窗口一个 Scope，不要为每个按钮挂 Scope。

## 热键

- 热键在 `OnEnable` 注册、`OnDisable` 注销；频繁开关窗口会产生 Scope 注册成本，但不会每帧发生。
- 同一 Holder 的同一 action/pressType 多个注册是允许的，但会增加分发候选；能用单个组件解决时不要重复挂。
- 热键动作必须来自 `InputService` 的同一份资产；不要为每个组件创建独立资产。

## 输入图标与反馈

- `InputGlyphImage` / `InputGlyphText` 已监听 Profile 和绑定变化；不要额外每帧刷新。
- `InputFeedback` 按“Cue + 设备”匹配规则，开销低；主题规则不要无限增长，按 Cue/设备合并。
- 没有 `IAudioService` 时反馈直接跳过，不会反复报错。

## 用工具确认

- Profiler 中观察 `GC.Alloc`，分别测打开窗口、滚动列表、切换设备、重绑定和导航刷新。
- `InputVisualizer` 只在 Editor/Development Build 编译，正式包不包含。
- 先测真实业务路径，不要用单次冷启动数据推断稳态开销。

## 注意事项

1. 输入层重建后模块实例是新的，旧缓存引用需要重新获取。
2. 首次查询、首次打开窗口、首次重绑定有初始化成本，不能按稳态 0 B 估算。
3. 批量数据变化后只刷新一次 UI。
4. 诊断组件和日志格式化会产生额外字符串，发布包关闭。
