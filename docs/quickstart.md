# 快速开始

从安装框架、接入启动场景到跑通第一个界面，这一页按顺序走完。更细的模块 API 见左侧各模块文档。


## 安装

<div class="steps" markdown>

1. **添加 Installer**：Unity Package Manager → `+` → Add package from git URL，填 `https://github.com/AlicizaX/FramworkInstaller.git`。

2. **打开安装器**：菜单栏 `AlicizaX / Installer`，会自动检测并补齐所需的 OpenUPM Scoped Registry 和 Scopes。

3. **安装核心库**：点 `Install Core` 安装 `com.alicizax.unity.framework`，装好后模板入口才会解锁。

4. **选择模板**：`Normal` 用于标准项目开发，`Hybrid` 用于带热更工程的混合模式。

5. **按需加扩展包**：UI 扩展、调试面板、多语言都是独立 UPM 包，见下表，用不到可以不装。

</div>


## 可选扩展包

| 包 | 用途 | 建议 |
| :--- | :--- | :--- |
| `com.alicizax.unity.framework` | 核心框架：服务容器、资源、UI、事件、计时、对象池 | **必装** |
| `com.alicizax.unity.ui.extension` | UI 扩展组件：`UXButton`、`UXImage`、`RecyclerView` 等 | 有 UI 就装 |
| `com.alicizax.unity.editor.extension` | 编辑器扩展工具（Aliciza UI Toolkit 控件、快捷面板） | 推荐 |
| `com.alicizax.unity.input` | 输入扩展：设备监听、快捷键、UI 导航、按键图标 | 需要手柄/键盘导航时 |
| `com.alicizax.unity.debugger` | 运行时调试面板：日志、资源、内存池、音频、计时器、Event Monitor | 开发期推荐 |
| `com.alicizax.unity.localization` | 基于 Luban 的多语言运行时、TMP 绑定与编辑器预览 | 多语言项目 |
| `com.alicizax.unity.installer` | 安装器本身：装核心包、导入模板 | 安装期需要 |

依赖关系要注意两点：

- `com.alicizax.unity.input` 依赖 Unity **Input System**，装之前先启用该包，否则输入类扩展（`Navigation`、`HotkeyComponent`、`InputGlyph`）不可用；
- 需要手柄/键盘导航的列表（`RecyclerView` 的 `UXNavigationScope`）同样依赖 input 包，只做触屏可以跳过。


## 三步上手

<div class="steps" markdown>

1. **跑通启动链路**：按下面的「启动链路」从 RootModule 启动开始走一遍，先让第一个界面亮起来。

2. **按需求挑模块**：每篇文档都是「使用前提 → 用法 → API 速查 → 注意事项」四段式，先确认前提组件挂上了，再抄示例。

3. **接入自己的流程**：注册自定义服务、打开业务 UI、申请资源租约；需要热更时再按 HybridCLR 的流程接。

</div>

## 启动链路

项目当前的推荐启动顺序如下：

1. 启动场景中放置框架根节点，挂载 `RootModule` 和各模块 `Component`。
2. `RootModule.Awake()` 创建 `ServiceWorld`，之后各模块组件在 `Awake()` 中注册服务。
3. `ResourceComponent.Awake()` 初始化 YooAsset 并创建默认资源包。
4. `ProcedureEntry.Start()` 等待 `YooAssets.Initialized`，然后注册启动流程。
5. 启动流程完成资源包初始化、版本检查、资源下载、缓存清理。
6. 如果开启 HybridCLR，加载热更程序集并反射调用 `GameLogic.HotfixEntry.Entrance`。
7. 业务层通过 `GameApp` 或 `AppServices` 调用 UI、资源、音频、场景等模块。

项目已有入口代码：

```csharp
using System.Collections.Generic;
using AlicizaX;
using Cysharp.Threading.Tasks;
using UnityEngine;

namespace Unity.Startup.Procedure
{
    public class ProcedureEntry : MonoBehaviour
    {
        private async UniTaskVoid Start()
        {
            await UniTask.WaitUntil(() => YooAsset.YooAssets.Initialized);

            ProcedureBuilder.InitializeProcedure(
                new List<ProcedureBase>
                {
                    new ProcedureEntryState(),
                    new ProcedureGetAppVersionInfoState(),
                    new ProcedureInitPackageState(),
                    new ProcedureDownloadBundleState(),
                    new ProcedurePatchDoneState(),
#if ENABLE_HYBRIDCLR
                    new ProcedureLoadAssembly(),
#endif
                    new ProcedureUpdateFinishState(),
                },
                typeof(ProcedureEntryState));

            Destroy(gameObject);
        }
    }
}
```
