# Procedure 流程

Procedure 是一个单线程的流程状态机：`ProcedureBuilder` 创建一个常驻的 `ProcedureRunner`，持有一组 `ProcedureBase` 实例，任何时刻只有一个"当前流程"。切换流程就是"上一个 `OnLeave` → 下一个 `OnEnter`"，用来编排启动、热更新、登录这类有严格先后顺序的阶段。

边界：同一时刻只有一个流程、没有子流程栈、没有自动回滚、没有超时与重试框架，中断与重试都要写进流程自己。

源码位置：

- `Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Procedure`
- `Client/Assets/Scripts/Startup/ProcedureEntry.cs`
- `Client/Assets/Scripts/Startup/Procedure`

## 适用场景

- 启动链路：版本检查 → 初始化资源包 → 下载 → 加载程序集 → 进入游戏。
- 需要按阶段串行推进、每阶段有明确进入/离开时机的业务流程（登录、匹配、结算）。
- 需要每帧轮询某个异步状态、轮询结束再进入下一阶段（此时 `OnUpdate` 比 `await` 更直观）。

不适合并行分支、可回退的向导，以及需要多个流程实例同时运行的场合。

## 使用前提

- 需要一个挂 `ProcedureEntry`（或自己调用 `ProcedureBuilder`）的启动场景。模板里是 `Client/Assets/Scenes/Main.unity` 中名为 `Procedure` 的节点。
- 启动模板的流程依赖 `Entry.prefab`：`ProcedureEntryState` 与后续流程会访问 `GameApp.ResourcePackage`，也就是 `ResourceComponent` 注册的 `IResourcePackageService`。`Entry` 的 `Awake` 早于 `ProcedureEntry.Start`，顺序天然成立。
- 模板在 `InitializeProcedure` 之前先等 `YooAssets.IsInitialized`，并加载启动本地化表；自建流程时同样要把这些前置条件放在第一个流程之前。
- 异步写法使用 UniTask（`Cysharp.Threading.Tasks`）。热更程序集加载走 `ENABLE_HYBRIDCLR` 宏，未开启时从当前 `AppDomain` 里找程序集。

## 最短接入

1. 继承 `ProcedureBase`，在 `OnEnter` 里启动该阶段的工作。
2. 用 `SwitchProcedure<T>()` 推进到下一阶段。
3. 在启动点调用 `ProcedureBuilder.InitializeProcedure` 并指定默认流程。

```csharp
using System.Collections.Generic;
using AlicizaX;
using Cysharp.Threading.Tasks;

public sealed class ProcedureLogin : ProcedureBase
{
    protected override void OnEnter()
    {
        LoginAsync().Forget();
    }

    private async UniTaskVoid LoginAsync()
    {
        await UniTask.Delay(500);
        SwitchProcedure<ProcedureLobby>();
    }
}

public sealed class ProcedureLobby : ProcedureBase
{
    protected override void OnEnter()
    {
        // 进入大厅
    }
}

public static class Launcher
{
    public static void Start()
    {
        ProcedureBuilder.InitializeProcedure(
            new List<ProcedureBase> { new ProcedureLogin(), new ProcedureLobby() },
            typeof(ProcedureLogin));
    }
}
```

`InitializeProcedure` 返回时，默认流程的 `OnEnter` 已经同步执行完毕（`ProcedureLogin.OnEnter` 只启动了异步任务，所以不会卡住调用方）。

## 核心用法：生命周期钩子

| 钩子 | 调用时机 |
| --- | --- |
| `OnInit()` | 流程实例被 `AddProcedure` 加入时，紧随构造之后，按列表顺序调用 |
| `OnEnter()` | 成为当前流程时；默认流程在 `InitializeProcedure` 内部就会触发 |
| `OnLeave()` | 离开当前流程时，早于下一个流程的 `OnEnter` |
| `OnUpdate()` | `ProcedureRunner.Update` 每帧调用当前流程 |
| `OnDestroy()` | `ClearAllProcedures` / `Shutdown` 时对每个流程调用一次 |

五个钩子都是 `protected internal virtual`，子类写 `protected override` 即可。`ProcedureBase` 另外提供 `protected ResourceScope Res`：首次访问时用 `AppServices.Require<IResourceService>().CreateScope(GetType().Name)` 创建，`OnLeave` 后由基类释放，适合"跟着流程走"的资源加载。

```csharp
protected override void OnEnter()
{
    Res.Load<UnityEngine.GameObject>("UI/LoginPanel");
}
```

## 核心用法：InitializeProcedure

三个重载只是容器类型不同，语义一致：

```csharp
void InitializeProcedure(IEnumerable<ProcedureBase> availableProcedures, Type defaultProcedureType);
void InitializeProcedure(ProcedureBase[] availableProcedures, Type defaultProcedureType);
void InitializeProcedure(List<ProcedureBase> availableProcedures, Type defaultProcedureType);
```

调用时会：

1. 先 `DestroyProcedure()` 清掉已有的运行器（重复调用等于重启流程系统）。
2. 新建名为 `[ProcedureRunner]` 的 `GameObject` 并 `DontDestroyOnLoad`，挂上内部 `ProcedureRunner`。
3. 依次 `AddProcedure` 并调用每个实例的 `OnInit()`。
4. 找到 `defaultProcedureType` 对应的实例并进入它；找不到就什么都不进入。

`availableProcedures` 或 `defaultProcedureType` 为 `null` 时，第 3、4 步直接跳过，`IsRunning` 仍是 `true`，但 `CurrentProcedureType` 为 `null`。

## 核心用法：切换流程

```csharp
// 流程内部：忽略返回值
SwitchProcedure<ProcedureLobby>();

// 流程外部：返回是否切换成功
bool ok = ProcedureBuilder.SwitchProcedure<ProcedureLobby>();
bool ok2 = ProcedureBuilder.SwitchProcedure(typeof(ProcedureLobby));

Type current = ProcedureBuilder.CurrentProcedureType;
bool contains = ProcedureBuilder.ContainsProcedure<ProcedureLobby>();
```

切换规则：

- 目标类型已注册且不是当前实例：先 `previous.Leave()`（`OnLeave` + 释放 `Res`），再 `next.OnEnter()`。
- 目标就是当前实例：直接返回 `true`，不重复调用钩子。
- 目标类型未注册：**静默回退到默认流程**；默认流程也不存在时返回 `false`。
- `DestroyProcedure()` 之后运行器进入销毁状态，所有切换返回 `false`。
- 列表里出现同类型的两个实例时，后者替换前者：旧实例收到 `OnDestroy()`，槽位保留，新实例收到 `OnInit()`。

## 核心用法：异步流程写法

```csharp
// 1) 纯异步推进：OnEnter 启动，完成后切下一个流程
protected override void OnEnter() => DownloadAsync().Forget();

private async UniTask DownloadAsync()
{
    await DoDownloadAsync();
    if (ProcedureBuilder.CurrentProcedureType != typeof(ProcedureDownloadBundleState)) return;
    SwitchProcedure<ProcedurePatchDoneState>();
}

// 2) 轮询推进：OnUpdate 里检查标志位（模板的 ProcedureLoadAssembly）
protected override void OnUpdate()
{
    if (_done) SwitchProcedure<ProcedureUpdateFinishState>();
}

// 3) 串行等待：OnEnter 里连续 await 多个步骤（模板的 ProcedureInitPackageState）
private async UniTask InitAsync()
{
    await GameApp.ResourcePackage.InitPackageAsync(string.Empty, hostUrl, hostUrl);
    await UniTask.DelayFrame();
}
```

流程离开后异步任务不会自动取消，回调里要自己判断是否还在当前流程（如上例），或持有 `CancellationToken`。`OnEnter` 里切换流程是允许的：模板的 `ProcedureEntryState` 就在 `OnEnter` 中直接 `SwitchProcedure` 到下一阶段。

## 核心用法：启动模板的 7 个流程

`ProcedureEntry.Start()` 的真实结构：

```csharp
private async UniTaskVoid Start()
{
    LocalizationKit.Initialize(Resources.Load<LocalizationSettings>("Localization/Settings"));
    if (!await LocalizationKit.AddTableAsync("StartupTable", new ResourcesTableSource("Localization/StartupTable_{locale}")))
        throw new InvalidOperationException("Startup localization could not be loaded.");

    await UniTask.WaitUntil(() => YooAsset.YooAssets.IsInitialized);
    ProcedureBuilder.InitializeProcedure(
        new List<ProcedureBase>
        {
            new ProcedureEntryState(),
            new ProcedureGetAppVersionInfoState(),
            new ProcedureInitPackageState(),
            new ProcedureDownloadBundleState(),
            new ProcedurePatchDoneState(),
            new ProcedureLoadAssembly(),
            new ProcedureUpdateFinishState(),
        },
        typeof(ProcedureEntryState)
    );
    Destroy(gameObject);
}
```

| 顺序 | 流程 | 职责 |
| --- | --- | --- |
| 1 | `ProcedureEntryState` | 注册下载进度订阅；`OfflinePlayMode` / `EditorSimulateMode` / `WebPlayMode` 直接进初始化，否则先取远端版本 |
| 2 | `ProcedureGetAppVersionInfoState` | 比对 `Application.version` 与 `StartupSetting.Version`，不一致就 `Utility.Platform.Quit()` 并跳转下载页 |
| 3 | `ProcedureInitPackageState` | `InitPackageAsync` → 请求包版本 → 加载 manifest，失败最多重试 3 次 |
| 4 | `ProcedureDownloadBundleState` | 创建下载器；无更新直接跳过；有更新则下载并用 `EventBus` 广播进度（`OnLeave` 里 `CancelDownload`） |
| 5 | `ProcedurePatchDoneState` | `ClearCacheAsync(ClearCacheMethods.ClearUnusedBundleFiles)` 完成后进入下一阶段 |
| 6 | `ProcedureLoadAssembly` | `ENABLE_HYBRIDCLR` 下加载 AOT 元数据与热更 DLL，`OnUpdate` 中反射调用 `HotfixEntry.Entrance` |
| 7 | `ProcedureUpdateFinishState` | `UpdateProgressUtils.Dispose()`，然后 `ProcedureBuilder.DestroyProcedure()` |

第 6 步反射到的入口就是 `Client/Assets/Scripts/Hotfix/GameLogic/HotfixEntry.cs` 的 `Entrance(object[] objects)`，启动链路在热更层继续。

## 核心用法：销毁流程

```csharp
ProcedureBuilder.DestroyProcedure();
```

执行顺序：`Shutdown()` 标记运行器销毁 → 当前流程 `OnLeave()` 并释放 `Res` → 所有流程 `OnDestroy()` → 销毁 `[ProcedureRunner]` 的 `GameObject` → `_runner` 置空。

模板在最后一个流程里自毁：`ProcedureUpdateFinishState.OnEnter` 调用 `DestroyProcedure()`。此时 `OnEnter` 剩余代码仍会执行完，但流程系统已停机，之后所有 `SwitchProcedure` 都返回 `false`。

## 容易踩的坑

1. 重复调用 `InitializeProcedure` 会先销毁旧运行器，旧流程实例全部 `OnDestroy`；不要把它当成"注册更多流程"。
2. `defaultProcedureType` 不在列表里不会报错，只是没有任何流程被进入（`CurrentProcedureType` 为 `null`），表现为"启动后一片安静"。
3. 切到拼错或未注册的类型不会报错，会静默回退到默认流程；症状是"总是跳回第一个流程"。
4. `.Forget()` 的异步任务不会随流程离开而取消。回调里要判断 `ProcedureBuilder.CurrentProcedureType`，否则离开流程后仍会切换流程或操作已释放资源。
5. 流程离开后 `Res` 已被释放；如果之后异步回调再次访问 `Res`，会新建一个作用域，而它不会再被自动释放，等于泄漏到容器关闭。
6. `DestroyProcedure()` 之后没有"重新开始"的捷径，只能重新 `InitializeProcedure`（会新建运行器与新实例）。

## API 速查

```csharp
// ProcedureBuilder（静态）
void InitializeProcedure(IEnumerable<ProcedureBase> availableProcedures, Type defaultProcedureType);
void InitializeProcedure(ProcedureBase[] availableProcedures, Type defaultProcedureType);
void InitializeProcedure(List<ProcedureBase> availableProcedures, Type defaultProcedureType);
bool SwitchProcedure<T>() where T : ProcedureBase;
bool SwitchProcedure(Type procedureType);
bool ContainsProcedure<T>() where T : ProcedureBase;
bool ContainsProcedure(Type procedureType);
void DestroyProcedure();
Type CurrentProcedureType { get; }
bool IsRunning { get; }

// ProcedureBase（子类可重写）
protected internal virtual void OnInit();
protected internal virtual void OnEnter();
protected internal virtual void OnLeave();
protected internal virtual void OnUpdate();
protected internal virtual void OnDestroy();
protected ResourceScope Res { get; }
protected internal void SwitchProcedure<T>() where T : ProcedureBase;
```

相关页面：[Service 服务容器](../service/index.md)、[资源模块](../resource/index.md)、[快速开始](../../quickstart.md)。
