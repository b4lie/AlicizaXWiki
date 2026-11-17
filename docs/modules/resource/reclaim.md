# 回收、统计与调试

引用归零不等于立刻卸载：记录先进 Idle，等 TTL 到期或被手动回收才释放底层句柄。这一页讲清这条时间线、三条手动回收 API 的差别、自动回收由谁驱动，以及怎么在 Inspector 里看出还有谁拿着资源。

## 适用场景

- 排查「界面关了内存没降」。
- 决定什么时候调用 `UnloadUnusedAssets` / `ForceUnloadAllAssets`。
- 接入低内存处理、切大场景前主动释放。
- 确认租约、作用域、组件绑定是否成对释放。
- 判断某条资源是被 Idle TTL 卡住，还是还有引用没还。

不适用：YooAsset 内部的 bundle 引用计数与缓存淘汰（那是 YooAsset 的策略），单资源加载写法（见[加载、租赁与绑定](loading.md)）。

## 使用前提

- 服务由 `ResourceComponent` 注册。组件的 `Update` 负责周期回收与 `Resources.UnloadUnusedAssets`，服务的 tick 负责 Idle 过期扫描。
- `ForceUnloadUnusedAssets` 依赖组件在 `Awake` 里注入的回调（`SetForceUnloadUnusedAssetsAction`）。没有组件时它是**空操作**，不报错也不释放。
- 运行时诊断面板需要 Play Mode，且选中的是场景里的 `ResourceComponent` 实例；Prefab 资产上只会显示占位提示。
- 想在面板里看到已释放的记录，要打开 Editor 区的 `Show Released Slots`。

## 最短接入

1. 先打开 Inspector 面板确认现状，再决定回收强度：

```text
Play Mode → 选中 Entry/Resources → Resource Debug → Assets
看 References 行的 Scope / Lease / Binding / Waiter 哪个不为 0
```

2. 确认无引用后手动回收：

```csharp
// 清掉全部无引用记录（不等 TTL），再让每个有效包卸载未使用 bundle
GameApp.Resource.UnloadUnusedAssets(releaseIdleAssets: true);

// 走组件路径：releaseIdle + Resources.UnloadUnusedAssets + 可选 GC.Collect
GameApp.Resource.ForceUnloadUnusedAssets(performGCCollect: true);

// 全部释放：切大场景、回登录、退出玩法时用
GameApp.Resource.ForceUnloadAllAssets();
```

3. 低内存不用自己接，`Application.lowMemory` 已经由组件接到服务：`ResourceService.OnLowMemory()` 打一条警告并触发强制回收（`performGCCollect: true`）。
4. 需要观察释放结果就再看一次面板，`Assets` 里的状态应从 `Active` 变成 `Idle`，再到 `Released`。

## 核心用法

### 引用归零之后：Idle 时间线

1. 最后一条引用归还（`lease.Dispose()`、`scope.Dispose()`、绑定释放、实例销毁）时，记录的 `RefCount` 归零，状态 `Active → Idle`，并记下入队时刻。
2. 服务每帧 tick 调用 `ReleaseExpiredIdleAssets`：从 Idle 队首按先进先出检查，超过 `idleAssetExpireTime` 就释放，单帧最多处理 `IdleReleaseCountPerTick` 条，遇到第一条没到期的就停。
3. Idle 记录数超过 `IdleAssetCapacity` 时立刻释放队首，不等到期。
4. 「释放」指的是释放这条资源记录持有的 YooAsset 句柄；bundle 文件是否真正卸载由 YooAsset 的引用计数与 `autoUnloadBundleWhenUnused` 决定。
5. TTL 未到期前再加载同一资源会命中 Idle：`Idle → Active`，不发起后端加载（面板里记为 `IdleHit`）。

时间是 `Time.unscaledTime`，不受 `Time.timeScale` 影响；暂停游戏不会冻结 Idle 过期。

| 参数 | 组件字段 | 默认 | 作用 |
| :--- | :--- | :--- | :--- |
| Idle 过期秒数 | `idleAssetExpireTime` | 60 | 无引用记录多久后被释放 |
| Idle 数量上限 | `idleAssetCapacity` | 256 | 超出后立刻释放队首 |
| 每帧释放上限 | `expireProcessCountPerFrame` | 16 | 控制单帧释放造成的尖峰 |
| 自动回收集间隔 | `maxUnloadUnusedAssetsInterval` | 300 | 周期性调用 `UnloadUnusedAssets(false)` |
| 系统卸载 | `useSystemUnloadUnusedAssets` | true | 强制回收时是否调 `Resources.UnloadUnusedAssets()` |
| GC 最小间隔 | `minGCCollectInterval` | 30 | 两次 `GC.Collect()` 的最小间隔 |

### 手动回收的三条路

| API | 会清 Idle | 会卸 bundle | 会调系统卸载 | 会 GC | 影响正在使用的引用 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `UnloadUnusedAssets(releaseIdleAssets)` | 传 `true` 时全清 | 是 | 否 | 否 | 不会，只动无引用记录 |
| `ForceUnloadUnusedAssets(performGCCollect)` | 是 | 是 | 是（需 ≥ 上一次完成） | `true` 时 | 不会，只清无引用记录 |
| `ForceUnloadAllAssets()` | 是 | 是（`UnloadAllAssetsAsync`） | 否 | 否 | 会，全部失效 |

细节：

- `UnloadUnusedAssets(false)` 只让每个「已初始化且 `PackageValid`」的包执行 `UnloadUnusedAssetsAsync()`，Idle 记录本身还留着。上一次批量卸载没完成时不会重复发起。
- `UnloadUnusedAssets(true)` 先把 Idle 链表全部清空，再走上面那步，适合切场景前的「轻量清一下」。
- `ForceUnloadUnusedAssets` 只是**请求**：它通过组件注入的回调置标志位，真正的释放发生在组件下一次 `Update`。它会把 `releaseIdleAssets` 置为 true，`performGCCollect` 还会额外请求 `Resources.UnloadUnusedAssets()` 与 `GC.Collect()`。
- `ForceUnloadAllAssets()` 同步开始重置：释放所有加载中的等待者、清空所有作用域条目、释放全部记录、清空全部绑定，再对每个有效包发起 `UnloadAllAssetsAsync()`，随后回到 `Running`。
  - 重置期间 `CanLoadResources == false`，`LoadLease*` 返回空租约、`Instantiate` 返回 `null`。
  - 作用域对象不会被销毁：`scope.IsAlive` 仍为 true，重置完成后可以继续 `Load`。
  - 已经发出去的租约全部失效（`IsValid == false`），再 `Dispose()` 不会误减引用。
  - WebGL 不支持这个接口，只会打印一条 `WebGL not support invoke ForceUnloadAllAssets` 警告后返回。

### 自动回收

`ResourceComponent.Update` 每帧累计 `Time.unscaledDeltaTime`，满足下面任一条就执行一次回收：

- 收到强制回收请求（`ForceUnloadUnusedAssets` / 低内存）。
- 距上次回收超过 `maxUnloadUnusedAssetsInterval`。

执行时的分支：

1. 调 `ResourceService.UnloadUnusedAssets(releaseIdleAssets)`，`releaseIdleAssets` 只在强制路径为 true。
2. 强制路径且 `useSystemUnloadUnusedAssets` 为 true 时，启动 `Resources.UnloadUnusedAssets()`；周期路径**不会**调系统卸载。
3. `performGCCollect` 的请求要等上一次系统卸载完成，且距上次 GC 至少 `minGCCollectInterval` 才真正 `GC.Collect()`。

也就是说：周期回收只负责「把没用的 bundle 卸掉」，真正让 Idle 记录消失的是每帧的 TTL 扫描，两者不要混为一谈。

### 统计与调试面板

入口只有一个：Play Mode 下选中场景里带 `ResourceComponent` 的物体，Inspector 底部会多出运行时区块（约 250 ms 刷新一次，Prefab 资产上不显示）。

| 区块 | 内容 |
| :--- | :--- |
| Runtime Status | `Unload Unused Assets`（已过秒数 / 间隔）、`Game Version`（`ApplicableGameVersion`，未就绪显示 `<Unknown>`） |
| Resource Debug → Assets | 记录总数；`Visible Assets`（Active / Idle / Loading / Released 计数）；`References`（Total / Scope / Lease / Binding / Waiter）；`Requests`（Total / Hit / IdleHit / Join / Miss / HitRate） |
| 单条记录行 | `#id` + `State | Package/Location | Type(SubAssets) | Ref:N | Idle:Ns` |
| Resource Debug → Scopes | `Scope N` + `Manual / Instance / Binding | Gen:n Entries:n Bindings:n`，带 Owner Object |
| Resource Debug → Bindings | 槽位名 + `Scope / Target(TargetId) / Asset / Ver / Runtime`，带 Target Object |

- `Total / Scope / Lease / Binding / Waiter` 是定位泄漏最快的一行：`Waiter` 不为 0 说明还有加载没完成，`Scope` / `Lease` 不为 0 说明还有作用域或租约没放。
- `Hit / IdleHit / Join / Miss` 能看出缓存是否生效：`Miss` 持续增长说明地址写错或缓存被频繁清掉。
- `Show Released Slots` 打开后，`Released` 状态的记录才会出现在列表里；这个开关按机器存在 EditorPrefs，不是场景配置。
- Play Mode 下 `Load Mode` 与 `Resource Cache` 两组字段是禁用的，改配置要退出 Play Mode。

诊断数据结构（`ResourceAssetInfo`、`ResourceScopeInfo`、`ResourceBindingInfo`、`ResourceStats`）与 `ResourceService.GetStats()` / `GetAssetInfos()` / `GetScopeInfos()` 都是 `internal`，业务程序集读不到；`IResourceService` 也没有统计属性。业务侧能用的判断只有 `CanLoadResources`、`AssetLease.IsValid`、`ResourceScope.IsAlive`。

### 常见泄漏原因

| 现象 | 原因 | 怎么确认 | 怎么修 |
| :--- | :--- | :--- | :--- |
| `Lease` 不归零 | `await` 之后忘了 `Dispose`，或异常路径提前 return | References 行 `Lease` 持续大于 0 | 一律用 `using`，或 try/finally 释放 |
| `Scope` 不归零 | 手动 `CreateScope` 后没 `Dispose` | Scopes 里出现 `Manual: <Unnamed>` 且 Entries 不降 | 给 scope 起名，用完 `Dispose` |
| 关了界面内存不降 | 窗口进了缓存（`CacheTime` 为 `-1` 或大于 0），缓存期间作用域不释放 | Scopes 里该界面的 scope 仍存活 | 缓存到期或 `CacheTime = UICacheTime.None` |
| 切场景后旧资源还在 | 静态字段 / 单例持有 `lease.Asset`，或绑定挂在 `DontDestroyOnLoad` 物体上 | Bindings 的 Target 指向常驻物体 | 显式释放并清空静态引用 |
| 已销毁物体的绑定迟迟不释放 | 物体未激活（`SetActive(false)`）时 `OnDestroy` 不触发，只能等扫描（每帧最多 32 个） | Bindings 的 Target 显示 `<Missing Target>` | 用 `Destroy` 真正销毁，或手动 `scope.Dispose()` |
| 加载中的请求堆积 | 发起后没取消也没等结果，`Waiter` 一直挂着 | References 行 `Waiter` 不为 0 | 传 `CancellationToken`，随界面 / 流程取消 |

### 排查步骤

1. 复现后看 `Assets` 的 `References`：`Waiter` → 加载没结束；`Lease` → 租约没放；`Scope` → 作用域没放；`Binding` → 绑定目标还活着。
2. 到 `Scopes` 找对应 scope，看 Owner Object 是不是该销毁却没销毁。
3. 到 `Bindings` 找具体组件，确认它挂在哪、是不是常驻物体。
4. 用 `UnloadUnusedAssets(true)` 排除 Idle 干扰：调用后仍 `Active` 的就是还有引用的。
5. 还找不到就全局搜 `static` 字段与单例里缓存的资源对象。

## 常见配置

| 目标 | 调什么 | 代价 |
| :--- | :--- | :--- |
| 很少重复加载，优先省内存 | `idleAssetExpireTime` 调小、`idleAssetCapacity` 调小 | 重复加载变多，可能卡顿 |
| 频繁切换界面 | `idleAssetExpireTime` 调大、`idleAssetCapacity` 调大 | 常驻内存变高 |
| 释放尖峰太抖 | `expireProcessCountPerFrame` 调小 | 释放更慢，Idle 停留更久 |
| 切场景前清干净 | 手动 `UnloadUnusedAssets(true)` 或 `ForceUnloadAllAssets()` | 下次进入要重新加载 |
| 想少调系统卸载 | `useSystemUnloadUnusedAssets` 关掉 | 依赖 GC 自己回收托管引用 |
| 减少 GC 抖动 | `minGCCollectInterval` 调大 | 强制回收后的内存回落更慢 |

## 容易踩的坑

1. **`ForceUnloadAllAssets` 不是「立即回收无用资源」**。它连正在用的引用一起作废，加载中的请求全部取消；只在切大场景、回登录这类明确的断点用。
2. **`UnloadUnusedAssets(false)` 不会缩短 Idle**。想立刻释放无引用记录必须传 `true`，或改 `idleAssetExpireTime`。
3. **`ForceUnloadUnusedAssets` 是异步生效的**。它只置标志位，组件下一次 `Update` 才真正动手；调用完立刻查内存不会有变化。
4. **没有组件时强制回收是空操作**。自己 `new ResourceService()` 或组件已被销毁的情况下，调用不会释放任何东西。
5. **`scope.Dispose()` 之后旧句柄不会复活**。槽位被新作用域复用时靠代际号区分，旧 scope 的 `Dispose` 不会影响新作用域。
6. **面板看不到就等于没泄漏，是错觉**。面板只显示 `ResourceService` 记录内的资源；静态字段直接持有的 Unity 对象、`Resources` 目录资源都不在其中。

## API 速查

`IResourceService`（`GameApp.Resource`）：

```csharp
bool CanLoadResources { get; }
float IdleAssetExpireTime { get; set; }
int IdleAssetCapacity { get; set; }
int IdleReleaseCountPerTick { get; set; }
ResourceScope App { get; }

void UnloadUnusedAssets(bool releaseIdleAssets);
void ForceUnloadAllAssets();
void ForceUnloadUnusedAssets(bool performGCCollect);
void SetForceUnloadUnusedAssetsAction(Action<bool> action);
void OnLowMemory();
```

`ResourceComponent`（组件上的手动入口与观测值）：

```csharp
float MaxUnloadUnusedAssetsInterval { get; set; }
bool UseSystemUnloadUnusedAssets { get; set; }
float LastUnloadUnusedAssetsOperationElapseSeconds { get; }
float IdleAssetExpireTime { get; set; }
int IdleAssetCapacity { get; set; }
int ExpireProcessCountPerFrame { get; set; }
string ApplicableGameVersion { get; }
int InternalResourceVersion { get; }

void ForceUnloadUnusedAssets(bool performGCCollect);
```

`ResourceScope` / `AssetLease<T>` 的自检属性：

```csharp
bool ResourceScope.IsAlive { get; }
bool AssetLease<T>.IsValid { get; }
```
