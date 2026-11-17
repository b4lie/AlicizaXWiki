# UXDraggable

`UXDraggable` 把 UGUI 的拖拽事件转发成三个 `UnityEvent<PointerEventData>`。它**只做事件转发**：不移动节点、不做边界限制、不做惯性，位移逻辑要自己写在回调里。

## 适用场景

- 可拖拽的弹窗标题栏、悬浮球、道具格子。
- 需要拿到 `PointerEventData`（拖拽位置、按键、`delta`）再做自定义处理。
- 给美术/策划在 Inspector 里直接挂回调，不用写代码。

## 使用前提

- 已安装 `com.alicizax.unity.ui.extension`。
- 场景里有 `EventSystem`，Canvas 上有 `GraphicRaycaster`。
- 挂载节点自身或子节点要有可接收射线的 `Graphic`（`Raycast Target` 打开），否则收不到拖拽事件。
- 同一节点不要和 `ScrollRect` 的拖拽区域重叠，否则事件会被 `ScrollRect` 优先消费。

## 最短接入

1. 给要拖拽的节点挂 `UXDraggable`。
2. Inspector 里出现 `On Begin Drag`、`On Drag`、`On End Drag` 三个事件。
3. 在 `On Drag` 里把 `PointerEventData.delta` 应用到目标 `RectTransform`：

```csharp
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;

public sealed class WindowDrag : MonoBehaviour
{
    [SerializeField] private RectTransform _target;
    [SerializeField] private Canvas _canvas;
    [SerializeField] private UXDraggable _draggable;

    private Vector2 _startPosition;

    private void Awake()
    {
        if (_target == null) _target = (RectTransform)transform;
        if (_canvas == null) _canvas = GetComponentInParent<Canvas>();
        _draggable.onBeginDrag.AddListener(OnBeginDrag);
        _draggable.onDrag.AddListener(OnDrag);
        _draggable.onEndDrag.AddListener(OnEndDrag);
    }

    private void OnDestroy()
    {
        _draggable.onBeginDrag.RemoveListener(OnBeginDrag);
        _draggable.onDrag.RemoveListener(OnDrag);
        _draggable.onEndDrag.RemoveListener(OnEndDrag);
    }

    private void OnBeginDrag(PointerEventData eventData) => _startPosition = _target.anchoredPosition;

    private void OnDrag(PointerEventData eventData)
    {
        Vector2 next = _startPosition + eventData.delta / _canvas.scaleFactor;
        _target.anchoredPosition = next;
    }

    private void OnEndDrag(PointerEventData eventData) { }
}
```

`eventData.delta` 是屏幕像素，除以 `Canvas.scaleFactor` 才能得到 UI 本地坐标的位移。

## 三个事件

| 事件 | 触发时机 | 常用参数 |
| :--- | :--- | :--- |
| `onBeginDrag` | 按下并开始拖动 | `position`、`pressEventCamera` |
| `onDrag` | 拖动过程中每帧 | `delta`、`position` |
| `onEndDrag` | 松开或取消拖动 | `position`、`pointerCurrentRaycast` |

事件类型是 `UnityEvent<PointerEventData>`，Inspector 里只能绑定带 `PointerEventData` 参数的方法，或使用无参方法（Unity 会忽略参数）。

## 与 ScrollRect / 其它拖拽组件的冲突

- 父级有 `ScrollRect` 且开启水平/垂直拖动时，子节点上的 `UXDraggable` 仍会收到事件，但 `ScrollRect` 也可能同时滚动；用 `eventData.pointerDrag` 判断，或在 `onBeginDrag` 里临时关闭 `ScrollRect.enabled`。
- 同一节点挂多个 `IDragHandler` 时，Unity 会把事件发给所有实现者，注意不要重复位移。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 限制在父节点内 | 在 `onDrag` 里对 `anchoredPosition` 做 `Mathf.Clamp`，用父 `RectTransform.rect` 计算边界 |
| 只允许水平拖 | 只应用 `delta.x` |
| 拖动时置顶 | 在 `onBeginDrag` 里 `transform.SetAsLastSibling()` |
| 拖动结束吸附 | 在 `onEndDrag` 里对位置取整或补间到最近锚点 |
| 禁用拖拽 | `_draggable.enabled = false` |

## 容易踩的坑

1. **以为挂了就能拖**：组件不移动任何东西，必须自己写位移逻辑。
2. **节点没有 Raycast Target**：收不到任何事件；检查挂载节点或子节点的 `Graphic`。
3. **Canvas 缩放导致位移过快/过慢**：`delta` 是屏幕像素，要除以 Canvas 的 `localScale`（或使用 `eventData.delta / canvas.scaleFactor`）。
4. **在 `ScrollRect` 列表项上直接用**：会与列表滚动抢事件，需要按业务决定谁优先。
5. **忘记移除监听**：`UnityEvent` 持有方法引用，节点销毁前不移除会阻止对象回收（长生命周期组件尤其注意）。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `onBeginDrag` | `UnityEvent<PointerEventData>`，开始拖拽 |
| `onDrag` | `UnityEvent<PointerEventData>`，拖拽中 |
| `onEndDrag` | `UnityEvent<PointerEventData>`，结束拖拽 |
| 实现的接口 | `IBeginDragHandler`、`IDragHandler`、`IEndDragHandler` |

相关：[UXSelectable 基础](uxselectable.md)｜[UI 界面](../ui/index.md)
