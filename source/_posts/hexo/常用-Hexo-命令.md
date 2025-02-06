---
title: 常用 Hexo 命令
categories:
  - hexo
tags:
  - Hexo
  - 常用命令
abbrlink: a0f4bf82
date: 2025-02-06 10:30:34
---

Hexo 是一个静态博客框架，以下是一些常用的 Hexo 命令：

## 1. `hexo init`
初始化一个新的 Hexo 项目，创建基本的目录结构，并安装相关依赖。

## 2. `hexo new post <title>`
创建一篇新的文章，`<title>` 是文章的标题。

## 3. `hexo generate` 或 `hexo g`
生成静态文件，将 Markdown 文件转换为 HTML 文件，存放在 `public` 目录中。

## 4. `hexo server` 或 `hexo s`
启动本地服务器，默认在 `http://localhost:4000`，用于查看博客。

## 5. `hexo deploy` 或 `hexo d`
将博客部署到远程服务器，按照 `_config.yml` 中的部署配置上传静态文件。

## 如何创建一篇新的文章

要创建一篇新的文章，只需运行以下命令：

```bash
hexo new post "文章标题"