# 项目模板与生成目录

安装器带两套模板：`NormalTemplate` 与 `HybridTemplate`。这一页列出它们复制进项目后的真实目录、改了哪些项目设置，以及重复安装时的行为。

## 适用场景

- 想知道“装完模板后 `Assets/` 里多了什么”。
- 准备手动补文件，先确认哪些目录归模板管。
- 对比 Normal 与 Hybrid 的差异，决定用哪套。

## 模板来源

```text
Packages/com.alicizax.unity.installer/Editor/Installer/Templates~
├─ NormalTemplate
└─ HybridTemplate
```

`Templates~` 以 `~` 结尾，Unity 不会把它当资源导入，所以模板不会被编译进项目。安装时按相对路径复制到当前项目的 `Assets/` 下。

## 生成目录

Normal 模板复制后：

```text
Assets
├─ Art
│  └─ Atlas
├─ Bundles
│  ├─ Audios
│  ├─ Configs
│  │  ├─ bytes           # empty.txt 占位，运行时由 Config 管线写入
│  │  ├─ json
│  │  └─ sciptableObject  # 目录名就是源码里的拼写
│  ├─ Effects
│  ├─ Entity
│  ├─ Materials
│  ├─ Scenes
│  ├─ Shaders
│  ├─ ShaderVariants
│  ├─ UI
│  └─ UIRaw
│     ├─ Atlas
│     └─ Raw
├─ Scenes
│  ├─ Main.unity          # 启动场景
│  └─ Desingn.unity       # 场景名沿用模板拼写
├─ Scripts
│  └─ Startup
│     ├─ ProcedureEntry.cs
│     ├─ StartupSetting.cs
│     ├─ UpdateProgressUtils.cs
│     ├─ Unity.Startup.asmdef
│     ├─ Event
│     │  └─ AssetDownloadProgressUpdateEventArgs.cs
│     └─ Procedure
│        ├─ ProcedureEntryState.cs
│        ├─ ProcedureGetAppVersionInfoState.cs
│        ├─ ProcedureInitPackageState.cs
│        ├─ ProcedureDownloadBundleState.cs
│        ├─ ProcedurePatchDoneState.cs
│        ├─ ProcedureLoadAssembly.cs
│        └─ ProcedureUpdateFinishState.cs
└─ YooAsset
   └─ AssetBundleCollectorSetting.asset
```

Hybrid 模板在 Normal 基础上多出：

```text
Assets
├─ HybridCLRGenerate
│  ├─ AOTGenericReferences.cs
│  ├─ HybridCLRGenerate.asmdef
│  └─ link.xml
├─ Bundles
│  └─ DLL
└─ Scripts
   └─ Hotfix
      ├─ GameBase
      │  ├─ GameBase.cs
      │  └─ GameBase.asmdef
      ├─ GameLib
      │  ├─ GameLib.cs
      │  └─ GameLib.asmdef
      ├─ GameLogic
      │  ├─ HotfixEntry.cs
      │  └─ GameLogic.asmdef
      └─ GameProto
         ├─ GameProto.asmdef
         └─ Config
            └─ Extension
               └─ ExternalTypeUtil.cs
```

## 各目录职责

| 目录 | 作用 | 相关文档 |
| :--- | :--- | :--- |
| `Assets/Scenes/Main.unity` | 启动场景，挂 `ProcedureEntry`；`InputService` 等组件按需自行挂载 | [Procedure 流程](../procedure/index.md) |
| `Assets/Scripts/Startup` | 启动流程状态机、下载进度、版本信息 | [Procedure 流程](../procedure/index.md) |
| `Assets/Scripts/Hotfix` | 热更程序集，入口 `GameLogic.HotfixEntry` | [Procedure 流程](../procedure/index.md) |
| `Assets/HybridCLRGenerate` | AOT 泛型引用与 `link.xml`，HybridCLR 生成物 | [Editor Extension](../editor-extension/index.md) |
| `Assets/Bundles` | YooAsset 收集目录，按类型分组打包 | [Resources 资源](../resource/index.md) |
| `Assets/YooAsset/AssetBundleCollectorSetting.asset` | 收集器配置：分组、收集路径、打包与地址规则 | [Resources 资源](../resource/index.md) |
| `Assets/Bundles/Configs/bytes` | 配置表 `.bytes` 输出位置，由 Config 管线写入 | [Config 配置管线](../config/index.md) |
| `Assets/Bundles/Configs/sciptableObject` | 本地化 `.loclang` 资源 | [Localization 本地化](../localization/index.md) |

模板里的 `Startup` 脚本只是**可运行的起点**：`ProcedureEntry` 会初始化 `LocalizationKit`、等待 YooAsset、再把 `ProcedureBuilder` 的流程列表跑起来。业务代码应该替换这些状态，而不是在模板文件上堆逻辑。

## 项目设置改动

| 设置 | Normal | Hybrid | 说明 |
| :--- | :--- | :--- | :--- |
| `PlayerSettings.allowUnsafeCode` | `true` | `true` | 框架运行时代码需要 |
| 脚本符号 `ENABLE_LOG` | 追加 | 追加 | 控制日志开关 |
| 脚本符号 `ENABLE_HYBRIDCLR` | — | 追加 | 控制热更分支 |
| Scripting Backend | 不改 | 询问后设为 IL2CPP | HybridCLR 要求 |
| Api Compatibility Level | 不改 | 询问后设为 .NET Framework | HybridCLR 要求 |
| Incremental GC | 不改 | 询问后关闭 | HybridCLR 要求 |

Hybrid 的设置不是强制的：确认框里选 `Manual` 会保留你当前的 Player Settings，需要自己按上表设置。选 `Cancel` 会中止整个模板安装。

## 重复安装与文件冲突

- **目录先创建**：模板里的空目录（只放了 `.gitkeep`）也会在 `Assets/` 下建出来。
- **占位文件不复制**：`.keep` / `.gitkeep` 会被跳过。
- **已存在的文件不覆盖**：只打一条警告 `AlicizaX installer skipped existing file: …`，然后继续下一个文件。
- **不会删除**任何已有文件。

所以“重装模板”不能用来修复被改坏的启动脚本；要恢复只能从包里的 `Templates~` 手动拷回。

## 安装状态如何判定

安装器不只看 JSON，还会核对模板留下的目录标记：

| 状态 | 需要的标记（任一命中） |
| :--- | :--- |
| `NormalTemplate` | `Assets/Scripts/Startup`、`Assets/Bundles`、`Assets/YooAsset` |
| `HybridTemplate` | `Assets/Scripts/Hotfix`、`Assets/HybridCLRGenerate`、`Assets/Bundles/DLL` |

标记缺失时，JSON 里的状态会被降级（Hybrid → Normal → NotInstalled），避免“显示已安装但实际没有文件”。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 只用框架，不要模板场景 | 安装 Core 后点 `Use Custom`，自己搭启动场景 |
| 模板文件想改 | 直接改 `Assets/` 下的副本；模板只复制一次，不会回写 |
| 换一套模板 | 先按[升级、修复与卸载](maintain.md)清理，再重装 |
| 加自己的启动逻辑 | 在 `Assets/Scripts/Startup/Procedure` 增删状态类，再改 `ProcedureEntry` 的列表 |

## 容易踩的坑

1. **以为模板会自动更新**：模板是快照，升级 installer 包不会更新已复制的文件。
2. **改 `Templates~` 里的文件**：那是包内只读内容，下次 Package Manager 更新会被覆盖；要改就改 `Assets/` 下的副本。
3. **Hybrid 只装了文件没改后端**：确认框选 `Manual` 后忘了设 IL2CPP，打包才会暴露问题。
4. **删除 `Assets/Bundles/DLL`**：Hybrid 状态判定会降级，安装器可能重新允许安装模板。
5. **在已有同类文件的项目里装 Normal**：模板文件被跳过，旧文件保留，可能出现新旧脚本混用。

## 下一步

- 安装完成后的验证与升级：[升级、修复与卸载](maintain.md)
- 启动链路：[快速开始](../../quickstart.md)
