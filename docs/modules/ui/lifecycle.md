# 窗口声明与生命周期

一个窗口由三部分组成：逻辑类（`UIWindow<Holder>`）、元数据（`[Window]`，也可以手动注册）、Holder（`UIHolderObjectBase`，带 `[UIRes]` 资源地址）。这一页讲它们怎么声明、生成器怎么产出 Holder 与 `ui_*` 脚本、以及钩子的真实执行顺序。

## 适用场景

- 新增一个界面：从声明、生成到第一次打开。
- 判断某段代码该放 `OnInitialize`、`OnOpen`、`OnRefresh` 还是 `OnDestroy`。
- 排查「事件没退订」「缓存实例被复用了但状态不对」「窗口没走 `OnOpen`」。
- 手动注册动态窗口，或绕过 UI 服务自己实例化 Holder（弹字、多实例 Tips）。

## 使用前提

- 已按 [UI 界面](index.md) 挂好 `UIComponent` 与 `UIRoot.prefab`，`GameApp.UI` 可用。
- 生成器需要配置文件 `ProjectSettings/UIGenerateConfiguration.asset`（用 `AlicizaX/UISetting Window` 打开）。
- 要生成的节点必须是 **prefab 资产**，且资产路径位于该生成项的 `Prefab Root Path` 之下。
- Holder 的加载走 `IResourceService`：`[UIRes]` 的 `AssetBundle` 用 `Instantiate/InstantiateAsync`，`Resources` 用 `Resources.Load/Resources.LoadAsync`。
- 可选依赖：Element Mapping 里把 `Text` 映射到 `TMPro.TextMeshProUGUI` 需要 TextMeshPro；把 `Btn` 映射到 `UXButton` 需要 `com.alicizax.unity.ui.extension`。UI 核心代码不引用它们。

## 最短接入

1. 打开生成配置：菜单 `AlicizaX/UISetting Window`，在 `Script Generation` 页签新增一条生成项。

   | 字段 | 本模板的配置 | 说明 |
   | --- | --- | --- |
   | `Project Name` | `Hotfix` | 仅用于区分工程 |
   | `Namespace` | `Game.UI` | 生成脚本的命名空间 |
   | `Holder Code Path` | `Assets/Scripts/Hotfix/GameLogic/UI/Generate` | `ui_*` 输出目录 |
   | `Prefab Root Path` | `Assets/Bundles/UI` | 允许生成的 prefab 根目录（窗口放在其 `Window/` 子目录） |
   | `Load Type` | `AssetBundle` | 写入 `[UIRes]` 的加载方式 |

2. 在 prefab 上给要绑定的节点改名，格式是 `组件前缀@名字`。模板里的真实例子：`Btn@Shop`、`Text@Title`、`ScrollView@GoodsList`、`Tog@Recommend`、`Dropdown@Languages`、`Img@BackGround`。前缀到组件类型的映射在 `Element Mapping` 页签里配（默认规则见下面的「常见配置」）。

3. 选中 prefab 根节点，执行菜单 `GameObject/UI生成/绑定`。编辑器会：

   - 按 `GeneratePrefix` + prefab 名生成类名，例如 `ui_UIHomeWindow`；
   - 把脚本写到 `Holder Code Path`，内容基于 `Editor/Modules/UI/Res/UITemplate.txt`；
   - 编译后自动把 Holder 组件挂到该节点，回填所有 `[SerializeField]` 字段，并刷新默认转场引用。

   只想要字段声明、不想生成文件时，用 `GameObject/UI生成/仅复制属性`，内容进剪贴板。

4. 声明逻辑类，使用 `baseui` 访问生成出来的字段：

   ```csharp
   using AlicizaX.UI.Runtime;
   using Game.UI;

   namespace GameLogic.UI
   {
       [Window(UILayer.UI, 30)]
       public class UIHomeWindow : UIWindow<ui_UIHomeWindow>
       {
           protected override void OnInitialize()
           {
               baseui.BtnShop.onClick.AddListener(OnBtnShopClick);
           }

           private void OnBtnShopClick()
           {
           }

           protected override void OnOpen() { }
           protected override void OnClose() { }
           protected override void OnDestroy() { }
       }
   }
   ```

5. 打开它。声明完成后，打开/关闭见 [打开、关闭与查询](window.md)；示例代码在 `Client/Assets/Scripts/Hotfix/GameLogic/HotfixEntry.cs`，只有一行 `await GameApp.UI.ShowUI<UIHomeWindow>();`。

## 核心用法

### 声明窗口：`[Window]`

```csharp
[Window(UILayer.UI, 30)]      // 关闭后缓存 30 秒
public class UIHomeWindow : UIWindow<ui_UIHomeWindow> { }

[Window(UILayer.Popup, -1)]   // 常驻缓存，不销毁
public class UIBuyAlertWindow : UIWindow<ui_UIBuyAlertWindow> { }

[Window(UILayer.UI, 0)]       // 关闭即销毁（等同 UICacheTime.None）
public class UIRecyclerViewTestHubWindow : UIWindow<ui_UIRecyclerViewTestHubWindow> { }
```

| 参数 | 说明 |
| --- | --- |
| `windowLayer` | `UILayer` 之一，决定窗口挂在哪个层 |
| `cacheTime` | `UICacheTime.None`（0）关闭即销毁；`UICacheTime.Forever`（-1，任意负数同理）常驻缓存；正数表示关闭后保留的秒数 |

注册后这些值会固化进 `UIMetaRegistry.UIMetaInfo`（`ViewType` / `HolderType` / `Handle` / `Name` / `Layer` / `CacheTime` / `HasUpdate` / `IsWindow`，全部只读）。`HasUpdate` 在反射兜底里按「是否重写了 `OnUpdate`」计算，手动注册时要自己传对。

### 声明 Holder 与资源

生成器产出的 Holder 长这样（`Client/Assets/Scripts/Hotfix/GameLogic/UI/Generate/ui_UIHomeWindow.cs`）：

```csharp
using AlicizaX.UI.Runtime;

namespace Game.UI
{
    [UIRes(ui_UIHomeWindow.ResTag, EUIResLoadType.AssetBundle)]
    public partial class ui_UIHomeWindow : UIHolderObjectBase
    {
        public const string ResTag = "UIHomeWindow";

        [SerializeField] private UXButton mBtnShop;
        public UXButton BtnShop => mBtnShop;
    }
}
```

- `[UIRes(location, loadType)]` 只写在 Holder 上；`EUIResLoadType` 只有 `Resources` 和 `AssetBundle` 两个值。
- `ResTag` 由生成时的 Resource Path Resolver 算出：YooAsset 包开启可寻址时用 prefab 名（本模板就是 `UIHomeWindow`），否则用去掉扩展名的资源全路径。
- 加载失败或预制体上找不到 Holder 组件时，`UIHolderFactory` 记录 `UI resource could not be loaded` / `missing holder`，并销毁已实例化的节点，`ShowUI` 返回 `null`。

### Holder 与逻辑类的关系

- Holder 是 `MonoBehaviour` 节点，逻辑类不是：逻辑类由注册时给的工厂 `new` 出来，两者通过 `Bind` 关联，`UIWindow<T>.baseui` 就是 `(T)Holder`。
- Holder 提供 `OnInitEvent` / `OnShowEvent` / `OnCloseEvent` / `OnDestroyEvent` 四个事件，在对应钩子 **之前** 触发，异常只记日志。
- Holder 通过 `UITransition` 提供转场，`SetTransition(IUITransition)` 可在运行时覆盖，传 `null` 恢复序列化的默认值。默认值（`_defaultTransition`）由编辑器自动解析：优先自身组件，其次深度优先找子节点，遇到嵌套 Holder 截断；Holder 的 Inspector 整体只读，不要在面板里改。
- Holder 被外部销毁时，会回调所属视图走销毁流程（`DestroySelf`）。

### 生命周期钩子与真实顺序

| 钩子 | 触发时状态 | 执行次数 | 典型用途 |
| --- | --- | --- | --- |
| `OnInitialize` | `Closed`（Holder 已绑定、仍隐藏） | 每个实例一次 | 绑定按钮、建常驻 Widget、初始化事件代理 |
| `OnOpen` | `Opening`（转场尚未开始） | 每次打开 | 读取 `UserData`、刷新界面 |
| `OnRefresh` | `Opening` / `Opened` | 已打开时再次 `Show` | 用新参数刷新，不重播转场 |
| `OnUpdate` | `Opening` / `Opened` 且父级启用 | 每帧（重写才启用） | 逐帧逻辑 |
| `OnClose` | `Closing`（转场尚未开始） | 每次关闭 / 销毁前补一次 | 停协程、保存状态 |
| `OnDestroy` | `Destroyed` | 实例一次 | 退订外部事件、释放引用 |

首次打开的完整顺序：

```text
资源加载(Loading) → 绑定 Holder(Closed) → Holder.OnInitEvent → OnInitialize
→ 先隐藏一次 → 置为可见并进入 Opening → Holder.OnShowEvent → OnOpen
→ 开场转场 → Opened（射线开启）
```

其余路径：

- 缓存中的窗口再次打开：`OnOpen` → 转场，不重跑 `OnInitialize`，不重新加载资源。
- 已打开（或正在开场）时再次 `ShowUI`：只执行 `OnRefresh`，不重播开场转场。
- 关闭：状态先变 `Closing`、射线关闭 → `Holder.OnCloseEvent` → `OnClose` → 关闭转场 → `Closed` → 进缓存或销毁。
- 销毁：`Destroyed`；如果销毁前是 `Opening`/`Opened`，会先补一次 `OnClose`，再 `Holder.OnDestroyEvent` → `OnDestroy` → 销毁子 Widget → 回收节点。

钩子里的开关操作立即生效，原流程不再继续：例如在 `OnOpen` 里 `CloseSelf()`，开场转场不会播放；在 `OnClose` 里再次打开，窗口会停在 `Opened`。钩子抛异常只记录日志，生命周期继续推进。

### 缓存与销毁

- 关闭转场结束后才决定去向：`force` 或 `CacheTime == 0` 直接销毁，否则从层级里移除并进入缓存。
- 进入缓存时 `UserData` 会被清空；`CacheTime` 为正数时用 unscaled 计时器计时，到期自动销毁；负数常驻。
- `UIService` 释放时（`AppServices.Shutdown`）会结束导航、销毁全部窗口并销毁 UI 根节点；之后 `ShowUI` 只记录 `UIService is not running.`。

### 读取 `UserData`

`ShowUI` 只传一个对象，多个字段请自己定义参数类。真实例子（`UIBuyAlertWindow`）在 `OnOpen` 里读：

```csharp
protected override void OnOpen()
{
    _goodsData = UserData as ShopGoodsData;
    RefreshView();
}
```

- 传 `null` 不覆盖已有参数；非 `null` 才替换。
- 参数在进入缓存和销毁时清空；缓存的窗口重新打开时 `UserData` 是 `null`，需要靠 `OnRefresh` 传新参数。

### 视图自带的能力

| 成员 | 用途 |
| --- | --- |
| `State` / `Visible` | 当前状态；`Visible` 为 `Opening`/`Opened`/`Closing` |
| `AwaitTransition()` | 等到转场结束（状态稳定） |
| `EventProxy` | 事件代理，随视图启停自动订阅/退订；销毁后再访问抛异常 |
| `Res` | 视图级 `ResourceScope`，随视图销毁一起释放 |
| `DestroyCancellationToken` | 视图销毁时取消，适合接业务异步 |
| `CloseSelf(bool force, bool snap)` | 窗口内关闭自己（`UIWindow` 的 protected 方法） |

```csharp
protected override void OnOpen()
{
    LoadAsync(DestroyCancellationToken).Forget();
}

private async UniTaskVoid LoadAsync(CancellationToken token)
{
    await UniTask.Delay(1000, cancellationToken: token);
}
```

### 手动注册

生成器没有覆盖面（动态类型、插件里的窗口）时手动注册，两步都要做：

```csharp
using AlicizaX.UI.Runtime;

UIMetaRegistry.RegisterWindow<MyWindow, ui_MyWindow>(
    () => new MyWindow(),
    UILayer.UI,
    UICacheTime.Forever,
    hasUpdate: false);

UIResRegistry.Register(typeof(ui_MyWindow), "UI/MyWindow", EUIResLoadType.AssetBundle);
```

- `hasUpdate` 必须与「是否重写 `OnUpdate`」一致，否则更新不会驱动。
- 未注册的类型首次打开会走反射兜底：从泛型实参解析 Holder、读 `[Window]`、用 public 无参构造，并打一条 `UI not pre-registered ... using reflection fallback.` 的 Warning；没有 `[Window]` 时默认 `UILayer.UI` + 关闭即销毁。
- 生成注册入口是程序集里的 `AlicizaX.UI.Runtime.UIGeneratedRegistry.RegisterAll`，存在才会被调用；本仓库的 UI 生成器只产出 Holder 脚本，没有生成注册表，所以模板里首次打开会看到上面那条 Warning，属正常兜底。

### 手动创建 Holder

不经过 `UIService`，自己实例化 Holder（适合多实例飘字、临时提示）：

```csharp
using AlicizaX.UI.Runtime;
using Cysharp.Threading.Tasks;

ui_UIHomeWindow holder = await UIHolderFactory.CreateUIHolderAsync<ui_UIHomeWindow>(parent);
ui_UIHomeWindow sync = UIHolderFactory.CreateUIHolderSync<ui_UIHomeWindow>(parent);
```

手动创建的 Holder 没有 owner：`IsEnabled` 恒为 `false`，不播转场、不自动解绑事件，节点生命周期由调用方负责。

## 常见配置

| 位置 | 字段 | 说明 |
| --- | --- | --- |
| UISetting `General` | 组件分割符 / 组件结尾符 | 默认 `#` / `@`：`@` 之后是字段名后缀，`#` 用于同一节点声明多个组件（`Btn#Img@Foo`） |
| UISetting `General` | 数组分割 | 默认 `*`，用于数组型绑定 |
| UISetting `General` | 生成脚本前缀 | 默认 `ui`，决定 `ui_XXX` 类名 |
| UISetting `General` | `Exclude Keywords` | 命中关键字的节点不生成，默认含 `ViewHolder` |
| UISetting `General` | 四个类型选择 | `Identifier Formatter` / `Resource Path Resolver` / `Script Code Emitter` / `Script File Writer` |
| UISetting `Script Generation` | 每项 5 个字段 | 见上面「最短接入」的实验表 |
| UISetting `Element Mapping` | 前缀 → 组件类型 | 默认文件与本模板配置包含 `Btn`→`UXButton`、`Text`→`TMPro.TextMeshProUGUI`、`Img`→`Image`、`ScrollView`→`AlicizaX.UI.RecyclerView`、`Tog`→`UXToggle`、`Input`→`TMP_InputField`、`Drag`→`UXDraggable`、`Dropdown`→`TMP_Dropdown` |
| Holder Inspector | `Transition` | 只读，编辑器自动维护；要改转场在预制体上换组件或 `SetTransition` |
| `UIPresetTransition` | `openPreset` / `closePreset` / `openDuration` / `closeDuration` / `useUnscaledTime` | 预设转场参数，详见 [路由与转场](router.md) |

## 容易踩的坑

1. `[Window]` 只能写在逻辑类上、`[UIRes]` 只能写在 Holder 上；写反了要么走反射兜底（丢层级/缓存设置），要么在加载时报 `UIResAttribute not found`。
2. 在场景里的临时节点上点生成不会有任何产出：生成器要求 prefab 资产，否则只打一条 `Please save the UI as a prefab asset before generating bindings.`。
3. 类名 = 前缀 + prefab 名；给 prefab 改名后要重新生成，并同步改逻辑类的泛型实参与 `using`，否则编译不过。
4. 缓存的窗口复用时不会重跑 `OnInitialize`，但仍然长期持有实例：普通 C# 事件（如 `LocalizationKit.Changed`）必须在 `OnDestroy` 里退订；`EventProxy` 的订阅只随启停挂起/恢复，不要和手动订阅混在一起。
5. 不要在 `OnDestroy` 之后访问 `EventProxy`，会抛 `InvalidOperationException`；也不要把一次性初始化放在 `OnOpen`，否则每次打开都会重复执行。
6. 手动创建的 Holder 没有 owner，不能指望转场和生命周期自动推进；需要受管生命周期就用 `UIMetaRegistry` 注册成窗口/Widget。

## API 速查

| 类型 / 成员 | 用途 |
| --- | --- |
| `UIWindow<T>` / `UITabWindow<T>` | 窗口基类（`T` 为 Holder 类型），`baseui` 取 Holder |
| `WindowAttribute(UILayer, int)` | 声明层级与关闭后缓存秒数 |
| `UICacheTime.None` / `UICacheTime.Forever` | 0 / -1 |
| `UIResAttribute(string, EUIResLoadType)` | Holder 的资源地址与加载方式 |
| `UIHolderObjectBase` | Holder 基类：`OnInitEvent` / `OnShowEvent` / `OnCloseEvent` / `OnDestroyEvent`、`UITransition`、`SetTransition` |
| `UIMetaRegistry.RegisterWindow` / `RegisterWidget` | 手动注册视图元数据 |
| `UIMetaRegistry.TryGet(string, out UIMetaInfo)` | 按类型全名查元数据 |
| `UIResRegistry.Register(Type, string, EUIResLoadType)` | 手动注册 Holder 资源 |
| `UIHolderFactory.CreateUIHolderAsync<T>` / `CreateUIHolderSync<T>` | 手动创建 Holder |
| `UIGenerateConfiguration` | 生成配置（`ProjectSettings/UIGenerateConfiguration.asset`） |
