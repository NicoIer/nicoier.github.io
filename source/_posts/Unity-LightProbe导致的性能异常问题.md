---
title: Unity-LightProbe导致的性能异常问题
date: 2025-08-07 11:12:10
tags: [Unity, LightProbe, 游戏开发]
categories: [Unity]
---

使用LightProbe烘焙间接光时，主要是有一些LightProbe的点出现了穿插，导致在运行时计算插值时出现了大量的计算开销，从而导致性能异常。

解决方案是让Probe不要出现穿插，避免计算时因为两个点之间存在Mesh导致计算开销过大以及错误结果


# 原因

LightProbe 本质上是用空间中的 Probe 点来近似间接光。

运行时物体需要根据当前位置，在周围的 Probe 之间做插值，得到当前点的光照信息。

当 Probe 布点不合理，比如：

- Probe 点穿进了 Mesh
- Probe 点分布过密但没有必要
- Probe 点在墙体、地面、封闭空间附近交错
- 两个 Probe 之间被场景几何体隔开

就可能导致插值结果不稳定，甚至让运行时计算出现额外开销。

LightProbe 不是点越多越好，关键是点要放在合理的位置。
