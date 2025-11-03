---
title: EntitiesGraphics
date: 2025-08-27 11:19:24
tags: [Unity, DOTS, ECS, 性能优化 , 渲染]
---

# Entities Graphics

优化后的SRP Batch


利用了Burst编译器(LLVM)

多线程收集数据 (EntityQuery)

无论多少相机，只需要上传一次数据

Data Persistent Model：这一部分用于在渲染系统中管理和上传数据，重点在于如何有效地管理渲染所需的各种数据，确保这些数据在不同渲染帧之间保持一致和可用。

Persistent Batches：这一部分涉及渲染批次的优化，以减少绘制调用的开销。通过在一定时间内保持批次的一致性，最大限度地减少了与每帧设置和执行这些批次相关的计算成本。



C++层的渲染逻辑被C#替代 (不再是黑盒 CullScriptable可以自己改造了!)

https://zhuanlan.zhihu.com/p/707811162