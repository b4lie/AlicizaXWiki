# 资产引用查找

引用查找窗口解决两个问题：**“这个资源被谁用了”**（引用模式）和**“这个资源用了谁”**（依赖模式）。它直接扫描工程里的 YAML 资源，不依赖 AssetDatabase 的深层查询，适合查 Prefab / 场景 / 材质级别的引用链。

## 适用场景

- 删贴图/材质前，确认没有 Prefab 或场景还在引用。
- 查某个 UI 图集被哪些窗口 Prefab 用到。
- 反向排查“这个 Prefab 依赖了哪些材质/贴图/动画”。
- 找出丢失引用（Missing）或 GUID 被改过的资产。

## 使用前提

- 已安装 `com.alicizax.unity.editor.extension`。
- 首次打开会扫描整个工程，工程越大越慢；扫描结果缓存在 `Library/ReferenceFinderCache`。
- 只索引这些扩展名：`.prefab`、`.unity`、`.mat`、`.asset`、`.anim`、`.controller`。纯脚本、图片、音频本身不作为“引用方”被扫描。

## 最短接入

1. 在 Project 面板选中要查的资源（可多选，也可以选文件夹）。
2. 按 `F10`，或菜单 `AlicizaX / Extension / 查找资产引用 _F10`。
3. 窗口打开后按当前选择构建引用树。
4. 需要追加/换目标时，把资源从 Project 面板**拖进窗口**。
5. 用工具栏切换模式、展开/折叠、刷新缓存。

窗口标题是 `查找资产引用`。

## 界面与操作

| 区域 | 功能 |
| :--- | :--- |
| 工具栏 `点击更新本地缓存` | 重新扫描全工程并刷新索引（会重新写 `Library/ReferenceFinderCache`） |
| 工具栏 `引用模式` / `依赖模式` | 切换方向；选择会记在 `PlayerPrefs`，下次打开保持 |
| 工具栏 `展开` / `折叠` | 展开或折叠整棵树 |
| 列 `名称` | 资产名，可排序（升序/降序切换） |
| 列 `路径` | 资产路径，可排序 |
| 列 `状态` | `Normal` / `Changed` / `Missing` / `Invalid` |
| 列 `引用数量` | 该资产被引用的次数 |
| 拖拽区 | 整个窗口接收 Project / 资源管理器拖入的资产或文件夹 |

选中文件夹时，会展开成文件夹内所有资产（跳过子文件夹本身）。

## 两种模式

| 模式 | 树的含义 | 典型用途 |
| :--- | :--- | :--- |
| 引用模式（默认） | 根 = 你选的资产，子节点 = 引用它的资产，逐层向上 | 判断能不能删 |
| 依赖模式 | 根 = 你选的资产，子节点 = 它依赖的资产，逐层向下 | 评估加载一个 Prefab 会带上哪些资源 |

模式切换会立即重建树，不会重新扫描工程。

## 状态含义

| 状态 | 含义 | 触发条件 |
| :--- | :--- | :--- |
| `Normal` | 索引有效 | 文件修改时间与缓存记录一致 |
| `Changed` | 文件在缓存之后被改过 | `File.GetLastWriteTime` 与缓存里的哈希字符串不一致 |
| `Missing` | 引用的资产不存在 | 索引里有 GUID，但磁盘上找不到对应文件 |
| `Invalid` | 无法解析 | 资源读取失败或 GUID 无效 |

出现大量 `Changed` 时点一次 `点击更新本地缓存` 重建索引。

## 扫描与缓存

- 扫描是多线程的：线程数 = `max(8, CPU 核数)`，每个线程一次处理 100 个文件后汇总。
- 每个资产记录：`path`、`name`、`dependencies`、`references`、`assetDependencyHashString`、`state`。
- 缓存文件：`Library/ReferenceFinderCache`。删除 `Library` 或该文件后，下次打开会重新全量扫描。
- 索引只解析 YAML 里的 `guid: <32 位>` 形式，因此**代码里 `Resources.Load` 的字符串路径、Addressables/YooAsset 的地址不会被识别为引用**。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 只看某个目录 | 在 Project 里选中文件夹再按 `F10` |
| 换一批目标 | 直接把新资源拖进窗口，会替换当前根集合 |
| 索引过期 | 点 `点击更新本地缓存` |
| 默认打开就是依赖模式 | 切一次模式即可，选择会记住 |

## 容易踩的坑

1. **把“引用数量 0”当成可以安全删除**：通过资源地址（YooAsset/Resources）动态加载的资源不会被索引，删除前还要全局搜地址字符串。
2. **首次打开很慢**：全工程扫描需要时间，期间窗口可能没有内容；等扫描结束或先点一次更新缓存。
3. **缓存与真实文件不同步**：在 Unity 外修改了 Prefab/场景，状态可能仍显示 `Normal`；点 `点击更新本地缓存`。
4. **脚本引用查不到**：`.cs` 不在索引扩展名里，脚本之间的依赖请用 IDE。
5. **移动/重命名资产后**：GUID 不变则引用关系不变；如果用了“强制文本序列化 + 手改 GUID”，会出现 `Missing`。

## API 速查

| 类型/成员 | 说明 |
| :--- | :--- |
| `ResourceReferenceInfo` | 引用查找窗口（`EditorWindow`） |
| `ResourceReferenceInfo.FindRef()` | 菜单入口，按当前 `Selection` 打开窗口 |
| `ReferenceFinderData` | 索引数据：`CollectDependenciesInfo()`、`ReadFromCache()`、`UpdateAssetState(guid)` |
| `ReferenceFinderData.AssetState` | `Normal` / `Changed` / `Missing` / `Invalid` |
| `AssetDescription` | 单个资产的依赖、被引用列表与状态 |
| `SortType` | `None` / `AscByName` / `DescByName` / `AscByPath` / `DescByPath` |
| `DragAreaGetObject.GetObjects()` | 窗口拖拽接收 |

相关：[图集工具](atlas.md)｜[Resources 资源](../resource/index.md)
