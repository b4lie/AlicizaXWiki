# GameObjectPool 实例池

GameObjectPool 只池化 Unity `GameObject` 实例：按 YooAsset location 命中规则、`Instantiate` Prefab、按策略回收与销毁。Prefab 的资源租约由资源模块持有；普通 C# 引用对象请用 [MemoryPool](../memorypool/index.md)。

源码：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/GameObjectPool`、`Client/Packages/com.alicizax.unity.framework/Editor/Modules/GameObjectPool`。

## 适用场景

- 高频生成/销毁的 Prefab：特效、弹道、掉落物、飘字、列表小控件。
- 有明确容量上限、需要空闲回收策略的对象（战斗高峰期突增、战斗后回落）。
- 希望复用预热实例、避免反复 `Instantiate` 的循环体。

不适用：整窗 UI、`NetworkObject`、一次性物体（走 `GameApp.Resource.Instantiate`）、纯 C# 对象（走 `MemoryPool`）。

## 使用前提

1. **挂载位置**：服务根 Prefab（模板 `Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab`）下的 `GameObjectPool` 子节点挂 `GameObjectPoolComponent`（菜单 `Game Framework/GameObjectPool`）。该组件**没有任何 Inspector 字段**，`Awake` 里注册 `IGameObjectPoolService` 与 `IGameObjectPoolDebugService`，并把自身 `transform` 当作空闲实例容器根。
2. **依赖资源模块**：同根节点必须有 `ResourceComponent`。`ResourceComponent` 是 `[DefaultExecutionOrder(-700)]`，`GameObjectPoolComponent` 无执行顺序（0），注册时 `OnInitialize` 立即 `Require<IResourceService>()`。
3. **必须先加载目录**：组件不会自动读 PoolConfig，模板里也**没有内置 `GameObjectPoolConfig.asset`**。未加载目录时所有 `Spawn` / `LoadPrefab` 都返回 `null`，Editor 与 Development Build 下额外打 `[GameObjectPool] Location is not in PoolConfig`；Release 包静默返回 `null`。
4. **获取服务**：

```csharp
using AlicizaX;

IGameObjectPoolService pool = GameApp.GameObjectPool;
```

`GameApp.GameObjectPool` 是 `AppServices.Require<IGameObjectPoolService>()` 的静态缓存；服务未注册时抛 `InvalidOperationException`。

## 最短接入

1. 建配置资源：Project 窗口右键 `Create > AlicizaX > PoolConfig`（默认文件名 `GameObjectPoolConfig`）。
2. 放进被 YooAsset 收集的目录，模板为 `Client/Assets/Bundles/Configs/sciptableObject/`（Collector `AddressByFileName`）→ location 就是**文件名去扩展名**。
3. 资源包初始化完成后加载目录：

```csharp
GameApp.GameObjectPool.LoadCatalog("GameObjectPoolConfig");
// 已经持有资产引用时也可以直接传资产
// GameApp.GameObjectPool.LoadCatalog(configAsset);
```

4. 先把 Prefab 钉进内存，再同步生成：

```csharp
GameObject prefab = GameApp.GameObjectPool.LoadPrefab("Effects/Hit");   // 只钉 Prefab，不建实例
// 或 GameObject prefab = await GameApp.GameObjectPool.LoadPrefabAsync("Effects/Hit");
if (prefab == null)
{
    return;
}

GameObject instance = GameApp.GameObjectPool.Spawn("Effects/Hit", transform); // parent 可省
GameApp.GameObjectPool.Despawn(instance);
```

5. 回收一律走 `Despawn`，不要对池实例调 `Destroy`。

## 核心用法

### 生成：谁加载 Prefab

| 方法 | 加载 Prefab | 建实例 | Prefab 未加载时 | 满 `hardCapacity` 时 |
| --- | --- | --- | --- | --- |
| `Spawn(location, parent)` | 否 | 有空闲弹尾，否则新建 | 返回 `null` | 返回 `null` + `HardCapacity reached` 警告 |
| `Spawn<T>(location, parent)` | 否 | 同上 | `null` | `null` |
| `SpawnAsync(location, parent, ct)` | 是 | 先 `LoadPrefabAsync` 再 `Spawn` | 不适用 | `null` |
| `LoadPrefab` / `LoadPrefabAsync` | 是 | 否 | — | — |
| `WarmupAsync(location, count, ct)` | 是 | 只造空闲 | — | 停在 hard |

```csharp
GameObject fx = GameApp.GameObjectPool.Spawn("Effects/Hit", transform);
if (fx == null)
{
    fx = await GameApp.GameObjectPool.SpawnAsync("Effects/Hit", transform);
}
```

- 同步 `Spawn` 不会把失败的同步请求转异步；也不会加入进行中的异步加载（`LoadPrefabAsync` 未完成时 `LoadPrefab` 返回 `null`）。
- `Spawn<T>` 只做**根节点** `GetComponent<T>()`，子节点上的组件取不到——取不到会先 `Despawn` 再返回 `null`。
- `WarmupAsync` 数量夹在 `[0, hardCapacity]`，每帧最多 8 个或 1ms 就让出；取消后已造的实例留在空闲列表。

### 回收：`Despawn` 与句柄

池在实例上自动 `AddComponent<GameObjectPoolHandle>()`（`[DisallowMultipleComponent]`），两种 `Despawn` 等价：

```csharp
var handle = instance.GetComponent<GameObjectPoolHandle>();
GameApp.GameObjectPool.Despawn(handle);   // 与 Despawn(instance) 同一条路径
```

业务错误直接抛 `InvalidOperationException`，**不销毁对象、不改动池账本**：

- 对象上没有 `GameObjectPoolHandle`：`... is not a pooled instance`。
- 已回池（重复归还）：`... is not spawned`。
- 所属池已被新的 `LoadCatalog` 释放：`... no longer belongs to a pool`。

池外 `Destroy` 会经 `GameObjectPoolHandle.OnDestroy` 摘槽，Editor 与 Development Build 下打印 `destroyed outside pool` 警告；槽位会被下一个实例复用。

### 清理：`Flush` 系列

```csharp
GameApp.GameObjectPool.Flush("Effects/Hit");
GameApp.GameObjectPool.FlushGroup("FX");   // 空/空白参数回落 DefaultGroup
GameApp.GameObjectPool.FlushAll();
```

- 三者都走**低内存计划**：任何策略（含 `Sticky`）都按 `minIdle` 剪空闲，**绝不销毁在场实例**。
- 每次维护有剪裁预算：约 `softCapacity >> 2`，夹在 `1..16`；低内存时固定 16。所以一次 `Flush` 可能要多帧才剪完。
- `Application.lowMemory` 会对所有池执行一次低内存维护。
- 池被剪空且 `unloadPrefab = true` 时释放 Prefab 源租约（交给资源模块的 Idle 回收，不是立刻从内存抠掉）。

### 池化回调：`IGameObjectPoolable`

```csharp
using AlicizaX;
using UnityEngine;

public sealed class PooledMuzzle : MonoBehaviour, IGameObjectPoolable
{
    [SerializeField] private ParticleSystem _fx;

    public void OnSpawn(in PoolSpawnContext context)
    {
        _fx.Clear(true);
        _fx.Play(true);
    }

    public void OnDespawn()
    {
        _fx.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);
    }

    public void OnPooledDestroy()
    {
    }
}
```

- 实例创建时用 `GetComponentsInChildren(true, ...)` 收集**一次**并缓存，含未激活子节点。
- `PoolSpawnContext`：`Location`、`Group`、`Parent`、`SpawnFrame`（`Time.frameCount`）。
- **显隐接管**：根节点上存在任意 `IGameObjectPoolable` 时，池不再替它 `SetActive(false)`，`OnDespawn` 后实例保持激活；子节点上的实现只收回调，不改变显隐方式。
- 新建与预热出来的实例初始失活，第一次 `Spawn` 激活一次。
- 异常语义：`OnSpawn` 抛 → 回滚为空闲并失活，异常继续抛给调用方；`OnDespawn` 抛 → 实例留在场，修复后可再次 `Despawn`；`OnPooledDestroy` 抛 → 记日志后继续销毁，维护与关闭流程不中断。

### 规则匹配：`PoolEntry` 与 glob

- 先查精确表，再按 glob 列表匹配；glob 列表顺序由 `priority` 降序、`assetPath` 长度降序、`group` 升序决定。
- glob 语法：`*` 匹配单段，`**` 递归匹配 0 到多段（连续 `**` 会折叠），`?` 匹配单字符；首次 glob 命中的结果会写回精确表。
- `PoolEntry.NormalizeLocation`：去首尾空白、`\` 转 `/`、去尾部 `/`、去扩展名、去 `Assets/Bundles/` 或 `Assets/Bundle/` 前缀。规则与请求都走同一套归一化。
- `assetPath` 为空的条目、`null` 条目会被跳过；字面规则重复时，精确表只记住排序后的第一条（`priority` 高者优先）。
- `LoadCatalog` 会 `Shutdown` 并清空**全部**运行时池（含 `[Group]` 容器节点），然后按新目录重建。

### 自定义 `IPrefabLoader`（可选依赖）

```csharp
public interface IPrefabLoader
{
    GameObject LoadPrefab(string location);
    UniTask<GameObject> LoadPrefabAsync(string location, CancellationToken cancellationToken = default);
    void UnloadPrefab(GameObject prefab);
}
```

内置实现 `YooAssetPrefabLoader`，用 `LoadLease<GameObject>` 记引用计数，`UnloadPrefab` 归零时 `Dispose`。**说明**：接口是 public，但注入入口 `GameObjectPoolService(Transform, IPrefabLoader)` 是 `internal`（只对 `InternalsVisibleTo` 的测试程序集开放），`GameObjectPoolComponent` 固定使用内置加载器，业务程序集无法替换。

### 编辑器与调试

- **配置窗**：没有独立菜单项，双击 `PoolConfigScriptableObject` 资产，或点 Inspector 里的「在编辑器窗口中打开」，即可打开「对象池配置编辑器」。左栏是可搜索、可拖拽排序的规则列表（一行显示 `group / pattern  Policy  min-soft-hard`），右栏编辑选中规则；`priority` 由拖拽顺序自动维护，`idleSeconds` 仅在 `Burst` 下显示；重复 `assetPath` 会给出警告，工具栏保存按钮落盘。
- **运行时面板**：Play 模式选中场景里的 `GameObjectPool` 节点，每 250ms 刷新：
  - `Runtime Summary`：`Ready` / `Pools` / `Loaded Prefabs` / `Instances` / `Active` / `Inactive` / `Pending Maintenance`。
  - `Pools`：每池一张卡，标题 `[Group] entryName`，摘要 `A{active}/I{inactive}/T{total} | Hit {hit}/{spawn}` 加策略徽标；展开可看 `Policy`、`Min Idle`、`Retain Target`、`Soft / Hard`、`Next Maintenance`、`Spawn/Despawn/Hit/Miss/Expand/Destroy/Peak Active` 计数与实例列表。
  - `Flush This` / `Flush Group` 两个按钮直接触发清理。
- **Hierarchy**：容器根下可见 `[FX]`、`[HUD]` 等分组节点与休眠实例。计数器只增不减，`Flush` 不清零。

### 测试覆盖的边界

`Client/Packages/com.alicizax.unity.framework/Tests/GameObjectPool`（EditMode + PlayMode）锁定了这些行为：

- 未登记 location → `Spawn` / `LoadPrefab` 返回 `null`，不建运行时池，也不调用 loader。
- 到达 `hardCapacity` → 新建返回 `null` 并打警告，但空闲命中仍然成功。
- `LoadCatalog`（含传 `null`、传错误资产类型）→ 清空全部池与 `[Group]` 节点，之后按新目录判定"未登记"。
- 回调异常：`OnSpawn` 回滚为空闲、`OnDespawn` 实例保留在场、`OnPooledDestroy` 记日志后继续销毁。
- 外部 `Destroy` 与场景卸载 → 摘槽、池账本归零，服务仍可继续 `LoadPrefab`。
- `Time.timeScale = 0` 下 `Burst` 的空闲回收仍然生效；`Flush` 不销毁在场实例。

## 常见配置

`PoolConfigScriptableObject` 只有一个字段 `entries`（`List<PoolEntry>`）；`PoolEntry` 字段：

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `entryName` | `PoolRule` | 调试名，Inspector 与配置窗显示这个 |
| `group` | `DefaultGroup` | 空闲实例挂在容器根下 `[Group]` 节点，按组懒创建 |
| `assetPath` | 空 | YooAsset location 或 glob；空则该条被跳过 |
| `policy` | `Burst` | `Fixed` / `Burst` / `Sticky` |
| `minIdle` | `0` | 维护后至少保留的空闲数（不约束在场实例） |
| `softCapacity` | `8` | 空闲修剪目标上限 |
| `hardCapacity` | `16` | 总实例硬顶（含在场），到达后新建返回 `null` |
| `idleSeconds` | `15` | 仅 `Burst` 生效，非 Burst 会被 `Normalize()` 清零 |
| `unloadPrefab` | `true` | 池剪空后是否释放 Prefab 源租约 |
| `priority` | `0` | 越大越先匹配；配置窗按列表顺序自动维护 |

`Normalize()` 的钳制：`minIdle >= 0`、`softCapacity >= 1`、`hardCapacity >= softCapacity`、`minIdle <= hardCapacity`。

策略行为（`retain` 目标由 `PoolPolicyPlanner` 计算）：

| 策略 | 空闲保留目标 | 何时剪 | 典型用途 |
| --- | --- | --- | --- |
| `Fixed` | `clamp(minIdle, 0, soft)` | 空闲数超 `retain` 立刻剪 | HUD、固定数量控件 |
| `Burst` | `clamp(minIdle, 0, soft)` | 未超 soft 时最老空闲超过 `idleSeconds` 再剪；`total > soft` 立刻剪 | 特效、弹体 |
| `Sticky` | `max(minIdle, 当前空闲数)` | 普通 Tick 不剪，等 `Flush` 或低内存 | 关卡常驻 |

## 容易踩的坑

1. 只写 `Spawn` 不写 `LoadPrefab`：`Spawn` 不加载资源，Prefab 不在内存时永远返回 `null`，而且不打错误日志。
2. 忘记 `LoadCatalog`：所有 location 都"未登记"，`Spawn` / `LoadPrefab` 全 `null`。`LoadCatalog(string)` 的参数是 YooAsset location，不是磁盘路径或 `Assets/...`。
3. `Spawn<T>` 只查根节点；组件在子节点上时改成 `Spawn` 后自己 `GetComponentInChildren<T>(true)`。
4. 不要在战斗中途 `LoadCatalog`：它会销毁所有在场池实例（走 `OnPooledDestroy` 后销毁），业务侧持有的引用会变成空引用。
5. 别自己 `Destroy` 池实例，也别把非池对象交给 `Despawn`（直接抛异常）。父节点被销毁时实例会摘槽，不会连带销毁池里的对象。
6. 维护使用 `Time.unscaledTime`，`Time.timeScale = 0` 不会冻结 `Burst` 的空闲回收；`minIdle` 只约束空闲数，不保证在场数。

## API 速查

```csharp
// IGameObjectPoolService（GameApp.GameObjectPool）
GameObject Spawn(string location, Transform parent = null);
T Spawn<T>(string location, Transform parent = null) where T : Component;
UniTask<GameObject> SpawnAsync(string location, Transform parent = null, CancellationToken cancellationToken = default);
UniTask<T> SpawnAsync<T>(string location, Transform parent = null, CancellationToken cancellationToken = default) where T : Component;
GameObject LoadPrefab(string location);
UniTask<GameObject> LoadPrefabAsync(string location, CancellationToken cancellationToken = default);
UniTask WarmupAsync(string location, int count, CancellationToken cancellationToken = default);
void Despawn(GameObject instance);
void Despawn(GameObjectPoolHandle handle);
void Flush(string location);
void FlushGroup(string group);
void FlushAll();
void LoadCatalog(PoolConfigScriptableObject config);
void LoadCatalog(string poolConfigPath);
```
