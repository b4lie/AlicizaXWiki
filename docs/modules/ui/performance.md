# 性能与缓存

UI 的性能主要取决于三件事：多少视图在每帧更新、多少窗口留在缓存、每帧产生多少托管分配。先按使用方式控制这三项，再用 Profiler 验证。

## 只让需要的视图更新

`OnUpdate` 只对**重写了它**的窗口/Widget 生效，不需要额外标签：

```csharp
[Window(UILayer.UI, UICacheTime.None)]
public sealed class BattleHudWindow : UIWindow<ui_BattleHudWindow>
{
    protected override void OnUpdate(float deltaTime)
    {
        RefreshHpBar(deltaTime);
    }
}
```

- 没有重写 `OnUpdate` 的视图不会进入更新列表，不会产生每帧调用。
- 视图处于 `Opening` / `Opened` 且父级启用时才更新；`Closed`、`Closing`、父窗口隐藏时会暂停。
- 更新列表按视图进入顺序执行，不要依赖 Widget 之间的执行顺序。
- `OnUpdate` 内避免 `new`、LINQ、字符串拼接、闭包和装箱；高频文本改用缓存或 `SetText` 复用。

## 缓存窗口要设上限

`[Window]` 的 `cacheTime`：

| 值 | 行为 |
| --- | --- |
| `UICacheTime.None`（0） | 关闭即销毁 |
| `UICacheTime.Forever`（-1） | 常驻缓存 |
| 正数 | 关闭转场结束后按秒计时，到期销毁 |

建议：

- 高频开关的弹窗用正数缓存，不要全部 `Forever`。
- 大界面或带大量贴图的窗口优先 `None`，避免缓存长期占内存。
- 缓存窗口被再次 `Show` 时会清掉缓存计时并重新打开；不会重新执行 `OnInitialize`。
- 需要彻底回收时用 `CloseUI<T>(force: true)` 或在 Inspector 的 `Cached Windows` 中确认缓存数量。

## 避免每帧分配

- 事件订阅放在 `OnInitialize`，用 `EventProxy.AddUIEvent`；视图隐藏时自动暂停，不需要每帧判断。
- 不要在 `OnUpdate` 里申请临时集合、创建委托或拼接 UI 文本。
- 打开/关闭是低频操作；不要每帧调用 `ShowUI` / `CloseUI` 来模拟状态切换。
- 需要等待动画时使用 `AwaitTransition`；不需要结果时不要在每个高频路径上 `await` 关闭任务。
- 页签和 Widget 优先复用实例；不要每次打开都重新创建 Prefab。

## 层级与批次

- 每层最多 200 个窗口，超过会在编辑器中断言；打开大量弹窗时要拆分或及时关闭。
- 窗口的 `Canvas.sortingOrder` 由 UI 服务维护，不要手动修改。
- 同层 UI 混用多套材质或图集会增加批次；合并图集、复用材质通常比微调代码更有效。

## 隐藏窗口不会停止组件

窗口隐藏时只关闭 `Canvas` / `CanvasGroup` 的可见性，节点上的 `MonoBehaviour` 仍在运行，也不会触发 `OnEnable` / `OnDisable`。因此：

- 带 `Update`、粒子、Animator 或视频的组件，要在 `OnClose` 停止、`OnOpen` 恢复。
- 不要把“窗口不可见”当成“对象已禁用”。
- 需要彻底停止时，Widget 用 `Destroy()`，窗口用 `CloseUI(force: true)`。

## 用工具确认

- Play Mode 选中 `UIComponent`，查看 `UI RuntimeDebug` 的 Open / Cache / Update / Block 数量和缓存列表。
- `Navigation RuntimeDebug` 查看 history；history 不应无限增长。
- Profiler 中观察 `Canvas.BuildBatch`、`Canvas.SendWillRenderCanvases` 和 UI 相关 GC 分配。
- 框架的 UI 测试位于 `Client/Packages/com.alicizax.unity.framework/Tests/UI`，修改热路径后先跑相关测试，再在目标平台复测。

## 注意事项

1. 缓存不是越多越好；缓存窗口仍然占用内存，也会出现在 Inspector 的 Cached Windows 中。
2. 不要把窗口当作高频数据容器；列表数据用 RecyclerView，定期刷新的内容放在 Widget 里按需更新。
3. 只重写真正需要的 `OnUpdate`，不要为了“以后可能用到”给所有窗口加重写。
4. 转场的 alpha 预设需要 `CanvasGroup`；窗口的 Canvas 显隐方式不同，优先使用位置/缩放转场。
