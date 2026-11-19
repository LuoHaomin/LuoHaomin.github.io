# Hugo主题——Hextra
Hextra 是一个专注文档与博客体验的 Hugo 主题，支持全文搜索、暗色模式、文档侧边栏、短代码等“开箱即用”的功能，非常适合用来搭建知识库 + 博客的组合站点。

<!--more-->

本文结合我自己的博客实践，按“从安装到上线”的顺序梳理使用 Hextra 的关键步骤。

## 1. 准备一个 Hugo 站点

如果你还没有 Hugo 站点，可以先创建一个：

```bash
hugo new site my-site
cd my-site
```

建议 Hugo 使用扩展版（extended），方便处理 Tailwind / PostCSS 等相关功能。

## 2. 获取 Hextra 主题

一般有两种常见方式：

### 2.1 作为 Git 子模块引入

```bash
git init
git submodule add https://github.com/imfing/hextra.git themes/hextra
```

然后在 `hugo.toml` 或 `hugo.yaml` 中指定主题：

```yaml
theme: hextra
```

### 2.2 使用 Hugo Modules（推荐）

如果你熟悉 Go Modules / Hugo Modules，可以在项目中启用模块支持，将 Hextra 作为远程模块引入。好处是升级主题和管理依赖更方便。

## 3. 基础配置示例

下面是一个精简过的 `hugo.yaml` 示例，展示与 Hextra 配合时常用的全局配置：

```yaml
baseURL: https://example.com/
languageCode: en-us
title: 我的 Hugo 站点
theme: hextra

enableRobotsTXT: true
enableGitInfo: true
hasCJKLanguage: true

outputs:
  home: [html]
  page: [html, markdown]
  section: [html, rss, markdown]

params:
  navbar:
    displayTitle: true
    displayLogo: true
    width: wide

  theme:
    default: system
    displayToggle: true

  search:
    enable: true
    type: flexsearch
    flexsearch:
      index: content
      tokenize: forward

  blog:
    list:
      displayTags: true
      sortBy: date
      sortOrder: desc
    article:
      displayPagination: true
```

可以根据自己的需求再添加 `footer`、`editURL`、`comments` 等配置。

## 4. 组织内容：博客区与文档区

Hextra 对“文档型内容”和“博客型内容”都有良好支持：

- 博客区：通常使用 `content/blog/` 目录，文章有 `date`、`tags`、`description` 和 `<!--more-->` 摘要分隔。
- 文档区：推荐使用 `_index.md + 子页面` 的结构，并在 front matter 中设置 `type: docs` 或通过 `cascade` 继承。

### 4.1 博客文章示例

```markdown
---
title: 我的第一篇 Hugo 文章
date: 2026-02-07T20:00:00+08:00
tags:
  - Hugo
  - 随笔
description: 用 Hugo + Hextra 写下的第一篇文章
---

这里是文章的开头内容，会出现在博客列表摘要中。

<!--more-->

这里是正文的剩余部分。
```

### 4.2 文档章节示例

```markdown
---
title: Bevy 完整教程
cascade:
  type: docs
---

欢迎来到教程首页，这里可以作为整个文档系列的入口说明。
```

子页面只需要正常写内容，Hextra 会自动在左侧生成侧边栏导航，并在右侧生成小标题目录。

## 5. 自定义导航栏与侧边栏

导航栏菜单通过 `menu.main` 配置，例如：

```yaml
menu:
  main:
    - name: 博客
      pageRef: /blog
      weight: 1
    - name: 笔记
      pageRef: /notes
      weight: 2
    - name: 关于
      pageRef: /about
      weight: 3
    - name: 搜索
      weight: 4
      params:
        type: search
```

侧边栏的主要内容会根据目录结构自动生成，你也可以通过：

- 在 `_index.md` 中设置 `weight` 控制顺序；
- 使用 `menu.sidebar` 添加额外链接（例如跳转到特定文档或外部参考）。

## 6. 常用增强功能

- **全文搜索**：开启 `params.search` 后，Hextra 默认使用 FlexSearch 在前端实现全文检索。
- **代码高亮与复制按钮**：通过 `markup.highlight` 和 `params.highlight.copy` 进行配置。
- **暗色模式**：`params.theme.default` 设置为 `system` 或 `dark`，并启用 `displayToggle`。
- **短代码**：如 `callout`、`cards`、`tabs` 等，可以让文档更美观易读。

## 7. 本地预览与部署

本地开发通常用：

```bash
hugo server --buildDrafts --disableFastRender
```

确认一切正常后，使用：

```bash
hugo
```

在 `public/` 目录中得到最终的静态站点，然后部署到 GitHub Pages、Netlify 之类的平台即可。

---

总体体验下来，Hextra 对“文档 + 博客”这种混合场景非常友好：

- 博客区：列表清晰、支持标签与全文搜索；
- 文档区：侧边栏导航、右侧目录、短代码都很顺手；
- 配置上手成本不高，用少量 YAML 就能获得不错的默认效果。

如果你已经在用 Hugo，希望站点更适合写长期文档和教程，非常推荐尝试一下 Hextra。

