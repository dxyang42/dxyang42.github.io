---
title: 用 Hexo 和 GitHub Pages 搭建个人博客
date: 2026-07-08 23:23:23
tags:
  - Hexo
  - GitHub Pages
  - 博客
---

这篇文章记录我是如何从一个空文件夹开始，把个人博客搭起来并部署到 GitHub Pages 上的。初始环境非常简单：一个空目录，以及一个已经登录好的 `gh` CLI。

## 技术选型

- **Hexo**：静态博客生成器，基于 Node.js，主题多、生态成熟。
- **GitHub Pages**：免费托管静态站点，对个人博客来说足够用。
- **仓库名**：因为要做成个人主页，仓库必须命名为 `<用户名>.github.io`，这样页面会部署在根域名 `https://<用户名>.github.io`。

## 安装 Hexo CLI 并初始化

全局安装 Hexo 命令行工具：

```bash
npm install -g hexo-cli
```

在当前目录初始化博客：

```bash
hexo init .
npm install
```

执行完后，目录里会出现 `_config.yml`、`source/`、`themes/`、`scaffolds/` 等标准结构。

## 创建 GitHub 仓库

用 `gh` 创建仓库，选择公开仓库，命名为 `dxyang42.github.io`：

```bash
gh repo create dxyang42.github.io --public --description "Personal blog powered by Hexo"
```

## 分支策略：源码与站点分离

对于 `<用户名>.github.io` 这种用户站点，GitHub Pages 只能从 `master` 或 `main` 分支的根目录读取，不能从 `gh-pages` 分支读取。为了同时保留源码和生成的静态文件，我采用两条分支：

- `source` 分支：放 Hexo 源码、主题配置、文章 Markdown。
- `master` 分支：放 `hexo generate` 生成的静态站点，供 GitHub Pages 读取。

先把本地分支改名为 `source` 并推上去：

```bash
git branch -m source
git push -u origin source
```

## 配置 `_config.yml`

主要改了这几项：

```yaml
title: dxyang42's Blog
author: dxyang42
language: zh-CN
url: https://dxyang42.github.io

deploy:
  type: git
  repo: https://github.com/dxyang42/dxyang42.github.io.git
  branch: master
```

同时安装部署插件：

```bash
npm install hexo-deployer-git --save
```

## 踩坑：SSH 权限冲突

推送时遇到了一个权限错误，大意是 `dxyang42` 的仓库被拒绝访问。原因是 `gh` 登录的是 `dxyang42`，但本地 SSH key 却对应着另一个 GitHub 账户。解决办法是把 Git 远程协议从 SSH 换成 HTTPS，并让 `gh` 接管 Git 凭据：

```bash
gh auth setup-git
git remote set-url origin https://github.com/dxyang42/dxyang42.github.io.git
```

之后推送就正常了。

## 部署

生成并部署：

```bash
npx hexo clean
npx hexo deploy
```

`hexo-deployer-git` 会把 `public/` 目录下的内容推送到 `master` 分支。然后在 GitHub 仓库设置里确认 Pages 的 Source 是 `master` 分支即可。

## 最终效果

- 博客地址：`https://dxyang42.github.io/`
- 源码分支：`source`
- 部署分支：`master`

这篇就是在这个新博客上写的第一篇文章，算是给自己的搭建过程留个档，也顺便验证一下写作和部署流程是否通畅。
