---
title: FrozenHot (Eurosys 23)
date: 2023-6-10 18:52:57
description: FrozenHot (Eurosys 23) 论文阅读
tags:
- Cache
- 论文阅读
categories:
- Cache
hidden: true
---

**Abstract**

1. CPU 核心数增加，但由于基于链式结构的内存缓存锁争用，命中路径的扩展性没有跟上；
2. 在缓存工作负载具有高度倾斜的数据流行度和短期热点稳定性的动机下，将缓存分为两个区域：frozen 和 dynamic。前者不进行命中提升、不加锁，以极低的命中时延服务热数据；后者利用现有缓存设计实现负载自适应；
3. 在 HHVM 和 RocksDB 上进行实现，吞吐量和尾延时得到大大改善。

# Design

## Overview

逻辑上将缓存空间分区成 Frozen 和 Dynamic，Frozen 区域使用无锁哈希表索引 FC 缓存对象，DC 区域使用传统的缓存设计。

1. 请求到来，首先 Lookup FC，如果命中直接返回；
2. FC 缺失，则 Lookup DC，如果命中则在 DC 中进行 Promotion 并返回；
3. DC 也缺失，则 Insert DC，视 DC 空间情况进行对应缓存策略的 Evict。

{% asset_img overview.jpg overview %}

FC 的元数据链表只能进行读操作（FC 的缓存数据和元数据本身仍然是可以修改的），且不存在 Promotion，因此无需加锁避免了锁争用；同时，因为只读，可以选用 fast，immutable 的哈希表。

**如何设置 FC 和 DC 的大小？并且需要定期修改它们之间的比例才能适应工作负载的变化，从而不会导致命中率的急剧下降**

## Adapt, Resize and Reconstruct

**Learning Frozen Ratio**

在学习开始前，将前一阶段的 FC 和 DC 合并成一整个链表，再将一个 Marker 插入到链表头部，然后进入学习阶段；在此阶段中，新对象插入到 Marker 的右侧，Marker 右侧对象命中提升到链表头部，Marker 左侧对象命中保持不变，定期（例如，1s）监视 Marker 左侧对象的命中次数和总体缺失次数，Marker 左侧容量逐渐增加，每 2% cache capacity 的步长进行一次记录（Marker 左侧对象的命中次数，总体缺失次数），此时利用下述公式可以绘制一条 "Frozen Ratio" 到 "平均延时" 的曲线，计算得到满足最低 "平均延时" 的 "Frozen Ratio"。 

{% asset_img 平均延时.jpg 平均延时 %}

**construct and deconstruct FC**

学习到 FC 比例后，将整个链表的前 Frozen Ratio 对象添加到 FC 哈希中，而在 DC 哈希中不删除；这样的话，在 deconstruct 阶段，只需删除 FC 哈希中的所有键值对即可，因为 FC 中的对象在 DC 哈希中仍然存在。

**How often should the FC be reconstructed?**

三种机制

1. Performance monitoring

    持续监控平均延时和带宽等性能，如果发现性能低于上一学习阶段观察到的 "全 DC 性能"，则重新学习最佳 Frozen Ratio 并进行重建。

2. Periodic refresh

    用户配置 Frozen 持续时间（比如设置成重建时间的 20 倍），以避免上一学习阶段观察到的 "全 DC 性能" 非常差导致长时间无法重建。

3. Regression to baseline

    如果学习到的 Frozen Ratio 无法优于全 DC 模式，则会使用全 DC，全 DC 的持续时间随着连续失败而呈指数级增长。