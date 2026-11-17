# Audio 音频

Audio 模块基于 Unity `AudioSource` 提供统一的播放服务：5 个音频分类（`AudioType`）、句柄控制、淡入淡出、2D/3D/跟随播放、Mixer 分组音量、Clip 缓存与预热。它不是 Wwise 级别的中间件：没有事件库、RTPC、Switch/State、Bank、虚拟声部和音乐转场。

源码：`Client/Packages/com.alicizax.unity.framework/Runtime/Modules/Audio`、`Client/Packages/com.alicizax.unity.framework/Editor/Modules/Audio`。

## 适用场景

- UI 点击/弹窗音效、技能特效声、BGM、角色语音、环境声与机关循环声。
- 需要"分类音量 + 分类开关 + 并发上限"的场合（设置面板、静音音乐但保留音效）。
- 需要按 3D 位置或跟随目标播放，并带遮挡衰减的声音。

不适用：需要自定义 DSP 图、音乐段落无缝转场、几十路精确虚拟声部管理的项目。

## 使用前提

1. **挂载位置**：服务根 Prefab（`Client/Packages/com.alicizax.unity.framework/Runtime/Prefabs/Entry.prefab`）的 `Audio` 节点上，`AudioComponent` 与 `AudioListener` 必须挂在**同一个** `GameObject`。`AudioComponent` 的四个序列化字段都要填：`m_AudioMixer`（`AudioMixer`）、`m_AudioListener`（`AudioListener`）、`m_AudioGroupConfigs`（`AudioGroupConfig[]`，必须覆盖全部 `AudioType`，每个枚举值恰好一条）、`m_ServiceConfig`（`AudioServiceConfig`，Clip 缓存容量/TTL/默认策略）。
2. **运行时硬校验**：`AudioComponent.Awake` 构造 `AudioService`，以下任一情况抛 `GameFrameworkException` 导致注册失败（之后访问 `GameApp.Audio` 会抛 "Service ... is not registered"，而不是返回 `null`）：Mixer 为空、Listener 为空、`m_AudioGroupConfigs` 为 `null`/空数组、某个 `AudioType` 缺失、同一 `AudioType` 出现两次、任一条的 `MixerGroup` 为空。
3. **依赖资源模块**：`AudioComponent` 是 `[DefaultExecutionOrder(-400)]`，`ResourceComponent` 是 `-700`，服务根 `AppServiceRoot`（`RootModule`）是 `-32000`，顺序有保证；`AudioService.OnInitialize` 会 `Require<IResourceService>()`，地址播放依赖资源包已初始化（location 规则见 [Resource 模块](../resource/index.md)）。
4. **获取服务**：

```csharp
using AlicizaX;
using AlicizaX.Audio.Runtime;

IAudioService audio = GameApp.Audio;
```

`GameApp.Audio` 是 `AppServices.Require<IAudioService>()` 的静态缓存；初始化早于 `AudioComponent.Awake` 时用 `AppServices.TryGet<IAudioService>(out var audio)` 判空。

## 最短接入

1. 确认 `Entry/Audio` 上同时有 `AudioComponent` 与 `AudioListener`。打开 Inspector 会自动补齐 5 条分组配置，`m_AudioMixer` 为空时绑定框架内置的 `Runtime/Modules/Audio/Resources/AudioMixer.mixer`，再按 `Master/<Name>` 补齐空的 `MixerGroup`。
2. 检查 `Audio Groups` 里每张卡的 `Mixer Group` 都不为空（`Reset All` 会重置默认值并重新绑定）。
3. 音频资源放进被 YooAsset 收集的目录。模板是 `Client/Assets/Bundles/Audios/`，Collector 为 `AddressByFileName` 且开启 `SupportExtensionless` → **地址就是文件名去扩展名**，例如 `Client/Assets/Bundles/Audios/UI_MouseClick_01_A.ogg` 对应 `UI_MouseClick_01_A`。
4. 播放与停止：

```csharp
using AlicizaX;
using AlicizaX.Audio.Runtime;

IAudioService audio = GameApp.Audio;
ulong handle = audio.Play(AudioType.UISound, "UI_MouseClick_01_A");
if (handle == 0UL)
{
    return; // 播放失败
}

// 需要精确控制生命周期时保存句柄
audio.Stop(handle);
```

## 核心用法

### 播放 API 一览

| 方法族 | 定位 | 空间化 | 地址重载 | `AudioClip` 重载 | 异步变体 |
| --- | --- | --- | --- | --- | --- |
| `Play` | 2D | `spatialBlend = 0` | 有 | 有 | 仅地址版 `PlayAsync` |
| `Play3D` | 世界坐标 | 用位置 + 3D 参数 | 有 | 有 | 仅地址版 `Play3DAsync` |
| `PlayFollow` | 跟随目标 | 每帧 `target.TransformPoint(localOffset)` | 有 | 有 | 仅地址版 `PlayFollowAsync` |

- 每个方法族有两档：`(type, path|clip, loop = false, volume = 1f)` 与带 `in AudioSpatialOptions` / `in AudioPlayOptions` 的完整档（跟随族第二个参数换成 `target` + `localOffset`）。三个 Async 变体只是把 `AudioPlayOptions.Async` 置为 `true`；`AudioClip` 重载没有异步变体（`Async` 对 `Clip` 重载无效，直接 `StartClip`）。
- 所有方法返回 `ulong` 句柄，`0UL` 表示失败。**异步地址播放即使随后加载失败也会先返回非 0 句柄**，要用 `IsPlaying(handle)` 判断是否真的在播。
- `PlayFollow` 的 `target` 为 `null` 时直接返回 `0UL`；目标被销毁或失活时该路声音自动停止。

```csharp
// 3D：默认使用分组配置
audio.Play3D(AudioType.Sound, "Skill_Cast_01", transform.position);

// 3D：单次覆盖空间参数
var spatial = new AudioSpatialOptions
{
    Override = true,
    SpatialBlend = 1f,
    MinDistance = 2f,
    MaxDistance = 40f,
    RolloffMode = AudioRolloffMode.Logarithmic
};
audio.Play3D(AudioType.Sound, "Skill_Cast_01", transform.position, false, 1f, spatial, AudioPlayOptions.Default);

// 跟随：localOffset 在目标本地空间
ulong engine = audio.PlayFollow(AudioType.Ambient, "Engine_Loop_01", transform, Vector3.zero, loop: true, volume: 0.7f);
```

### `AudioPlayOptions` 字段

| 字段 | 类型 | 默认 | 行为 |
| --- | --- | --- | --- |
| `Async` | `bool` | `false` | 地址请求走异步加载 |
| `CachePolicy` | `AudioCachePolicy` | `Default` | 该次请求的缓存策略，`Default` 回落到 `AudioServiceConfig.DefaultClipCachePolicy` |
| `Pitch` | `float` | `0` | `<= 0` 时按 `1` 处理 |
| `FadeInSeconds` | `float` | `0` | `> 0` 时从 0 淡入 |
| `FadeOutSeconds` | `float` | `0` | 供 `Stop(handle, fadeout: true)` 使用；`<= 0` 时用内置 0.15s |
| `Priority` | `int` | `0` | `> 0` 时作为播放优先级（上限 256）；否则按 `256 - AudioGroupConfig.SourcePriority` 推算 |

淡入淡出与 Clip 缓存 TTL 都跑在**非缩放时间**上，`Time.timeScale = 0` 时仍会推进。

### `AudioSpatialOptions` 字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `Override` | `bool` | `false`（默认值）时整套 3D 参数取 `AudioGroupConfig`；`true` 时用下面四项 |
| `SpatialBlend` | `float` | `0..1`，越界会被夹紧 |
| `MinDistance` | `float` | `< 0` 会被夹到 `0` |
| `MaxDistance` | `float` | 小于 `MinDistance` 时被抬到 `MinDistance` |
| `RolloffMode` | `AudioRolloffMode` | 衰减曲线 |

### 句柄控制

```csharp
bool playing = audio.IsPlaying(handle);
audio.SetVolume(handle, 0.4f, fadeSeconds: 0.5f);  // 用户音量倍率，与淡入淡出、遮挡音量相乘
audio.Stop(handle, fadeout: true);                  // 用 AudioPlayOptions.FadeOutSeconds
audio.Stop(handle, fadeOutSeconds: 0.3f);           // 显式淡出时长
```

句柄是"索引 + 代际"的 64 位值，声音结束、槽位复用后旧句柄自然失效：`Stop` / `SetVolume` / `IsPlaying` 都返回 `false`。传 `0UL` 一律返回 `false`。

### 分组音量与启停

```csharp
audio.Volume = 0.8f;                              // 写 AudioListener.volume，clamp 0..1
audio.Enable = false;                             // 静音但不停播放
audio.SetCategoryVolume(AudioType.Music, 0.5f);   // clamp 到 0.0001..1，换算 dB 写 Mixer 参数
audio.SetCategoryEnable(AudioType.Voice, false);  // 写 -80dB 并停止该组正在播的声音
audio.Stop(AudioType.Sound, fadeout: true);
audio.StopAll(fadeout: true);
```

- 分组音量的 Mixer 参数名取 `AudioGroupConfig.m_ExposedVolumeParameter`，为空时用内置默认 `SoundVolume`、`UISoundVolume`、`MusicVolume`、`VoiceVolume`、`AmbientVolume`（框架内置 Mixer 已暴露这些参数）。
- 分组音量下限是 `0.0001`（约 -80dB），**不是 0**；真正的"关掉"要用 `SetCategoryEnable(false)`。
- `Enable = false` 只改总音量，正在播的声音仍在播放，恢复后回到之前的 `Volume`。

### 并发上限与抢占

- 每组 `MaxSourceCount` 是并发上限，`InitialSourceCount` 是启动时预建的 `AudioSource` 数量；句柄表容量 = 所有分组 `MaxSourceCount` 之和。
- 组内满载时按播放优先级抢占：取当前最低优先级里最早播放的那一路；若新声音优先级比所有活跃声音都低，本次播放失败并返回 `0UL`。
- `audio.Warmup(type, count)` 只预建 `AudioSource`（数量夹在 `[0, MaxSourceCount]`），**不加载任何 Clip**；非法 `AudioType` 是空操作。

### 缓存与预加载

```csharp
bool ok = audio.Preload("Bgm_Main_01");                     // 默认 AudioCachePolicy.Ttl
audio.PreloadAsync("Bgm_Main_01", AudioCachePolicy.Pin, success => { });
audio.Unload("Bgm_Main_01");                                // 有引用时不摘
audio.ClearCache(force: true);                              // 连 Pin 一起清
```

| 策略 | 行为 |
| --- | --- |
| `Default` | 回落到 `AudioServiceConfig.DefaultClipCachePolicy` |
| `None` | 引用归零后立刻释放，不缓存 |
| `Ttl` | 引用归零后进 LRU，超过 `ClipCacheTtl`（非缩放时间）或容量压力时淘汰 |
| `Pin` | 常驻，不参与 LRU/TTL；只能 `Unload` / `ClearCache(true)` 释放 |

- `AudioServiceConfig`：`ClipCacheCapacity`（默认 128，最小 1）、`ClipCacheTtl`（默认 30 秒）、`DefaultClipCachePolicy`（默认 `Ttl`）。
- 容量满时先淘汰 LRU 头；若槽位全被 Pin 或加载中占满，打 `Audio clip cache is full ...` 警告并播放失败。
- `Unload` / `ClearCache` **不会**摘掉正在播放、正在加载或仍有等待请求的 Clip；`Application.lowMemory` 会自动 `ClearCache()`。`Preload` 在条目加载中返回 `false`，`Unload(address, force: true)` 也只允许取消"只有 preload 等待者"的条目。

### `AudioEmitter` 组件

挂在场景物体上的声明式播放器（菜单 `Game Framework/Audio/Audio Emitter`）。

| 分组 | 字段 | 说明 |
| --- | --- | --- |
| Playback | `m_AudioType` / `m_ClipMode` / `m_Address` / `m_Clip` | 分类（默认 `Ambient`）、`Address` 或 `Clip` 模式与对应资源 |
| | `m_PlayOnEnable` / `m_Loop` / `m_Volume` / `m_StopWithFadeout` | 启用自动播放、循环、音量、停止时淡出 |
| | `m_Async` / `m_CachePolicy` | 地址模式异步加载；播放时透传给 `AudioPlayOptions.CachePolicy` |
| Spatial | `m_FollowSelf` / `m_FollowOffset` | 跟随自身 Transform 及本地偏移 |
| | `m_SpatialBlend` / `m_RolloffMode` / `m_MinDistance` / `m_MaxDistance` | 单次覆盖的 3D 参数 |
| Trigger | `m_UseTriggerRange` / `m_TriggerRange` / `m_TriggerHysteresis` | 按 Listener 距离自动播放/停止，半径与回滞（避免边界抖动） |
| Gizmos | `m_DrawGizmos` / `m_DrawOnlyWhenSelected` / `m_TriggerColor` / `m_MinDistanceColor` / `m_MaxDistanceColor` | 场景里画触发半径与远近衰减球 |

```csharp
emitter.Play();
emitter.Stop();
ulong handle = emitter.Handle;
bool playing = emitter.IsPlaying;
```

`AudioEmitter` 通过 `AppServices.TryGet(out IAudioService ...)` 绑定服务，因此在 `AudioComponent.Awake` 之后启用才有声音；`OnDisable` 会停止播放并解绑。启用 `Use Trigger Range` 后由 Listener 距离驱动，Listener 不存在则停止播放。

### 编辑器调试面板

- `AudioComponent` Inspector（非 Play 可编辑，Play 只读）：`Audio Component`（`Listener`、`Audio Mixer`）、`Clip Cache`（`Capacity`、`TTL Seconds`、`Default Policy` + `Reset`/`Mobile`/`Desktop` 预设）、`Audio Groups`（每个 `AudioType` 一张折叠卡，含 `Type`、`Name`、`Mixer Group`、`Mute`、`Volume`、`Initial Sources`、`Max Sources`、`Volume Param`、`Spatial Blend`、`Doppler`、`Spread`、`Priority`、`Reverb Mix`、`Rolloff`、`Min/Max Distance`、`Occlusion` 细节，以及单卡 `Reset` 和 `Expand All`/`Collapse All`/`Reset All`）。
- `AudioComponent` 运行时面板（Play 模式，250ms 刷新）：`Runtime Service`（初始状态、`Volume`、`Listener`、`Instance Root`、`Sources`、`Active Agents`、`Audio Groups`、`Handle Capacity`、`Clip Cache`、`Cache TTL`、`Default Cache Policy`、策略分布）、`Runtime Groups`（每组 `Active/Capacity`、`ON/OFF`、音量）、`Active Agents`、`Clip Cache`（逐条 `Ref` / `Policy` / 状态）。
- `AudioEmitter` Inspector：`Playback` / `Spatial` / `Trigger` / `Gizmos` 四段，`Clip` 与 `Address` 按模式显隐并给出缺参警告，`Max Distance < Min Distance` 时提示会被抬升。

## 常见配置

`AudioGroupConfig` 字段（`[Serializable]`，序列化名带 `m_` 前缀；`AudioType`、`audioRolloffMode`、`minDistance`、`maxDistance` 无前缀）：

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `m_Name` / `m_MixerGroup` | 空 / 空 | 显示名；`AudioSource.outputAudioMixerGroup`（必填） |
| `m_Mute` / `m_Volume` | `false` / `1` | 启动时是否禁用、初始分组音量 |
| `m_InitialSourceCount` / `m_MaxSourceCount` | `2` / `8` | 预建数量（夹到 `[0, Max]`）；并发上限，同时决定句柄表容量 |
| `m_ExposedVolumeParameter` | 空 | Mixer 暴露参数名，空则用内置名 |
| `m_SpatialBlend` / `m_DopplerLevel` / `m_Spread` | `1` / `1` / `0` | 3D 声源参数 |
| `m_SourcePriority` / `m_ReverbZoneMix` | `128` / `1` | 反向映射播放优先级（`256 - m_SourcePriority`）；混响混合 |
| `m_OcclusionEnabled` / `m_OcclusionMask` | `false` / `~0` | 是否做遮挡（开启后才创建 `AudioLowPassFilter`）；遮挡射线层遮罩 |
| `m_OcclusionCheckInterval` / `m_OcclusionLowPassCutoff` / `m_OcclusionVolumeMultiplier` | `0.12` / `1200` / `0.55` | 检测间隔（最小 0.02）、低通截止频率、遮挡音量倍率 |
| `AudioType` | `Sound` | 该条配置对应的分类 |
| `audioRolloffMode` / `minDistance` / `maxDistance` | `Logarithmic` / `1` / `500` | 衰减曲线与距离（模板里 Music/UISound 为 1/25，Sound 为 2/80） |

`AudioServiceConfig`：`ClipCacheCapacity = 128`、`ClipCacheTtl = 30f`、`DefaultClipCachePolicy = AudioCachePolicy.Ttl`。

## 容易踩的坑

1. Mixer / Listener / 任一条 `MixerGroup` 为空：`AudioService` 构造直接抛 `GameFrameworkException`，`AudioComponent` 注册失败，之后 `GameApp.Audio` 抛 "Service ... is not registered"，而不是返回 `null`。
2. 地址必须是 YooAsset location（模板是**文件名去扩展名**），写 `Assets/Bundles/Audios/xxx.wav` 或磁盘绝对路径都加载不到。
3. `Play` 对未缓存地址是**同步**加载，首帧可能卡顿；热路径用 Async 变体，或提前 `Preload` / `PreloadAsync`。同时记住 `PlayAsync` 返回非 0 句柄不代表已经在响，资源加载失败时句柄仍会返回，用 `IsPlaying(handle)` 校验。
4. `Enable = false` 只是 `AudioListener.volume = 0`，声音仍在播；分类静音要用 `SetCategoryEnable(type, false)`，音量下限也只会到 `0.0001`。
5. `AudioEmitter` 的 `Cache Clip` 与 `CachePolicy` 并存：`m_CachePolicyMigrated` 为 `true` 后，`Cache Clip` 不再参与缓存决策，以 `m_CachePolicy` 为准；要精确控制就在 `AudioPlayOptions.CachePolicy` 或 `Preload` 上显式传。
6. 工程里 `Game.Config.AudioType`（配置表生成的音频格式枚举）与框架的 `AlicizaX.Audio.Runtime.AudioType` 同名，同时 `using` 会冲突；用别名 `using AudioType = AlicizaX.Audio.Runtime.AudioType;`。

## API 速查

```csharp
// IAudioService（GameApp.Audio）：全局与分组
float Volume { get; set; }
bool Enable { get; set; }
float GetCategoryVolume(AudioType type);
void SetCategoryVolume(AudioType type, float value);
bool GetCategoryEnable(AudioType type);
void SetCategoryEnable(AudioType type, bool value);
// 2D：地址版与 Clip 版各有简版 / 带 options 版
ulong Play(AudioType type, string path, bool loop = false, float volume = 1f);
ulong PlayAsync(AudioType type, string path, bool loop = false, float volume = 1f);
ulong Play(AudioType type, string path, bool loop, float volume, in AudioPlayOptions options);
ulong Play(AudioType type, AudioClip clip, bool loop = false, float volume = 1f);
ulong Play(AudioType type, AudioClip clip, bool loop, float volume, in AudioPlayOptions options);
// 3D：同上，位置用 in Vector3，另有 Play3DAsync 地址版
ulong Play3D(AudioType type, string path, in Vector3 position, bool loop = false, float volume = 1f);
ulong Play3DAsync(AudioType type, string path, in Vector3 position, bool loop = false, float volume = 1f);
ulong Play3D(AudioType type, string path, in Vector3 position, bool loop, float volume, in AudioSpatialOptions spatial, in AudioPlayOptions options);
ulong Play3D(AudioType type, AudioClip clip, in Vector3 position, bool loop = false, float volume = 1f);
ulong Play3D(AudioType type, AudioClip clip, in Vector3 position, bool loop, float volume, in AudioSpatialOptions spatial, in AudioPlayOptions options);

// 跟随：同上，第二个参数换成 target + in Vector3 localOffset，另有 PlayFollowAsync 地址版
ulong PlayFollow(AudioType type, string path, Transform target, in Vector3 localOffset, bool loop = false, float volume = 1f);
ulong PlayFollowAsync(AudioType type, string path, Transform target, in Vector3 localOffset, bool loop = false, float volume = 1f);
ulong PlayFollow(AudioType type, string path, Transform target, in Vector3 localOffset, bool loop, float volume, in AudioSpatialOptions spatial, in AudioPlayOptions options);
ulong PlayFollow(AudioType type, AudioClip clip, Transform target, in Vector3 localOffset, bool loop = false, float volume = 1f);
ulong PlayFollow(AudioType type, AudioClip clip, Transform target, in Vector3 localOffset, bool loop, float volume, in AudioSpatialOptions spatial, in AudioPlayOptions options);

// 控制与缓存
bool Stop(ulong handle, bool fadeout = false);
bool Stop(ulong handle, float fadeOutSeconds);
bool SetVolume(ulong handle, float volume, float fadeSeconds = 0f);
bool IsPlaying(ulong handle);
void Stop(AudioType type, bool fadeout);
void StopAll(bool fadeout);
void Warmup(AudioType type, int count);
bool Preload(string address, AudioCachePolicy policy = AudioCachePolicy.Ttl);
void PreloadAsync(string address, AudioCachePolicy policy, Action<bool> completed = null);
bool Unload(string address, bool force = false);
void ClearCache(bool force = false);
```
