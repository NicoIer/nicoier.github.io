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

- Data Persistent Model：这一部分用于在渲染系统中管理和上传数据，重点在于如何有效地管理渲染所需的各种数据，确保这些数据在不同渲染帧之间保持一致和可用。

- Persistent Batches：这一部分涉及渲染批次的优化，以减少绘制调用的开销。通过在一定时间内保持批次的一致性，最大限度地减少了与每帧设置和执行这些批次相关的计算成本。






# SRP Batcher

Unity是一款非常灵活的渲染引擎，允许在一帧的任何时间点修改Material属性。然而，这种灵活性带来了一些弊端。
比如，当一个DrawCall（绘制调用）需要使用新的Material时，需要进行大量的处理。
因此，场景中的Material数量越多，为设置GPU端的数据而占用的CPU资源就越多。
传统解决此问题的方法是减少DrawCall的数量，以优化CPU的渲染消耗。
因为在Unity中，发出DrawCall之前需要进行大量的设置，这些设置导致了实际的CPU消耗，而不是GPU DrawCall本身的消耗。
GPU DrawCall本身只是在渲染循环中向GPU命令缓冲区推送一些字节。
当一个新的Material被使用时，CPU会收集所有的属性，并在GPU内存中设置不同的Constant Buffers。
GPU缓冲区的数量取决于Shader如何声明其CBUFFERs。





```text
# 参考文章
https://zhuanlan.zhihu.com/p/707811162
```
