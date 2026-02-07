# Hugo使用速查表
这是一份围绕 Hugo 日常使用场景整理的速查表，适合已经安装好 Hugo、想要快速查命令和常见配置的你。

<!--more-->

## 1. 安装与创建站点

### 1.1 安装 Hugo（推荐扩展版）

- macOS（Homebrew）：

```bash
brew install hugo
# 或指定扩展版
brew install hugo --HEAD
```

- 验证版本：

```bash
hugo version
```

### 1.2 创建新站点

```bash
hugo new site my-blog
cd my-blog
```

常见目录：

- `content/`：文章内容
- `layouts/`：自定义模板
- `static/`：静态资源（图片、js、css）
- `hugo.toml|yaml|json`：站点配置

## 2. 常用命令速查

### 2.1 本地开发与预览

```bash
hugo server --buildDrafts --disableFastRender
# 简写
hugo server -D
```

- `-D / --buildDrafts`：包含 `draft: true` 的文章
- 默认监听 `http://localhost:1313`

### 2.2 生成静态文件

```bash
hugo
# 指定输出目录
hugo -d public
```

生成后的静态站点位于 `public/` 目录，可直接部署到任意静态托管平台。

### 2.3 新建内容

```bash
# 新建一篇博客文章
hugo new blog/my-first-post.md

# 新建一篇笔记
hugo new notes/my-note.md
```

新建的文章会带有默认 front matter，通常初始为 `draft: true`。

## 3. Front Matter 速记

Hugo 支持 YAML / TOML / JSON 形式的 front matter，以下为 YAML 示例：

```yaml
---
title: 我的第一篇文章
date: 2026-02-07T20:00:00+08:00
tags:
  - 随笔
  - Hugo
categories:
  - Blog
draft: false
---
```

常见字段：

- `title`：文章标题
- `date`：创建/发布日期
- `lastmod`：最后修改时间（配合 `enableGitInfo` 使用更佳）
- `tags` / `categories`：标签与分类
- `draft`：草稿标记

## 4. 多环境配置要点

### 4.1 基本配置示例（hugo.yaml）

```yaml
baseURL: https://example.com/
title: 我的博客
theme: hextra

enableRobotsTXT: true
enableGitInfo: true
hasCJKLanguage: true
```

常见问题：

- 本地预览 `baseURL` 无所谓，但部署到生产环境时应设置为真实网址，否则 RSS / Canonical URL 可能异常。

### 4.2 输出格式与 RSS

```yaml
outputs:
  home: [html, rss]
  section: [html, rss]
  page: [html]
```

## 5. 常见调试与排错

### 5.1 查看所有页面列表

```bash
hugo list all
```

可以快速确认某篇文章是否被 Hugo 识别、是否为 draft。

### 5.2 检查草稿 / 未来文章

```bash
hugo list drafts
hugo list future
```

如果线上站点看不到某篇文章，优先排查：

- 是否 `draft: true`
- `date` 是否在未来
- 是否在 `content/` 下的正确子目录

## 6. 与主题协同使用的小贴士

- 大部分主题会在文档中给出推荐的 front matter 字段，可以直接参照。
- 对于文档型内容，优先使用 `type: docs` 和 `_index.md + 子页面` 的方式组织。
- 熟悉 `params` 配置（如导航栏、搜索、评论系统等）能极大提升 Hugo 的使用体验。

## 7. 进一步学习

- 官方文档：https://gohugo.io/documentation/
- 主题站点示例与配置：在所用主题仓库的 `exampleSite/` 目录中寻找

