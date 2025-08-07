---
title: Unity-Bloom-锯齿优化
date: 2025-07-15 15:19:10
tags: [ Unity, Shader,后处理, Bloom,锯齿 ]
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

<script>
MathJax = {
  tex: {
    inlineMath: [['$', '$'], ['\\(', '\\)']]
  }
};
</script>
<script id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js">
</script>