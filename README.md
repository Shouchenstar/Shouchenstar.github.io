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
content/posts/     文章（Markdown）
content/about.md   关于页
content/search.md  搜索页
static/            图片、_headers 等静态文件
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

推送后 GitHub Pages 和 Cloudflare Workers 会各自自动构建发布。

## 部署配置备忘

### GitHub Pages

- 仓库 Settings → Pages → Source 必须是 **GitHub Actions**（不是分支发布）
- 构建由 `.github/workflows/hugo.yml` 完成，无需手动操作

### Cloudflare Workers

- 仓库根目录的 `wrangler.jsonc` 指定了静态资源目录 `./public`
- Workers 构建配置：
  - Build command：`curl -sL https://github.com/gohugoio/hugo/releases/download/v0.168.0/hugo_extended_0.168.0_linux-amd64.tar.gz | tar -xz -C /tmp hugo && /tmp/hugo --minify --baseURL https://shouchenstar-github-io.2190298686.workers.dev/`
  - Deploy command：`npx wrangler deploy`
- 访问地址：`https://shouchenstar-github-io.2190298686.workers.dev`
- 注意：`*.workers.dev` 在国内被 DNS 污染，直连打不开，需要绑自定义域名才能稳定访问
- 换成自定义域名后，记得把命令里的 `--baseURL` 一起改掉

## 修改站点信息

- 站名、描述、作者：`hugo.toml` 的 `title` 和 `[params]`
- 首页大标题和简介：`[params.homeInfoParams]`
- 导航菜单：`[[menus.main]]`
- 头像：放一张 `static/avatar.jpg`，然后在 `[params.homeInfoParams]` 里加 `image = "avatar.jpg"`
