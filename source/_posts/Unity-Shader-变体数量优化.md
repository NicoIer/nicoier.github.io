---
title: Unity Shader 变体数量优化
date: 2025-08-27 11:14:53
tags: [Unity, Shader, 性能优化]
---


在使用URP之后，有时候会发现Shader变体数量暴增，导致打包体积过大，加载时间过长等问题。

通常这个情况出现在使用了Shader Graph，或者在自定义Shader中使用了多种关键词（Keywords）时。

Unity自带了一个工具去检查工程中使用到的变体，并且可以生成一个变体列表文件（.shadervariants）

在构建AB包/打包时，可以指定这个文件，来剔除未使用的变体。


multi_compile 和 shader_feature 在编译时的行为完全不一样，multi_compile 会强制编译所有变体，而 shader_feature 则只会编译使用到的变体。


至于剔除则是在编译完Shader之后，根据变体列表文件来剔除未使用的变体。

所以要节约打包时间，最好使用 shader_feature 来替代 multi_compile。


要完整的跑一遍游戏所有的功能，才能确保变体列表文件的完整性。因为有时候Shader的某个变体只在游戏的某个分支中使用到，如果没有跑到这个分支，那么这个变体就不会被记录下来，导致显示异常。
