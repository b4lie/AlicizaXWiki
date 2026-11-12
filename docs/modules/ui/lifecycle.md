# 窗口声明与生命周期

窗口逻辑类继承 `UIWindow<Holder>`，用 `[Window]` 声明层级和缓存时间；Holder 用 `[UIRes]` 声明资源位置。

## 声明窗口

```csharp
using AlicizaX.UI.Runtime;

[Window(UILayer.UI, UICacheTime.Forever)]
public sealed class HomeWindow : UIWindow<ui_HomeWindow>
{
    protected override void OnInitialize()
    {
        baseui.BtnClose.onClick.AddListener(() => CloseSelf());
    }
}
```

`WindowAttribute`：

| 参数 | 说明 |
| --- | --- |
| `windowLayer` | `UILayer` 之一 |
| `cacheTime` | `UICacheTime.None`（0）关闭即销毁；`UICacheTime.Forever`（-1）常驻缓存；正数按秒缓存 |

缓存从**关场转场结束**后开始计时。缓存复用时不会再次执行 `OnInitialize`，只会重新执行 `OnOpen`，必要时执行 `OnRefresh`。

## 声明 Holder 资源

Holder 通常由生成器生成，并带 `[UIRes]`：

```csharp
[UIRes(ui_HomeWindow.ResTag, EUIResLoadType.AssetBundle)]
public partial class ui_HomeWindow : UIHolderObjectBase
{
}
```

- `ResLocation`：资源地址。
- `EUIResLoadType`：`Resources` 或 `AssetBundle`。
- `[UIRes]` 写在 Holder 类上，不写在 `HomeWindow` 逻辑类上。

## 生命周期钩子

```csharp
[Window(UILayer.UI, UICacheTime.Forever)]
public sealed class ShopWindow : UIWindow<ui_ShopWindow>
{
    protected override void OnInitialize() { /* 只执行一次 */ }
    protected override void OnOpen() { /* 每次打开 */ }
    protected override void OnRefresh() { /* 已打开时再次 Show */ }
    protected override void OnUpdate(float deltaTime) { /* 每帧，仅重写时启用 */ }
    protected override void OnClose() { /* 逻辑关闭 */ }
    protected override void OnDestroy() { /* 实例销毁 */ }
}
```

| 钩子 | 状态 | 执行时机 |
| --- | --- | --- |
| `OnInitialize` | `Closed` | Holder 绑定后、首次打开前，只执行一次 |
| `OnOpen` | `Opening` | 逻辑打开，开场转场尚未开始 |
| `OnRefresh` | `Opening` / `Opened` | 已打开的窗口再次 `Show`，参数已更新 |
| `OnUpdate` | `Opening` / `Opened` | 重写了 `OnUpdate` 且父级启用时，每帧调用 |
| `OnClose` | `Closing` | 逻辑关闭，`IsOpen` 已为 `false`，关闭转场尚未开始 |
| `OnDestroy` | `Destroyed` | 实例销毁前，只执行一次 |

补充：

- `OnUpdate` 不需要任何标签；只要重写了 `OnUpdate`，该视图就会被加入更新列表。父窗口关闭、隐藏或销毁时，Widget 的更新和事件会暂停。
- 钩子内调用的打开、关闭、销毁会立即改变状态，原流程不会继续。
- 业务钩子抛出的异常会被捕获并记录，生命周期继续推进。

## 读取打开参数

`ShowUI` 只传一个 `userData` 对象。需要多个字段时定义一个参数类：

```csharp
public sealed class ShopOpenArgs
{
    public int ShopId;
    public string From;
}

[Window(UILayer.Popup, UICacheTime.Forever)]
public sealed class ShopWindow : UIWindow<ui_ShopWindow>
{
    protected override void OnOpen()
    {
        if (UserData is ShopOpenArgs args)
        {
            Refresh(args.ShopId);
        }
    }
}
```

打开时传入：

```csharp
await GameApp.UI.ShowUI<ShopWindow>(new ShopOpenArgs { ShopId = 1001, From = "bag" });
```

`UserData` 在视图销毁或进入缓存时清空。已打开时再次 `Show` 传 `null` 会保留原参数。

## 异步与销毁取消

视图销毁时，`DestroyCancellationToken` 会取消：

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;

protected override void OnOpen()
{
    LoadAsync(DestroyCancellationToken).Forget();
}

private async UniTaskVoid LoadAsync(CancellationToken token)
{
    await SomeAsync(token);
}
```

窗口或 Widget 销毁、缓存淘汰时，这个令牌会取消；业务异步操作应把它传给 `UniTask`。

## Holder 事件

Holder 暴露四个事件：

```csharp
baseui.OnInitEvent += OnHolderInit;
baseui.OnShowEvent += OnHolderShow;
baseui.OnCloseEvent += OnHolderClose;
baseui.OnDestroyEvent += OnHolderDestroy;
```

- 事件在对应生命周期钩子之前触发，顺序固定。
- 单个订阅者抛异常只记录日志，不影响其他订阅者。
- Holder 被外部销毁时，会通知所属视图按销毁流程处理。

事件适合外部模块感知窗口显示/隐藏；业务自己的逻辑优先写在 `OnOpen` / `OnClose`。

## 手动注册

正常使用生成器时不需要手动注册。动态类型或生成器覆盖不到的类型可以手动注册：

```csharp
using AlicizaX.UI.Runtime;

UIMetaRegistry.RegisterWindow<MyWindow, ui_MyWindow>(
    () => new MyWindow(),
    UILayer.UI,
    UICacheTime.Forever,
    hasUpdate: true);

UIMetaRegistry.RegisterWidget<MyWidget, ui_MyWidget>(
    () => new MyWidget(),
    hasUpdate: false);

UIResRegistry.Register(typeof(ui_MyWindow), "UI/MyWindow", EUIResLoadType.AssetBundle);
UIResRegistry.Register(typeof(ui_MyWidget), "UI/MyWidget", EUIResLoadType.AssetBundle);
```

- `hasUpdate` 要和是否重写 `OnUpdate` 保持一致；生成器会自动判断。
- 未注册类型首次使用会走反射兜底并输出 `UI not pre-registered` Warning；反射要求 public 无参构造。
- `RegisterWindow` 的 Holder 类型由泛型约束保证，填写错误的类型关系会编译失败。

## 手动创建 Holder

少数场景需要自己创建 Holder（例如临时 Tips 多实例）：

```csharp
ui_UILogicTestAlert holder =
    await UIHolderFactory.CreateUIHolderAsync<ui_UILogicTestAlert>(parent);

ui_UILogicTestAlert sync =
    UIHolderFactory.CreateUIHolderSync<ui_UILogicTestAlert>(parent);
```

手动 Holder 没有窗口 owner，不会自动播放转场，也不会自动解绑事件；生命周期由调用方维护。资源加载失败或 prefab 缺少 Holder 时返回 `null`，已实例化的节点会被销毁。

## UISetting 与生成流程

生成器的配置入口是编辑器窗口 `AlicizaX / UISetting Window`，三个页签分别管命名、输出和元素映射。

![UISetting - General](../../assets/ui/ui-setting-general.png)

*General：命名前缀、`Identifier Formatter`、`Resource Path Resolver`、`Script Code Emitter / Writer`、`Exclude Keywords`，下方实时预览生成的脚本与组件。*

![UISetting - Script Generation](../../assets/ui/ui-setting-script-generation.png)

*Script Generation：按工程分别配置 `Namespace`、Holder 代码输出目录、Prefab 根目录和加载类型（`Resources` / `Asset Bundle`）。*

![UISetting - Element Mapping](../../assets/ui/ui-setting-element-mapping.png)

*Element Mapping：节点名前缀到组件类型的映射，例如 `Btn` → `UXButton`、`Img` → `Image`、`ScrollView` → `RecyclerView`、`Drag` → `UXDraggable`。*

在 Hierarchy 里选中节点右键即可生成：

![UI 生成菜单](../../assets/ui/ui-generate-context-menu.png){ width="240" }

*右键菜单里的 `UI生成 / 绑定` 与 `UI生成 / 仅复制属性` 两个入口。*

生成的 Holder 和窗口关系会由生成注册表在启动时自动注册，不需要在每个窗口里写注册代码。

## 注意事项

1. `[Window]` 只写在窗口逻辑类上；`[UIRes]` 只写在 Holder 类上。
2. 缓存窗口复用时不会重新执行 `OnInitialize`；把一次性初始化放在 `OnInitialize`，把每次打开的状态放在 `OnOpen`。
3. `OnClose` 执行时关闭转场还没开始；不要在 `OnClose` 里再起一套动画。
4. 手动 Holder 不会自动解绑事件，也不播放转场，业务要自己管理生命周期。
5. 生成器修改后需要重新生成 Holder 和注册表；生成器诊断中的类型错误要先修复，避免运行时走反射兜底。
