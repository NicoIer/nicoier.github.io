---
title: 通过Pre-Z减少AlphaClip绘制的开销
date: 2025-08-07 11:12:42
tags: [Unity, Shader, Pre-Z, AlphaClip]
categories: [Shader]
---

# 手机上AlphaClip为什么会很卡

本质上是由于 OverDraw （过度绘制）引起的。AlphaClip 会丢弃像素，但这些像素在片元着色器阶段之前已经被处理过了，因此仍然会消耗 GPU 资源。尤其是在移动设备上，GPU 资源有限，过度绘制会导致性能下降。

# Pre-Z优化方案

- 再Opaque绘制之前，先绘制一遍AlphaClip的物体，只进行深度写入
- 在AlphaClip绘制时，开启深度测试 ZTest Equal （只有深度值相等的像素才会被绘制）
- 没有被丢弃的像素就不会进入片元着色器阶段，从而减少了GPU的负担



Unity中可以使用RenderObjects实现一次Pre-Z绘制

然后在自己的AlphaClip Shader中开启ZTest Equal

```shaderlab
ZTest Equal
```


# 代价

代价就是所有的AlphaClip物体都需要绘制两次，一次Pre-Z，一次正常绘制。