# 大内密探 · peterhu.github.io 重构 Coding Plan

> 定位：英文为主 + 中文归档。保留 Jekyll 3.8.5 + tmaize-blog 主题。
> 状态：待 review，用户批准后再动手。
> 注意：本文件只是计划，部署前删除或加入 `_config.yml` 的 `exclude`。

## 现状（已确认）

- Jekyll 3.8.5 + tmaize-blog 主题（手搓极简主题，无框架）
- `_posts/` = 文章 markdown（~33 篇中文，2020~2021 + 1 篇 2026 串口乱码）
- `posts/` = 文章图片（按 `YYYY/MM/DD` 嵌套，255 张，theme 约定，暂不动）
- 中文市场残留 + 品牌未换 + 根目录杂物

---

## Phase 1 · 清理残留（删 + 关）

1. **删根目录杂物**：
   - `2020-01-20-DDR Memory.md`（与 `_posts/2020-01-20-DDR Memory工作原理.md` 重复的遗留）
   - `cli.bat`（中文注释已乱码的 Windows 启动脚本）
   - `CNAME`（空文件，用户站不需要）
2. **删中文互动页**：`pages/chat.html`（留言）、`pages/links.html`（友链）
3. **删中文站残留 include**：`_includes/ext-adsense.html`、`ext-baidu.html`、`ext-mta.html`、`ext-mathjax.html`
4. **`_config.yml` 关开关**：`extClickEffect`/`extMTA`/`extBaidu`/`extAdsense`/`extMath` 全设 `false`；删 `tucaoUrl`（吐糟）；删 `links`（友链）
5. **`_includes/script.html`**：删掉点击特效脚本（富强/民主…那 12 个字）、extMTA/extBaidu/extMath 的 include 引用

## Phase 2 · 换品牌（英文）

1. **`_config.yml`**：
   - `title`、`description`、`keywords`、`author` 改英文
   - `menu` 改英文：Home /、Categories、Search、About（删留言/友链）
   - `footerText` 保留（已是英文 "Powered by My Mind"）
2. **`_includes/head.html`**：`content-language` 从 `zh-CN` → `en`
3. **`_includes/footer.html`**：`RSS订阅` → `RSS`
4. **`pages/about.md`**：重写英文自我介绍（18 年固件 / 内存训练专家）
5. **`pages/categories.html`**：`文章分类`→`Categories`、`所有分类`→`All Categories`
6. **`pages/search.html`**：中文文案改英文（如有）

## Phase 3 · 内容分区（英文为主 + 中文归档）

1. **老中文文批量加 front matter**：每篇加 `lang: zh`（~33 篇，机械批量，可用脚本）
2. **顺手修一个小 bug**：`_posts/2020-11-08- PCIe 体系结构简介.md` 文件名前多了个空格，去掉
3. **新英文案卷**：`lang: en` + `categories: [DDR5]` + `layout: mypost`（大内密探案卷系列）
4. **`index.html` 改**：首页只列英文文（`{% unless post.lang == 'zh' %}` 过滤），中文不进首页
5. **新增 `pages/archive-zh.html`**：中文旧文归档页（列出所有 `lang: zh` 的 post）
6. **`_config.yml` menu 加一项**：Archive → `/pages/archive-zh.html`

## Phase 4 · 加动图区

1. 新增 `animations/` 目录，放 HTML 动图（自包含，直接可访问）
2. 帖子 / 首页 / about 里链接动图
3. （可选）`static/img/` 换 logo / favicon

## Phase 5 · 部署

- 当前 `D:\Code\githubio\peterhu.github.io-master` 是 **ZIP 解压、非 git 仓库**
- 需要：`git init` + `git remote add origin https://github.com/peterhu/peterhu.github.io.git` + 提交 + push
- **网络风险**：之前 git clone 到 GitHub 被 `Connection reset`（VPN 没代理 git 流量），push 可能同样失败，需 VPN 或走 web 上传
- push 前删除本 `CODING_PLAN.md`，或加入 `_config.yml` 的 `exclude`

---

## 需要你拍板的 3 个决定

| # | 决定 | 我的建议 |
|---|---|---|
| 1 | 站点 `title` 用什么？ | `Forbidden City Cop — DDR5 Memory Training Case Files`（让 DDR5 博客成为站点身份，author 用 Peter Hu）；或保守用 `Peter Hu · Memory Firmware Engineer` |
| 2 | 首页英文列表：只列英文，还是英文 + 中文分标签？ | 只列英文（中文全进 Archive），最干净 |
| 3 | `about.md` 英文自我介绍：要不要我把你现有中文简历翻译成英文？ | 我翻译成英文，突出「内存训练专家」定位 |

## 不做的事（明确边界）

- 不删任何老中文文内容，只加 `lang: zh` 标记 + 移到归档入口
- 不动 `posts/` 图片目录（改了会断 255 张图链接）
- 不换主题、不升级 Jekyll 版本（避免引入新问题）
- 老中文文**不迁移**到知乎/CSDN（除非你另行要求）
