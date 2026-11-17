# 性能与分配

本页按模块说明输入热路径的分配来源、零分配写法、绑定缓存的失效条件，以及包内如何在编辑器中量化分配。结论只写源码与用例能证明的部分：凡是没有零分配断言的路径，这里不会写成「0 GC」。

## 适用场景

- 在 `Update` 里高频读取输入，或大量 UI 使用输入图标与导航。
- 需要在开发期确认某段输入代码是否产生 GC。
- 排查设备插拔、重绑定之后输入缓存重建造成的抖动。
- 决定哪些查询可以每帧做、哪些必须缓存或事件驱动。

## 使用前提

- 分配数据来自 `AlicizaX.Input.Tests` 里的 PlayMode 用例，只在编辑器下通过 Profiler 采样；用例文件用 `#if INPUTSYSTEM_SUPPORT && UNITY_EDITOR` 包裹，测试 asmdef 还带 `defineConstraints: ["INPUTSYSTEM_SUPPORT"]`。
- 输入层必须先初始化；`InputDeviceTestBase` 会创建测试资产（`Gameplay/Jump`、`Gameplay/Move` 等）并启用动作。
- 文中「零分配」指该用例覆盖的调用路径与迭代次数内实测 0 字节、0 次分配，不覆盖首次调用、缓存扩容、重绑定操作、文件 IO 和音频播放。

## 最短接入

1. 动作与组合部件句柄只解析一次，存成字段；稳态读取只用 `InputAction` / `InputCompositePart` 重载。
2. 边沿与开关用字段形式的 `InputReader.ButtonState`。
3. 用下节的用例名在 Test Runner（PlayMode）里跑一遍，确认自己的调用组合没有引入分配。
4. 需要新用例时复用 `InputAllocationCapture.Measure`，用 `Assert.That(bytes, Is.Zero)` 收口。

在 Unity 中运行：Window → General → Test Runner → PlayMode → 过滤 `InputAllocation` 或 `InputOptimization` → Run All。失败信息会带上用例名与 `INPUT_ALLOC` 实测行，便于对比改动前后。

```csharp
using AlicizaX.Input;
using UnityEngine;
using UnityEngine.InputSystem;

public sealed class ZeroAllocInput : MonoBehaviour
{
    private InputAction _move;
    private InputAction _jump;
    private InputCompositePart _moveUp;
    private InputReader.ButtonState _jumpState;

    private void Awake()
    {
        _move = UXInput.Reader.ResolveAction("Gameplay/Move");
        _jump = UXInput.Reader.ResolveAction("Gameplay/Jump");
        _moveUp = UXInput.Reader.ResolveCompositePart(_move, "Up");
    }

    private void Update()
    {
        Vector2 move = UXInput.Reader.ReadValue<Vector2>(_move);  // 不查字典、不建字符串
        bool up = UXInput.Reader.IsPressed(_moveUp);              // 句柄缓存
        if (_jumpState.Once(_jump))                               // 无闭包、无装箱
            Jump(move, up);
    }

    private void Jump(Vector2 move, bool up) { }
}
```

## 核心用法

### `Reader`：稳态读取零分配

`ReaderSteadyStateAllocatesZeroBytes` 在 2000 次循环里调用 `IsPressed`（两次）、`ReadValue<Vector2>`、`ButtonState.Once`、`ButtonState.Toggle`、`IsCompositePartPressed`，断言 `bytes == 0` 且 `allocations == 0`。首次建立缓存另有上限，由 `FirstUseAndSteadyStateAreMeasuredSeparately` 断言：

| 测量项 | 断言 |
| --- | --- |
| 首次组合部件查询 `reader-first-composite` | ≤ 16384 B（固定的小缓存成本） |
| 稳态组合部件读取 `reader-steady-composite` | 0 B |
| 首次图标键 `glyph-first-new-key` | ≤ 4096 B（只缓存字符串） |
| 首次显示名 `glyph-first-display-name` | ≤ 262144 B（Input System 显示名表，约 130 KB，一次性） |
| 首次重绑定显示文本 `rebind-first-display` | ≤ 16384 B |
| 以上各项的稳态查询 | 0 B |

分配来源集中在首次解析：`Reader` 的完整路径字典、组合部件句柄缓存、`InputCompositePart` 的控件数组（按需扩容）。稳态读取本身不再分配。字符串重载每次要做一次字典查找，功能上没问题，但高频路径应使用缓存字段。

### `Watch`：按设备缓存上下文

- `WatchContextResolutionAllocatesZeroBytes`：对同一设备 2000 次解析上下文并读 `CurrentProfileId`，断言 0 B。
- `WatchAlternatingDevicesAllocateZeroBytes`：键盘 / 鼠标 / 手柄交替 2000 轮，断言 0 B。
- `WatchSameDeviceCallbacksAllocateZeroBytes`：2000 次按下+松开，`OnInputActivity` 收到 2000 次，断言 0 B。
- `WatchInputCallbacksAllocateZeroBytes`：键鼠与手柄交替共 4000 次活动，断言 0 B。
- `WatchMeasuresNewDeviceTypeAndCapacityGrowth`：新增设备类型与 20 个设备扩容单独测量，20 设备稳态 100 轮断言 0 B。

上下文由 `InputContext` 结构体承载，事件只传结构体与稳定字符串；`CurrentProfileId` 返回 `InputProfileCatalog` 里的共享字符串（用例断言两次读取是同一引用）。`BuildContext` 是公开的、不做缓存的构造方法，源码注释说明它只组装现有字符串与值类型，不产生托管分配；高频场景仍应读 `Current` / `CurrentInputType` / `CurrentInputProfile`，而不是每帧重建上下文。

### `Glyph`：显示文本、图标键、绑定索引三层缓存

- `GlyphQueriesAllocateZeroBytes`：预热后 2000 轮 `CurrentProfileId`、`GetDisplayName("<Keyboard>/space")`、`GetGlyphKeyFromControlPath`、`GetDisplayName(action)`、`GetBindingControlPath(action)`，断言 0 B。
- `BindingCacheMeasuresCapacityGrowth`：首个绑定键与新键单独测量，随后扩容到 80 个动作；80 个动作 25 轮稳态断言 0 B。
- `SpriteAndTmpQueriesMeasureFirstUseAndSteadyState`：`TryGetSprite` / `TryGetTag` 首次测量，2000 轮混跑断言 0 B；命中时标签是缓存的 `<sprite name="...">` 字符串。

`InputGlyphImage` / `InputGlyphText` 通过基类订阅 `Watch.OnInputProfileChanged` 与 `Rebind.OnBindingsChanged` 刷新，不需要也不应该每帧调用 `RefreshGlyph`。

### `Rebind`：低频、有分配

重绑定不是热路径。`RebindAndSessionCostsAreMeasured` 只记录首次与 20 次 `Begin`+`Cancel`、`PrepareRebind`+`ConfirmApply`（含存盘）、`Shutdown`、`Initialize`（含读档）的数值，不断言为 0；断言的是效果：覆盖已应用、动作在初始化后已启用。

可以安全高频使用的只有查询类接口：`RebindRequestAndDisplayStringAllocateZeroBytes` 在预热后 2000 轮调用 `RebindRequest.ForTarget(...)` 与 `GetDisplayString(action)`，断言 0 B。`RebindRequest` 是结构体，`With*` 派生新值；`ExcludedControls` 等默认数组是共享静态数据，不要修改。

会产生分配或 IO 的操作：`PerformInteractiveRebinding` 的操作对象与委托、冲突收集的 `List<RebindChange>`、`OnApply` 快照数组、`ExportBindingsJson` 的 JSON 字符串、`SaveBindings` 的文件写入、`LoadBindings` 的文件读取。设置页面里不要每帧调 `Begin`/`Cancel`，批量改键后统一确认一次。

### `Navigation`：稳态刷新零分配

- `NavigationRefreshAllocatesZeroBytes`：2000 次 `Refresh()` 断言 0 B；200 次「清空选中 + Refresh」的焦点恢复同样 0 B。
- `NavigationMultipleScopesAndParentChainAllocateZeroBytes`：16 个 Scope × 32 个候选控件、带三层嵌套 Holder，2000 次刷新断言 0 B、0 次分配；再叠一个 `overrideSorting` 的遮挡层，2000 次刷新仍是 0。

结构变化（Scope 注册/注销、Holder 显示关闭、控件增删）会同步全量刷新；稳态每帧只做快路径检查：选中对象是否变化、Scope 是否存活、当前选中是否仍可用、补选策略是否因输入类型切换而改变。运行时改 `Canvas.sortingOrder`、或改动非当前选中域的 `interactable` 不会被快路径发现，需要调用 `NavigationScope.NotifySelectableStateChanged`。

### `Hotkey`：分发零分配，注册有成本

`HotkeyRegistrationAndDispatchAreMeasured`：首个注册与首次提交单独测量；2000 次分发与 2000 次「同 Holder 双 trigger」分发都断言 0 B，并校验最后一次注册的 trigger 消费按键；20 次注销+注册只测量不断言。

`InputHotkey` 在构造时缓存了三个 `Action<InputAction.CallbackContext>` 回调委托，分发路径走 `ReadOnlySpan` 的 Holder 链，不再产生委托或闭包。注册会创建/复用 `HotkeyScope`、字典条目与订阅，属于结构性操作，应在 `OnEnable`/`OnDisable` 成对处理，不要每帧调用。

### `Feedback`：源码层无分配，但没有零分配用例

`InputFeedback.Play` 先按当前 `InputType` 映射到 `InputFeedbackDevice`，再用 `InputFeedbackProfile.TryResolve` 线性扫描规则数组并返回结构体；这一层没有字符串拼接或集合分配。真正的播放交给 `IAudioService`，开销取决于音频模块的缓存策略；没有音频服务时直接跳过。包内没有针对 Feedback 的分配用例，因此这里只给源码结论，不宣称 0 GC。

### 绑定缓存与版本号

内部 `InputBindingVersion` 维护两个粒度，选中哪个粒度决定缓存是否被设备插拔清空：

| 版本 | 何时推进 | 谁依赖它 |
| --- | --- | --- |
| `Value` | 只有 `Rebind` 的显式变更（应用、重置、加载、导入）会 `Bump` | `InputBindingDisplayCache`（绑定显示文本）、`InputGlyph` 的绑定索引缓存 |
| `ControlsValue` | `Value` 推进时同步推进；另外跟随 Input System 的 `BoundControlsChanged`（设备增删、直接 `ApplyBindingOverride`） | `InputCompositePart` 的控件列表重建 |

因此插拔设备不会清空显示文本缓存、不会重新分配字符串；只有组合部件句柄需要重新收集控件。批量改键后刷新一次 UI（监听 `Rebind.OnBindingsChanged`），不要在每个动作改完后各刷一遍。

### 用例如何断言

- 稳态零分配成对断言：`Assert.That(bytes, Is.Zero)` 与 `Assert.That(allocations, Is.Zero)`，例如 `ReaderSteadyStateAllocatesZeroBytes`、`WatchInputCallbacksAllocateZeroBytes`、`NavigationRefreshAllocatesZeroBytes`。
- 首次成本用上界：组合部件 `Is.LessThanOrEqualTo(16384L)`、图标键 `4096L`、显示名 `262144L`，都在 `FirstUseAndSteadyStateAreMeasuredSeparately` 里。
- 分配断言必须配行为断言，否则「什么都没做所以 0 B」也会通过：`WatchSameDeviceCallbacksAllocateZeroBytes` 断言 `activityCount == 2000`，`HotkeyRegistrationAndDispatchAreMeasuredSeparately` 断言提交计数从 1 涨到 2001，`SpriteAndTmpQueriesMeasureFirstUseAndSteadyState` 断言标签等于 `<sprite name="SpaceButton">`，`VisualizerRecordsContinuousReadsWithoutAllocations` 断言同帧 2000 次记录只分配 1 次。
- 每轮测量都会用 `TestContext.WriteLine` 和 `Debug.Log` 输出 `INPUT_ALLOC,name,bytes,allocations,ms`；首次用途例另输出 `FIRST_USE,<name>,<bytes>`，方便记录基线与回归对比。

### 诊断工具 `InputVisualizer`

只在 `#if INPUTSYSTEM_SUPPORT && (UNITY_EDITOR || DEVELOPMENT_BUILD)` 下编译，正式包不包含。通过 Add Component → Input → Input Visualizer 添加；`OnEnable` 要求 `UXInput` 已初始化，否则断言失败并直接返回。

| Inspector 字段 | 作用 |
| --- | --- |
| `logInputActions` | 订阅 `InputSystem.onActionChange`，只记录 `ActionPerformed`，忽略 Watch 内部动作与 pointer 的 `position`/`delta` |
| `logDeviceActivity` | 订阅 `Watch.OnInputActivity`；`logRepeatedDeviceActivity` 与 `repeatedDeviceActivityInterval` 控制同设备重复记录 |
| `logContextChanges` | 订阅 `Watch.OnContextChanged` |
| `logReaderReads` / `logReaderContinuousReads` | 订阅 `Reader.OnRead` / `OnContinuousRead`；后者可能每帧多次 |
| `showOnScreen`、`maxHistoryEntries`、`entryLifetime`、`entryHeight` | IMGUI 面板与历史条目 |
| `showTimestamps`、`mergeDuplicateEntries`、`duplicateMergeWindow` | 行文本时间戳与重复合并 |

公开接口：`ShowOnScreen`、`EntryCount`、`LogInput(actionName, value, isActive)`、`LogMessage(message)`、`ClearHistory()`。每条历史只在首次取文本时拼接一次并缓存；`showTimestamps` 下同一帧的重复记录会合并成一条（`VisualizerRecordsContinuousReadsWithoutAllocations` 断言 2000 次同帧记录只产生 1 次分配）；订阅了连续读取后 2000 次读取仍断言 0 B。

## 常见配置

| 项 | 建议 | 依据 |
| --- | --- | --- |
| 读取重载 | 用 `InputAction` / `InputCompositePart` 重载 | 零分配用例覆盖的路径 |
| `ButtonState` | 声明为字段 | 属性/局部变量会丢边沿状态 |
| 组合部件句柄 | 缓存 `ResolveCompositePart` 结果 | 每次调用都做一次字典查找 |
| 动作路径 | 缓存 `InputAction`，不要每帧拼路径字符串 | 字符串拼接本身就是分配 |
| 图标刷新 | 订阅 `OnInputProfileChanged` / `OnBindingsChanged` | 组件基类已订阅，无需每帧查 |
| 绑定版本 | 只依赖组合句柄自动重建 | `ControlsValue` 已覆盖设备增删 |
| 分配验证 | 编辑器 PlayMode 跑 `InputAllocationPlayModeTests` | 依赖 Profiler，非运行时可用 |
| 诊断组件 | 只在编辑器/开发版启用 | 正式包不编译 |

## 容易踩的坑

1. **把首次成本当稳态。** 首次显示名要初始化 Input System 的显示名表（用例上限 256 KB），首次组合部件查询最多 16 KB；用首次数据推断每帧开销会严重高估。
2. **每帧调用 `ResolveCompositePart`。** 它要做一次字典查找，还可能触发首次缓存分配；句柄本就可以长期持有。
3. **每帧拼动作路径。** `"Gameplay/" + name` 每次都新建字符串并重新哈希，既分配又慢；存 `InputAction` 或至少缓存路径字面量。
4. **设备插拔后手动清 Glyph 缓存。** 显示文本缓存按 `Value` 失效，插拔只推进 `ControlsValue`；手动清反而造成重新分配。
5. **发布包订阅 `OnRead`。** 这两个事件在非编辑器/非开发版根本不存在，是编译错误；开发版长期打开 `logReaderContinuousReads` 也会持续产生回调与历史条目。
6. **用 `GC.GetAllocatedBytesForCurrentThread` 自测。** 当前 Mono 运行时不支持（恒为 0），包内改用 Profiler 原始帧数据里的 `GC.Alloc` 样本。

## API 速查

| 入口 | 说明 |
| --- | --- |
| `InputAllocationCapture.Measure(name, action, verify)` | 采样一次工作负载：开启 Profiler 的 CPU/Memory 区域，用 `InputAudit.Measure.N` 标记包裹 `action`，统计其子样本里的 `GC.Alloc` 字节数与次数，回调 `verify(bytes, allocations)` |
| 日志格式 | 控制台/测试输出 `INPUT_ALLOC,name,bytes,allocations,ms`；首次用途例额外输出 `FIRST_USE,<name>,<bytes>` |
| `AllocationCaptureDetectsKnownAllocation` | 自检：分配 4096 B 数组必须被采样到，防止测量器失效 |
| `InputAllocationPlayModeTests` | Reader、Watch、Glyph、Rebind、Navigation 的零分配与首次成本用例 |
| `InputOptimizationAllocationTests` | 图标 Sprite/TMP、绑定缓存扩容、视觉化器、热键、重绑定与会话成本的用例 |
