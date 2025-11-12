---
title: hexo命令速记
date: 2025-11-12 13:00:49
tags: [hexo]
---

```bash
# 初始化 Hexo 博客
hexo init <folder>  
# 在指定文件夹中初始化一个新的 Hexo 博客
# 进入博客目录
cd <folder>
# 安装依赖包
npm install
# 生成静态文件
hexo generate 或 hexo g
# 预览本地服务器
hexo server 或 hexo s
# 部署到远程服务器
hexo deploy 或 hexo d
# 清除生成的静态文件
hexo clean
# 创建新文章
hexo new <post_name> 或 hexo n <post_name>
# 创建新草稿
hexo new draft <draft_name> 或 hexo n draft <draft_name>
# 发布草稿
hexo publish <draft_name> 或 hexo p <draft_name>
# 查看 Hexo 版本
hexo version 或 hexo -v
```