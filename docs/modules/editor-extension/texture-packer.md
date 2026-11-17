# Texture Packer

Texture Packer 把多张贴图打成一个 PNG 图集，并导出一份 TexturePacker 格式的 JSON。它的目标很具体：**给 TextMeshPro 生成可用的多 Sprite 图集**（例如表情、图标字体）。

## 适用场景

- TMP 的表情图集 / 图标集，需要“一张大图 + 多个 Sprite + JSON 描述”。
- 临时把散图合并，交给美术或外部工具核对。
- 需要在不改源图导入设置的前提下快速出图。

不适用：UI 常规图集请用[图集工具](atlas.md)，它会跟随目录自动维护。

## 使用前提

- 已安装 `com.alicizax.unity.editor.extension`。
- 贴图能被 Unity 读取（Project 里的 `Texture2D`，或从资源管理器拖入）。
- 输出目录在 `Assets/` 下，否则不会被 Unity 导入。

## 最短接入

1. 打开窗口：`AlicizaX / Extension / Texture Packer`。
2. 把贴图拖进窗口的拖拽区（支持 Project 面板与系统资源管理器）。
3. 调整顺序：`▲` / `▼` 改变打入顺序，`Remove` 移除。
4. 设置输出：
   - `Output Folder`：默认 `Assets/TexturePackerOutput`
   - `Atlas Filename`：默认 `atlas.png`
   - `JSON Filename`：默认 `atlas.json`
   - `Max Atlas Size`：1024 / 2048 / 4096，默认 4096
   - `Padding (px)`：默认 2
   - `Overwrite Existing`：默认开启
5. 点 `Build Atlas`。
6. 成功后弹窗显示图集与 JSON 路径，窗口下方会展示 JSON 内容预览。

## 输出内容

```text
Assets/TexturePackerOutput
├─ atlas.png     # 未压缩 PNG，导入为 Sprite (Multiple)
└─ atlas.json    # TexturePacker 格式的 frames + meta
```

- 图集贴在打包时按 `Max Atlas Size` 自动换行/换页；`PackTextures` 返回的矩形决定每个 Sprite 的位置。
- PNG 以**未压缩**方式导入，Sprite 的 pivot 统一为 `(0.5, 0.5)`、对齐方式 `Center`。
- JSON 结构：

```json
{
  "frames": [
    {
      "filename": "icon_1.png",
      "frame": { "x": 0, "y": 0, "w": 64, "h": 64 },
      "rotated": false,
      "trimmed": false,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 64, "h": 64 },
      "sourceSize": { "w": 64, "h": 64 },
      "pivot": { "x": 0.5, "y": 0.5 }
    }
  ],
  "meta": {
    "app": "Unity TexturePacker Export",
    "version": "1.0",
    "image": "atlas.png",
    "format": "RGBA8888",
    "size": { "w": 256, "h": 256 },
    "scale": "1"
  }
}
```

`frame.y` 已经转换成“左下角原点”的坐标（`atlasH - (y + h)`），与 TexturePacker 的习惯一致。

## 打包时对源图做了什么

为了能读像素，`BuildAtlas` 会临时改源图导入设置：

1. 记录每张源图的 `textureType`、`textureCompression`、`maxTextureSize`。
2. 改成 `Default` + `Uncompressed`，并把 `maxTextureSize` 调到不小于原图尺寸。
3. `SaveAndReimport()` 后读取像素、打包。
4. 在 `finally` 里恢复所有源图的导入设置并 `AssetDatabase.Refresh()`。

所以打包过程中会触发一次资源重导入，大图集耗时会明显变长；中途取消也会走恢复流程。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 控制图集大小 | 调小 `Max Atlas Size`，或减少贴图数量 |
| 避免图元粘连 | 增大 `Padding` |
| 固定输出位置 | 改 `Output Folder`，建议放 `Assets/Art/` 下 |
| 保留旧文件 | 关闭 `Overwrite Existing`，存在同名文件时会弹确认框 |
| 只要 PNG 不要 JSON | 目前不能关；JSON 一定会生成 |

## 容易踩的坑

1. **源图不是可读贴图**：压缩格式或平台设置导致读不到像素时，工具会临时改成 Uncompressed；如果项目有强制的纹理规范，打包后记得确认设置已恢复。
2. **输出目录不在 `Assets` 下**：Unity 不会导入，`AssetDatabase` 相关步骤会失败。
3. **同名覆盖**：`Overwrite Existing` 关闭时只弹确认框，不会自动改名。
4. **图集超过 `Max Atlas Size`**：`PackTextures` 会尽量塞，塞不下会返回失败或产生超大图；减少贴图或调大上限。
5. **把生成的图集再交给图集工具**：两份工具都管 `Sprite` 导入，容易互相覆盖设置；生成的图集应放到独立的 `Assets/TexturePackerOutput` 或 `Assets/Art` 目录。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `UnityTexturePackerEditor` | 窗口类，仅编辑器 |
| `Build Atlas` 按钮 | 执行 `BuildAtlas()`：临时改导入 → 打包 → 写 PNG/JSON → 恢复 |
| `Max Atlas Size` | 1024 / 2048 / 4096 |
| 输出 | `<Output Folder>/<Atlas Filename>` 与 `<Output Folder>/<JSON Filename>` |

相关：[图集工具](atlas.md)｜[UI 界面](../ui/index.md)
