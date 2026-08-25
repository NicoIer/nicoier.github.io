---
title: 何为mipmap
date: 2026-08-25 15:56:49
tags:
---

起因是在蛋仔派对的三面中 面试官问我mipmap是什么 我说mipmap是一种采样技术 是为了解决远处的摩尔纹的一种技术 远处的像素会采样低级mipmap 近处的会采样高级mipmap
然后他问我 一个像素点具体是怎么选择mipmap level的 由于没有系统性的研究过 所以我说我不知道 结果他就拷打我 让我想一想 我说距离 他说不对

然后我就开始查资料，发现mipmap的level选择是根据纹理坐标的偏导数来计算的。具体来说，GPU会计算当前像素在屏幕空间中对应的纹理坐标的变化率，然后根据这个变化率来选择合适的mipmap level。变化率越大，说明纹理在屏幕上占据的面积越小，因此会选择更低级别的mipmap；反之，变化率越小，说明纹理在屏幕上占据的面积越大，会选择更高级别的mipmap。

其实还是没懂，干脆系统性的研究一下

什么叫footprint?
如何计算出footprint?

摩尔纹是什么?

什么叫欠采样?

屏幕上的一个像素点的颜色是如何来的?
不考虑抗锯齿和其他各种效果，一个覆盖屏幕的大平板Mesh，一张颜色Texture，一个屏幕上的点的uv会通过顶点插值得到，然后用这个uv去采样纹理，得到这个像素点的颜色。

![moire-pixel-uv-texture-explainer.svg](../img/moire-pixel-uv-texture-explainer.svg)


如何避免欠采样?

mipmap是什么？

解决了什么问题?
优势是什么
代价是什么
如何生成mipmap

各向异性是什么？
各向异性过滤Mipmap是什么
如何生成


https://zhuanlan.zhihu.com/p/144332091