# 图集工具

图集工具把 `Assets/Bundles/UIRaw/Atlas` 下的 Sprite 按子目录自动打成一个或多个 `SpriteAtlas`，并在贴图增删时增量更新。适合 UI 图标、通用小图这类“目录即图集”的资产管理方式。

## 适用场景

- UI 图标按目录分类（Common、Shop、Item…），每类一个图集。
- 不想每次加图都手动拖进 `SpriteAtlas`。
- 需要按平台设置不同的压缩格式（Android/iOS/WebGL）。

## 使用前提

- 已安装 `com.alicizax.unity.editor.extension` 与 `com.unity.2d.sprite`。
- 源图放在 `AtlasConfiguration.sourceAtlasRoot` 的子目录里（默认 `Assets/Bundles/UIRaw/Atlas`）。
- 图片格式为 Unity 支持的贴图（`.png` / `.jpg` / `.tga` 等，按 `ShouldProcess` 判断）。
- 不要在 `Packages/` 下放源图，处理器会跳过。

## 最短接入

1. 打开配置窗口：`AlicizaX / Extension / 图集工具 / Configuration Panel`。
2. 设置两个目录：
   - `Source Root`：`Assets/Bundles/UIRaw/Atlas`
   - `Output Directory`：`Assets/Art/Atlas`
3. 设置 `Platform Settings` 的 Android / iOS / WebGL 格式。
4. 设置 `Packing Settings`（一般保持默认）。
5. 把图片放到 `Assets/Bundles/UIRaw/Atlas/Common/`、`Assets/Bundles/UIRaw/Atlas/Shop/` 这样的子目录。
6. 保持 `Auto Generate` 勾选，Unity 导入时自动生成；也可以点按钮 `Generate All Atlases` 或菜单 `ForceGenerateAll` 立即重建。

生成结果：

```text
Assets/Art/Atlas
├─ Atlas_Common.spriteatlasv2
└─ Atlas_Shop.spriteatlasv2
```

（`Use V2 Packing` 打开时后缀是 `.spriteatlasv2`，关闭时是 `.spriteatlas`。）

## 配置项

配置文件是 `ProjectSettings/AtlasConfiguration.asset`，窗口里改完立即保存。

| 字段 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `outputAtlasDir` | `Assets/Art/Atlas` | 图集输出目录 |
| `sourceAtlasRoot` | `Assets/Bundles/UIRaw/Atlas` | 源图根目录，**只扫描它的子目录** |
| `androidFormat` | `ASTC_6x6` | Android 平台图集格式 |
| `iosFormat` | `ASTC_5x5` | iOS 平台图集格式 |
| `webglFormat` | `ASTC_6x6` | WebGL 平台图集格式 |
| `padding` | `2` | 图元间距，窗口里只能选 2 / 4 / 8 |
| `enableRotation` | `true` | 允许旋转图元 |
| `blockOffset` | `1` | 块偏移 |
| `tightPacking` | `true` | 紧凑打包（窗口显示为 `Trim Transparency`） |
| `compressionQuality` | `50` | 平台压缩质量，写入图集 importer |
| `autoGenerate` | `true` | 是否在导入/删除/移动时自动处理 |
| `enableLogging` | `true` | 输出处理日志 |
| `enableV2` | `true` | 使用 SpriteAtlas V2（`.spriteatlasv2`） |
| `excludeKeywords` | `_Delete`, `_Temp` | 路径包含这些关键词的图片被排除 |
| `excludeFolders` | `Assets/Bundles/UIRaw/Raw` | 这些目录（含子目录）被排除 |

另外，`enableAlphaDilation` 在代码里固定为 `true`，窗口不暴露。

## 图集命名规则

图集名 = 源根目录名 + 相对子目录路径（用 `_` 连接）：

| 源文件 | 图集名 |
| :--- | :--- |
| `Assets/Bundles/UIRaw/Atlas/Common/Icon_1.png` | `Atlas_Common` |
| `Assets/Bundles/UIRaw/Atlas/Shop/Item/Icon_2.png` | `Atlas_Shop_Item` |
| `Assets/Bundles/UIRaw/Atlas/Icon_3.png` | **不生成**（见下方坑） |

源根目录名来自 `sourceAtlasRoot` 的最后一段（默认 `Atlas`），改名源目录会改变所有图集名。

## 自动流程

`SpritePostprocessor` 挂在资源导入管线里：

| 阶段 | 行为 |
| :--- | :--- |
| `OnPreprocessTexture` | 把源图 `textureType` 设为 `Sprite`、`spriteImportMode` 设为 `Single`、关闭 fallback physics shape |
| `OnPostprocessAllAssets`（导入） | 新图加入图集映射，标记脏，延迟处理 |
| `OnPostprocessAllAssets`（删除） | 从映射移除；图集空了就删除图集资产 |
| `OnPostprocessAllAssets`（移动） | 旧路径按删除处理，新路径按导入处理 |
| 延迟处理 | 合并同一帧的多次变更，只重建脏图集 |

`GenerateAtlas` 的优化：

- 源图还在导入（`PendingImport`）时跳过本次，等下一轮。
- 现有图集的 packables 与期望一致时直接跳过，不重写资产。
- 图集资产存在但还没导入完成时跳过。
- 图集没有任何 Sprite 时，删除当前与另一版本后缀的旧图集。

## 手动操作

| 操作 | 入口 | 说明 |
| :--- | :--- | :--- |
| 全量重建 | 窗口按钮 `Generate All Atlases` | 忽略 packables 比较，强制重建 |
| 全量重建 | 菜单 `AlicizaX/Extension/图集工具/ForceGenerateAll` | 同上，不开窗口也能执行 |
| 清缓存 | 窗口按钮 `Clear Cache` | 清空内存里的脏标记集合，不删图集资产 |

`ForceGenerateAll` 用于：改了平台格式、改了打包参数、图集与目录不同步、从版本控制拉了新图但没触发导入。

## 运行时的关系

- 源图仍在 `Assets/Bundles/UIRaw/Atlas`，由 YooAsset 的 `UIRaw` 收集器打包；运行时加载的是 Sprite，不是图集资产。
- 图集输出到 `Assets/Art/Atlas`，只负责构建期合批，**不要**把它移进 `Assets/Bundles`，否则会被重复收集。
- 图集的平台格式与压缩质量只影响图集本身；源图的导入设置由 `PrepareSpriteImporter` 统一为 Sprite。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 一类图一个图集 | 在 `sourceAtlasRoot` 下按分类建子目录 |
| 排除测试图 | 文件名加 `_Delete` / `_Temp`，或把目录加进 `excludeFolders` |
| 关闭自动生成 | 取消 `Auto Generate`，之后手动 `ForceGenerateAll` |
| 改用 `.spriteatlas` 格式 | 关闭 `Use V2 Packing`，重新全量生成 |
| 调整图集数量 | 拆/合子目录；图集粒度就是“最深层子目录” |

## 容易踩的坑

1. **图片直接放源根目录**：`GetAtlasName` 要求相对路径至少有“子目录 + 文件名”两段，根目录直属图片会被忽略，既不报错也不进图集。
2. **改了源目录名**：图集名随之变化，旧图集不会自动删（除非同名的空图集触发清理），需要手动删 `Assets/Art/Atlas` 下的旧资产。
3. **自动生成期间频繁改图**：处理是延迟合并的，连续改动可能只保留最后一轮；等导入完成再检查结果。
4. **在 `Packages/` 下放 UI 图**：处理器直接跳过，不会生成图集。
5. **把输出目录设进 `Assets/Bundles`**：图集资产会被 YooAsset 当资源再打一次，包体变大。
6. **手动改图集 packables**：下一次自动处理会按目录重算并覆盖你的修改。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `AtlasConfiguration.Instance` | 配置单例 |
| `AtlasConfiguration.Save(true)` | 保存到 `ProjectSettings/AtlasConfiguration.asset`（窗口自动调用） |
| `EditorSpriteSaveInfo.ForceGenerateAll()` | 强制重建全部图集 |
| `EditorSpriteSaveInfo.ClearCache()` | 清空脏标记 |
| `EditorSpriteSaveInfo.ShouldProcess(path)` | 判断路径是否受管 |
| `EditorSpriteSaveInfo.PrepareSpriteImporter(importer, path)` | 统一 Sprite 导入设置 |
| `EditorSpriteSaveInfo.MarkParentAtlasesDirty(path)` | 目录变更时标记父级图集 |

相关：[Texture Packer](texture-packer.md)｜[Resources 资源](../resource/index.md)
