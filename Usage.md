# Usage

这是这个 Hexo 博客仓库的维护和写作说明。

## 环境要求

- Node.js 20
- npm
- Git

建议直接使用和 GitHub Actions 一致的 Node.js 版本，避免本地和 CI 构建结果不一致。

## 首次使用

1. 克隆仓库

```bash
git clone https://github.com/ZHLHZHU/ZHLHZHU.github.io.git
cd ZHLHZHU.github.io
```

2. 安装依赖

```bash
npm ci
```

如果只是本地临时折腾，`npm install` 也可以；如果想和 CI 保持一致，优先用 `npm ci`。

3. 启动本地预览

```bash
npm run server
```

默认会启动本地 Hexo 服务，通常访问 [http://localhost:4000](http://localhost:4000) 就能看到页面。

## 常用命令

```bash
# 清理缓存和旧的生成文件
npm run clean

# 生成静态页面
npm run build

# 本地启动预览服务
npm run server
```

如果本机装了 Hexo CLI，也可以直接用这些命令：

```bash
npx hexo clean
npx hexo generate
npx hexo server
```

## 新写一篇文章

可以直接手动在 `source/_posts` 下面新建 Markdown 文件，也可以用 Hexo 命令生成：

```bash
npx hexo new post "文章标题"
```

这个仓库当前使用了：

- `post_asset_folder: true`
- `marked.postAsset: true`

所以一篇文章通常对应这两部分：

- `source/_posts/xxx.md`
- `source/_posts/xxx/`，用于放图片等资源

例如：

```text
source/_posts/fix-linptech-es3.md
source/_posts/fix-linptech-es3/es3.webp
```

正文里直接这样引用图片即可：

```md
![封面](es3.webp)
```

## Front Matter

文章头部建议至少包含这些字段：

```yaml
---
title: 文章标题
author: AAA智能家居专修
date: 2026-05-31 17:31:46
categories: [折腾]
tags: [Hexo, 博客]
---
```

如果需要摘要，可以在正文里加入：

```md
<!--more-->
```

## 发布流程

这个仓库已经配置好了 GitHub Actions，工作流文件是 `.github/workflows/pages.yml`。

当前发布逻辑如下：

1. 推送到 `main`
2. GitHub Actions 自动执行 `npm ci`
3. 自动执行 `npm run build`
4. 将 `public` 目录作为构建产物上传
5. 自动部署到 GitHub Pages

也就是说，日常更新文章的流程通常就是：

```bash
git add .
git commit -m "feat: update posts"
git push origin main
```

推送完成后，等 Actions 跑完就会自动更新线上站点。

## 目录说明

```text
.
|-- source/
|   |-- _posts/          # 博文和文章资源
|   |-- categories/      # 分类页
|   `-- tags/            # 标签页
|-- scaffolds/           # Hexo 模板
|-- _config.yml          # Hexo 主配置
|-- _config.next.yml     # NexT 主题配置
|-- package.json         # 脚本和依赖
`-- .github/workflows/   # GitHub Actions
```

## 备注

- 本地生成结果在 `public`
- 一般不需要手动提交 `public`
- 如果页面样式或配置有变动，优先检查 `_config.yml` 和 `_config.next.yml`
