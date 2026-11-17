# 分组列表

`UGGroupList<TData>` 把一份平铺的子项数据按 `IGroupViewData.Type` 投影成「组头 + 组内项」的显示列表，并负责展开/折叠。组头固定使用构造时传入的 `groupTemplateId` 模板，子项使用自己的 `TemplateId`。这一页讲怎么把它接起来，布局与滚动参数见 [配置与 API](config.md)。

## 适用场景

- 任务、背包、商店、设置这类需要「分类头 + 若干子项」的列表。
- 组头要能点击或手柄提交来展开/折叠。
- 组头和子项尺寸不同，需要两套模板。

## 使用前提

- `RecyclerView` 已配好 `Content`、至少两个模板、布局管理器和 `Scroller`，搭法见 [配置与 API](config.md)。
- 数据类型实现 `IGroupViewData`（即 `TemplateId` + `Expanded` + `Type`），并且有公共无参构造：`UGGroupList` 的约束是 `where TData : class, IGroupViewData, new()`。
- 一个 `RecyclerView` 只能被创建一次列表。`UGListBase` 构造时调用 `RecyclerView.SetAdapter`，再次绑定会抛 `InvalidOperationException`；复用一个列表请改数据，不要重建。
- 布局建议用 `MixedLayoutManager`：它按模板取高度，组头与子项可以不同高。`LinearLayoutManager` 定高模式所有行都取「第 0 行模板」的尺寸。

## 最短接入

1. 在预制体上准备两个模板并拖进 `Item Templates`：`#0` 组头、`#1` 子项（顺序只为举例，`groupTemplateId` 传几都行）。
2. 定义实现 `IGroupViewData` 的数据类。
3. 用 `UGListCreateHelper.CreateGroup<TData>(recyclerView, groupTemplateId)` 建列表，`groupTemplateId` 传组头模板下标。
4. `SetData` 灌子项，再显式 `Expand` 需要的组（默认全部折叠）。

```csharp
using System.Collections.Generic;
using AlicizaX.UI;

// Type 就是分组键：值相同的数据归到同一组。TemplateId 只区分组头/子项模板。
public sealed class TaskData : IGroupViewData
{
    public int TemplateId { get; set; }
    public bool Expanded { get; set; }
    public int Type { get; set; }
    public string Text;
}

// 0 是组头模板下标；子项用 1。列表一个 RecyclerView 只创建一次。
UGGroupList<TaskData> groupList = UGListCreateHelper.CreateGroup<TaskData>(recyclerView, 0);

groupList.SetData(new List<TaskData>
{
    new() { TemplateId = 1, Type = 0, Text = "寻找刻晴" },
    new() { TemplateId = 1, Type = 0, Text = "拜访师父" },
    new() { TemplateId = 1, Type = 1, Text = "为甘雨采摘10朵清心" },
});

// 参数是显示索引：第 0 行是第一个组头。不调用就全是折叠态。
groupList.Expand(0);
```

## 核心用法

### 数据结构：源列表平铺，组由 `Type` 派生

没有「组对象列表 + 子项列表」这种嵌套结构。你只维护一份子项列表，适配器按 `Type` 把它投影成显示列表：

```text
源列表（你维护，长度=子项数）        显示列表（适配器派生，长度会变）
[0] Type=0 子项                    [0] 组头 Type=0  TemplateId=groupTemplateId
[1] Type=0 子项                    [1] 子项 0        TemplateId=1
[2] Type=1 子项                    [2] 子项 1        TemplateId=1
                                   [3] 组头 Type=1  TemplateId=groupTemplateId
                                   [4] 子项 2        TemplateId=1
```

由此带来的三条硬事实：

- 组对象是适配器 `new TData { TemplateId = groupTemplateId, Type = 组键 }` 造出来的，**不在你的 `List<TData>` 里**，业务字段（如 `Text`）是默认值。组头文案要靠 `Type` 自己格式化，真实写法见 `RvGroupItemViewHolder`。
- 组的先后顺序 = 该 `Type` 在源列表中第一次出现的顺序。
- `GetItemCount()` 与 `GetRealCount()` 都等于显示行数：滚动、选中、导航全部用**显示索引**。

### 两套 ViewHolder

| 模板 | 继承 | 职责 |
| :--- | :--- | :--- |
| 组头（`groupTemplateId`） | `ViewHolder<TData>`，需要点击就加 `IPointerClickHandler` | 画标题与箭头，按 `data.Type` 取文案，按 `data.Expanded` 画状态 |
| 子项（其他 `TemplateId`） | `ViewHolder<TData>` | 正常绑定业务字段 |

组头点一下切展开/折叠，真实窗口就是这么写的：

```csharp
public sealed class RvGroupHeaderHolder : ViewHolder<TaskData>, IPointerClickHandler
{
    [SerializeField] private TextMeshProUGUI title;

    protected override void OnBind(TaskData data, int index)
    {
        // 组头没有业务文本，按 Type 自己格式化
        title.text = data.Type switch { 0 => "每日", 1 => "世界", _ => "分类 " + data.Type };
    }

    public void OnPointerClick(PointerEventData eventData)
    {
        // CurrentIndex 是显示索引，Activate 同样接受显示索引
        TaskWindow.Current?.ToggleGroup(CurrentIndex);
    }
}
```

### 展开与折叠（真实 API）

| API | 参数 | 行为 |
| :--- | :--- | :--- |
| `Activate(int index)` | 显示索引 | 组头 → 切换展开；子项 → `ChoiceIndex = index` |
| `Expand(int index)` / `Collapse(int index)` | 组头显示索引 | 展开/折叠；非组头索引无效 |
| `SetExpanded(int index, bool expanded)` | 组头显示索引 | 返回是否命中组头；状态没变时返回 `true` 但不重绑 |
| `IsGroupIndex(int index)` | 显示索引 | 该行是不是组头 |
| `TryGetDisplayData(int index, out TData data)` | 显示索引 | 取该行绑定的对象（组头返回适配器造的那个组对象） |

展开会往显示列表插入子项行，折叠会移除；两者都通过结构通知增量处理，滚动位置不会跳。展开状态下 `Add` 同组子项会直接出现在组尾；折叠状态下只增加组内计数，展开时才出现。

### 分组导航

组头也是一行，所以导航按显示行移动，没有单独的「组导航」通道：

- 组头模板要参与导航，就继承 `InteractiveViewHolder<TData>`（或 `NavigationViewHolder<TData>`），否则该行不可聚焦。
- 提交走 `OnItemClick` → `Activate(显示索引)`，即组头切展开、子项设选中。组头一般希望 `SelectOnActivate => false`，别把组头写进业务选中。
- 展开/折叠会改变后面所有行的显示索引；控制器只按数字索引把焦点修到最近可聚焦行，不保证还是同一业务项。折叠动作后建议对新组头索引调用 `RestoreFocus`。
- 想整行禁用（例如空组），用 `RecyclerViewNavigationController.SetFocusFilter` 跳过对应的显示索引。

```csharp
public sealed class RvGroupHeaderNavHolder : InteractiveViewHolder<TaskData>
{
    protected override bool SelectOnActivate => false;      // 组头只切展开，不做业务选中
    protected override void OnBind(TaskData data, int index) { /* 画标题与箭头 */ }
    protected override void OnItemClick(TaskData data, int index) => TaskWindow.Current?.ToggleGroup(index);
}
```

完整的挂载、边界与遮挡规则见 [列表导航](navigation.md)。

## 常见配置

| 项目 | 建议值 | 说明 |
| :--- | :--- | :--- |
| `Layout Type` | `MixedLayoutManager` | 按模板取高度，组头/子项可不同高；`Variable Item Length` 可关 |
| `Item Templates` | `#0` 组头、`#1` 子项 | 子项不得使用 `groupTemplateId`，否则会被当成组头 |
| `Spacing` | `(0, 10)` | 真实分组示例值；间距只影响内容尺寸 |
| `Padding` | `(16, 12)` | 上下 padding 参与内容高度 |
| `Refresh Mode` | `Immediate` | 分组一般不在滚动中改数据；聊天式流才用 `AfterSettle` |
| `Scrollbar Visibility` | `AlwaysHide` | 示例窗口未接滚动条 |
| 构造参数 | `groupTemplateId = 0` | 必须小于 `Templates` 长度 |

## 容易踩的坑

1. **别指望数据里的 `Expanded` 预置展开态。** 组对象是适配器新建的，`Expanded` 默认 `false`；想默认展开必须在 `SetData` 之后 `Expand`。真实窗口 `ShowGroup()` 每次都 `SetData` 再 `Expand(0)`。
2. **`SetData` 会清空所有组对象**，展开态随之丢失；它还会清空视图池并重建显示列表，别在每次小改动时调。
3. **`Move` 不能跨组**：`GroupAdapter.Move` 在起止项 `Type` 不同时抛 `InvalidOperationException`，显示顺序由分组规则派生，只能组内重排。
4. **源索引 ≠ 显示索引**：`Add`/`Insert`/`RemoveAt`/`RemoveRange`/`ReplaceAt`/`Move`/`Sort` 作用于源列表；`Activate`/`Expand`/`Collapse`/`SetExpanded`/`IsGroupIndex`/`TryGetDisplayData`/`RefreshItem`/`ChoiceIndex` 用显示索引。
5. **`groupTemplateId` 与子项 `TemplateId` 都必须落在 `Templates` 范围内**，否则 `ViewProvider` 报 `adapter returned an invalid template id` 并少一行。
6. **`LinearLayoutManager` 定高会把所有行做成同样高度**（取显示列表第 0 行——也就是第一个组头——的模板尺寸），子项会被撑成组头高度；要么换 `MixedLayoutManager`，要么打开 `Variable Item Length` 并让数据实现 `IMeasuredViewData`。
7. **`SetExpanded` 的返回 `false` 不是异常**：索引越界、不是组头都会返回 `false`；`Expand`/`Collapse` 没有返回值，需要判断就用 `IsGroupIndex`。
8. **源列表不要放 `null`**：`GroupAdapter` 遇到 `null` 会 `Debug.Assert` 并跳过，显示行会少一条。
9. **`ReplaceAt` 改 `Type` 会整表刷新**（组顺序可能变），只换文本则只重绑该行。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `UGGroupList<TData>(RecyclerView, int groupTemplateId)` | 构造并绑定，组头模板下标必填 |
| `UGListCreateHelper.CreateGroup<TData>(RecyclerView, int)` | 等价工厂方法 |
| `SetData(List<TData>, ListScrollPolicy)` | 整表替换并重建分组；默认 `KeepAnchor` |
| `Expand` / `Collapse` / `SetExpanded` / `Activate` | 展开、折叠、设置、按行类型激活 |
| `IsGroupIndex` / `TryGetDisplayData` | 行类型判断与显示数据 |
| `RefreshItem(int index, bool relayout)` | 按显示索引刷新单行；改高传 `true` |
| `ChoiceIndex` / `ClearChoice()` / `OnChoiceIndexChanged` | 业务选中（显示索引） |
| `DataCount` / `TryGetVisibleRange` / `ScrollTo(...)` | 继承自 `UGListBase`，见 [配置与 API](config.md) |
