# 循环列表（UGLoopList）

适用场景：轮播图、角色预览、循环选择器、无限横向滚动。

`UGLoopList<TData>` 使用 `LoopAdapter<TData>`，当真实数据数量大于 0 时，显示数量返回 `int.MaxValue`。绑定时会用 `index % list.Count` 映射回真实数据。

## Prefab 配置

```text
ScrollView
├── Content
└── Templates
    └── BannerTemplate  // Templates[0]，挂 BannerHolder
```

推荐配置：

| 项 | 值 |
| --- | --- |
| `LayoutManager` | `LinearLayoutManager`、`PageLayoutManager` 或 `CircleLayoutManager` |
| `Direction` | 轮播通常用 `Horizontal` |
| `Snap` | 通常开启 |
| `Inertia` | 按手感决定 |

## 数据

```csharp
using AlicizaX.UI;

public sealed class BannerData : ISimpleViewData
{
    public int BannerId;
    public string Title;
    public string ImagePath;
}
```

## Holder

```csharp
using AlicizaX.UI;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public sealed class BannerHolder : ViewHolder<BannerData>
{
    [SerializeField] private Image image;
    [SerializeField] private TextMeshProUGUI titleText;
    [SerializeField] private GameObject selectedFrame;

    protected override void OnBind(BannerData data, int index)
    {
        titleText.text = data.Title;

        uint version = CurrentBindingVersion;
        LoadImageAsync(data.ImagePath, sprite =>
        {
            if (IsBindingCurrent(version))
            {
                image.sprite = sprite;
            }
        });
    }

    protected override void OnSelectionChange(bool select)
    {
        selectedFrame.SetActive(select);
    }

    protected override void OnClear()
    {
        image.sprite = null;
        selectedFrame.SetActive(false);
    }

    public void OnClick()
    {
        SetSelect();
    }

    private void LoadImageAsync(string path, System.Action<Sprite> onLoaded)
    {
        // 接入项目自己的异步图片加载逻辑。
        onLoaded?.Invoke(null);
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

[UIUpdate]
[Window(UILayer.UI)]
public sealed class BannerWindow : UIWindow<ui_BannerWindow>
{
    private UGLoopList<BannerData> _banners;
    private float _autoScrollTimer;

    protected override void OnInitialize()
    {
        _banners = UGListCreateHelper.CreateLoop<BannerData>(baseui.ScrollViewBannerList);
        _banners.OnChoiceIndexChanged += OnBannerSelected;
        _banners.ScrollStopped += OnScrollStopped;
        _banners.Data = CreateBanners();

        if (_banners.DataCount > 0)
        {
            _banners.ChoiceIndex = 0;
            _banners.ScrollToIndex(0);
        }
    }

    protected override void OnClose()
    {
        if (_banners == null)
        {
            return;
        }

        _banners.OnChoiceIndexChanged -= OnBannerSelected;
        _banners.ScrollStopped -= OnScrollStopped;
    }

    protected override void OnUpdate()
    {
        if (_banners == null || _banners.DataCount <= 0)
        {
            return;
        }

        _autoScrollTimer += Time.deltaTime;
        if (_autoScrollTimer < 3f)
        {
            return;
        }

        _autoScrollTimer = 0f;
        int next = (_banners.ChoiceIndex + 1) % _banners.DataCount;
        _banners.ChoiceIndex = next;
        _banners.ScrollToCenter(next, smooth: true, duration: 0.25f);
    }

    private void OnBannerSelected(int index)
    {
        BannerData banner = _banners.Adapter.GetData(index);
        if (banner != null)
        {
            Log.Info($"Select banner: {banner.BannerId}");
        }
    }

    private void OnScrollStopped()
    {
        // 开启 Snap 时，RecyclerView 会在停止后吸附到最近项。
        // 如果需要把当前停靠项同步成业务选择，可在这里按业务规则计算并设置 ChoiceIndex。
    }

    private static List<BannerData> CreateBanners()
    {
        return new List<BannerData>
        {
            new BannerData { BannerId = 1, Title = "Event A", ImagePath = "event_a" },
            new BannerData { BannerId = 2, Title = "Event B", ImagePath = "event_b" },
            new BannerData { BannerId = 3, Title = "Event C", ImagePath = "event_c" },
        };
    }
}
```

## 常见操作

```csharp
// 循环列表的滚动索引传真实数据索引即可。
_banners.ScrollToCenter(2, smooth: true);

// 替换轮播数据后重新定位。
_banners.Data = CreateBanners();
_banners.ChoiceIndex = 0;
_banners.ScrollToIndex(0);
```

注意：循环列表真实数据数量来自 `DataCount` 和 `Adapter.GetRealCount()`。业务逻辑不要遍历 `Adapter.GetItemCount()`，因为循环列表显示数量可能是 `int.MaxValue`。
