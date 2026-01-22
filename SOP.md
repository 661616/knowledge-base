# 内容发布 SOP (Standard Operating Procedure)

## 1. 撰写新文章

在项目根目录下，使用终端运行以下命令创建新文件：

```bash
# 格式: hugo new content/posts/英文文件名.md
hugo new content/posts/my-new-post.md
```

或者直接在 `content/posts/` 目录下新建 Markdown 文件。

## 2. 编辑 FrontMatter (文章头信息)

打开新建的 `.md` 文件，修改头部信息：

```yaml
---
title: "这里写文章标题"
date: 2026-01-22T10:00:00+08:00
draft: false  # 设为 false 才会发布，true 为草稿
categories: ["技术", "随笔"]
tags: ["Hugo", "教程"]
---
```

## 3. 本地预览

在终端运行：

```bash
hugo server
```

打开浏览器访问 `http://localhost:1313`，实时查看效果。

## 4. 发布上线

确认无误后，执行 Git 命令推送：

```bash
git add .
git commit -m "Add post: 文章标题"
git push
```

Cloudflare Pages 会自动触发构建，约 1-2 分钟后线上即可看到更新。

