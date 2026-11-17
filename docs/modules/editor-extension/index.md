# Editor Extension 编辑器扩展

`com.alicizax.unity.editor.extension` 是一个**纯编辑器工具包**：图集自动生成、Texture Packer、资产引用查找、HybridCLR 构建命令。它不包含任何运行时组件，也不影响打包后的程序集。

## 适用场景

- UI 图集想跟着 Sprite 增删自动更新，而不是手工拖 `SpriteAtlas`。
- 需要把零散贴图打成 TMP 能用的图集（含 JSON 描述）。
- 要查“这个材质/贴图被哪些 Prefab 引用”，或反向查依赖。
- 用 HybridCLR 时，需要一键编译热更 DLL 并复制 AOT 补充程序集。

## 使用前提

| 项 | 要求 |
| :--- | :--- |
| Unity | 2022.3+ |
| 包依赖 | `com.unity.2d.sprite`（SpriteAtlas 相关 API） |
| HybridCLR 功能 | 需安装 `com.code-philosophy.hybridclr`，包内通过 `HYBIRDCLR_SUPPORT` 宏（注意源码里的拼写）条件编译 |
| 平台 | 仅 Editor；程序集 `AlicizaX.EditorExtension.Editor` 的 `includePlatforms` 为 `Editor` |

安装方式：通过 OpenUPM（scope `com.alicizax.unity`）按包名安装，或从仓库 git URL 添加：

```text
com.alicizax.unity.editor.extension
```

## 菜单总览

| 菜单 | 窗口/动作 | 页面 |
| :--- | :--- | :--- |
| `AlicizaX/Extension/图集工具/Configuration Panel` | 打开图集配置窗口 | [图集工具](atlas.md) |
| `AlicizaX/Extension/图集工具/ForceGenerateAll` | 强制重建全部图集 | [图集工具](atlas.md) |
| `AlicizaX/Extension/Texture Packer` | 打开贴图打包窗口 | [Texture Packer](texture-packer.md) |
| `AlicizaX/Extension/查找资产引用 _F10` | 打开引用查找窗口（快捷键 `F10`） | [资产引用查找](reference-finder.md) |
| `HybridCLR/Tools/Enable HybridCLR` | 加 `ENABLE_HYBRIDCLR` 并启用 HybridCLR | [HybridCLR 构建工具](hybridclr.md) |
| `HybridCLR/Tools/Disable HybridCLR` | 移除宏并关闭 HybridCLR | [HybridCLR 构建工具](hybridclr.md) |
| `HybridCLR/Tools/BuildAssets And CopyTo AssemblyTextAssetPath` | 编译热更 DLL + 复制 AOT 补充程序集 | [HybridCLR 构建工具](hybridclr.md) |

## 与 Runtime 包的边界

| 能力 | 所在包 | 文档 |
| :--- | :--- | :--- |
| 图集在运行时的加载与释放 | `com.alicizax.unity.framework` | [Resources 资源](../resource/index.md) |
| `AlicizaX/UISetting Window`、`GameObject/UI生成` | `com.alicizax.unity.framework`（Editor 目录） | [UI 界面](../ui/index.md) |
| `AlicizaX/Open Folder/*` | `com.alicizax.unity.framework`（Editor 目录） | [Resources 资源](../resource/index.md) |
| 组件 Inspector 增强 | 各运行时包自带 | 对应模块页面 |
| 图集生成、引用查找、HybridCLR 命令 | **本包** | 本模块各页 |

本包只改 `Assets/`、`ProjectSettings/`、`Library/` 下的编辑器数据，不参与运行时服务注册。

## 最短接入

1. 安装 `com.alicizax.unity.editor.extension`，等待编译。
2. 打开 `AlicizaX/Extension/图集工具/Configuration Panel`，确认 `Source Root` 指向 `Assets/Bundles/UIRaw/Atlas`，`Output Directory` 指向 `Assets/Art/Atlas`。
3. 往 `Assets/Bundles/UIRaw/Atlas/<子目录>/` 放 Sprite，图集自动生成。
4. 需要查引用时按 `F10` 打开引用查找窗口，把资产拖进窗口。
5. 需要热更时用 `HybridCLR/Tools/BuildAssets And CopyTo AssemblyTextAssetPath`。

## 常见配置

| 设置 | 位置 | 说明 |
| :--- | :--- | :--- |
| 图集源目录/输出目录 | 图集工具窗口 `Directory Settings` | 存于 `ProjectSettings/AtlasConfiguration.asset` |
| 平台压缩格式 | 图集工具窗口 `Platform Settings` | Android / iOS / WebGL 分别配置 |
| 打包参数 | 图集工具窗口 `Packing Settings` | Padding、旋转、紧凑打包、Block Offset |
| 排除规则 | 图集工具窗口 `Exclusion Settings` | 关键词与目录 |
| Texture Packer 输出 | 窗口内 `Output Folder` | 默认 `Assets/TexturePackerOutput` |

## 容易踩的坑

1. **以为它是运行时包**：本包没有 Runtime 程序集，打包后不生效；运行时逻辑请找对应模块。
2. **HybridCLR 宏拼写**：源码里的宏是 `HYBIRDCLR_SUPPORT`（少一个 `D`），自定义脚本判断时要与它一致。
3. **图集工具会自动改导入设置**：`SpritePostprocessor` 在导入时把贴图转成 Sprite；不想被改就把 `Auto Generate` 关掉。
4. **引用查找首次很慢**：要扫描全工程并写 `Library/ReferenceFinderCache`，大项目耐心等或先点“点击更新本地缓存”。
5. **HybridCLR 菜单不出现**：没装 HybridCLR 包，`HYBIRDCLR_SUPPORT` 未定义，整个 `HybridCLR/` 菜单都不会注册。

## 本模块页面

- [图集工具](atlas.md)：自动生成流程、图集命名、平台格式、排除规则。
- [Texture Packer](texture-packer.md)：TMP 兼容图集与 JSON 输出。
- [资产引用查找](reference-finder.md)：引用/依赖模式、缓存、状态标记。
- [HybridCLR 构建工具](hybridclr.md)：启用/禁用、编译热更 DLL、复制 AOT 程序集。

## API 速查

| 类型 | 位置 | 说明 |
| :--- | :--- | :--- |
| `AtlasConfiguration` | `Editor/AtlasBuilder/AtlasConfiguration.cs` | 图集配置单例，存 `ProjectSettings/AtlasConfiguration.asset` |
| `AtlasConfigWindow` | `Editor/AtlasBuilder/AtlasEditorWindow.cs` | 配置窗口 |
| `EditorSpriteSaveInfo` | `Editor/AtlasBuilder/EditorSpriteSaveInfo.cs` | 图集生成入口：`ForceGenerateAll` / `ClearCache` / `ShouldProcess` |
| `SpritePostprocessor` | `Editor/AtlasBuilder/SpritePostprocessor.cs` | 资源导入钩子 |
| `UnityTexturePackerEditor` | `Editor/TexturePacker/UnityTexturePackEditor.cs` | 贴图打包窗口 |
| `ReferenceFinderData` | `Editor/ReferenceFinder/ReferenceFinderData.cs` | 引用索引与缓存 |
| `ResourceReferenceInfo` | `Editor/ReferenceFinder/ResourceReferenceInfo.cs` | 引用查找窗口 |
| `SortType` / `AssetState` | `Editor/ReferenceFinder` | 排序方式与资产状态 |
| `HybridClrBuild` / `BuildDLLCommand` | `Editor/HybridCLR` | 热更编译与 AOT 复制 |
