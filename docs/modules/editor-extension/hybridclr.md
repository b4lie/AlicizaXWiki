# HybridCLR 构建工具

这个页面只讲 `com.alicizax.unity.editor.extension` 提供的三个 HybridCLR 菜单：启用/禁用宏、编译热更 DLL 并复制 AOT 补充程序集。运行时如何加载热更程序集见 [Procedure 流程](../procedure/index.md)。

## 适用场景

- 项目已切到 Hybrid 模板，需要产出热更 DLL。
- 改了 `GameLogic` / `GameProto` 等热更程序集，要重新出包。
- 需要临时关闭 HybridCLR（改回纯 AOT 流程）做对比测试。

## 使用前提

| 项 | 要求 |
| :--- | :--- |
| HybridCLR 包 | 已安装 `com.code-philosophy.hybridclr`；否则 `HYBIRDCLR_SUPPORT` 未定义，整个 `HybridCLR/` 菜单不会出现 |
| Player Settings | Scripting Backend = IL2CPP，Api Compatibility Level = .NET Framework，关闭 Incremental GC |
| 程序集划分 | 热更程序集（`GameLib`、`GameProto`、`GameBase`、`GameLogic`）已在 HybridCLR Settings 里标记为 Hot Update |
| AOT 引用 | 已生成 `AOTGenericReferences.cs`（路径由 HybridCLR Settings 的 `outputAOTGenericReferenceFile` 决定） |
| 资源收集 | `Assets/Bundles/DLL` 在 YooAsset 收集器里（模板已配置 `DLL` 组） |

## 菜单

| 菜单 | 作用 |
| :--- | :--- |
| `HybridCLR/Tools/Enable HybridCLR` | 移除再添加脚本符号 `ENABLE_HYBRIDCLR`，并把 `HybridCLR.SettingsUtil.Enable` 设为 `true` |
| `HybridCLR/Tools/Disable HybridCLR` | 移除 `ENABLE_HYBRIDCLR`，并把 `SettingsUtil.Enable` 设为 `false` |
| `HybridCLR/Tools/BuildAssets And CopyTo AssemblyTextAssetPath` | 编译当前平台的热更 DLL + 复制 AOT 补充程序集到 `Assets/Bundles/DLL` |

## 最短接入

1. 确认 HybridCLR 已安装，`HybridCLR/Tools/Enable HybridCLR` 可用。
2. 执行一次 `HybridCLR/Tools/Enable HybridCLR`。会加宏并启用 HybridCLR，等编译结束。
3. 用 HybridCLR 自带的生成入口生成 AOT 引用（`AOTGenericReferences.cs`）。这一步必须在构建 DLL 之前完成。
4. 执行 `HybridCLR/Tools/BuildAssets And CopyTo AssemblyTextAssetPath`。
5. 检查 `Assets/Bundles/DLL` 下的 `.bytes` 文件：

    ```text
    Assets/Bundles/DLL
    ├─ GameLib.dll.bytes
    ├─ GameProto.dll.bytes
    ├─ GameBase.dll.bytes
    ├─ GameLogic.dll.bytes
    └─ <AOT 补充程序集>.dll.bytes
    ```

6. 按正常流程打 AssetBundle / 出包。热更程序集与 AOT 补充程序集都会被打进 `DLL` 包。

## 构建时做了什么

`BuildAndCopyDlls` 依次调用：

```csharp
HybridClrBuild.CompileHotUpdate(EditorUserBuildSettings.activeBuildTarget);
HybridClrBuild.CopyAot(EditorUserBuildSettings.activeBuildTarget);
```

### 1. 编译热更 DLL

- `CompileDllCommand.CompileDll(target)` 编译当前激活平台的热更程序集。
- 从 `SettingsUtil.HotUpdateAssemblyFilesExcludePreserved` 取热更程序集列表。
- 从 `SettingsUtil.GetHotUpdateDllsOutputDirByTarget(target)` 复制到 `Assets/Bundles/DLL`，并重命名为 `<dll>.bytes`。
- 复制后 `AssetDatabase.Refresh()`。

### 2. 复制 AOT 补充程序集

- 读取 `StartupSetting` 同目录的 `AOTGenericReferences` 类型（文件名由 `outputAOTGenericReferenceFile` 决定）。
- 通过反射取静态字段 `PatchedAOTAssemblyList` 得到需要补充元数据的程序集列表。
- 从 `SettingsUtil.GetAssembliesPostIl2CppStripDir(target)` 复制到同一目录，同样加 `.bytes` 后缀。

`.bytes` 后缀是为了让 Unity 把它们当 `TextAsset` 打进 AssetBundle。

## 运行时衔接

热更程序集与入口在 `Assets/Scripts/Startup/StartupSetting.cs` 里配置：

```csharp
public const string EntranceDll = "GameLogic.dll";
public const string EntranceClass = "GameLogic.HotfixEntry";
public const string EntranceMethod = "Entrance";

public static readonly List<string> HotUpdateAssemblies = new List<string>()
    { "GameLib.dll", "GameProto.dll", "GameBase.dll", "GameLogic.dll" };
```

启动流程的 `ProcedureLoadAssembly` 会：

1. 调用 `RuntimeApi.LoadMetadataForAOTAssembly` 为 AOT 补充程序集加载元数据。
2. 通过 `GameApp.Resource.LoadLeaseAsync<TextAsset>` 按地址加载 `GameLib.dll` 等 DLL 字节。
3. `Assembly.Load(bytes)` 载入热更程序集。
4. 反射调用 `GameLogic.HotfixEntry.Entrance`，进入热更逻辑。

地址就是去掉 `.bytes` 的文件名（YooAsset `AddressByFileName`），因此 `Assets/Bundles/DLL/GameLogic.dll.bytes` 的地址是 `GameLogic.dll`。

改入口或热更程序集列表时，要同步改 `StartupSetting` 与 HybridCLR Settings，否则运行时会报“找不到入口类型/程序集”。

## 常见错误

| 报错 | 原因 | 处理 |
| :--- | :--- | :--- |
| `热更程序集列表为空` | HybridCLR Settings 里没有标记 Hot Update 程序集 | 在 HybridCLR Settings 勾选热更程序集后重试 |
| `AOT 补充程序集列表为空` | `AOTGenericReferences.PatchedAOTAssemblyList` 为空 | 重新生成 AOT 引用，或检查 HybridCLR 生成流程 |
| `AOT 引用文件不存在` | `outputAOTGenericReferenceFile` 指向的文件没生成 | 先跑 HybridCLR 的生成入口 |
| `找不到 AOT 引用类型: AOTGenericReferences` | 生成的脚本没编译，或文件名/类型名不一致 | 等编译完成；确认生成的类型名 |
| `… 缺少 PatchedAOTAssemblyList` | 生成脚本版本不匹配 | 重新生成 AOT 引用 |
| `DLL 不存在: GameLogic.dll` | 热更编译输出目录里没有该 DLL | 确认程序集名与 HybridCLR 配置；先编译成功再复制 |
| `HybridCLR 安装失败` | HybridCLR 未正确安装 | 用 HybridCLR 自带安装器完成初始化 |

## 常见配置

| 需求 | 改哪里 |
| :--- | :--- |
| 增减热更程序集 | HybridCLR Settings 的 Hot Update 列表 + `StartupSetting.HotUpdateAssemblies` |
| 换入口类/方法 | `StartupSetting.EntranceDll` / `EntranceClass` / `EntranceMethod` |
| 换 DLL 输出目录 | `HybridClrBuild.DllAssetDirectory`（源码固定为 `Assets/Bundles/DLL`） |
| 关闭热更做对比 | `HybridCLR/Tools/Disable HybridCLR` |
| 换目标平台 | 切换 Unity 的 Active Build Target 后再执行构建菜单 |

## 容易踩的坑

1. **先构建 DLL 后生成 AOT 引用**：`PatchedAOTAssemblyList` 是旧数据，补充元数据会缺；顺序不能反。
2. **宏名拼写**：包内宏是 `HYBIRDCLR_SUPPORT`（少一个 `D`），自定义脚本判断时要用这个名字。
3. **只在编辑器验证热更**：编辑器下 `Assembly.Load(bytes)` 会与已加载程序集冲突，`ProcedureLoadAssembly` 在 `EditorSimulateMode` 下跳过这一步；热更必须出包验证。
4. **忘了把 DLL 打进包**：`Assets/Bundles/DLL` 必须在收集器里，且改完 DLL 要重新打包。
5. **切平台不重新构建**：`.bytes` 是按平台编译的，换平台后要重新执行构建菜单。
6. **IL2CPP 设置被改回 Mono**：HybridCLR 直接不可用，出包前检查 Player Settings。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `BuildDLLCommand.Enable()` / `Disable()` | 菜单实现：宏与 `SettingsUtil.Enable` |
| `BuildDLLCommand.BuildAndCopyDlls()` | 编译热更 + 复制 AOT |
| `BuildDLLCommand.GenerateAll()` | 调用 `PrebuildCommand.GenerateAll()`（无菜单入口） |
| `HybridClrBuild.CompileHotUpdate(target)` | 编译并复制热更 DLL |
| `HybridClrBuild.CopyAot(target)` | 复制 AOT 补充程序集 |
| `HybridClrBuild.Generate()` | 生成 HybridCLR 相关文件 |

相关：[Procedure 流程](../procedure/index.md)｜[Installer 安装器](../installer/index.md)｜[项目模板与生成目录](../installer/templates.md)
