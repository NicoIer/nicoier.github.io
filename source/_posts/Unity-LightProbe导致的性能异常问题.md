---
title: Unity-LightProbe导致的性能异常问题
date: 2025-08-07 11:12:10
tags: [ Unity, LightProbe ]
---

使用LightProbe烘焙间接光时，主要是有一些LightProbe的点出现了穿插，导致在运行时计算插值时出现了大量的计算开销，从而导致性能异常。

解决方案是让Probe不要出现穿插，避免计算时因为两个点之间存在Mesh导致计算开销过大以及错误结果