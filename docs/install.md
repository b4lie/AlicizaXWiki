# 安装与可选扩展包

AlicizaX 用 **Framework Installer** 统一安装：它负责补 OpenUPM Scoped Registry、装核心包、导入模板，不用手动一个个加 Git URL。

## 安装框架

### 1. 添加 Installer

Unity **Package Manager** → `+` → **Add package from git URL**：

```text
https://github.com/AlicizaX/FramworkInstaller.git
```

### 2. 打开安装器

菜单栏 **`AlicizaX / Installer`**。安装器会自动检测并补齐所需的 OpenUPM Scoped Registry 和 Scopes。

### 3. 安装核心库

点 **`Install Core`**，安装：

```text
com.alicizax.unity.framework
```

装好后模板入口才会解锁。

### 4. 选择模板

| 模板 | 适用场景 |
| :--- | :--- |
| **Normal** | 标准项目开发 |
| **Hybrid** | 混合开发模式（含热更工程） |

### 5. 按需加扩展包

见下表。UI 扩展、调试面板、多语言都是独立的 UPM 包，用不到可以不装。

## 可选扩展包

| 包名 | 版本 | 用途 | 装不装 |
| :--- | :--- | :--- | :--- |
| [`com.alicizax.unity.framework`](https://github.com/AlicizaX/AlicizaXTemplate) | 2.3.6 | 核心框架：服务容器、资源、UI、事件、计时、对象池 | **必装** |
| [`com.alicizax.unity.ui.extension`](https://github.com/AlicizaX/com.alicizax.unity.ui.extension) | 3.0.1 | UI 扩展组件（UXButton / UXImage / RecyclerView 等） | 有 UI 就装 |
| [`com.alicizax.unity.editor.extension`](https://github.com/AlicizaX/com.alicizax.unity.editor.extension) | 2.1.0 | 编辑器扩展工具（Aliciza UI Toolkit 控件、快捷面板） | 推荐 |
| [`com.alicizax.unity.input`](https://github.com/AlicizaX/com.alicizax.unity.input) | 3.0.1 | 输入扩展：设备监听、快捷键、UI 导航、按键图标、震动 | 需要手柄/键盘导航时 |
| [`com.alicizax.unity.debugger`](https://github.com/AlicizaX/com.alicizax.unity.debugger) | 2.1.5 | 运行时调试面板：日志、资源、内存池、音频、计时器、Event Monitor | 开发期推荐 |
| [`com.alicizax.unity.localization`](https://github.com/AlicizaX/com.alicizax.unity.localization) | 1.0.1 | 基于 Luban 的多语言运行时、TMP 绑定与编辑器预览 | 多语言项目 |
| [`com.alicizax.unity.installer`](https://github.com/AlicizaX/FramworkInstaller) | 1.0.0 | 安装器本身：装核心包、导入模板 | 安装期需要 |

依赖关系要注意两点：

- `com.alicizax.unity.input` 依赖 Unity **Input System**，装之前先启用该包，否则输入类扩展（`Navigation`、`HotkeyComponent`、`InputGlyph`）不可用；
- 需要手柄/键盘导航的列表（`RecyclerView` 的 `UXNavigationScope`）同样依赖 input 包，只做触屏可以跳过。

## 相关文档

- [快速入门](quickstart.md)：装完之后的启动链路
- [UI Extension 概览](extension/index.md)：扩展包内各组件的文档索引
- [UI 模块](modules/ui/index.md)：窗口、路由、转场
