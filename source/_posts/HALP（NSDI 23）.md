---
title: HALP（NSDI 23）
date: 2023-4-25 17:52:57
description: HALP（NSDI 23）论文阅读
tags:
- Cache
- ML
- 论文阅读
categories:
- Cache
---

本文的主要想法是：通过启发式算法选择驱逐候选集，再使用学习策略从中选择最佳驱逐对象。这样做可以减少 ML 预测的计算开销。

# Introduction

learned caches 部署在大规模生产环境中的问题：

1. 对比于启发式算法，learned caches 训练和预测的计算开销太大了；
2. 大规模的 CDN 生产环境，即使是小部分区域的缓存性能下降也是不能接受的，因此，在 learned caches 效果不佳时，需要做好兜底操作；
3. 生产环境中即使是同一机架的机器缓存缺失率也不一样，如何准确评估一个新算法在 production noise 上的效果。

# Design

{% asset_img design.jpg design %}

利用启发式算法从缓存尾部选取 4 个候选集，再以锦标赛方式进行 3 对比较选择最佳驱逐对象。

1% 的机器使用原有启发式算法，其余机器使用 HALP，因此，可以监控 HALP 对性能的影响，从而及时切换为原有启发式算法。

