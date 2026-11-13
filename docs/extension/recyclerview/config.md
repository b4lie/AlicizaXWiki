# 配置与 API

`RecyclerView` 的主要配置项：

| 配置 | 说明 |
| --- | --- |
| `Direction` | `Vertical`、`Horizontal`、`Custom` |
| `Center Cross Axis` | 内容不足一屏时是否在交叉轴居中 |
| `Spacing` | 项间距 |
| `Padding` | 内容内边距 |
| `Movement Type` | `Elastic` 回弹或 `Clamped` 硬限制 |
| `Scroll Mode` | `AlwaysDisable`、`AlwaysEnable`、`WhenScrollable` |
| `Snap` | 停止滚动后吸附到最近项 |
| `Snap Duration` | 吸附动画时长 |
| `Refresh Mode` | `Immediate` 或 `AfterSettle`（滚动停止后刷新） |
| `Inertia` | 是否启用惯性滑动 |
| `DecelerationRate` | 惯性减速率，范围 `[0.001, 0.999]`，值越小减速越快 |
| `ScrollSpeed` | 平滑滚动速度系数 |
| `WheelSpeed` | 鼠标滚轮速度系数 |
| `ScrollbarVisibility` | `AlwaysHide`、`AlwaysShow`、`WhenScrollable` |
| `Templates` | 模板数组，混合和分组列表通过下标选择模板 |

布局类型：

| 类型 | 说明 |
| --- | --- |
| `LinearLayoutManager` | 单列或单行。定高读模板尺寸；打开 `Variable Item Length` 后按每条声明/实测高度排 |
| `GridLayoutManager` | 定高网格，`cellCount` 控制每行或每列数量。可见窗口按行头步进 |
| `PageLayoutManager` | 定高分页，继承线性布局并对可见项做近大远小缩放。横向封面卡需要 `Snap` |
| `MixedLayoutManager` | 多模板列表。未开变高时按 `TemplateId` 读模板长度；打开变高后同模板也可不同高 |
| `CircleLayoutManager` | 定高圆形布局，使用虚拟布局区间并按角度摆放 |

滚动条在 `WhenScrollable` 模式下只会在内容尺寸大于视口尺寸时显示并允许交互。对 `Direction.Custom`，溢出检测不生效。

## API 速查

| API | 说明 |
| --- | --- |
| `UGListCreateHelper.Create<T>(RecyclerView)` | 创建普通列表 |
| `UGListCreateHelper.CreateMixed<T>(RecyclerView)` | 创建混合模板列表 |
| `UGListCreateHelper.CreateLoop<T>(RecyclerView)` | 创建循环列表 |
| `UGListCreateHelper.CreateGroup<T>(RecyclerView, int)` | 创建分组列表，第二个参数是组头模板下标 |
| `UGListBase.SetData(list, policy)` | 替换数据源；`KeepAnchor` 保可见位置，`ResetScroll` 回起点 |
| `UGListBase.Add` / `Insert` / `RemoveAt` | 结构增量更新 |
| `UGListBase.RefreshItem(index, relayout)` | 重绑可见项；`relayout: true` 时重测高度 |
| `UGListBase.StickToEnd()` | 贴底，聊天发消息用 |
| `UGListBase.IsAtEnd` / `IsScrolling` | 是否贴底 / 是否正在滚动 |
| `UGListBase.ScrollDataIndex` | 当前滚动对应的数据索引 |
| `UGListBase.ChoiceIndex` | 业务选择索引 |
| `UGListBase.FocusedDataIndex` | 导航焦点索引；未接导航包时为 `-1` |
| `ViewHolder.RequestResize()` | 绑定后实测高度写回 |
| `RecyclerView.TrimInactivePool()` | 裁剪对象池中的非活动实例 |
| `RecyclerView.PoolStats` | 开发环境下查看对象池统计 |
