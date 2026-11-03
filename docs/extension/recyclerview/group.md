# UGGroupList 分组列表

适用场景：任务分组、背包分类、设置页分段、成就分类等“组头 + 子项”的列表。

`UGGroupList<TData>` 要求数据实现 `IGroupViewData`，并且 `TData` 必须有无参构造函数。创建时需要传入组头模板下标 `groupTemplateId`。

## Prefab 配置

```text
ScrollView
├── Content
└── Templates
    ├── QuestGroupTemplate  // Templates[0]，挂 QuestGroupHolder
    └── QuestItemTemplate   // Templates[1]，挂 QuestItemHolder
```

推荐使用 `MixedLayoutManager`，因为组头和子项通常高度不同。

## 数据

原始 `Data` 只放真实子项。`GroupAdapter` 会扫描子项的 `Type`，为每个类型创建一个临时组头数据：

```csharp
new TData
{
    TemplateId = groupTemplateId,
    Type = type
};
```

```csharp
using AlicizaX.UI;

public enum QuestGroupType
{
    Main = 1,
    Daily = 2,
    Achievement = 3,
}

public sealed class QuestRowData : IGroupViewData
{
    public int QuestId;
    public string Title;
    public int Current;
    public int Target;
    public bool Completed;

    public int TemplateId { get; set; }
    public bool Expanded { get; set; }
    public int Type { get; set; }

    public QuestGroupType Group => (QuestGroupType)Type;

    public static QuestRowData Item(
        QuestGroupType group,
        int questId,
        string title,
        int current,
        int target,
        bool completed = false)
    {
        return new QuestRowData
        {
            QuestId = questId,
            Title = title,
            Current = current,
            Target = target,
            Completed = completed,
            Type = (int)group,
            TemplateId = 1,
        };
    }
}
```

## Holder

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public sealed class QuestGroupHolder : ViewHolder<QuestRowData>
{
    [SerializeField] private TextMeshProUGUI titleText;
    [SerializeField] private Image arrow;
    [SerializeField] private GameObject selectedFrame;

    protected override void OnBind(QuestRowData data, int index)
    {
        titleText.text = GetGroupTitle(data.Group);
        arrow.rectTransform.localEulerAngles = data.Expanded
            ? new Vector3(0f, 0f, 90f)
            : Vector3.zero;
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
    }

    protected override void OnClear()
    {
        arrow.rectTransform.localEulerAngles = Vector3.zero;
        selectedFrame.SetActive(false);
    }

    public void OnClick()
    {
        SetSelect();
    }

    private static string GetGroupTitle(QuestGroupType group)
    {
        return group switch
        {
            QuestGroupType.Main => "主线任务",
            QuestGroupType.Daily => "日常任务",
            QuestGroupType.Achievement => "成就任务",
            _ => "其他任务",
        };
    }
}
```

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public sealed class QuestItemHolder : ViewHolder<QuestRowData>
{
    [SerializeField] private TextMeshProUGUI titleText;
    [SerializeField] private TextMeshProUGUI progressText;
    [SerializeField] private Button rewardButton;
    [SerializeField] private GameObject selectedFrame;

    protected override void OnBind(QuestRowData data, int index)
    {
        titleText.text = data.Title;
        progressText.text = $"{data.Current}/{data.Target}";
        rewardButton.interactable = data.Completed;
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
    }

    protected override void OnClear()
    {
        selectedFrame.SetActive(false);
        rewardButton.interactable = false;
    }

    public void OnClick()
    {
        SetSelect();
    }
}
```

## 窗口使用

```csharp
using System.Collections.Generic;
using AlicizaX;
using AlicizaX.UI;
using AlicizaX.UI.Runtime;
using Game.UI;
using UnityEngine;

[Window(UILayer.UI)]
public sealed class QuestWindow : UIWindow<ui_QuestWindow>
{
    private const int GroupTemplateId = 0;

    private UGGroupList<QuestRowData> _quests;

    protected override void OnInitialize()
    {
        _quests = UGListCreateHelper.CreateGroup<QuestRowData>(
            baseui.ScrollViewQuestList,
            groupTemplateId: GroupTemplateId);

        _quests.OnChoiceIndexChanged += OnQuestChoiceChanged;
        _quests.Data = CreateQuestRows();

        ExpandFirstGroup();
    }

    protected override void OnClose()
    {
        if (_quests != null)
        {
            _quests.OnChoiceIndexChanged -= OnQuestChoiceChanged;
        }
    }

    private void OnQuestChoiceChanged(int displayIndex)
    {
        if (!_quests.TryGetDisplayData(displayIndex, out QuestRowData row))
        {
            return;
        }

        if (_quests.IsGroupIndex(displayIndex))
        {
            _quests.Activate(displayIndex);
            return;
        }

        Log.Info($"Open quest detail: {row.QuestId}");
    }

    private void ExpandFirstGroup()
    {
        if (_quests.TryGetDisplayData(0, out _) && _quests.IsGroupIndex(0))
        {
            _quests.Expand(0);
        }
    }

    public void AddDailyQuest()
    {
        _quests.Data.Add(QuestRowData.Item(
            QuestGroupType.Daily,
            questId: 2003,
            title: "完成一次钓鱼",
            current: 0,
            target: 1));

        _quests.Adapter.NotifyDataChanged();
    }

    private static List<QuestRowData> CreateQuestRows()
    {
        return new List<QuestRowData>
        {
            QuestRowData.Item(QuestGroupType.Main, 1001, "前往王城", 1, 1, completed: true),
            QuestRowData.Item(QuestGroupType.Main, 1002, "拜访骑士团长", 0, 1),
            QuestRowData.Item(QuestGroupType.Daily, 2001, "完成三次副本", 1, 3),
            QuestRowData.Item(QuestGroupType.Daily, 2002, "赠送一次礼物", 0, 1),
            QuestRowData.Item(QuestGroupType.Achievement, 3001, "累计登录七天", 4, 7),
        };
    }
}
```

## 常见操作

```csharp
// 展开或收起显示索引上的组头。
_quests.Expand(0);
_quests.Collapse(0);
_quests.SetExpanded(0, true);

// Activate：组头会切换展开，子项会设置 ChoiceIndex。
_quests.Activate(displayIndex);

// 获取当前显示列表数据。显示列表包含组头和已展开子项。
if (_quests.TryGetDisplayData(displayIndex, out QuestRowData row))
{
    bool isGroup = _quests.IsGroupIndex(displayIndex);
}

// 分组数据变更后通常使用全量刷新，因为显示列表需要重建。
_quests.Adapter.NotifyDataChanged();
```

注意：

1. `groupTemplateId` 是组头模板下标，不是显示文本。
2. 子项的 `TemplateId` 不要等于 `groupTemplateId`，否则 `IsGroupIndex` 会把子项识别为组头。
3. `UGGroupList` 的索引是显示索引，不是原始 `Data` 的子项索引。展开和收起会改变显示索引。
4. 仅靠 `SetSelect()` 时，同一个已选中的组头再次点击不会触发 `OnChoiceIndexChanged`。如果业务要求重复点击同一组头也切换，需要让窗口层直接调用 `_quests.Activate(displayIndex)`。
