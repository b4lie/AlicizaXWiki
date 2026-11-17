# 安装失败排查

按安装阶段查：窗口打不开 → 环境检测不通过 → 核心包装不上 → 模板装不上 → 装完编译报错。

## 适用场景

- 安装器窗口只显示一个错误框，或一直停在 “AlicizaX Installer”。
- 环境面板某一行是红的，安装按钮点不了。
- 模板提示安装成功，但场景跑不起来。

## 阶段一：窗口启动

窗口打开时会先补齐 OpenUPM 作用域，再等 Package Manager 解析。

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| 顶部只有 `AlicizaX Installer` 和一个错误框 | 写 `Packages/manifest.json` 失败 | 看错误文本；确认 `manifest.json` 存在且可写（版本控制未锁定、磁盘未满） |
| `Package manifest is missing: Packages/manifest.json` | 不在有效 Unity 项目根目录打开，或 manifest 被删 | 用 Unity Hub 打开正确项目；从版本控制恢复 manifest |
| `Package Manager resolve failed: …` | 写入了作用域但 Package Manager 解析失败（网络/DNS/代理） | 修好网络后点 `Retry`；必要时手动在 Package Manager 里刷新 |
| 一直显示 `OpenUPM scoped registry was updated. Waiting for Package Manager to resolve...` | 解析还没结束 | 等几秒；长时间不动就关掉窗口重开，或检查 Editor.log |

作用域要求（源码常量）：

```text
registry: https://package.openupm.com
scopes:   com.cysharp
          com.alicizax.unity
          com.tuyoogame
```

`manifest.json` 里已有同名 registry 但 URL 或 scope 不全时，安装器会改写；如果你们的规范不允许自动改 manifest，先手工补齐再打开窗口。

## 阶段二：环境检测

`Environment` 面板每行对应一个检查项：

| 行 | 红色时含义 | 处理 |
| :--- | :--- | :--- |
| `Core package` | `Packages/manifest.json` 里没有 `com.alicizax.unity.framework`，也没有内嵌包 | 点 `Install Core`，或手动在 Package Manager 安装 |
| `URP package` | 缺 `com.unity.render-pipelines.universal` | Package Manager 安装 URP |
| `HybridCLR package` | 缺 `com.code-philosophy.hybridclr` | 只在需要 Hybrid 时安装 |
| `Installer state` | 状态文件与目录标记对不上 | 见[升级、修复与卸载](maintain.md)的修复章节 |
| `Normal template folder` | 包内 `Templates~/NormalTemplate` 不存在 | 重新安装 installer 包 |
| `Hybrid template folder` | 包内 `Templates~/HybridTemplate` 不存在 | 同上 |

`Unity version must be 2022.3.x or newer.` 说明当前编辑器版本低于 2022.3，升级 Unity 后再装。

## 阶段三：安装核心包

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| 弹窗 `Failed to install Core: …` | Package Manager 解析失败：网络、磁盘、包版本冲突 | 按弹窗文本处理；也可以在 Package Manager 里手动 Add package by name `com.alicizax.unity.framework` |
| 点 `Install Core` 没反应 | 正在安装中（按钮变 `Installing Core...`），或又在补作用域 | 等安装结束；补作用域后会中断本次点击，需要再点一次 |
| 装完仍显示 `Not installed` | manifest 没写入成功，或包被解析到别的路径 | 打开 `Packages/manifest.json` 确认依赖存在；点 `Refresh Check` |

## 阶段四：安装模板

按钮禁用时，窗口会显示具体原因，对照表见 [Installer 安装器](index.md) 的“安装模板的阻塞条件”。

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| `Template folder is missing: …` | installer 包被移动、或被包管理器缓存成裁剪版本 | 移除后重新添加 installer 包 |
| 点安装后 Console 大量 `AlicizaX installer skipped existing file: …` | 目标项目已有同名文件，安装器不覆盖 | 正常保护行为；确认旧文件是否要保留，必要时先删除再装 |
| 弹窗后什么都没发生 | 在确认框点了 `Cancel`，或 Hybrid 设置框点了 `Cancel` | 重新点安装 |
| 安装“成功”但 `Assets/Scripts/Startup` 不存在 | 安装被异常中断，或模板目录不完整 | 检查安装器包完整性；从 `Templates~` 手动拷贝 |
| 状态文件写入失败 | `ProjectSettings` 只读/被占用 | 关闭占用程序，重试；或手动创建该 JSON（见[项目模板与生成目录](templates.md)） |

## 阶段五：安装之后

| 现象 | 原因 | 处理 |
| :--- | :--- | :--- |
| 编译报缺 `RootModule` / `GameApp` | 核心包没装成功或程序集未刷新 | 确认 `com.alicizax.unity.framework` 在 manifest 中；`Assets > Reimport All` |
| 编译报 HybridCLR 类型缺失 | 选了 Hybrid 但没装 HybridCLR | 安装 `com.code-philosophy.hybridclr`，或改用 Normal |
| 打包报 IL2CPP / Api Compatibility 错误 | Hybrid 设置选了 `Manual` 但没配 | 按 [项目模板与生成目录](templates.md) 的设置表补齐 |
| 运行时报缺配置/本地化资源 | 模板只给了目录，还没有生成数据 | 先跑 Config 管线（[Config 配置管线](../config/index.md)），再打资源包 |
| 启动场景没有 `InputService` | 模板场景只含 `ProcedureEntry` | 按 [Input 输入](../input/index.md) 自行挂载 |
| Console 警告 `Failed to read AlicizaX installer state: …` | 状态 JSON 损坏 | 删掉 `ProjectSettings/AlicizaXFrameworkInstaller.json` 后 `Refresh Check` |

## 手动恢复

```text
1. 关掉安装器窗口
2. 删除 ProjectSettings/AlicizaXFrameworkInstaller.json
3. 检查 Packages/manifest.json：
   - dependencies 里有 com.alicizax.unity.framework
   - scopedRegistries 里有 https://package.openupm.com 与三个 scope
4. 重开 AlicizaX / Installer
5. 点 Refresh Check
```

如果状态被推断成“已安装”导致不能重装：按[升级、修复与卸载](maintain.md)删掉对应的标记目录（`Assets/Scripts/Startup` 等），或直接把缺失的模板文件补齐。

## 容易踩的坑

1. **在包管理器还在解析时反复点按钮**：会中断当前流程，等状态稳定再操作。
2. **项目开启了只读文件保护**：`manifest.json` 与 `ProjectSettings` 必须可写，否则安装器无法记录状态。
3. **手动改了包内 `Templates~`**：Package Manager 更新会覆盖；而且 `Template folder` 检查会通过，但内容已不是官方版本。
4. **只看 Console 最后一行**：模板跳过的文件是 `LogWarning`，真正失败通常是弹窗或 `Debug.LogError`。
5. **用旧版 installer 装新版 framework**：包版本不匹配时先更新 installer，再重装模板。

## 相关页面

- 安装流程与窗口：[Installer 安装器](index.md)
- 模板内容与设置：[项目模板与生成目录](templates.md)
- 升级/修复/卸载：[升级、修复与卸载](maintain.md)
