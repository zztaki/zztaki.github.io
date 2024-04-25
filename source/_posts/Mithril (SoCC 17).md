---
title: Mithril (SoCC 17)
date: 2023-8-7 18:52:57
description: Mithril (SoCC 17) 论文阅读
tags:
- Cache
- prefetch
- 论文阅读
categories:
- Cache
hidden: true

---

缓存替换和缓存准入技术在提高存储层性能方面被讨论得较多，而缓存预取在软件层讨论较少，有很大的改良机会。

# Introduction

当前缓存预取技术分为两派：

1. 顺序预取技术（sequential prefetching techniques），例如 AMP，这种方法假设连续块将得到访问，但依赖于 block I/O with progressive data layout.
2. 基于历史预取（history-based prefetching），这种方法探索过去访问的更深层次关联，但需要更多的计算开销。

作者认为，块存储系统需用通用的预取技术，并满足一下三个条件：

1. 探索历史信息。由于存储系统的较低层次已经包含顺序预取（CPU cache line），因此更应该在时间和空间维度挖掘更加复杂的重用模式；
2. 时间和空间开销低；
3. 向后兼容，只实现标准遗留接口，而将存储系统的其它部分视为黑盒子。

中心思想：**只跟踪中等频率对象**，因为高频对象会被缓存替换算法捕获，而低频对象无需占用缓存；

Mithril 的优势：

1. 无需应用程序的提示，只需时间戳即可挖掘块访问的关联性；
2. 在多租户场景下，能够以较低的开销发现不连续的交错请求之间的关系；
3. Mithril 性能之所以好，是因为它只捕获中频块的信息。

# Background and Motivation

1. 缓存替换和缓存准入方面的研究很多，然而预取很少，预取有很大的改善机会；
2. 基于顺序的预取技术在操作系统层级应用广泛，然而，对于高层存储级，效果有限；
3. 现有基于历史的预取技术，要么时间和空间开销过大，要么需要应用程序级的提示，不够通用；
4. 挖掘时间局部性。

# MITHRIL

当缓存缺失（rflag=true, pflag=false）或缓存命中（rflag=false, pflag=true）时，守护进程调用算法3，分别记录块的访问序号，并周期性地进行 mining 更新预取表，以及查询预取表进行预取操作。

{% asset_img prefetch_api.jpg prefetch_api %}

{% asset_img mining.jpg mining %}

{% asset_img check_association.jpg check_association %}

**随后作者还仔细的讲述了时间和空间上做的优化，不再赘述，这种写法值得学习。**

