# 加载与绑定

新代码一律用 `LoadLease` / `LoadLeaseAsync`。租约是一次明确的所有权；`Dispose()` 只交还所有权，不会立刻把资源从内存里抠掉，引用归零后走 Idle TTL。

```csharp
using AlicizaX;
using AlicizaX.Resource.Runtime;
using UnityEngine;

Texture2D icon;

using (AssetLease<Texture2D> lease = GameApp.Resource.LoadLease<Texture2D>("icon_start"))
{
    icon = lease.Asset;
}
```

异步版本建议传 `CancellationToken`：

```csharp
using System.Threading;
using AlicizaX;
using AlicizaX.Resource.Runtime;
using Cysharp.Threading.Tasks;
using UnityEngine;

public sealed class LoadSpriteExample : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private AssetLease<Sprite> _avatar;

    private async UniTaskVoid OnEnable()
    {
        _cts = new CancellationTokenSource();
        _avatar = await GameApp.Resource.LoadLeaseAsync<Sprite>("avatar_default", _cts.Token);
    }

    private void OnDisable()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;

        if (_avatar.IsValid)
        {
            _avatar.Dispose();
        }
    }
}
```

需要复用一个地址时，先取 `AssetRef<T>`，再反复按 key 加载，省掉字符串查表：

```csharp
AssetRef<Sprite> key = GameApp.Resource.Key<Sprite>("avatar_default");
AssetLease<Sprite> lease = GameApp.Resource.LoadLease(key);
```

同一地址正在异步加载时，同步 `LoadLease` 会加入同一个 Provider，最终只保留一条记录。

## 作用域加载

一批资源如果同生共死（一次战斗、一个界面），用作用域比逐个 `Dispose` 更省心：

```csharp
using AlicizaX.Resource.Runtime;
using UnityEngine;

using ResourceScope scope = GameApp.Resource.CreateScope("Battle");

var atlas = scope.Load<Texture2D>("battle_atlas");
var config = await scope.LoadAsync<TextAsset>("battle_config");
// 离开 using 时，scope 内所有条目一起释放
```

`GameApp.Resource.App` 是进程级的全局作用域，框架自身也挂在上面。

## 加载并实例化 Prefab

`Instantiate()` / `InstantiateAsync()` 加载 Prefab 并实例化。实例会自动挂载或复用 `ResourceOwner`，由它持有源 Prefab 的引用；`Destroy(instance)` 时自动解除。

```csharp
using AlicizaX;
using UnityEngine;

GameObject window = GameApp.Resource.Instantiate("UIHomeWindow", parent);
```

```csharp
GameObject window = await GameApp.Resource.InstantiateAsync("UIHomeWindow", parent, cancellationToken);
```

两个方法在服务未就绪、取消或加载失败时返回 `null`，调用方要判空。

对象池不要走这两条 API。`GameObjectPool` 内部用 `LoadLease<GameObject>` 持有源 Prefab，按池策略 `Dispose`。

## 组件资源绑定

绑定服务把资源绑到组件属性上，生命周期交给 `ResourceOwner`。物体销毁或重新绑定时自动释放，不需要手动 `Dispose`。

```csharp
using AlicizaX.Resource.Runtime;
using UnityEngine.UI;

image.SetSprite("icon_start", ResourceBindingOptions.SetNativeSize);
image.SetSubSprite("CommonAtlas", "btn_start");
image.SetMaterial("ui_mask");

spriteRenderer.SetSprite("player");

renderer.SetSharedMaterial("shared");
renderer.SetMaterialInstance("role", ResourceBindingOptions.Synchronous);
```

可选项 `ResourceBindingOptions`（可组合）：

- `None`：默认，异步加载。
- `SetNativeSize`：绑定成功后对 `Image` 调 `SetNativeSize()`。
- `Synchronous`：要求同步完成绑定。

绑定槽类型：

- `ImageSprite`：绑定 `Image.sprite`。
- `SubSprite`：从图集子资源取 Sprite，绑到 `Image.sprite` 或 `SpriteRenderer.sprite`。
- `SpriteRendererSprite`：绑定 `SpriteRenderer.sprite`。
- `ImageMaterial`：绑定 `Image.material`。
- `RendererSharedMaterial`：绑定 `Renderer.sharedMaterial`。
- `RendererMaterialInstance`：从材质创建运行时实例，绑到 `Renderer.sharedMaterial`。
- `PrefabSource`：Prefab 实例和源 Prefab 之间的绑定。

需要显式回收时，销毁挂 `ResourceOwner` 的物体，或直接 `owner.Scope.Dispose()`。
