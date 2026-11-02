---
hide:
  - navigation
  - toc
---

<div class="hero" markdown>

![AlicizaX](assets/logo.png){ width="200" }

# 面向商业化项目的 Unity 游戏框架

服务容器、资源管线、UI 管理、事件与计时、对象池、热更——收敛成一套装好即用的骨架。

[开始阅读](QuickStart.md){ .md-button .md-button--primary }
[框架仓库](https://github.com/AlicizaX/AlicizaXTemplate){ .md-button }

</div>

## 快速入口

<div class="grid cards" markdown>

-   :material-rocket-launch-outline:{ .lg .middle } __快速入门__

    ---

    启动链路、场景根节点、资源初始化、打开第一个 UI。

    [:octicons-arrow-right-24: QuickStart](QuickStart.md)

-   :material-puzzle-outline:{ .lg .middle } __核心服务__

    ---

    服务容器、四级时间轮、零 GC 事件总线、流程调度。

    [:octicons-arrow-right-24: Service](Service.md)

-   :material-package-variant-closed:{ .lg .middle } __资源与对象__

    ---

    基于 YooAsset 的资源租约、场景加载、实例池与内存池。

    [:octicons-arrow-right-24: Resources](Resources.md)

-   :material-window-restore:{ .lg .middle } __UI 与表现__

    ---

    窗口栈与层级、UIRouter、Widget / Tab、音频与本地化。

    [:octicons-arrow-right-24: UI](UI.md)

</div>

## 核心能力

| 模块 | 核心价值 | 亮点特性 |
| :--- | :--- | :--- |
| **Event** 事件系统 | 0 GC 高频通信 | 无堆分配设计 · 自动解绑 · 主线程同步派发 |
| **Timer** 时间轮 | 极性能计时 | 四级时间轮算法 · 无全量扫描 · 技能 CD / 心跳 |
| **MemoryPool** 内存池 | 低 GC 引用管理 | O(1) 环形缓存 · 近期峰值窗口回收 · 软硬上限预算 |
| **GameObjectPool** 实例池 | 全生命周期管理 | Fixed / Burst / Sticky 策略 · 到期维护堆 |
| **Resources** 资源服务 | 智能缓存策略 | 基于 YooAsset · 资源租约 · 热资源保活 |
| **HybridCLR** 热更 | 无缝 C# 热更 | AOT / Hotfix 分层 · 反射调用示例 |
| **UI** 界面系统 | 高效窗口管理 | 窗口栈与层级 · Holder 自动生成 · 逻辑视图分离 |
| **RecyclerView** 虚拟列表 | 海量数据承载 | 可见区增量刷新 · 循环 / 分组 / 网格 / 分页 |
| **UI Extension** UI 扩展 | 开箱即用组件 | 封装 Button / Switch / Image · 拖拽导航 |
| **Navigation** 导航热键 | 多设备输入 | New Input System · 拓扑感知 · 顶层焦点栈 |

## 文档怎么读

每篇模块文档都按同一套结构组织，可以按需要跳读：

- **使用前提** —— 需要哪些组件、要不要先初始化
- **用法** —— 从获取服务到完整示例，按调用顺序排列
- **API 速查** —— 一屏看完常用方法签名
- **注意事项** —— 边界条件、性能陷阱、已废弃接口

建议的阅读顺序：先读 [QuickStart 快速入门](QuickStart.md) 跑通启动流程，再按项目需要挑模块——做战斗看 **Timer / Event / GameObjectPool**，做界面看 **UI / RecyclerView**，做资源与热更看 **Resources / HybridCLR**。

!!! note "关于本文档"

    本站在内容上与框架仓库 `AlicizaXTemplate/Books/` 保持 1:1 一致，仅将两处已失效的链接降级为纯文本。
    发现文档问题请到 [文档仓库](https://github.com/b4lie/AlicizaXWiki/issues) 提 issue，框架本身的问题请提到 [AlicizaXTemplate](https://github.com/AlicizaX/AlicizaXTemplate/issues)。
