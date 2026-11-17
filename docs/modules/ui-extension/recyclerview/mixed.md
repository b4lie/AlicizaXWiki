# 多模板列表

多模板列表由 `UGMixedList<T>` + `MixedAdapter<T>` + `MixedLayoutManager` 组成：同一条列表里，每一项通过数据的 `TemplateId` 选择 `Item Templates` 中的一套模板。它适合“短卡片、长卡片、广告位混排”这类行高不一致的列表。

源码位置：`Client/Packages/com.alicizax.unity.ui.extension/Runtime/RecyclerView/Adapter/MixedAdapter.cs`、`Layout/MixedLayoutManager.cs`、`UGList.cs`。

## 适用场景

- 信息流里混排短文本、图文卡片、整宽 Banner 或分割块。
- 聊天列表里系统消息、普通消息、操作提示使用不同结构与高度。
- 同一份数据需要按类型切换布局，但不想拆成多个 `RecyclerView`。
- 不需要按类型分组、折叠；那是 [分组列表](group.md) 的职责。

## 使用前提

- 必须使用 `AlicizaX.UI.MixedLayoutManager`。`UGMixedList` 内部是 `MixedAdapter<T>`，换成普通 `LinearLayoutManager` 会在创建时抛 `InvalidOperationException`。
- 数据实现 `IMixedViewData`，并给每一项填写 `TemplateId`；它是 `Item Templates` 的数组下标，不是模板名。
- `Item Templates` 至少包含 `TemplateId` 会用到的最大的下标 + 1 项，且每一项都挂 `ViewHolder<TData>` 子类。
- 模板顺序在编辑期固定；运行时不能换模板数组，也不能换适配器。
- 其他前提与 [普通列表](list.md) 相同：`Content`、`Scroller`、`Layout Type` 必须先配好。

## 最短接入

1. 做三个模板 Prefab（例如短、中、高），根节点分别挂 `ViewHolder<RvMixedData>` 子类，设置不同的固定主轴高度。
2. 把三个模板按顺序拖进 `Item Templates`：`#0` 短、`#1` 中、`#2` 高。
3. `Layout Type` 选择 `AlicizaX.UI.MixedLayoutManager`。需要按数据声明/实测高度时，再打开该布局的 `Variable Item Length`；示例 Prefab `ScrollView@Mixed` 打开了该开关。
4. 数据实现 `IMixedViewData`，按行填写 `TemplateId`。
5. 窗口里创建 `UGMixedList` 并 `SetData`：

```csharp
using System.Collections.Generic;
using AlicizaX.UI;

// UIWindow<...> 内
private UGMixedList<RvMixedData> _mixed;

protected override void OnOpen()
{
    _mixed ??= UGListCreateHelper.CreateMixed<RvMixedData>(baseui.ScrollViewMixed);
    _mixed.SetData(CreateMixedItems(30));
}

private List<RvMixedData> CreateMixedItems(int count)
{
    var items = new List<RvMixedData>(count);
    for (int i = 0; i < count; i++)
    {
        int templateId = i % 3;
        items.Add(new RvMixedData { TemplateId = templateId, Text = "Item " + i });
    }

    return items;
}
```

## 核心用法

### 数据与 TemplateId

```csharp
public sealed class RvMixedData : IMixedViewData
{
    public int TemplateId { get; set; }
    public string Text;
}
```

`MixedAdapter.GetTemplateId(index)` 直接返回 `list[index].TemplateId`。因此：

- `0` 对应 `Item Templates[0]`；
- `1` 对应 `Item Templates[1]`；
- 超出数组范围或对应模板为空时，该行无法创建 Holder，控制台会报 `adapter returned an invalid template id`，列表表现为空行或高度异常。

### 多套 ViewHolder

有两种等价写法：

- **每个模板一个 Holder 类**：为不同高度的卡片各写一个 `ViewHolder<RvMixedData>` 子类（类名由项目自定），`Item Templates` 对应位置拖不同 Prefab。
- **一套 Holder 处理多个模板**：仓库示例 `RvMixedItemViewHolder` 在多个模板间复用，`OnBind` 里按 `data.TemplateId` 切换颜色或内容。

```csharp
protected override void OnBind(RvMixedData data, int index)
{
    title.text = data.Text;
    background.color = data.TemplateId switch
    {
        1 => new Color(0.18f, 0.32f, 0.22f, 0.95f),
        2 => new Color(0.28f, 0.22f, 0.12f, 0.95f),
        _ => new Color(0.16f, 0.2f, 0.34f, 0.95f)
    };
}
```

不需要注册工厂：模板 Prefab 本身就是 Holder 的来源，`ViewProvider` / `ViewHolderFactory` 在内部按 `TemplateId` 取用。

### 模板高度与测量

- **`Variable Item Length` 关闭**：每一行的主轴长度取它所选模板的 `RectTransform` 尺寸。模板 `#0/#1/#2` 分别是多高，列表就按多高排。`MixedLayoutManager` 仍会为每行建偏移表，所以三种模板可以有三种固定高度。
- **`Variable Item Length` 打开**：数据可实现 `IMeasuredViewData.TryGetItemLength` 给出声明长度；未声明时先回退到模板高度。Holder 绑定后，框架会实测并回写长度。
- 在 `OnBind` 里改了 Holder 根节点高度时，改完调用 `RequestResize()`；已绑定数据从外部改高时用 `RefreshItem(index, true)`。
- 模板高度为 0 会让该类型的行没有主轴占位；先检查模板 RectTransform 的 `sizeDelta` 和锚点。

### 真实示例

示例窗口 `Client/Assets/Scripts/Hotfix/GameLogic/UI/RecyclerViewTest/UIRecyclerViewExampleWindow.cs` 的 `ShowMixed` 路径：

- `UGListCreateHelper.CreateMixed<RvMixedData>(baseui.ScrollViewMixed)` 创建列表；
- 30 条数据按 `i % 3` 填 `TemplateId`；
- 按钮 A 追加 `TemplateId = 0` 的短项，按钮 B 追加 `TemplateId = 2` 的高项，按钮 C 对第 1 条执行 `RefreshItem(1, true)`；
- 对应 Prefab 在 `Client/Assets/Bundles/UI/Window/UIRecyclerViewExampleWindow.prefab`，`ScrollView@Mixed` 配了 3 个模板和 `MixedLayoutManager`。

## 常见配置

| Inspector 字段 | 源码字段 | 默认值 | 说明 |
| --- | --- | --- | --- |
| Layout Type | `layoutManager` | None | 必须选 `MixedLayoutManager` |
| Item Templates | `templates` | 空 | 顺序对应 `TemplateId`，示例为 3 项 |
| Variable Item Length | `variableItemLength` | false | 按声明/实测高度排版时打开 |
| Direction | `direction` | Vertical | 主轴方向 |
| Content | `content` | 空 | 列表项父节点，必须显式指定 |
| Spacing | `spacing` | (0, 0) | 行间距；垂直列表取 `y` |
| Padding | `padding` | (0, 0) | 内容留白 |
| Refresh Mode | `refreshMode` | Immediate | 行高频繁变化时可考虑 `AfterSettle` |
| Scroller Type | `scroller` | None | 同节点上的 Scroller 组件 |

## 容易踩的坑

- 用了 `UGList<T>`：普通 `Adapter<T>` 的 `GetTemplateId` 恒为 0，多模板数据只会反复用模板 `#0`。混排必须用 `UGMixedList<T>`。
- `TemplateId` 越界、为负或对应模板为空时不会自动回退到 `#0`，该行直接创建失败；在数据构造处就把 ID 限死在合法范围。
- `MixedAdapter` 配 `LinearLayoutManager` 会被 `UGListBase` 直接拒绝，这不是运行期警告，而是构造时异常。
- 模板高度改了但没打开 `Variable Item Length`，或者没 `RefreshItem(index, true)`，行位置和总高度会保持旧值。
- `Templates` 是编辑期配置；运行时换 Prefab、换数组顺序都不会被支持。
- 一个模板可以共用同一套 Holder，但 Holder 的 `TData` 必须与列表的 `T` 一致，否则绑定失败并清空该行。
- 模板在主轴方向拉伸锚点会被启动校验拒绝；不同模板要显式设置不同高度。

## API 速查

| 成员 | 用途 |
| --- | --- |
| `IMixedViewData.TemplateId` | 指定该行使用的模板下标 |
| `UGMixedList<T>` | 绑定 `MixedAdapter<T>` 的业务封装 |
| `UGListCreateHelper.CreateMixed<T>` | 创建多模板列表 |
| `SetData(list, policy)` / `Data` | 替换数据并刷新 |
| `RefreshItem(index, true)` | 重绑并重排某一行 |
| `Adapter.NotifyDataChanged()` | 全量失效并重绑 |
| `ViewHolder.RequestResize()` | 绑定后 Holder 尺寸变化时回写 |
| `ScrollToIndex` / `ScrollToCenter` | 定位到某条数据 |
| `TryGetVisibleRange` / `TryGetViewHolder` | 查询可见项 |

选择、点击和局部刷新的统一入口见 [数据更新与交互](interaction.md)；全部字段见 [配置与 API](config.md)。
