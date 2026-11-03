# UGMixedList 混合模板列表

适用场景：一条列表中混合标题、普通项、奖励项、广告位、分割线等不同模板。

## Prefab 配置

```text
ScrollView
├── Content
└── Templates
    ├── MailTextTemplate    // Templates[0]，挂 MailTextHolder
    └── MailRewardTemplate  // Templates[1]，挂 MailRewardHolder
```

推荐使用 `MixedLayoutManager`，它会按每一项的 `TemplateId` 读取对应模板尺寸。

## 数据

```csharp
using AlicizaX.UI;

public enum MailTemplate
{
    Text = 0,
    Reward = 1,
}

public sealed class MailData : IMixedViewData
{
    public int MailId;
    public string Title;
    public string Content;
    public bool HasAttachment;
    public bool Claimed;
    public int TemplateId { get; set; }

    public static MailData Text(int id, string title, string content)
    {
        return new MailData
        {
            MailId = id,
            Title = title,
            Content = content,
            TemplateId = (int)MailTemplate.Text,
        };
    }

    public static MailData Reward(int id, string title, string content, bool claimed)
    {
        return new MailData
        {
            MailId = id,
            Title = title,
            Content = content,
            HasAttachment = true,
            Claimed = claimed,
            TemplateId = (int)MailTemplate.Reward,
        };
    }
}
```

## Holder

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;

public sealed class MailTextHolder : ViewHolder<MailData>
{
    [SerializeField] private TextMeshProUGUI titleText;
    [SerializeField] private TextMeshProUGUI contentText;
    [SerializeField] private GameObject selectedFrame;

    protected override void OnBind(MailData data, int index)
    {
        titleText.text = data.Title;
        contentText.text = data.Content;
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
    }

    protected override void OnClear()
    {
        selectedFrame.SetActive(false);
    }

    public void OnClick()
    {
        SetSelect();
    }
}
```

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public sealed class MailRewardHolder : ViewHolder<MailData>
{
    [SerializeField] private TextMeshProUGUI titleText;
    [SerializeField] private TextMeshProUGUI contentText;
    [SerializeField] private Button claimButton;
    [SerializeField] private GameObject claimedMark;
    [SerializeField] private GameObject selectedFrame;

    protected override void OnBind(MailData data, int index)
    {
        titleText.text = data.Title;
        contentText.text = data.Content;
        claimButton.interactable = !data.Claimed;
        claimedMark.SetActive(data.Claimed);
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
    }

    protected override void OnClear()
    {
        claimButton.interactable = false;
        claimedMark.SetActive(false);
        selectedFrame.SetActive(false);
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
public sealed class MailWindow : UIWindow<ui_MailWindow>
{
    private UGMixedList<MailData> _mails;

    protected override void OnInitialize()
    {
        _mails = UGListCreateHelper.CreateMixed<MailData>(baseui.ScrollViewMailList);
        _mails.OnChoiceIndexChanged += OnMailSelected;
        _mails.Data = CreateMails();
    }

    protected override void OnClose()
    {
        if (_mails != null)
        {
            _mails.OnChoiceIndexChanged -= OnMailSelected;
        }
    }

    private void OnMailSelected(int index)
    {
        MailData mail = _mails.Adapter.GetData(index);
        if (mail == null)
        {
            return;
        }

        Log.Info($"Open mail: {mail.MailId}");
    }

    public void ClaimSelectedReward()
    {
        int index = _mails.ChoiceIndex;
        MailData mail = _mails.Adapter.GetData(index);
        if (mail == null || !mail.HasAttachment || mail.Claimed)
        {
            return;
        }

        mail.Claimed = true;
        _mails.Adapter.NotifyItemChanged(index);
    }

    private static List<MailData> CreateMails()
    {
        return new List<MailData>
        {
            MailData.Text(1, "System Notice", "Server maintenance completed."),
            MailData.Reward(2, "Login Reward", "Claim your daily reward.", claimed: false),
            MailData.Text(3, "Arena Result", "You reached rank 12."),
        };
    }
}
```

## 常见操作

```csharp
// 插入一个奖励邮件。TemplateId 决定使用哪个模板。
_mails.Adapter.Insert(0, MailData.Reward(4, "Compensation", "Thanks for waiting.", claimed: false));

// 修改模板类型时需要重布局，因为可见实例可能要换模板。
MailData mail = _mails.Adapter.GetData(1);
if (mail != null)
{
    mail.TemplateId = (int)MailTemplate.Reward;
    mail.HasAttachment = true;
    _mails.Adapter.NotifyItemChanged(1, relayout: true);
}
```

注意：`TemplateId` 必须是 `Templates` 数组的合法下标。`Templates[0]`、`Templates[1]` 的顺序改了，数据里的枚举值也要同步调整。
