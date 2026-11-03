# Widget 与 Tab

```csharp
protected T CreateWidgetSync<T>(Transform parent, bool visible = true) where T : UIWidget;
protected T CreateWidgetSync<T>(UIHolderObjectBase holder, bool destroyHolderOnDispose = true, bool visible = true) where T : UIWidget;
protected UniTask<T> CreateWidgetAsync<T>(Transform parent, bool visible = true, CancellationToken cancellationToken = default);
protected UniTask<UIWidget> CreateWidgetAsync(string typeName, Transform parent, bool visible = true, CancellationToken cancellationToken = default);
public UniTask RemoveWidget(UIWidget widget);

widget.Open(params object[] userDatas);
widget.Close();
await widget.Destroy();
await widget.AwaitTransition();
```

语义：

- `CreateWidget*`：资源 + Init +（`visible` 时）逻辑 `OnOpen`；开场转场后台推进。
- Widget 必须属于父窗口 Holder 的 Transform 子树，否则创建失败并报 Error。
- 每个 Widget 自己持有加载取消源：父窗口销毁会取消所有子创建；Widget 自身销毁也会立即取消自己的加载。
- `Open`：已 `Opened` 只刷新参数；`Opening` 返回成功；`Closing` 返回 false（需要重开请在关闭完成后再调）。
- `Close`：`Initialized`（从未打开）时是 no-op；其余状态走完整关闭流程。
- 可见性：`IsVisible = Visible && Parent.IsVisible`；父级不可见时子 Widget 逻辑状态不变，只暂停事件/更新。

## Tab 窗口

```csharp
public abstract class UITabWindow<T> : UIWindow<T> where T : UIHolderObjectBase
{
    protected void InitTabVirtuallyView<TTab>(Transform parent = null) where TTab : UIWidget;
    public UniTask<UIWidget> SwitchTab(int index, params object[] userDatas);
}
```

- Tab 首次选中时异步创建，之后复用实例。
- 切换是 latest wins：旧请求返回 `null`，旧 Tab 关闭完成后才打开新 Tab。
- Tab 加载中窗口被关闭/销毁时请求会以 `null` 结束，迟到资源被销毁。

## UI 事件自动解绑

```csharp
protected override void OnRegisterEvent(EventListenerProxy proxy)
{
    proxy.AddUIEvent<LocalizationChangeEvent>(OnLanguageChanged);
}

private void OnLanguageChanged(in LocalizationChangeEvent evt) => RefreshText();
```

- `OnRegisterEvent` 在每个视图的生命周期内只调用一次，发生在首次 `Effective` 时。之后不可见或关闭只暂停订阅，重新 `Effective` 时恢复，不再调用它。依赖每次打开的逻辑写在 `OnOpen` 或 `OnRefresh`。
- 销毁时释放订阅并把代理归还 `MemoryPool`。
- 在 `OnRegisterEvent` 内销毁自身是允许的：事件代理会延迟到回调返回后释放，不会泄漏句柄。
