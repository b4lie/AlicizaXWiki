# 配置与 API

这一页是 `RecyclerView` 的配置总表：Inspector 上的每个序列化字段、五种布局管理器的参数、`ScrollAlignment`、性能开关，以及几套可直接照抄的组合。具体怎么用某个列表，见 [普通列表](list.md) / [多模板列表](mixed.md) / [循环列表](loop.md) / [分组列表](group.md)。

## 适用场景

- 第一次在预制体上搭一个 `RecyclerView`，想知道每个格子填什么。
- 列表能显示但滚不动、滚动条不出来、行高不对，需要对着字段排查。
- 要给某个业务场景选布局与参数（定高列表、背包格、聊天流、分组、翻页、环形）。
- 想知道哪些字段运行期还能改。

## 使用前提

- `RecyclerView` 挂在滚动区域的根节点上，同物体还要有 `Scroller`（`Scroller Type` 选它），否则拖拽、滚轮、惯性全部不工作。
- 必须有 `Content`（承载实例的 `RectTransform`）与非空 `Item Templates`，否则 `Awake` 校验失败，列表什么都不显示。
- 模板根节点在滚动轴上**不能是拉伸锚点**（垂直列表要求 `anchorMin.y == anchorMax.y`），否则校验报 `template is stretched on the scroll axis`。
- 需要滚动条时，`Scrollbar` 物体上必须再挂 `ScrollbarEx`，否则运行期报 `ScrollbarEx is missing`。

## 最短接入

1. 摆层级。模板与实例的关系如下，实例由对象池在 `Content` 下创建：

   ```text
   ScrollView                     RecyclerView + Scroller（+ 导航控制器，可选）
   ├── Content                    Content 字段指向它
   │   ├── ItemTemplate           挂 ViewHolder 子类，就是 Templates[0]
   │   └── HeaderTemplate         挂 ViewHolder 子类，就是 Templates[1]
   └── Scrollbar                  可选，Scrollbar + ScrollbarEx
   ```

2. Inspector 里最少填四项：`Layout Type`（选布局）、`Content`、`Item Templates`（把模板拖进来，顺序即 `TemplateId`）、`Scroller Type`（选 `Scroller`）。
3. 代码里用 `UGListCreateHelper` 建列表并灌数据，不要直接 `SetAdapter`（它是 `internal` 且只允许绑定一次）：

   ```csharp
   using AlicizaX.UI;

   UGList<ItemData> list = UGListCreateHelper.Create<ItemData>(recyclerView);
   list.SetData(items);
   ```

4. 要滚动条就把 `Scrollbar Visibility` 从 `AlwaysHide` 改成 `WhenScrollable`，编辑器会按 `Direction` 自动挂一个带 `ScrollbarEx` 的滚动条。

## 核心用法

### RecyclerView 字段总表

| 序列化字段 | Inspector | 说明 |
| :--- | :--- | :--- |
| `layoutManager` | `Layout Type` | `[SerializeReference]` 布局实例；旁边 `_layoutManagerTypeName` 存类型名用于修复。改了会重建布局 |
| `scroller` | `Scroller Type` | 同物体上的 `Scroller` 或 `CircleScroller`；`_scrollerTypeName` 存类型名 |
| `direction` | `Direction` | `Vertical`(0) / `Horizontal`(1)，主轴方向 |
| `centerCrossAxis` | `Center Cross Axis` | 内容不足一屏时在交叉轴居中 |
| `content` | `Content` | 实例父节点，必填 |
| `spacing` | `Spacing` | 主轴/交叉轴间距；只影响内容尺寸 |
| `padding` | `Padding` | 内容内边距，参与内容尺寸与首项起点 |
| `scroll` | `Scroll Mode` | `AlwaysDisable` / `AlwaysEnable`(默认) / `WhenScrollable` |
| `scrollbarVisibility` | `Scrollbar Visibility` | `AlwaysHide`(默认) / `AlwaysShow` / `WhenScrollable` |
| `scrollbar` | （不直接编辑） | 由 `Scrollbar Visibility` 自动创建/清空，需配 `ScrollbarEx` |
| `snap` / `snapDuration` | `Snap` / `Snap Duration` | 停止后吸附到最近项，时长默认 `0.12`（最小 `0.01`） |
| `refreshMode` | `Refresh Mode` | `Immediate`(默认) / `AfterSettle`，见 [数据更新与交互](interaction.md) |
| `movementType` | `Movement Type` | `Elastic`(默认，越界回弹) / `Clamped`(硬边界) |
| `inertia` | `Inertia` | 默认开；关掉松手即停 |
| `decelerationRate` | `Deceleration Rate` | `0.001`–`0.999`，默认 `0.135`；越小减速越快 |
| `scrollSpeed` | `Scroll Speed` | `0.5`–`50`，默认 `7`；平滑滚动的速度系数（时长按距离推算） |
| `wheelSpeed` | `Wheel Speed` | `1`–`50`，默认 `30`；滚轮每格位移 |
| `templates` | `Item Templates` | `ViewHolder[]`，下标即 `TemplateId`；运行期禁止更换 |
| `viewProvider`（非序列化） | — | 内部对象池，无 Inspector 字段；用 `PoolStats` 观察、`TrimInactivePool()` 裁剪 |

Inspector 顶部还有两块只读区：`Setup` 诊断（缺 `Content`/`Templates`/`Layout Manager`/`Scroller`、变长支持情况、模板拉伸警告，正常时显示 `OK`）与 `Runtime`（Play Mode 下显示 `Visible Range`、`Scroll Data Index`、`Motion`、`Pool`）。缺引用时会出现 `Missing References` 修复按钮（`Restore Layout Manager` / `Restore Scroller Component` / `Add ScrollbarEx`）。

### 布局管理器参数

`Layout Type` 的下拉项来自工程里所有实现 `ILayoutManager` 的类型；选中后下面自动列出该布局的序列化参数。

| 布局 | 关键参数 | 适用与限制 |
| :--- | :--- | :--- |
| `LinearLayoutManager` | `Variable Item Length` | 单列线性。定高时所有行取显示列表第 0 行的模板尺寸；变长需数据实现 `IMeasuredViewData` 或绑定后实测 |
| `GridLayoutManager` | `Cell Count`（默认 1） | 网格，`Cell Count` 就是每行/列格数；单元格尺寸取第 0 行模板；不支持变长 |
| `PageLayoutManager` | `Min Scale`、`Scale Falloff`，以及**不能开**的 `Variable Item Length` | 翻页，近大远小；固定格长，常配 `Snap` 与横向 |
| `MixedLayoutManager` | `Variable Item Length` | 按模板取高度，多模板与分组首选；固定长度也按模板算，所以组头/子项可不同高 |
| `CircleLayoutManager` | `Circle Direction`、`Interval Angle`、`Max Visible Item Count`、`Initial Angle` | 环形；`Interval Angle <= 0` 时按 `360/数量` 均分；配 `CircleScroller` |

`PageLayoutManager` 继承自 `LinearLayoutManager`，Inspector 会显示 `Variable Item Length`，但它 `SupportsVariableItemLength == false`，打开会抛 `NotSupportedException`，别动它。

### `ScrollAlignment`

| 值 | 数值 | 定位结果 |
| :--- | :--- | :--- |
| `Start` | 0 | 目标项顶边贴视口起始边（`ScrollToIndex` 用的就是它） |
| `Center` | 1 | 目标项居中；导航控制器默认对齐方式 |
| `End` | 2 | 目标项底边贴视口结束边；商店列表示例用 `End` |

`offset` 叠在对齐结果上，最终都会被 `Scroller.ClampPosition` 夹进 `0..MaxPosition`。

### 运行期能改什么

Play Mode 下 Inspector 会锁住 `Layout Type`、`Scroller Type`、`Direction`、`Center Cross Axis`、`Content`、`Scroll Mode`、`Scrollbar Visibility` 与模板拖拽；其余数值项改动会立即生效（编辑器会调 `ApplySerializedScrollerSettings` 并同步 `Spacing`/`Padding`/`Snap`/`SnapDuration`/`RefreshMode`）。这些都对应同名公共属性，代码里也能改。

### 性能相关的开关

| 开关 | 代价 / 收益 |
| :--- | :--- |
| `Refresh Mode = AfterSettle` | 拖拽/惯性期间不做重建，落定后补一次；代价是期间几何滞后 |
| `Variable Item Length` 关闭 | 不建偏移表、不逐项测量，定高列表应关 |
| `Scrollbar Visibility = AlwaysHide` | 不创建滚动条，也不做滚动条尺寸计算 |
| `Scroll Mode = WhenScrollable` | 内容不溢出时禁手势；`AlwaysDisable` 连 `ScrollTo` 也一起禁 |
| `Inertia = false` | 省掉惯性推进，松手立即停 |
| `TrimInactivePool()` | 裁剪空闲池；`PoolStats` 只在编辑器/开发包有值 |
| `Grid Cell Count` | 决定布局单元 `Unit`；池预热按「可见数 + Unit×2」（上限 128） |

### 限制

- `Content` / `Templates` / `Layout Type` / `Scroller Type` 运行期不可更换；`SetAdapter` 是 `internal`，一个 `RecyclerView` 只允许绑一次。
- 变长只支持 `LinearLayoutManager` 与 `MixedLayoutManager`；`Grid` / `Page` / `Circle` 保持定高。
- `LoopAdapter` 不能配 `MixedLayoutManager` 或 `CircleLayoutManager`；`MixedAdapter` 必须配 `MixedLayoutManager`，组合非法会在建列表时直接抛异常。
- 圆形布局不支持对齐滚动：`UGList.ScrollTo` / `ScrollToStart` / `ScrollToCenter` / `ScrollToEnd` 会抛 `NotSupportedException`。
- 内容不足一屏时 `MaxPosition == 0`，所有滚动 API 都没有实际位移。
- 模板下标就是 `TemplateId`，越界会让 `ViewProvider` 报 `adapter returned an invalid template id`。

## 常见配置

| 场景 | `Layout Type` | 关键参数 | 其他 |
| :--- | :--- | :--- | :--- |
| 定高单列 | `Linear` | `Variable Item Length` 关 | `Spacing (12,10)`、`Padding (16,12)`、滚动条 `AlwaysHide` |
| 背包网格 | `Grid` | `Cell Count 2`（示例）/ `3`（商店） | `Spacing (90,10)` 或 `(35,20)`、`Scrollbar WhenScrollable` |
| 聊天流 | `Linear` | `Variable Item Length` 开 | `Spacing (0,12)`、`Refresh Mode AfterSettle`、贴底 |
| 分组列表 | `Mixed` | 两个模板，`Variable Item Length` 关 | `Spacing (0,10)`、`groupTemplateId = 0` |
| 多模板流 | `Mixed` | 三个模板，`Variable Item Length` 开 | `Spacing (12,10)`、`Refresh Mode Immediate` |
| 翻页 | `Page` | `Min Scale 0.9`、`Scale Falloff 0.0006` | `Direction Horizontal`、`Snap` 开、`Spacing (30,0)` |
| 环形 | `Circle` | `Interval Angle 0`、`Max Visible 32`、`Initial Angle 90` | `Scroller Type = CircleScroller`、`Padding (8,8)` |

以上数值取自 `UIRecyclerViewExampleWindow` 与 `UIShopWindow` 的预制体，可以直接当基线再微调。

## 容易踩的坑

1. **只挂 `RecyclerView` 不挂 `Scroller`**：列表能显示但拖不动、滚轮无反应；`Scroller Type` 保持 `None` 时最容易被忽略。
2. **模板在滚动轴上被拉伸**：`Content` 下用 stretch 锚点的模板会让校验失败（`template is stretched on the scroll axis`），整张列表不渲染。
3. **校验失败在 Release 下是静默的**：`LogValidationError` 只在编辑器与 `DEVELOPMENT_BUILD` 输出，正式包表现为「列表空的」，先回编辑器看 `Setup` 面板。
4. **`Content` 填成了模板本身或 `ScrollView` 根**：实例会跟着模板一起被隐藏，或挂到错误父节点导致位置错乱；`Content` 应是专门的空节点。
5. **给 `Page` 打开 `Variable Item Length`**：直接抛 `NotSupportedException`。`Grid`/`Circle` 没有这个字段（本来就不支持变长）。
6. **滚动条物体缺 `ScrollbarEx`**：运行期报错且拖拽结束不触发吸附，`AlwaysHide` 之外的可见性都可能踩到。
7. **`Spacing` 与 `Padding` 混用错误轴向**：垂直列表主轴是 `y`、交叉轴是 `x`，写反会得到奇怪的行距与左右留白。
8. **以为改 `Layout Type` 会保留原布局参数**：切换类型是新建布局实例，旧参数不会带过去，切换后逐项复查。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `Direction` / `CenterCrossAxis` / `Spacing` / `Padding` | 布局设置，改完登记重排 |
| `Scroll` / `ScrollbarVisibility` / `Scrollbar` | 滚动能力与滚动条 |
| `MovementType` / `Inertia` / `DecelerationRate` / `ScrollSpeed` / `WheelSpeed` | 手感参数，直接下发给 `Scroller` |
| `Snap` / `SnapDuration` / `RefreshMode` | 吸附与刷新模式 |
| `Templates` / `Content` / `LayoutManager` / `Scroller` | 只读引用，便于运行期查询 |
| `PoolStats` / `TrimInactivePool()` | 池统计与裁剪 |
| `ScrollToAligned(int, ScrollAlignment, float, bool, float)` | 按对齐方式定位 |
| `UGListCreateHelper.Create/CreateMixed/CreateLoop/CreateGroup` | 业务层列表工厂 |
