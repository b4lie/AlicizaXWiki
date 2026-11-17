# Installer 安装器

Installer 是 `com.alicizax.unity.installer` 包提供的**项目铺设工具**：一个编辑器窗口，负责补齐 OpenUPM 作用域、安装核心包、把 Normal 或 Hybrid 模板复制进当前 Unity 项目。它不参与运行时，也不做包版本管理。

## 适用场景

- 新建 Unity 项目，要一次性铺好 AlicizaX 的目录结构、启动场景和默认脚本。
- 已有项目想换成 Hybrid（热更）结构。
- 安装状态丢失、需要判断“这个项目到底装没装过模板”。

不适用：给已有项目升级框架版本（用 Package Manager）；删除模板（安装器没有卸载入口，见[升级、修复与卸载](maintain.md)）。

## 使用前提

| 项 | 要求 | 说明 |
| :--- | :--- | :--- |
| Unity | **2022.3 或更新** | 窗口会用版本号做门槛，低于 2022.3 直接禁止安装模板 |
| URP | `com.unity.render-pipelines.universal` 必须已安装 | 窗口强制检查，缺失时模板安装按钮禁用 |
| 网络 | 能访问 OpenUPM（`https://package.openupm.com`）与 GitHub | 安装核心包走 Package Manager |
| HybridCLR | 仅 Hybrid 模板需要 `com.code-philosophy.hybridclr` | 未安装时 Hybrid 选项禁用 |
| 安装器自身 | `https://github.com/AlicizaX/FramworkInstaller.git` | 通过 Package Manager 的 git URL 添加 |

## 最短接入

1. **添加安装器**：Package Manager → `+` → Add package from git URL，填：

    ```text
    https://github.com/AlicizaX/FramworkInstaller.git
    ```

2. **打开窗口**：菜单 `AlicizaX / Installer`。

3. **等作用域就绪**：窗口打开时会检查 `Packages/manifest.json` 里的 OpenUPM 作用域，缺 `com.cysharp`、`com.alicizax.unity`、`com.tuyoogame` 会自动写入并等待 Package Manager 解析。解析完成前不显示主面板。

4. **Install Core**：点击后通过 `Client.Add("com.alicizax.unity.framework")` 安装核心包，完成后弹窗提示，模板入口解锁。

5. **选模板**：
   - `Normal Template`：标准项目结构。
   - `Hybrid Template`：在 Normal 基础上加热更目录与 `ENABLE_HYBRIDCLR`。

6. **Install Template**：确认后把模板复制进 `Assets/`，然后自动设置脚本符号与 Player Settings。

7. **等待编译**，打开模板的启动场景（`Assets/Scenes/Main.unity`）试运行。

## 窗口结构

窗口分四块，自上而下：

| 面板 | 内容 | 判断依据 |
| :--- | :--- | :--- |
| `Environment` | Core 包、URP、HybridCLR、安装状态、Normal/Hybrid 模板目录 | 读 `Packages/manifest.json`、内嵌 `package.json`、`Packages/com.alicizax.unity.installer/Editor/Installer/Templates~` |
| `Core` | 安装核心包 | 未装 Core 时只显示 `Install Core` |
| `Template` | 选择 Normal / Hybrid | 未装 Core 时锁定；已装 Hybrid 时锁定；已装 Normal 时只允许升级 Hybrid |
| `Actions` | `Refresh Check`、`Use Custom`、`Install Template` / `Upgrade Template` | 见下方阻塞条件 |

### 安装模板的阻塞条件

只要命中一条，按钮就禁用并显示原因（文本来自源码）：

| 提示 | 含义 |
| :--- | :--- |
| `OpenUPM scoped registry is still being configured.` | 作用域还没解析完，等几秒重新 `Refresh Check` |
| `Unity version must be 2022.3.x or newer.` | Unity 版本过低 |
| `Install Core before importing templates.` | 还没装 `com.alicizax.unity.framework` |
| `URP package is required before installing AlicizaX framework templates.` | 缺 URP |
| `Hybrid template is already initialized.` | 已是 Hybrid，安装器不再改动 |
| `Normal template is already initialized and cannot be overwritten.` | 已是 Normal，只能升级 Hybrid |
| `Hybrid template requires HybridCLR package.` | 选了 Hybrid 但没装 HybridCLR |
| `Template folder is missing: …` | 安装器包被移动/裁剪，`Templates~` 不存在 |

## 安装核心包

点击 `Install Core` 后：

1. 再次确认 OpenUPM 作用域；缺失就先写 `manifest.json` 并中断本次安装。
2. 调 `Client.Add("com.alicizax.unity.framework")`，由 Package Manager 解析依赖（YooAsset、ZString、UniTask）。
3. 完成后 `AssetDatabase.Refresh()`，重新检测环境并提示可以装模板。

失败时弹窗显示 Package Manager 的错误信息，环境面板保持原状态。

## 安装模板时实际做了什么

按顺序四步（源码 `InstallSelectedTemplate`）：

1. **复制文件**：把 `Templates~/NormalTemplate` 或 `Templates~/HybridTemplate` 下的目录与文件复制到 `Assets/`。
   - 跳过 `.keep` / `.gitkeep` 占位文件。
   - **目标已存在的文件不会覆盖**，只在 Console 打 `AlicizaX installer skipped existing file: …`。
2. **Player Settings**：把 `PlayerSettings.allowUnsafeCode` 设为 `true`（框架运行时代码需要）。
3. **脚本符号**：追加 `ENABLE_LOG`；Hybrid 再追加 `ENABLE_HYBRIDCLR`。
4. **写安装状态**：`ProjectSettings/AlicizaXFrameworkInstaller.json`。

Hybrid 还会额外弹一次确认框，询问是否应用热更所需的 Player Settings（IL2CPP、.NET Framework、关闭 Incremental GC）。选 `Manual` 只复制文件不改进设置；选 `Cancel` 则整个安装中止。

## 安装状态文件

`ProjectSettings/AlicizaXFrameworkInstaller.json`：

```json
{
  "installerState": "NormalTemplate",
  "template": "Normal",
  "unityVersion": "2022.3.62f1",
  "projectPath": "D:/Projects/MyGame",
  "updatedAt": "2026-11-17 10:30:00"
}
```

- `installerState` 取值：`NotInstalled`、`Custom`、`NormalTemplate`、`HybridTemplate`。
- 读取时会做二次校验：Normal 必须还能找到 `Assets/Scripts/Startup`、`Assets/Bundles`、`Assets/YooAsset` 之一；Hybrid 必须能找到 `Assets/Scripts/Hotfix`、`Assets/HybridCLRGenerate`、`Assets/Bundles/DLL` 之一。对不上就降级或回到 `NotInstalled`。
- 状态文件丢失时，安装器会用脚本符号（`ENABLE_LOG` / `ENABLE_HYBRIDCLR`）和上面这些目录做兼容性推断。
- `Custom` 由 `Use Custom` 写入，表示“这个项目不需要模板”，之后仍可再装。

## 边界：Installer 不管什么

| 不属于 Installer | 应该去哪里 |
| :--- | :--- |
| 框架版本升级、依赖冲突 | Unity Package Manager |
| 运行时启动链路（RootModule、服务容器） | [Service 服务容器](../service/index.md) |
| 启动流程与热更程序集加载 | [Procedure 流程](../procedure/index.md) |
| 配置表生成 | [Config 配置管线](../config/index.md) |
| 多语言词库 | [Localization 本地化](../localization/index.md) |

安装器只在编辑器里跑，程序集 `AlicizaX.Installer.Editor` 仅 Editor 平台、无运行时引用。

## 容易踩的坑

1. **先点了模板再装 Core**：模板面板在未装 Core 时是锁定的，不要试图用旧版本窗口绕过；先 `Install Core`。
2. **在已有工程上重装模板**：已存在的文件一律跳过，所以“重装”不会修复被改坏的启动脚本，也不会覆盖你的业务代码。
3. **Unity 版本 2021**：窗口直接拒绝安装模板，升级 Unity 后再来。
4. **Hybrid 只选了 Manual 设置**：文件装好了但后端仍是 Mono，构建热更时会失败；按 [升级、修复与卸载](maintain.md) 补设置。
5. **删掉状态文件又删了 `Assets/Bundles`**：安装器会判定为未安装，再次安装时旧文件已存在会被跳过，导致结构半新半旧。

## 相关页面

- 模板内容与生成目录：[项目模板与生成目录](templates.md)
- 升级、修复、卸载：[升级、修复与卸载](maintain.md)
- 安装失败：[安装失败排查](troubleshooting.md)
