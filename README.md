# AlicizaX Wiki

AlicizaX 框架的在线文档站，基于 **MkDocs + Material for MkDocs** 构建，部署在 GitHub Pages。

- 线上地址：<https://b4lie.github.io/AlicizaXWiki/>
- 框架仓库：[AlicizaX/AlicizaXTemplate](https://github.com/AlicizaX/AlicizaXTemplate)

## 本地预览

需要 Python 3.9+。

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows PowerShell
# source .venv/bin/activate     # macOS / Linux

pip install -r requirements.txt
mkdocs serve                    # http://127.0.0.1:8000/AlicizaXWiki/
```

> 注意本地地址带 `/AlicizaXWiki/` 前缀，因为 `site_url` 指向项目子路径，本地预览和线上路径保持一致，
> 这样相对链接、静态资源在本地就能验证到位。

国内网络如果 PyPI 慢，可加镜像参数：

```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

## 构建

```bash
mkdocs build --strict
```

`--strict` 会把所有告警（页面不在导航里、相对链接失效等）升为错误，CI 用的就是这条命令。

## 目录结构

```text
.
├─ mkdocs.yml                  # 站点配置：主题、导航、扩展、中文搜索
├─ requirements.txt            # 依赖锁
├─ docs/
│  ├─ index.md                 # 首页
│  ├─ QuickStart.md            # 以下 12 篇与框架仓库 Books/ 一一对应
│  ├─ Service.md / Timer.md / Event.md / Procedure.md
│  ├─ Resources.md / Scene.md / GameObjectPool.md / MemoryPool.md
│  ├─ UI.md / Audio.md / Localization.md
│  └─ assets/                  # logo、favicon、extra.css
└─ .github/workflows/deploy.yml # 推送到 main 自动构建并发布 Pages
```

## 部署

推送到 `main` 后，GitHub Actions 会自动执行 `mkdocs build --strict` 并把 `site/` 发布到 Pages。

仓库首次部署前需要在 **Settings → Pages → Source** 选择 **GitHub Actions**（本项目已配置）。
如果首次运行报 `Get Pages site failed`，等设置生效后在 Actions 页面重跑一次即可。

## 文档来源约定

`docs/` 下的 12 篇模块文档是框架仓库 `Books/` 目录的镜像，当前保持 1:1 一致，唯一差异是
`Event.md` 末尾两处指向不存在文件的链接已被降级为纯文本，避免站内 404。

修改文档内容时请注意两侧同步；如果后续决定以本仓库为唯一写入点，再调整 `Books/` 的同步策略。

## 双语支持

首版仅提供简体中文。`mkdocs.yml` 末尾已保留 `mkdocs-static-i18n` 的配置骨架与启用步骤，
后续加英文版时按注释迁移到 `docs/zh/` + `docs/en/` 即可，不需要重构导航。

## 技术栈说明

- **MkDocs 1.6 + Material for MkDocs 9.8**，版本在 `requirements.txt` 中锁定
- **中文搜索**：`theme.language: zh` 配 `jieba` 分词（构建期往索引里插零宽空格，查询期由 `tinyseg` 还原分词），
  已验证「对象池」「热更」「MemoryPool」等中英文关键词都能命中
- **字体**：关闭 Google Fonts，改用系统字体栈（Segoe UI / 微软雅黑 / 苹方），国内访问不依赖外部 CDN
- **图标**：使用 Material 内置的 Material Icons / Octicons，无额外资源请求

> Material for MkDocs 上游已公布于 **2027-05-05 停止维护（EOL）**，同团队的新一代 **Zensical** 兼容现有
> `mkdocs.yml`，届时可按 [迁移指南](https://zensical.org/docs/compatibility/mkdocs/migration/) 平滑切换；
> 在此之前当前版本仍会收到关键修复与安全更新。
