# 列表导航

`RecyclerViewNavigationController` 让虚拟列表加入手柄/键盘导航。列表项是可回收实例，所以焦点绑的是**数据索引**而不是某个 `GameObject`；控制器自己是 EventSystem 里唯一的选中对象，行只是它的"光标"。这一页讲挂载、移动规则、边界与遮挡；点击与选中语义见 [数据更新与交互](interaction.md)。

## 适用场景

- 手柄或键盘要在列表里上下/左右选，选中项要有高亮框。
- 从按钮/页签按方向键进入列表，到头再按方向键离开列表。
- 弹窗盖住列表后要暂停导航，关掉后恢复原来那一行。
- 网格背包要按行列走，而不是一条线走到底。

## 使用前提

- 装了 `com.alicizax.unity.input`（提供 `NavigationScope` 与 `UXInput`）。控制器整体包在 `#if INPUTSYSTEM_SUPPORT && UXNAVIGATION_SUPPORT` 里：宏由 asmdef 的 versionDefines 生成，依赖 `com.unity.inputsystem >= 1.0.0` 与 `com.alicizax.unity.input >= 1.0.0`。缺任一个，`RecyclerViewNavigationController` 类型根本不存在。
- 控制器与 `RecyclerView` 必须同物体（`[RequireComponent(typeof(RecyclerView))]`，`AddComponentMenu` 是 `UI/RecyclerView Navigation Controller`）。
- 层级里要有 `NavigationScope`（窗口/面板根上），并把控制器填进它的 `Scope` 字段；运行期不会自动向上查找，只有编辑器 `OnValidate` 会填。
- 需要参与导航的模板必须继承 `NavigationViewHolder<TData>` 或 `InteractiveViewHolder<TData>`；普通 `ViewHolder<TData>` 的行会被跳过。

## 最短接入

1. 组件摆位：`RecyclerView` + `Scroller` + `RecyclerViewNavigationController` 放同一个物体，`Content` 与模板照常。
2. 模板继承 `InteractiveViewHolder<TData>`（同时要鼠标点击）或 `NavigationViewHolder<TData>`（只要导航），重写 `OnBind`，需要焦点视觉就重写 `OnNavigationFocused`。
3. 窗口/面板根的 `NavigationScope` 里把这个控制器加进 `Baked Selectables`（或运行期设 `DefaultSelectable`），`Default` 选它：打开窗口后 EventSystem 选中的就是列表，第一次方向键把焦点建立到首屏。控制器与所有可导航按钮都要在同一个 Scope 里。
4. 到达边界要离开列表时，配控制器的 `Navigation`（推荐 `Explicit` + `Select On` 各方向指向外层控件）。

```csharp
public sealed class BagItemHolder : InteractiveViewHolder<BagItemData>
{
    [SerializeField] private GameObject focusFrame;

    protected override void OnBind(BagItemData data, int index)
    {
        focusFrame.SetActive(false);          // 新绑定不持有焦点
    }

    protected override void OnNavigationFocused(bool focused)
    {
        focusFrame.SetActive(focused);        // 只画焦点视觉
    }

    protected override void OnItemClick(BagItemData data, int index)
    {
        // 鼠标左键与手柄提交共用这一个入口
    }
}
```

## 核心用法

### 挂载与 Inspector 字段

| Inspector | 序列化字段 | 说明 |
| :--- | :--- | :--- |
| `RecyclerView` | `recyclerView` | 同物体上的组件，`Awake` 自动补 |
| `Scope` | `navigationScope` | 父级最近的 `NavigationScope`，编辑器自动填；运行期不查找 |
| `Wrap` | `wrapNavigation` | 普通列表首尾是否环绕；环形布局与 `UGLoopList` 恒环绕 |
| `Keep Focus` | `keepFocusVisibleWhenSuppressed` | 被上层遮挡时是否保留焦点视觉，默认开 |
| `Smooth` / `Duration` | `smoothScroll` / `smoothScrollDuration` | 焦点移动是否平滑滚，默认开 / `0.18` |
| `Alignment` | `focusAlignment` | 焦点项对齐方式，默认 `Center`；`Start` 时改走 `ScrollToIndex` |
| `Navigation` | 基类 `m_Navigation` | `Mode`、`Wrap Around`、`Select Up/Down/Left/Right`，边界出口在这里 |

控制器在 `OnEnable` 里向 `Scope` 注册自己（`RegisterSelectable(this, true)`），并订阅 `NavigationSuppressionChanged`；`OnDisable` 注销并清掉焦点视觉。焦点索引本身在禁用期间保留。

### 焦点进入列表

- 控制器被 EventSystem 选中（`OnSelect`）后，方向输入才走它的 `OnMove`。
- 还没有焦点时，第一次方向输入**只用来建立焦点**，不会移动：起点是 `ScrollDataIndex`，没有就用首屏第一行的数据索引。
- 业务主动进入用 `RestoreFocus(index)`：越界会改投最近的可聚焦项，并顺手把 EventSystem 选中切到列表。只改位置用 `SetFocus(index, scroll: true)`，它不抢选中。
- 鼠标点击某行也会把焦点记到该行并选中列表，但鼠标模式下**不显示光标**；之后一按方向键光标才出现。

```csharp
[SerializeField] private RecyclerViewNavigationController navigation;

// 打开窗口后直接把焦点送进列表第 12 行（越界自动取最近可聚焦项）
navigation.RestoreFocus(12);

// 只改焦点位置，不动 EventSystem 选中
navigation.SetFocus(3, scroll: true);

// 让某些行不可聚焦（空槽位、装饰行）
navigation.SetFocusFilter(index => index % 5 != 0);
```

### 列表内移动规则

| 布局 | 上/下 | 左/右 | 环绕 |
| :--- | :--- | :--- | :--- |
| `LinearLayoutManager`（垂直） | ±1 | 不消费，交给 `Navigation` 离开 | 仅 `Wrap` / Loop |
| `LinearLayoutManager`（水平） | 不消费，离开 | ±1 | 仅 `Wrap` / Loop |
| `GridLayoutManager`（垂直） | ±`Cell Count`（跨行） | ±1（行内） | `Wrap` 时跳整行 |
| `GridLayoutManager`（水平） | ±1（行内） | ±`Cell Count`（跨列） | `Wrap` 时跳整列 |
| `CircleLayoutManager` | ±1 | ±1 | 恒环绕 |

细节：网格的"行内/跨行"由移动方向与布局方向共同判定，不看索引差；跨行越过首/末行是边界（`Wrap` 时补一整行，且落点必须可聚焦，否则仍算边界）。`Wrap` 在找不到可聚焦项时，线性/环形会消费输入但不移动，网格则按边界处理（可以离开列表）。

### 离开列表（边界出口）

移动算到边界时，控制器返回"不消费"，事件继续交给基类 `Selectable.OnMove` 按 `Navigation` 配置找下一个控件：

| 配置 | 结果 |
| :--- | :--- |
| `Mode = Explicit` + `Select On Right = 外部按钮` | 网格行内到头按右键 → 焦点去按钮，列表记住当前行；垂直单列列表则是任意左右键直接离开 |
| 没配任何 `Select On` | 输入不被消费，焦点停在原行（测试 `BoundaryWithoutConfiguredExitDoesNotConsumeTheInput` 固定这一行为） |
| `Wrap = true`、环形、`UGLoopList` | 没有边界，永远消费输入，只能靠提交或外部逻辑离开 |

`Select On` 的目标必须是 `Selectable`，并且它自己要能导航回来（反向配 `Select On Left` 指向控制器），否则焦点一去不回。

### 遮挡与 `NavigationScope`

`NavigationScope` 负责"哪一层可以导航"。控制器只认它给的抑制状态：

1. 上层窗口打开或同级更高层的 Scope 生效 → 本层 `NavigationSuppressed` 变 `true` → 控制器 `suppressed`。
2. 被抑制时 `HandleMove` / `HandleSubmit` 一律返回 `false`，不消费输入；焦点索引保留不变。
3. 是否还显示焦点视觉只由 `Keep Focus` 决定（默认保留）。
4. 抑制解除后重新选中控制器，光标回到原行；如果那一行已经不可聚焦（数据变少、模板换了），会改投最近的可聚焦项。

```csharp
// 被遮挡时不要自己清焦点，控制器会保留索引
bool blocked = navigation.IsSuppressed;
int lastRow = navigation.FocusedDataIndex;   // 遮挡期间依然可读
```

Scope 的默认选中、记忆上次焦点、阻断下层这些策略见 [UI 导航](../../input/navigation.md)。

### 焦点与业务选中

这是两套状态，别混：

| 状态 | 读写 | 变化入口 |
| :--- | :--- | :--- |
| 导航焦点 | `FocusedDataIndex` / `HasFocus` | `SetFocus` / `RestoreFocus` / 方向输入 / 鼠标点击 |
| 业务选中 | `ChoiceIndex` / `HasChoice` | `SetSelect()` / `ChoiceIndex = ...` / `InteractiveViewHolder.SelectOnActivate` |

`NavigationViewHolder.HandleNavigationSubmit` 默认返回 `false`，此时控制器回退为 `adapter.SetChoiceIndex(FocusedDataIndex)`；`InteractiveViewHolder` 覆盖了它并恒返回 `true`，提交走 `OnItemClick`。想让提交只是"选中"而不触发业务，就把模板换成 `NavigationViewHolder`，或重写 `SelectOnActivate => false`。

### 数据变化时的焦点修复

- `Adapter` 的结构变化会触发 `RecyclerView.OnLayoutChanged`，控制器再做一次 `FocusNearest`：焦点索引还在范围内就保持，越界或该行不可聚焦就找最近的。
- 它按**数字索引**修，不按数据身份；插入/删除/分组折叠让行整体位移时，焦点可能落在同一索引的另一条数据上。需要精确恢复时自己调 `RestoreFocus`。
- 数据清空后焦点变 `-1`；重新灌数据后要再 `RestoreFocus(0)`。

### 示例：从窗口进入列表再返回

`UIShopWindow` 的配法可以直接照搬：

1. 窗口根的 `NavigationScope`：`Default` 选关闭按钮，`Remember` 打开。
2. 列表控制器：`Alignment End`、`Smooth` 开，`Navigation` 设 `Explicit` + `Select On Right` 指向关闭按钮。
3. 关闭按钮：`Navigation` 设 `Explicit` + `Select On Left` 指向列表控制器。
4. 打开窗口 → 焦点在关闭按钮；按左进入列表，一次方向键建立焦点并显示光标；列表内自由移动；按右回到关闭按钮；再按左回到列表时停在原行（Scope 的 `Remember`，或自己 `RestoreFocus(last)`）。

## 常见配置

| 场景 | 控制器配置 | 外层配置 |
| :--- | :--- | :--- |
| 列表是窗口主体（商店） | `Alignment End`、`Smooth` 开、`Wrap` 关 | 控制器 `Select On Right` → 关闭按钮，按钮 `Select On Left` → 控制器，Scope `Default` 选按钮 |
| 网格背包 | `Wrap` 关（行内到头就离开） | 控制器 `Select On Up/Down` 指向页签或按钮 |
| 轮播/环形选择 | `Alignment Center`、`Wrap` 可关（环形恒环绕） | 通常不配出口，用提交确认 |
| 聊天/日志流 | `Alignment End`、`Smooth` 开、`Keep Focus` 开 | 只配上下出口 |
| 弹窗场景 | `Keep Focus` 视需求开关 | 弹窗自己要有 `NavigationScope` 才会遮住列表 |

示例值来自 `UIShopWindow.prefab`：`focusAlignment: 2`（`End`）、`smoothScrollDuration: 0.18`、`keepFocusVisibleWhenSuppressed: 1`、`wrapNavigation: 0`，Scope 开了 `Remember`。

## 容易踩的坑

1. **没装 input 包却写了 `RecyclerViewNavigationController`**：整个类型被宏裁掉，编译直接报找不到类型；先确认 `com.alicizax.unity.input` 已安装。
2. **控制器没挂 Scope**：`OnEnable` 不会注册，方向键永远进不了列表；`Scope` 字段只在编辑器自动填，实例化后的预制体也要检查。
3. **模板还是 `ViewHolder<TData>`**：这一行不可聚焦，方向键会直接跳过；整列都是普通模板时表现为"列表完全不响应导航"。
4. **只调 `SetFocus` 不选中列表**：光标要等 EventSystem 选中控制器后才画；要一步到位用 `RestoreFocus`。
5. **边界没配 `Select On`**：输入不被消费但也不移动，看起来像"卡住"；这是刻意设计，避免吞掉输入导致 EventSystem 永远出不去。
6. **`Wrap` 当成循环列表开关**：`Wrap` 只影响首尾；`UGLoopList` 会自动环绕，环形布局同理。
7. **靠焦点索引当业务身份**：插入/删除/分组折叠后同一索引可能换行；跨帧保存请存业务 id，恢复时再换回索引。
8. **遮挡期间点列表**：被抑制时点击不改焦点也不抢选中；同帧抑制也已生效，不要用点击去"抢回"焦点。
9. **自己实现 `IRecyclerViewNavigationViewHolder`**：那是 internal 契约，业务只能继承 `NavigationViewHolder` / `InteractiveViewHolder`。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `FocusedDataIndex` / `HasFocus` / `IsSuppressed` | 焦点行、是否有焦点、是否被遮挡 |
| `SetFocus(int dataIndex, bool scroll = true)` | 设焦点，越界清空；不抢 EventSystem 选中 |
| `RestoreFocus(int dataIndex)` | 设焦点（越界取最近）并把 EventSystem 选中切回列表 |
| `SetFocusFilter(Predicate<int>)` | 可聚焦过滤，`null` 表示不过滤；会立即修正当前焦点 |
| `HandleMove(AxisEventData)` / `HandleSubmit()` | 手动驱动一次移动/提交，返回是否消费 |
| `NavigationViewHolder.HandleNavigationMove` | 覆盖后返回 `true` 可让当前行吃掉方向输入 |
| `NavigationViewHolder.HandleNavigationSubmit` | 返回 `false` 时回退为业务选中 |
| `OnNavigationFocused(bool)` | 业务侧画焦点视觉的入口 |
| `UGList.FocusedDataIndex` / `SyncChoiceToFocus()` | 业务层读取焦点 / 把焦点同步成业务选中 |
