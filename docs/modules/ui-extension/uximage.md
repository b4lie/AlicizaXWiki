# UXImage

`UXImage` 继承 UGUI 的 `Image`，在同一套网格上增加四类能力：纯色/渐变、描边与阴影（自适应效果）、翻转/镜像、运行时图集采样。它保留 `Image` 的 `sprite`、`type`、`fillAmount`、`preserveAspect` 等全部行为。

## 适用场景

- 纯色底图或双色渐变底图，不想额外叠一张渐变贴图。
- 给图标加描边、投影，且希望阴影在放大/缩小时保持柔和（SDF 路径）。
- 同一张图需要水平/垂直/四角翻转，或做镜像拼接。
- 大量小图需要运行时合图采样，减少 draw call。

不适用：九宫格/平铺网格的复杂效果（`Sliced`/`Tiled` 会回退到普通贴图路径，见下文限制）。

## 使用前提

- 已安装 `com.alicizax.unity.ui.extension`（`UXImage` 在命名空间 `UnityEngine.UI`）。
- 挂载节点上已有 `CanvasRenderer`；`UXImage` 与 `Image` 一样需要父级 Canvas。
- 启用描边/阴影时，项目需要包含 `UI/UXImageAdaptiveEffect` 着色器（随包提供）；缺失时效果自动降级为普通贴图。
- 使用运行时图集时，Sprite 需要能按 `Sprite` 读取（图集/散图均可，走 GPU 采样，不要求 Read/Write）。

## 最短接入

1. 选中 UI 节点，`Add Component` → `UXImage`（或把已有 `Image` 替换为 `UXImage`）。
2. 设置 `Source Image`，按需选择 `Color Type`。
3. 需要描边/阴影时勾选 `Enable Outline` / `Enable Shadow`，调颜色、距离、Softness。
4. 需要翻转时设置 `Flip Mode` 与对应边。

```csharp
using UnityEngine;
using UnityEngine.UI;

public sealed class IconView : MonoBehaviour
{
    [SerializeField] private UXImage _icon;

    public void Show(Sprite sprite, bool highlighted)
    {
        _icon.sprite = sprite;
        _icon.enableOutline = highlighted;
        _icon.outlineEffectColor = Color.black;
        _icon.outlineEffectDistance = new Vector2(1.5f, 1.5f);
        _icon.outlineSoftness = 0.5f;

        // 水平翻转（镜像）
        _icon.flipMode = UXImage.FlipMode.Horziontal;
        _icon.flipEdgeHorizontal = UXImage.FlipEdgeHorizontal.Right;
    }
}
```

> 枚举名 `Horziontal` 是源码里的拼写，必须照写，写成 `Horizontal` 编译不过。

## 颜色模式

| 字段/属性 | 取值 | 说明 |
| :--- | :--- | :--- |
| `m_ColorType` | `Solid_Color` / `Gradient_Color` | 纯色或渐变 |
| `gradient` | `Gradient` | 渐变数据源；**替换引用**会自动刷新 |
| `SetGradientDirty()` | 方法 | 就地修改 `gradient` 的 `colorKeys` / `alphaKeys` / `mode` 后必须调用 |
| `Direction` | `Vertical` / `Horizontal` | 渐变方向 |

`Image.color` 仍然生效，会与渐变结果相乘（用于整体淡入淡出、置灰）。

## 描边与阴影

| 属性 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `enableOutline` | false | 开启描边 |
| `outlineEffectColor` | 黑 | 描边颜色 |
| `outlineEffectDistance` | `(1.5, 1.5)` | 描边偏移 |
| `outlineSoftness` | 0.5 | 描边柔化 |
| `enableShadow` | false | 开启阴影 |
| `shadowEffectColor` | 黑（50% alpha） | 阴影颜色 |
| `shadowEffectDistance` | `(2, -2)` | 阴影偏移 |
| `shadowSoftness` | 1.5 | 阴影柔化；`>= 8` 时切到 SDF 路径 |
| `useGraphicAlpha` | true | 把 `Graphic` 的 alpha 乘进效果 |

两条渲染路径：

- **普通路径**：直接扩展网格采样原贴图，适合大多数描边和轻阴影。
- **运行时 SDF 路径**：`shadowSoftness >= 8` 且其他条件满足时，GPU 把阴影烘到 2048×2048 的 SDF 图集页，再映射回图集 UV。烘焙是异步的，首次会先按普通路径渲染，完成后重建网格。

## 翻转与镜像

`Flip Mode` 决定翻转轴：

| 值 | 含义 |
| :--- | :--- |
| `None` | 不翻转 |
| `Horziontal` | 水平翻转 |
| `Vertical` | 垂直翻转 |
| `FourCorner` | 四角翻转（默认） |

配套字段：

- `flipWithCopy`：翻转时是否复制一份几何（用于镜像拼接）。
- `flipEdgeHorizontal`：`Left` / `Middle` / `Right`，水平翻转的基准边，默认 `Right`。
- `flipEdgeVertical`：`Up` / `Middle` / `Down`，垂直翻转的基准边，默认 `Down`。
- `flipEdge`：统一入口，按当前 `flipMode` 映射到水平或垂直边；`flipMode` 为 `None` 时返回 `FlipEdge.None`。
- `flipDirection`：另一套 `FlipDirection` 枚举（`None` / `Horziontal` / `Vertical` / `FourCorner` / `HorizontalNotCopy` / `VerticalNotCopy` / `DiagonalNotCopy`），默认 `FourCorner`。

新代码优先用 `flipMode` + `flipEdge*`；`flipDirection` 与它并存，改动都会触发 `SetVerticesDirty()`。

## 运行时图集与限制

`UXImage` 在满足条件时把 Sprite 复制进运行时图集（`UXImageRuntimeAtlas`），从而让多张小图共享材质：

- 只对 **`Simple` / `Filled` 且网格恰好 4 个顶点**的 Image 生效。
- `Sliced` / `Tiled` 继续走普通贴图路径，避免 UV 落入图集 padding 被裁掉。
- 图集页 2048×2048（ARGB32，约 16 MiB/页），最多 4 页；SDF 页最多 2 页。达到上限时静默回退普通路径，不抛异常、不超额分配。
- 单张 Sprite 可用上限为 `2048 - 2 × 128 = 1792` 像素，超出则回退。

手动回收：

```csharp
// 大版本 UI 切换或场景卸载后调用，释放图集页、SDF 页、缓存网格与共享材质
UXImage.ReleaseRuntimeEffectCaches();
```

材质按 `(Shader, SDF 贴图, 描边颜色, 阴影颜色)` 共享，缓存上限 32 个；状态相同的 Graphic 可以合批。

## 常见配置

| 需求 | 做法 |
| :--- | :--- |
| 整体变灰/变暗 | 改 `Image.color`，不要改渐变关键帧 |
| 渐变随动画变化 | 修改 `gradient` 后调 `SetGradientDirty()` |
| 关闭所有特效 | `enableOutline = false; enableShadow = false;` 材质会切回默认 |
| 大图不做运行时合图 | 保持 `Simple` 但超过 1792 像素会自动回退 |
| 九宫格按钮 | 用 `Sliced`，效果走普通路径，可接受再开 |

## 容易踩的坑

1. **就地改渐变不刷新**：必须调 `SetGradientDirty()`，只改 `gradient.colorKeys` 不会重新缓存。
2. **九宫格上期待 SDF 阴影**：`Sliced`/`Tiled` 不进入运行时图集，阴影走普通路径，柔化效果不同。
3. **单图超过 1792 像素**：静默回退，不会报错，但 draw call 合批失效。
4. **效果着色器被裁剪**：打包时 `UI/UXImageAdaptiveEffect` 没被引用会丢失，表现为描边/阴影消失；把它加入 Always Included Shaders 或用材质引用保底。
5. **频繁创建/销毁带效果的 Image**：共享材质与图集页是静态缓存，长期不释放会占显存；在场景切换后调 `ReleaseRuntimeEffectCaches()`。
6. **把 `flipMode` 写成 `Horizontal`**：源码拼写是 `Horziontal`。

## API 速查

| 成员 | 说明 |
| :--- | :--- |
| `gradient` / `SetGradientDirty()` | 渐变数据与刷新 |
| `Direction` | 渐变方向 |
| `enableOutline` / `outlineEffectColor` / `outlineEffectDistance` / `outlineSoftness` | 描边 |
| `enableShadow` / `shadowEffectColor` / `shadowEffectDistance` / `shadowSoftness` | 阴影 |
| `useGraphicAlpha` | 是否乘 Graphic alpha |
| `flipMode` / `flipWithCopy` / `flipEdgeHorizontal` / `flipEdgeVertical` / `flipEdge` | 翻转 |
| `flipDirection` | 另一套翻转枚举 |
| `ReleaseRuntimeEffectCaches()` | 释放运行时图集/SDF/材质缓存 |

相关：[UXSelectable 基础](uxselectable.md)｜[UI 界面](../ui/index.md)｜[Resources 资源](../resource/index.md)
