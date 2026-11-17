# 缓存与性能

UI 的开销主要看四件事：每帧有多少视图在更新、有多少窗口留在缓存、每帧产生多少托管分配、以及同时可见的 Canvas 数量。先按 API 设定边界，再用 `UIComponent` 的 Inspector 和 Profiler 验证。

源码位置：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/UI/Manager/UIService.cs`、`UIService.Cache.cs`、`UIService.Debug.cs`、`UIService.Layer.cs`。

## 适用场景

- 弹窗频繁开关，希望复用实例、避免重复加载 Prefab。
- 大界面或重资源窗口需要控制常驻内存。
- 战斗、列表等每帧刷新的界面出现 GC 或更新开销。
- 排查层级排序、Canvas 批次、缓存窗口数量异常。

## 使用前提

- UI 服务已初始化（`UIComponent`），见 [UI 模块](index.md)。
- 窗口用 `[Window(layer, cacheTime)]` 声明层级和缓存时间，见 [窗口声明与生命周期](lifecycle.md)。
- 运行时数据只在 Play Mode 的 `UIComponent` Inspector 中可见；编辑器下没有真实服务实例。
- Profiler 和 Debugger 包是可选工具；框架自身的调试入口不依赖 Debugger。

## 最短接入

1. 按窗口的开关频率和资源大小选择 `cacheTime`。本仓库现有窗口就是四种典型取值：

| 窗口 | 声明 | 效果 |
| --- | --- | --- |
| `UIHomeWindow` | `[Window(UILayer.UI, 30)]` | 关闭后缓存 30 秒，返回主界面通常不用重载 |
| `UIShopWindow` | `[Window(UILayer.UI, 3)]` | 关闭 3 秒后销毁，短暂来回切换可复用 |
| `UIBuyAlertWindow` | `[Window(UILayer.Popup, -1)]` | 常驻缓存，保留弹窗状态 |
| `UIRecyclerViewExampleWindow` | `[Window(UILayer.UI, 0)]` | 关闭即销毁，下次打开重新加载 |

2. 只给真正需要每帧更新的视图重写 `OnUpdate`。下面的 `UITestAWidget` 是验证更新驱动的示例，每帧写日志，不要照搬：

```csharp
protected override void OnUpdate(float deltaTime)
{
    base.OnUpdate(deltaTime);
    Log.Info("UITestAWidget.OnUpdate()");
}
```

3. 重资源窗口在退出场景或确定不再需要时用 `force` 关闭，绕开缓存：

```csharp
GameApp.UI.CloseUI<UIShopWindow>(force: true);
```

4. 进入 Play Mode，选中场景里的 `UIComponent`，展开 `UI RuntimeDebug` 查看 Open / Cache / Update 数量与缓存列表；展开 `Navigation RuntimeDebug` 查看 history。

## 核心用法

### 缓存策略

`[Window]` 的 `cacheTime` 是唯一的缓存时长配置，注册后保存在 `UIMetaInfo.CacheTime`：

- `UICacheTime.None`（0）：关闭转场结束后直接销毁。
- `UICacheTime.Forever`（-1，任意负数同理）：不启动计时，常驻缓存。
- 正数：关闭转场结束后启动 unscaled 计时，到期销毁。

实际流程是：关闭转场结束 → `ReleaseClosedWindow` → 需要缓存时移出层级并 `AddToCache`；`AddToCache` 清空该窗口的 `UserData`，正数时长启动计时。重新打开或销毁时 `RemoveFromCache` 取消计时；计时到期直接 `DestroyWindow`。

- 缓存窗口状态是 `Closed`，`GetUI<T>` 仍能拿到实例，`IsOpen<T>` 为 false。
- 缓存期间不重新执行 `OnInitialize`；重新打开只走 `OnOpen` / `OnRefresh`。
- Widget 不参与这套缓存：`RegisterWidget` 固定使用 `UICacheTime.None`，Widget 只有“保留实例（`Close`）”和“销毁（`Destroy`）”两种结果。

框架没有缓存数量上限、LRU 或全局预算字段，只有每个窗口自己的 `cacheTime`。Inspector 在缓存数超过 16 时给出 `Cached window count is high` 提示，`Cached Windows` 列表最多展示 64 条；这两个数字是调试阈值，不是运行时限制。

### 更新与事件

- 只有注册信息里 `HasUpdate` 为 true 的视图才会进入更新列表；生成器会填入该标记，反射兜底则用 `UIBase.OverridesUpdate` 判断是否重写了 `OnUpdate`。重写一个空 `OnUpdate` 也会被每帧调用。
- 视图在 `Opening` / `Opened` 且父级启用时才更新；`Closing`、缓存中的 `Closed`、父窗口隐藏时都会从更新列表移除。
- `IServiceTickable.Tick` 先把 `_updateViews` 拷贝到缓冲区再遍历，稳态零分配；缓冲区只在视图数超过当前容量时按 2 的幂扩容。
- `EventProxy` 的订阅在视图停用时 `Suspend`、启用时 `Resume`；代理从 `MemoryPool` 取用，销毁时归还并退订。
- 每帧调用 `ShowUI` 刷新已打开的窗口并不可取：已打开时 `OpenView` 会走 `OnRefresh`，等于每帧执行一次刷新逻辑。

### 层级与 Canvas

`UILayer` 共 6 层：`Background`、`Scene`、`UI`、`Popup`、`Tips`、`Top`。UI 服务为每层创建一个全屏 `RectTransform` 根节点（`Layer{index}-{name}`），窗口通过 `GetLayer(UILayer)` 取到对应根节点。

- 每个窗口绑定 Holder 时会确保有一个自己的 `Canvas`（`overrideSorting = true`）和一个 `GraphicRaycaster`（缺失时补上）；`sortingOrder = Depth`。
- `Depth = (int)layer * 4000 + layerIndex * 20`，即 `LAYER_DEEP = 200 * 20`、`WINDOW_DEEP = 20`。同层后打开的窗口排在前面，点击时也会被 `BringToFront` 提到最前。
- 每层最多 200 个窗口。超过时 `Debug.Assert` 报 `Too many windows in one layer`，因为第 201 个窗口的排序值会与下一层重叠；断言在非开发构建中不保留，发布包不会拦截。
- 隐藏或缓存一个窗口只关 `Canvas.enabled`（Widget 无 Canvas 时改 `CanvasGroup.alpha`），节点本身仍在层级里；缓存窗口已移出层级，不参与渲染。
- 每个窗口的 `Canvas` 本身就是独立的排序与批次边界。同时可见的窗口越多，Canvas 越多；不要把大量只有一个元素的小面板拆成很多窗口，也不要在窗口内部随手再加子 Canvas。

### 隐藏窗口对组件的影响

框架从不把窗口或 Widget 的节点 `SetActive(false)`，只切换 Canvas / CanvasGroup 和交互组件。因此：

- 节点上的 `MonoBehaviour.Update`、Animator、ParticleSystem、AudioSource、协程仍会运行，也不会触发 `OnEnable` / `OnDisable`。
- 框架的 `OnUpdate` 已停止、`EventProxy` 已暂停、`GraphicRaycaster` 已关闭；但业务组件自己的每帧逻辑不受影响。
- 需要随窗口隐藏一起停止的粒子、动画、视频，要在 `OnClose` 停、`OnOpen` 恢复，或直接销毁窗口。
- `Visible` 是视图自己的逻辑状态：父窗口已关闭时，Widget 仍可能报告 `Visible == true`；判断实际启用用 `Holder.IsEnabled`。

### 每帧分配来源

| 来源 | 说明 |
| --- | --- |
| 框架更新驱动 | `Tick` 拷贝列表、清槽位、调 `OnUpdate`，稳态零分配；测试 `TickWithEnabledViewsIsAllocationFree` 覆盖 |
| 业务 `OnUpdate` | 每帧执行，`new`、LINQ、字符串拼接、闭包、装箱都会直接变成每帧 GC |
| 重复订阅事件 | `AddUIEvent` 每次调用都会追加订阅项且不查重，容量翻倍时分配数组 |
| 捕获变量的谓词 | `TryGetTopVisibleHolder` 要求传静态委托才零分配；每次 new 一个捕获 lambda 会分配 |
| 每帧 `ShowUI` | 已打开窗口会每帧执行 `OnRefresh`，不是“只是查一下” |
| TMP 文本刷新 | 每帧改文本会触发重建；只在数值变化时 `SetText` / `SetKey` |
| 转场播放 | `UIPresetTransition` 每帧推进，但开关循环的分配测试为零；不需要动画时用 `snap: true` |
| 页面导航 | 已加载页面之间导航的分配测试为零；新页面首次加载才有资源与实例化开销 |
| 关闭窗口 | `CloseUI` 返回结构体；不需要等待结果时不要 `await`，注释明确“不等待时不分配” |

### 调试信息

`IUIDebugService` 与 `UIService.Debug.cs` 只在 `UNITY_EDITOR` 下编译，接口是 internal，业务代码不要调用。Play Mode 下选中 `UIComponent`：

| 面板 | 内容 |
| --- | --- |
| `UI RuntimeDebug` → `Service Summary` | Initialized、正交/透视、Open、Cache、Update、Block 数量 |
| `Open Windows` | 按层列出窗口：State、Visible、Updating、Depth、CacheTime / 剩余时间、Holder Transform |
| `Cached Windows` | 缓存窗口列表，数量超过 64 条时提示截断 |
| `Runtime Alerts` | 缓存数 >16、缓存窗口仍可见、Holder 丢失、可见窗口 Depth 重复 |
| `Navigation RuntimeDebug` | Current、Can Back、History 条数，以及每条 history 的类型、是否根页面、`UserData` 预览 |

两个卡片每 250 ms 刷新一次，且只读。Debugger 包没有 UI 模块专属页；需要看全局内存、引用池和计时器时，用它的 `Profiler/Summary`、`Profiler/Reference Pool`（`EventListenerProxy` 池）和 `Profiler/Timer`（缓存计时器），见 [Debugger 调试器](../debugger/index.md)。

### 优化清单

| 优化项 | 源码依据 |
| --- | --- |
| 只重写需要的 `OnUpdate`，不要留空实现 | `UIMetaInfo.HasUpdate`、`UIBase.SetEnabled` |
| 隐藏期间要停的粒子/Animator/协程，自己停 | `UIWindow.ApplyVisible` 只切 `Canvas.enabled`，从不 `SetActive(false)` |
| 高频小窗用正数缓存，重资源/低频窗口用 `None` 或 `force` | `WindowAttribute.CacheTime`、`ReleaseClosedWindow` |
| 不要把所有窗口设成 `Forever` | 缓存无数量上限，只有 `cacheTime`；Inspector 仅 >16 提示 |
| 同层窗口数控制在 200 以内 | `WindowsPerLayerLimit`、`LAYER_DEEP`，超出时排序与下一层重叠 |
| 事件在 `OnInitialize` 订阅一次 | `EventListenerProxy.AddSubscription` 不去重 |
| 查询用静态谓词，避免每次分配 lambda | `FindTopOpen` 注释与 `TopVisibleHolderQueryIsAllocationFree` 测试 |
| 页签和 Widget 复用实例，关闭用 `Close` 而非反复创建 | `UITabWindow.SwitchTab` 懒创建并保留已加载页签 |
| 不需要动画时 `snap: true` 或 `None` 预设 | `IUITransition.Open/Close(snap)`、`UIPresetTransition.Play` |

## 常见配置

| `cacheTime` 取值 | 关闭后行为 | 适用 |
| --- | --- | --- |
| `UICacheTime.None`（0） | 转场结束立即销毁 | 大界面、低频页面、希望彻底回收的窗口 |
| 正数（秒） | 转场结束后按 unscaled 计时，到期销毁 | 高频开关、希望保留滚动位置或输入状态的小窗 |
| `UICacheTime.Forever`（-1） | 不销毁，直到 `force` 关闭或服务释放 | 全局常驻、需要保留状态的弹窗 |

## 容易踩的坑

1. **`Forever` 不是免费的。** 缓存没有全局上限，也不做 LRU；所有窗口都常驻时内存只会增长，Inspector 的 >16 提示只是告警，不会自动清理。
2. **缓存/隐藏不等于停止。** 节点仍激活，Animator、粒子、`MonoBehaviour.Update` 和协程照常跑；只在 `OnClose` 停掉它们，或不再需要时销毁窗口。
3. **`Visible` 会骗人。** 它不检查父级；父窗口关闭后 Widget 仍可能是 `Visible == true`，实际启用状态看 `Holder.IsEnabled`。
4. **每层 200 个窗口的断言只在编辑器/开发构建生效。** 发布构建中超出的窗口排序会与下一层重叠，表现为弹窗被压在下面或点击错位。
5. **每帧 `ShowUI` 等于每帧 `OnRefresh`。** 刷新已打开窗口请用事件或显式数据变更，不要在 `Update` 里反复 `ShowUI`。
6. **每帧写日志的示例不能照抄。** `UITestAWidget.OnUpdate` 每帧 `Log.Info` 只用于验证更新驱动，线上会持续产生日志和托管分配。

## API 速查

| 成员 | 说明 |
| --- | --- |
| `[Window(UILayer windowLayer, int cacheTime = UICacheTime.None)]` | 声明窗口层级与缓存时长 |
| `UICacheTime.None` / `UICacheTime.Forever` | 关闭即销毁 / 常驻缓存 |
| `IUIService.CloseUI<T>(bool force = false, bool snap = false)` | `force` 跳过缓存直接销毁；`snap` 跳过关闭转场 |
| `CloseSelf(bool force = false, bool snap = false)` | 窗口逻辑类内关闭自己 |
| `IUIService.TryCloseTopAsync(Predicate<RuntimeTypeHandle>, bool force = false)` | 关闭最高层匹配窗口，可强制销毁 |
| `IUIService.IsOpen<T>()` / `GetUI<T>()` | 是否逻辑打开 / 取打开或缓存中的实例 |
| `UIHolderObjectBase.IsEnabled` | 视图及父级当前是否启用，比 `Visible` 更接近实际可见 |
| `IUIDebugService` | 仅编辑器 internal 调试接口，通过 `UIComponent` Inspector 使用 |
