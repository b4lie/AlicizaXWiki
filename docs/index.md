---
hide:
  - navigation
  - toc
---

<div class="hero" markdown>

<div class="hero-brand">
  <img class="only-light" src="assets/logo.png" alt="AlicizaX">
  <img class="only-dark" src="assets/logo-light.png" alt="AlicizaX">
</div>

# 面向商业化项目的 Unity 游戏框架

服务容器、资源管线、UI 管理、事件与计时、对象池、热更——收敛成一套装好即用的骨架。

[快速开始](QuickStart.md){ .md-button .md-button--primary }
[框架仓库](https://github.com/AlicizaX/AlicizaXTemplate){ .md-button }

<div class="hero-meta">
  <span><b>11</b> 个模块</span>
  <span><b>145</b> 段示例代码</span>
  <span><b>0 GC</b> 高频事件</span>
</div>

</div>

## 快速入口

<div class="grid cards" markdown>

-   :material-rocket-launch-outline:{ .lg .middle } **快速入门**

    ---

    从 RootModule 启动到打开第一个 UI，跑通最小链路。

    [:octicons-arrow-right-24: QuickStart](QuickStart.md)

-   :material-cog-outline:{ .lg .middle } **核心服务**

    ---

    服务容器、四级时间轮、零 GC 事件总线、流程调度。

    [:octicons-arrow-right-24: Service](Service.md)

-   :material-cube-outline:{ .lg .middle } **资源与对象**

    ---

    基于 YooAsset 的资源租约、场景加载、实例池与内存池。

    [:octicons-arrow-right-24: Resources](Resources.md)

-   :material-window-restore:{ .lg .middle } **UI 与表现**

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

## 三步上手

<div class="steps" markdown>

1. **跑通启动链路**：打开 [QuickStart](QuickStart.md)，从 RootModule、场景根节点到最小启动流程走一遍，先让第一个界面亮起来。

2. **按需求挑模块**：每篇文档都是「使用前提 → 用法 → API 速查 → 注意事项」四段式，先确认前提组件挂上了，再抄示例。

3. **接入自己的流程**：注册自定义服务、打开业务 UI、申请资源租约；需要热更时再按 HybridCLR 的流程接。

</div>

## 相关链接

<div class="grid cards links" markdown>

-   :octicons-repo-16: **框架源码**

    ---

    `AlicizaX/AlicizaXTemplate` —— 源码、Issues、Release 都在这里。

    [:octicons-arrow-right-24: 前往仓库](https://github.com/AlicizaX/AlicizaXTemplate)

-   :octicons-book-16: **文档仓库**

    ---

    本站的 Markdown 源文件，右上角「编辑此页」会跳到这里。

    [:octicons-arrow-right-24: 查看源码](https://github.com/b4lie/AlicizaXWiki)

-   :octicons-issue-opened-16: **问题反馈**

    ---

    文档写错了、示例跑不通、API 用法有疑问，都欢迎提 issue。

    [:octicons-arrow-right-24: 提个 issue](https://github.com/AlicizaX/AlicizaXTemplate/issues)

</div>

!!! note "关于本文档"

    模块文档与框架仓库 `AlicizaXTemplate/Books/` 同源，仅将两处已失效的链接降级为纯文本。
    站点使用 MkDocs + Material for MkDocs 构建，支持中英文混合检索。
