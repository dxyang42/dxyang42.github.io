# dxyang42's Blog

基于 [Hexo](https://hexo.io/) + [Butterfly](https://butterfly.js.org/) 主题搭建的个人博客，部署在 GitHub Pages 上。

访问地址：[https://dxyang42.github.io](https://dxyang42.github.io)

---

## 技术栈

- [Hexo 8.x](https://hexo.io/)：静态博客生成器
- [Butterfly 5.x](https://butterfly.js.org/)：博客主题
- [GitHub Pages](https://pages.github.com/)：静态站点托管

---

## 前置要求

本地需要安装：

- [Node.js](https://nodejs.org/)（建议 LTS 版本）
- [Git](https://git-scm.com/)

安装完成后，克隆本项目并安装依赖：

```bash
git clone https://github.com/dxyang42/dxyang42.github.io.git
cd dxyang42.github.io
npm install
```

> 注意：本项目的源码分支是 `source`，GitHub Pages 实际部署的是 `master` 分支（由 Hexo 自动生成）。

---

## 本地预览

运行以下命令启动本地服务器：

```bash
npm run server
```

默认访问 [http://localhost:4000](http://localhost:4000)。

修改文件后，Hexo 会自动刷新页面（部分配置修改需要重启服务器）。

---

## 项目结构

```
.
├── _config.yml              # 站点主配置（站点名称、URL、部署等）
├── _config.butterfly.yml    # Butterfly 主题配置（菜单、图片、侧边栏等）
├── package.json             # 项目依赖和脚本
├── source/
│   └── _posts/              # 博客文章目录
├── themes/                  # 主题目录（当前通过 npm 安装，此处为空）
└── public/                  # 生成的静态站点（自动生成，无需手动修改）
```

---

## 写文章

### 创建新文章

```bash
npx hexo new post "文章标题"
```

会在 `source/_posts/` 下生成一个 Markdown 文件，例如 `文章标题.md`。

打开文件后，顶部是文章的元信息（Front-matter）：

```markdown
---
title: 文章标题
date: 2026-07-08 15:00:00
tags:
  - Hexo
  - 博客
categories:
  - 技术
---

正文内容，支持 Markdown 语法。
```

常用 Front-matter 字段：

| 字段 | 说明 |
|------|------|
| `title` | 文章标题 |
| `date` | 发布时间 |
| `updated` | 更新时间 |
| `tags` | 标签（可多个）|
| `categories` | 分类 |
| `description` | 文章摘要（首页显示）|
| `cover` | 文章封面图链接 |
| `top_img` | 文章顶部大图 |

### 创建草稿

```bash
npx hexo new draft "草稿标题"
```

草稿不会发布，预览草稿需要：

```bash
npx hexo server --draft
```

---

## 修改站点配置

站点主配置文件是 `_config.yml`，常用修改项：

```yaml
# 站点信息
title: dxyang42's Blog
subtitle: ''
description: ''
author: dxyang42
language: zh-CN

# 站点地址
url: https://dxyang42.github.io

# 主题
theme: butterfly

# 分页
per_page: 10

# 部署配置
deploy:
  type: git
  repo: https://github.com/dxyang42/dxyang42.github.io.git
  branch: master
```

---

## 修改主题配置

Butterfly 主题的配置文件是 `_config.butterfly.yml`。当前已配置的内容包括：

- 固定顶部导航栏
- 首页/归档/标签/分类菜单
- 首页大图和打字机副标题
- 文章卡片交替布局
- 右侧边栏（作者卡片、公告、最新文章、标签云等）
- 深色模式切换

你可以修改的常用配置：

```yaml
# 首页大图（替换为自己的图片链接）
index_img: https://images.unsplash.com/photo-xxx

# 副标题
subtitle:
  enable: true
  sub:
    - 记录学习与思考
    - 分享技术与生活

# 作者描述
aside:
  card_author:
    description: 一个热爱技术的记录者

# 社交链接
social:
  fab fa-github: https://github.com/dxyang42 || Github || '#24292e'
```

更多配置请参考 [Butterfly 文档](https://butterfly.js.org/)。

---

## 生成站点

本地确认无误后，生成静态文件：

```bash
npm run clean
npm run build
```

生成的文件会输出到 `public/` 目录。

---

## 部署到 GitHub Pages

### 方式一：一键部署（推荐）

```bash
npm run deploy
```

Hexo 会自动把 `public/` 目录推送到 `master` 分支，GitHub Pages 随后会更新。

### 方式二：手动推送

如果你习惯手动操作：

```bash
# 提交源码变更
git add .
git commit -m "update blog"
git push origin source

# 部署站点
npm run deploy
```

---

## 常用命令速查

| 命令 | 作用 |
|------|------|
| `npm run server` | 启动本地预览服务器 |
| `npm run clean` | 清除缓存和生成的文件 |
| `npm run build` | 生成静态站点 |
| `npm run deploy` | 部署到 GitHub Pages |
| `npx hexo new post "标题"` | 创建新文章 |
| `npx hexo new draft "标题"` | 创建草稿 |
| `npx hexo server --draft` | 预览草稿 |

---

## 参考链接

- [Hexo 官方文档](https://hexo.io/docs/)
- [Butterfly 主题文档](https://butterfly.js.org/)
- [Markdown 语法指南](https://www.markdownguide.org/)

---

## 注意事项

1. `public/` 目录是自动生成的，不要手动修改里面的文件。
2. 图片资源可以放在 `source/img/` 目录下，引用时使用 `/img/xxx.png`。
3. 修改 `_config.yml` 或 `_config.butterfly.yml` 后，建议重启本地服务器再查看效果。
4. 部署前确保 `_config.yml` 中的 `url` 配置正确。
