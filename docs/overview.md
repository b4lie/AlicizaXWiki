# 概览

AlicizaX 把常用能力拆成 11 个模块：服务容器承住全局依赖，资源与对象池扛住加载和复用，UI 与表现负责界面、音频和文案。每个模块都可以单独接入，也可以按 [快速开始](quickstart.md) 的路径一次性跑通。

## 核心能力

| 模块 | 核心价值 | 亮点特性 |
| :--- | :--- | :--- |
| [**Event** 事件系统](modules/event/index.md) | 0 GC 高频通信 | 无堆分配设计 · 自动解绑 · 主线程同步派发 |
| [**Timer** 时间轮](modules/timer/index.md) | 极性能计时 | 四级时间轮算法 · 无全量扫描 · 技能 CD / 心跳 |
| [**MemoryPool** 内存池](modules/memorypool/index.md) | 低 GC 引用管理 | O(1) 环形缓存 · 近期峰值窗口回收 · 软硬上限预算 |
| [**GameObjectPool** 实例池](modules/gameobjectpool/index.md) | 全生命周期管理 | Fixed / Burst / Sticky 策略 · 到期维护堆 |
| [**Resources** 资源服务](modules/resource/index.md) | 智能缓存策略 | 基于 YooAsset · 资源租约 · 热资源保活 |
| **HybridCLR** 热更 | 无缝 C# 热更 | AOT / Hotfix 分层 · 反射调用示例 |
| [**UI** 界面系统](modules/ui/index.md) | 高效窗口管理 | 窗口栈与层级 · Holder 自动生成 · 逻辑视图分离 |
| **RecyclerView** 虚拟列表 | 海量数据承载 | 可见区增量刷新 · 循环 / 分组 / 网格 / 分页 |
| **UI Extension** UI 扩展 | 开箱即用组件 | 封装 Button / Switch / Image · 拖拽导航 |
| **Navigation** 导航热键 | 多设备输入 | New Input System · 拓扑感知 · 顶层焦点栈 |

## 三步上手

<div class="steps" markdown>

1. **跑通启动链路**：打开 [快速开始](quickstart.md)，从 RootModule、场景根节点到最小启动流程走一遍，先让第一个界面亮起来。

2. **按需求挑模块**：每篇文档都是「使用前提 → 用法 → API 速查 → 注意事项」四段式，先确认前提组件挂上了，再抄示例。

3. **接入自己的流程**：注册自定义服务、打开业务 UI、申请资源租约；需要热更时再按 HybridCLR 的流程接。

</div>
