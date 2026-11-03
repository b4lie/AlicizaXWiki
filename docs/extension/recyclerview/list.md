# 普通列表（UGList）

适用场景：背包、邮件列表、排行榜、任务子项列表等单一模板列表。

## Prefab 配置

```text
ScrollView
├── Content
└── Templates
    └── BagItemTemplate  // Templates[0]，挂 BagItemHolder
```

推荐配置：

| 项 | 值 |
| --- | --- |
| `Templates` | 只放一个模板 |
| `LayoutManager` | `LinearLayoutManager` 或 `GridLayoutManager` |
| `Direction` | 竖向列表用 `Vertical`，横向列表用 `Horizontal` |

## 数据

```csharp
using AlicizaX.UI;

public sealed class BagItemData : ISimpleViewData
{
    public int ItemId;
    public string Name;
    public int Count;
    public bool Locked;
}
```

## Holder

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public sealed class BagItemHolder : ViewHolder<BagItemData>
{
    [SerializeField] private Image icon;
    [SerializeField] private TextMeshProUGUI nameText;
    [SerializeField] private TextMeshProUGUI countText;
    [SerializeField] private GameObject selectedFrame;
    [SerializeField] private GameObject lockedMask;

    protected override void OnBind(BagItemData data, int index)
    {
        nameText.text = data.Name;
        countText.text = data.Count.ToString();
        lockedMask.SetActive(data.Locked);
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
        icon.color = select ? Color.cyan : Color.white;
    }

    protected override void OnClear()
    {
        selectedFrame.SetActive(false);
        lockedMask.SetActive(false);
        icon.color = Color.white;
    }

    public void OnClick()
    {
        if (!CurrentData.Locked)
        {
            SetSelect();
        }
    }
}
```

`OnClick` 可以绑定到模板上的 `Button.onClick`。如果没有按钮，也可以由其它点击组件调用这个公开方法。

## 窗口使用

```csharp
using System.Collections.Generic;
using AlicizaX;
using AlicizaX.UI;
using AlicizaX.UI.Runtime;
using Game.UI;
using UnityEngine;

[Window(UILayer.UI)]
public sealed class BagWindow : UIWindow<ui_BagWindow>
{
    private UGList<BagItemData> _items;

    protected override void OnInitialize()
    {
        _items = UGListCreateHelper.Create<BagItemData>(baseui.ScrollViewItemList);
        _items.OnChoiceIndexChanged += OnChoiceChanged;
        _items.ScrollStopped += OnScrollStopped;

        _items.SetData(CreateItems());
        _items.ChoiceIndex = 0;
        _items.ScrollToChoice(ScrollAlignment.Start);
    }

    protected override void OnClose()
    {
        if (_items == null)
        {
            return;
        }

        _items.OnChoiceIndexChanged -= OnChoiceChanged;
        _items.ScrollStopped -= OnScrollStopped;
    }

    private void OnChoiceChanged(int index)
    {
        BagItemData data = _items.Adapter.GetData(index);
        if (data == null)
        {
            return;
        }

        Log.Info($"Select item: {data.ItemId}");
    }

    private void OnScrollStopped()
    {
        Log.Info($"Scroll stopped at {_items.ScrollPosition}");
    }

    private static List<BagItemData> CreateItems()
    {
        return new List<BagItemData>
        {
            new BagItemData { ItemId = 1001, Name = "Potion", Count = 5 },
            new BagItemData { ItemId = 1002, Name = "Key", Count = 1 },
            new BagItemData { ItemId = 1003, Name = "Gem", Count = 3, Locked = true },
        };
    }
}
```

## 常见操作

```csharp
// 替换整份数据。
_items.Data = CreateItems();

// 添加单项。
_items.Adapter.Add(new BagItemData
{
    ItemId = 1004,
    Name = "Ticket",
    Count = 2,
});

// 修改已有数据，尺寸不变时只重绑可见项。
BagItemData item = _items.Adapter.GetData(1);
if (item != null)
{
    item.Count += 1;
    _items.Adapter.NotifyItemChanged(1);
}

// 如果修改会影响尺寸或布局，使用 relayout。
_items.Adapter.NotifyItemChanged(1, relayout: true);

// 滚动。
_items.ScrollToStart(0);
_items.ScrollToCenter(20, smooth: true);
_items.ScrollTo(20, ScrollAlignment.End, offset: 0f, smooth: true, duration: 0.3f);
```

## 变高与聊天列表

聊天对话框用定高 `LinearLayoutManager` 不够。需要同时满足：

1. Inspector 打开 `LinearLayoutManager.Variable Item Length`。
2. 数据实现 `IMeasuredViewData.TryGetItemLength`，给出预估高度。
3. Holder 绑定文本后改根节点高度，再调用 `RequestResize()`。
4. 刷新模式用 `ListRefreshMode.AfterSettle`，避免拖拽中反复改高闪一下。

```csharp
public sealed class ChatMessageData : IMeasuredViewData
{
    public string Text;
    public float DeclaredLength;

    public bool TryGetItemLength(out float length)
    {
        length = DeclaredLength;
        return DeclaredLength > 0f;
    }
}

_list = UGListCreateHelper.Create<ChatMessageData>(baseui.ScrollViewChat);
_list.ApplyUpdateOptions(new ListUpdateOptions
{
    ApplyRefreshMode = true,
    RefreshMode = ListRefreshMode.AfterSettle
});
_list.SetData(history, ListScrollPolicy.KeepAnchor);
_list.StickToEnd();

// 底部发消息：仅贴底时 StickToEnd。
if (_list.IsAtEnd)
{
    _list.Add(message);
    _list.StickToEnd();
}
else
{
    _list.Add(message);
}

// 顶部插入历史：KeepAnchor，当前可见消息屏幕位置不变。
_list.InsertRange(0, older);
```

`Content`、`Templates`、`Adapter`、`LayoutManager` 运行期不可更换。`SetAdapter` 只允许首次绑定。模板根节点不要在滚动轴上 Stretch。
