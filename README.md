# 量化投资与机器学习个人技术博客

这是我的个人技术博客：记录量化投资、机器学习、数学统计与 Python 工程化方向的学习、复现和实践。

线上地址：<https://jianglang740.github.io/my-blog/>

博客不是成熟的投研平台，也不提供任何投资建议。它更像一份持续更新的学习档案——把读过的理论、复现过的研报、写过的代码和踩过的坑沉淀下来，方便自己复盘，也希望能给同样在入门量化与机器学习的朋友一些参考。

## 内容版图

截至 2026 年 10 月，`content/post/` 下共有 70 篇长文，另有多因子、CTA 两个研究分区。

| 板块 | 路径 | 内容 |
| --- | --- | --- |
| Freqtrade 课程 | `/post/freqtrade/` | 30 课 36 篇，从环境搭建、回测、参数优化到实盘交易 |
| Freqtrade 策略库 | `/post/freqtrade-strategies/` | 21 个策略：12 个自研、5 个第三方精选、4 个故意保留未来函数的反面教材 |
| 华泰多因子研报复现 | `/post/华泰证券多因子系列研报复现项目/` | 4 篇，从因子体系理论、架构设计到数据预处理与策略实现 |
| 技术栈 | `/post/技术栈/` | 7 篇，BackTrader、CCXT、Streamlit、MySQL、项目结构等工具实践 |
| 数学教程 / 机器学习 | `/post/数学教程/`、`/post/机器学习/` | 统计量的矩视角、线性回归等基础内容 |
| 多因子研究 | `/factor/` | 研究方向仪表盘 + 轻量笔记 |
| CTA 研究 | `/cta/` | 研究方向仪表盘 + 轻量笔记 |
| 项目 | `/projects/` | 4 个项目展示页 |
| 书籍 | `/books/` | 书架：封面、简介与 PDF 在线阅读/下载 |

多因子和 CTA 不是简单的分类页，而是两套**研究仪表盘**：按研究方向组织笔记，展示每个方向的当前问题、最新结论、笔记数量与完成进度；笔记按 `group` 自动归类，数字全部由 Hugo 在构建时从页面计算，不写死。

- 多因子方向：已有因子、数据与预处理、因子构建与检验、收益预测与组合、回测与归因、研究工程化
- CTA 方向：宏观研究、趋势跟踪、截面动量、波动率与风险、组合与资金管理、执行与监控

## 站点功能

- **首页统计仪表盘**（`/`）：总文章数、总码字量、运行天数、分类分布、热门标签、年度贡献热力图、周/时段发布活跃度
- **文章列表**（`/blog/`）：卡片式文章流，按发布时间倒序分页
- **研究仪表盘**（`/factor/`、`/cta/`）：研究方向面板、KPI 统计、最近更新、相关长文
- **独立笔记分区**：研究笔记存放在 `content/factor/`、`content/cta/`，不进入 `/blog/`、归档和主 RSS，但已纳入站内搜索
- **书籍页**（`/books/`）：书籍卡片、封面上传、PDF 在线预览与下载
- **站内搜索**（`/search/`）：索引覆盖 `post`、`factor`、`cta`、`books` 四个分区
- **Freqtrade 课程与策略库**：课程章节与策略页各自有独立入口和目录
- 深浅色主题切换、Waline 评论、全文 RSS、KaTeX 数学公式（页面 front matter 中 `math: true` 开启）

## 技术栈

| 组件 | 说明 |
| --- | --- |
| [Hugo](https://gohugo.io/) Extended | 静态站点生成，本地开发使用 0.161.1 |
| [Hugo Theme Stack v4](https://github.com/CaiJimmy/hugo-theme-stack) | v4.0.0-beta.15，以 Hugo Module 方式引入 |
| Go | 1.25，用于管理主题模块 |
| Dart Sass | 编译主题与自定义 SCSS |
| GitHub Actions + GitHub Pages | 自动构建、部署与托管 |
| Waline | 评论与阅读量统计 |

项目没有前端框架，也没有单独的前端构建流程：Hugo 负责渲染页面、编译 SCSS 和打包资源，产物是纯静态文件。

## 目录结构

```text
.
├── archetypes/                  # 内容模板：研究笔记、因子条目等
├── assets/
│   ├── icons/                   # 侧边栏与自定义图标
│   ├── img/                     # 头像、favicon
│   └── scss/                    # 主题变量覆盖与自定义组件样式
├── config/_default/             # 站点、语言、菜单、主题参数
├── content/
│   ├── post/                    # 长文：freqtrade、策略库、华泰研报、技术栈、数学、机器学习
│   ├── factor/                  # 多因子研究方向与轻量笔记
│   ├── cta/                     # CTA 研究方向与轻量笔记
│   ├── books/                   # 书籍 page bundle（index.md + 封面 + PDF）
│   ├── projects/                # 项目展示
│   ├── blog/                    # /blog/ 文章列表页
│   ├── about/                   # 关于我
│   ├── archives/                # 归档
│   ├── search/                  # 搜索页
│   ├── stats/                   # 统计页
│   └── categories/              # 分类页
├── layouts/
│   ├── _partials/               # 统计仪表盘、研究仪表盘、侧边栏、页脚等
│   ├── _default/stats.html      # 统计页入口
│   ├── home.html                # 根路径 → 统计仪表盘
│   ├── blog/list.html           # /blog/ 文章列表
│   ├── factor/list.html         # 多因子仪表盘
│   ├── cta/list.html            # CTA 仪表盘
│   ├── books/                   # 书架与单本书目布局
│   ├── post/                    # 课程与策略库的章节/列表布局
│   ├── projects/list.html       # 项目列表
│   ├── page/search.json         # 搜索索引（含研究笔记）
│   └── shortcodes/              # title、timeline 等自定义 shortcode
├── i18n/zh.toml                 # 中文文案
├── static/                      # 静态资源
├── .github/workflows/deploy.yml # GitHub Pages 自动部署
└── 使用说明.md                  # 维护手册：路由规则、内容管理、排错
```

## 本地运行

需要安装：

- Hugo Extended（本地实测 0.161.1）
- Go 1.25 及以上（Hugo Module 需要）
- Dart Sass

克隆并启动：

```bash
git clone https://github.com/jianglang740/my-blog.git
cd my-blog
hugo mod download
hugo server
```

预览地址：<http://localhost:1313/>

构建静态文件：

```bash
hugo --gc --minify
```

产物在 `public/`，该目录已在 `.gitignore` 中，不需要提交。

## 写作与内容维护

### 普通长文

```bash
hugo new content post/技术栈/新文章.md
```

### 研究笔记

多因子与 CTA 的笔记使用独立模板，front matter 需要填写归属方向和状态：

```bash
hugo new content factor/因子去极值.md
hugo new content cta/我国期货市场现状.md

# 写「已有因子」条目时，用带固定正文骨架的模板
hugo new content --kind factor-library factor/momentum-12-1.md
```

| 字段 | 说明 |
| --- | --- |
| `group` | 归属方向 id，必须与 `_index.zh.md` 中 `directions[].id` 一致 |
| `status` | `exploring` / `in-progress` / `done` / `paused` |
| `summary` | 一句话结论，直接显示在仪表盘上 |

研究方向本身在 `content/factor/_index.zh.md`、`content/cta/_index.zh.md` 的 `directions` 里增删改。`group` 写错时构建会给出警告，页面上归入「未归类」。

### 书籍

书籍使用 page bundle，一本书一个目录：

```text
content/books/书名/
├── index.md      # 书名、作者、简介
├── cover.jpg     # 封面，可选；缺省时显示书名占位
└── book.pdf      # 正文 PDF
```

```bash
hugo new content books/书名/index.md
```

`author`、`publishYear`、`publisher`、`language`、`summary`、`tags` 会显示在书架和书目页；`cover`、`pdf` 默认指向 `cover.jpg`、`book.pdf`，文件名不同时改这两个字段即可。

PDF 按静态文件原样发布，Hugo 不会索引其内部文字，所以书名和简介要写在 `index.md` 里才能被站内搜索到。GitHub Pages 的站点上限约 1GB、单文件上限 100MB，单本建议控制在 80MB 以内。

更多细节（URL 规则、索引机制、常见问题）见 [使用说明.md](使用说明.md)。

## 部署

推送到 `main` 分支后，GitHub Actions 会自动安装 Hugo、Go 和 Dart Sass，执行 `hugo --gc --minify`，再把 `public/` 发布到 GitHub Pages。流程见 [.github/workflows/deploy.yml](.github/workflows/deploy.yml)。

迁移到新仓库时需要检查：

1. `config/_default/config.toml` 的 `baseurl` 是否与 GitHub Pages 地址一致
2. 仓库 Pages 的部署来源是否设为 GitHub Actions
3. `config/_default/params.toml` 中的 Waline `serverURL` 是否已替换为自己的服务
4. CI 安装的是最新版 Hugo Extended，与本地版本可能存在差异，升级主题后需要留意构建结果

## 关于我

我叫蒋浪，来自山东财经大学，对机器学习、量化投资、数学建模和各类技术栈都很感兴趣。

这个博客会持续记录我的学习见解、项目复现和实践总结。如果文章能帮到你，欢迎交流；如果有错误，也欢迎指出。

## 免责声明与许可

博客中涉及的量化策略、金融数据分析和回测内容仅用于学习与研究，不构成任何投资建议。市场有风险，投资需谨慎。

文章内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议；仓库代码遵循根目录 [LICENSE](LICENSE)（GPL-3.0）中的条款。
