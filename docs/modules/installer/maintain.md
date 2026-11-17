# 升级、修复与卸载

安装器只提供“装”和“Normal → Hybrid 升级”，**没有卸载按钮**。这一页说明三种升级的区别、状态异常怎么修、以及手动卸载的完整步骤。

## 适用场景

- 项目从 Normal 转 Hybrid。
- 安装状态显示不对（明明装了却显示未安装，或反过来）。
- 要彻底移除 AlicizaX 模板，换成自己的工程结构。

## 三种“升级”不要混

| 说法 | 实际做法 | 安装器参与吗 |
| :--- | :--- | :--- |
| 升级框架版本 | Package Manager 里更新 `com.alicizax.unity.framework` 等包 | 不参与 |
| Normal → Hybrid | 安装器窗口的 `Upgrade Template` | **参与** |
| 更新模板文件（改了启动脚本等） | 安装器不会覆盖已有文件，需手动比对 `Templates~` | 不参与 |

## Normal 升级到 Hybrid

前置：Unity 2022.3+、已装 URP、已装 `com.code-philosophy.hybridclr`、当前状态是 `NormalTemplate`。

1. Package Manager 安装 HybridCLR 包，等编译完成。
2. 打开 `AlicizaX / Installer`，点 `Refresh Check`。环境面板的 `HybridCLR package` 与 `Hybrid template folder` 都应变为已安装。
3. `Template` 面板只剩 `Hybrid Template` 可选（Normal 已锁定）。
4. 点 `Upgrade Template`，确认弹窗。
5. Hybrid 设置确认框：
   - `Apply`：自动设 IL2CPP、.NET Framework、关闭 Incremental GC。
   - `Manual`：只复制文件，Player Settings 自己设。
   - `Cancel`：中止升级。
6. 等待编译，检查 `Assets/Scripts/Hotfix`、`Assets/HybridCLRGenerate`、`Assets/Bundles/DLL` 是否出现。

升级只补 Hybrid 独有的文件与符号（`ENABLE_HYBRIDCLR`），**不会覆盖** Normal 已有的 `Startup` 脚本。如果这些脚本被改过，升级后仍是你的版本。

## 修复安装状态

### 状态文件丢失或损坏

```text
ProjectSettings/AlicizaXFrameworkInstaller.json
```

删掉它，回到窗口点 `Refresh Check`。安装器会用脚本符号和目录标记重新推断：

- 有 `ENABLE_HYBRIDCLR` 或 `Assets/HybridCLRGenerate` → Hybrid
- 有 `ENABLE_LOG` 或 `Assets/Scripts/Startup` → Normal
- 都没有 → 未安装

### 状态显示已安装但目录被删

`ReadInstallState` 会校验标记目录，对不上就降级。比如 Hybrid 的 `Assets/HybridCLRGenerate` 被删，会退成 Normal；Normal 的标记也没了，就回到未安装。

想让它重新可安装：把缺失的模板目录从 `Packages/com.alicizax.unity.installer/Editor/Installer/Templates~` 手动拷回 `Assets/`，或直接删状态文件后重装。

### 项目不需要模板

在未安装状态下点 `Use Custom`，状态写成 `Custom`，窗口不再提示安装模板，但仍保留后续安装入口。适合已经有自己启动结构的项目。

### 模板文件被改坏

安装器不会覆盖已有文件，只能手动恢复：

1. 在 `Packages/com.alicizax.unity.installer/Editor/Installer/Templates~/NormalTemplate`（或 `HybridTemplate`）里找到对应文件。
2. 拷到 `Assets/` 的同名路径，覆盖损坏文件。
3. 如果改的是 `.asmdef`，覆盖后等 Unity 重新编译。

## 卸载（手动）

安装器没有卸载入口。按下面顺序操作，避免删到业务资源：

1. **备份** `Assets/Bundles`、`Assets/Scenes`、`Assets/Scripts` 下自己的内容。
2. **删模板目录**（确认里面没有你的文件）：
   - `Assets/Scripts/Startup`
   - `Assets/Scripts/Hotfix`（Hybrid）
   - `Assets/HybridCLRGenerate`（Hybrid）
   - `Assets/YooAsset`
   - `Assets/Art/Atlas`
   - `Assets/Scenes/Main.unity`、`Assets/Scenes/Desingn.unity`（只删模板带来的场景）
   - `Assets/Bundles`（**内有资源，删前务必确认**）
3. **移除脚本符号**：Player Settings → Other Settings → Scripting Define Symbols，删掉 `ENABLE_LOG`、`ENABLE_HYBRIDCLR`。
4. **删安装状态**：`ProjectSettings/AlicizaXFrameworkInstaller.json`。
5. **移除包**：Package Manager 里移除 `com.alicizax.unity.installer`、`com.alicizax.unity.framework` 及其他 AlicizaX 包。
6. **清理 OpenUPM 作用域**：`Packages/manifest.json` 的 `scopedRegistries` 里，如果 `https://package.openupm.com` 的 `com.cysharp`、`com.alicizax.unity`、`com.tuyoogame` 没有其他包在用，可以移除。
7. **恢复 Player Settings**：按需把 `allowUnsafeCode`、Scripting Backend、Api Compatibility Level、Incremental GC 改回原值。

### 卸载顺序为什么重要

- 先删 `Assets` 再移除包：包被移除后 `Templates~` 就没了，想恢复只能重新下载。
- 先移除包再删 `Assets`：Unity 编译会因缺类型报一堆错，但不影响删除；只是很难确认哪些文件属于模板。
- `Assets/Bundles` 与 `Assets/Scenes` 可能已被业务使用，先比对模板原始文件再决定。

## 修复清单

| 症状 | 检查 | 处理 |
| :--- | :--- | :--- |
| 窗口显示 `Not initialized` 但项目里有模板 | 状态文件是否存在、标记目录是否被改名 | 删状态文件后 `Refresh Check` |
| 显示 `Normal template` 但 Hybrid 文件也在 | 是否手动拷过 Hybrid 目录 | 以标记为准；需要 Hybrid 就走升级流程 |
| `Template folder is missing` | installer 包是否被移动到别处/裁剪 | 重新添加 installer 包 |
| 升级后 `ENABLE_HYBRIDCLR` 没加 | 是否点了 `Cancel`/失败了 | 手动加符号，或删状态文件重走升级 |
| 升级后打包报 IL2CPP 相关错误 | Hybrid 设置是否选了 `Manual` 却没配 | 按 [项目模板与生成目录](templates.md) 的设置表补齐 |
| `allowUnsafeCode` 被改回 false | 团队设置或其它工具覆盖 | 改回 true，框架代码需要 |

## 容易踩的坑

1. **以为删状态文件等于卸载**：只删 JSON 不会删任何模板文件，下次 `Refresh Check` 还会靠标记推断成已安装。
2. **在 Hybrid 项目上重复点安装**：状态是 Hybrid 时按钮直接禁用，属于保护行为，不是 bug。
3. **手动删 `ENABLE_LOG` 后状态变了**：没有 JSON 时符号是推断依据之一，删符号可能让状态从 Normal 掉回未安装。
4. **把 `Assets/Bundles` 整个删掉**：里面通常已经有打包资源和收集器配置，删了要重建。
5. **卸载后残留 `.meta`**：Unity 会为孤立 `.meta` 报警告，按目录一起删干净。

## 相关页面

- 安装入口与窗口说明：[Installer 安装器](index.md)
- 模板内容：[项目模板与生成目录](templates.md)
- 安装失败排查：[安装失败排查](troubleshooting.md)
