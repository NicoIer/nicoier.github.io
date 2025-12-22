---
title: build-aseprite
date: 2025-12-22 11:28:12
tags: [aseprite, build, tutorial]
---

```text
本文主要介绍了一下如何编译 Aseprite，其实现在已经变成傻瓜式的编译了，官方文档也写得很清楚了，这里就不多赘述了。直接列出命令行
```

```shell
# 克隆 Aseprite 仓库
git clone https://github.com/aseprite/aseprite
# 进入仓库目录
cd aseprite
# 更新所有子模块
git submodule update --init --recursive
# 执行自带的构建脚本 按照提示一路Yes即可
./build.sh
```


构建完成后会出现一个 `build` 目录，里面包含了编译好的 Aseprite 可执行文件。



