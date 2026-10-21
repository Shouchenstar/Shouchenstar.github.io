# 个人博客

基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 的静态博客。

## 写一篇新文章

```bash
# 生成草稿（draft: true，本地能看到，线上不会发布）
hugo new content posts/my-new-post.md
```

然后编辑 `content/posts/my-new-post.md`，写完把 front matter 里的 `draft` 改成 `false`。

也可以直接手动新建文件，只要包含这几行：

```yaml
---
title: "文章标题"
date: 2026-10-21
draft: false
tags: ["标签"]
categories: ["分类"]
summary: "一句话摘要，显示在列表里"
---
```

## 给文章分类

在文章 front matter 里加 `categories`：

```yaml
categories: ["论文"]
```

现在有四个分类：**论文**、**游戏评价**、**灵感**、**日志**。打开「文章」页，标题下面就是分类标签，点进去只显示该分类的文章。

新增一个分类，在 `content/categories/<分类名>/_index.md` 建文件：

```yaml
---
title: "分类名"
description: "一句话说明"
weight: 50    # 数字越小，标签排得越靠前
---

{{< category-tabs >}}
```

之后再在文章里写 `categories: ["分类名"]` 就行。不建这个文件也能用，只是分类页没有说明文字和标签栏。

## 新增一条图片

「图片」栏目分摄影、UI 设计两个分类块，都是「一张图一条目」，每条一个文件夹：

```
content/gallery/photography/my-photo/
  index.md     ← 标题、日期、图片名、地点
  cover.jpg    ← 图片本身
```

UI 设计的条目放在 `content/gallery/design/` 下，格式完全一样。

`index.md` 内容：

```yaml
---
title: "外滩的晚上"
date: 2026-10-21
image: "cover.jpg"
location: "上海"
---
```

正文写一小段说明（拍摄时的想法、设计思路、素材来源），不写也行。

- 图片放在同一文件夹里，`image` 写文件名；不写 `image` 时默认找 `cover.*`
- 支持 `.jpg` / `.png` / `.webp`：构建时会自动生成 720px 缩略图（列表用）和 1600px 大图（详情页用），不用自己压
- `.svg` 不做缩放，直接原样输出
- 文件夹名就是网址，用英文或数字，别用中文和空格
- 图片版权归自己或已获授权再用；建议长边 2000px 以内、单张 500KB 左右，构建会快很多

想再加一个分类块，在 `content/gallery/` 下新建文件夹和 `_index.md`：

```yaml
---
title: "分类名"
description: "一句话说明"
layout: "gallery"
weight: 30
cascade:
  layout: "gallery-item"
---
```

「图片」页里就会多出一块，`weight` 越小排得越靠前。

## 本地预览

```bash
hugo server -D
```

打开 http://localhost:1313 ，改文件会自动刷新。`-D` 表示把草稿也一起显示。

## 构建

```bash
hugo --minify
```

产物在 `public/`，已加入 `.gitignore`，不需要提交。

## 目录说明

```
content/posts/        文章（Markdown）
content/categories/   文章分类页（论文 / 游戏评价 / 灵感 / 日志）
content/gallery/      图片栏目：photography/ 摄影、design/ UI 设计
content/about.md      关于页
content/search.md     搜索页
layouts/              图片版式（gallery.html / gallery-item.html）与分类标签短代码
assets/css/extended/  图集和分类标签样式，PaperMod 会自动合并
static/               图片、_headers 等静态文件
themes/PaperMod/   主题源码（已内置，勿改，升级时整体替换）
hugo.toml          站点配置：站名、菜单、个人信息
.github/workflows/ GitHub Pages 自动部署
```

## 发文章

```bash
git add .
git commit -m "post: 文章标题"
git push
```

推送后 GitHub Pages 和 Cloudflare Pages 会各自自动构建发布。

## 部署配置备忘

### GitHub Pages

- 仓库 Settings → Pages → Source 必须是 **GitHub Actions**（不是分支发布）
- 构建由 `.github/workflows/hugo.yml` 完成，无需手动操作

### Cloudflare Pages

- 项目名：`shouchenstar-github-io`
- 构建配置：
  - Framework preset：`None`
  - Build command：`curl -sL https://github.com/gohugoio/hugo/releases/download/v0.168.0/hugo_extended_0.168.0_linux-amd64.tar.gz | tar -xz -C /tmp hugo && /tmp/hugo --minify`
  - 注意：不要加 `--baseURL "$CF_PAGES_URL/"`，它返回的是带哈希的部署地址，会把站内链接全部指错。baseURL 统一写在 `hugo.toml` 里
  - Build output directory：`public`
- 访问地址：`https://shouchenstar-github-io.pages.dev`
- 每次 push 到 `main` 自动部署；其他分支会生成预览地址
- 根目录的 `wrangler.jsonc` 是给 Workers 用的，Pages 不读取，暂时保留
- 换成自定义域名后，记得把命令里的 `--baseURL` 一起改掉

## 修改站点信息

- 站名、描述、作者：`hugo.toml` 的 `title` 和 `[params]`
- 首页大标题和简介：`[params.homeInfoParams]`
- 导航菜单：`[[menus.main]]`
- 头像：放一张 `static/avatar.jpg`，然后在 `[params.homeInfoParams]` 里加 `image = "avatar.jpg"`
