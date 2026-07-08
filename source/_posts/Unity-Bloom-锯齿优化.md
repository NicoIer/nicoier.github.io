---
title: Unity-Bloom-锯齿优化
date: 2025-07-15 15:19:10
description: 记录 Unity Bloom 在上采样阶段用 X 型卷积核补回角落采样，降低锯齿的一个简单处理。
tags: [Unity, Shader, 后处理, Bloom, 锯齿, 游戏开发]
categories: [Unity]
math: true
---

Unity Bloom在降采样和上采样时使用的卷积核如下，是一个十字型的卷积核，这样会丢掉四个角落的像素信息，导致锯齿现象。
我们在上采样时,使用一个X型的卷积核心，将丢失的信息补充回来即可

<span>

$$
\begin{bmatrix}
0 & 1 & 0 \\
1 & 1 & 1 \\
0 & 1 & 0
\end{bmatrix}
$$
</span>

<span>

$$
\begin{bmatrix}
1 & 0 & 1 \\
0 & 1 & 0 \\
1 & 0 & 1
\end{bmatrix}
$$
</span>

这样做没有什么理论依据，但是效果很好
