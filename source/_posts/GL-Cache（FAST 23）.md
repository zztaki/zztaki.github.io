---
title: GL-Cache（FAST 23）
date: 2023-4-5 12:52:57
description: GL-Cache（FAST 23）论文阅读
tags:
- Cache
- ML
- 论文阅读
categories:
- Cache
---

**Abstract**

Web 应用程序严重依赖软件缓存从而实现低延迟和高带宽。为适应工作负载的变化，近年提出了三种学习缓存（学习驱逐）：object-level learning，learning-from-distribution 和 learning-from-simple-experts。但是，这三种方法要么粒度太细（object-level），计算和存储开销比较大，要么就像其它两种一样粒度太粗，无法捕捉对象之间的差异而留下较大的效率提升空间。

本文提出将相似对象集群到同一组并以组为单位执行学习和驱逐，`Learning at the group level accumulates more signals for learning, leverages more features with adaptive weights, and amortizes overheads over objects, thereby achieving both high efficiency and high throughput.`

与 SOTA 策略比较，GL-Cache 实现了高带宽和高命中率。

# Introduction

1. Large-scale cache deployments enable the success of today's Internet.

2. The main driving force of cache deployments is the cache's ability to serve data with high throughput and low latency. Retrieving data from a cache is thousands of times faster than retrieving it from the backend.

    - 但是缓存总是部署在价格昂贵、容量有限的介质中，缓存容量通常比数据集大小小得多，因此**决定哪些数据存储在缓存中**格外重要。
    - 高效缓存能够存储更有用得数据从而无需请求后端数据系统就能服务更多的请求。缓存有效性通常被命中率衡量。
    - 当缓存满时，需要使用驱逐算法决定哪些数据被保存，哪些数据被驱逐。因此，**驱逐策略对于缓存效率非常重要**。

3. 这些年许多驱逐算法利用不同的对象特征做驱逐决策。比如一些 LRU 的变体使用新近度的不同角度选择驱逐对象；频率+新近度；频率+对象大小；

    - 不同特征对于不同的工作负载具有不同程度的重要性，因此只是使用特定方式组合一个或者两个对象特征通常也就只能在一些工作负载中实现高效率。
    - **learned caches**: employed  machine learning to improve cache evictions.

4. 将 learned caches 分为三类

    - object-level learning（LRB）：learns the next access time for each object using dozens of object features and evicts the object with the furthest predicted request time.
        - **Advantage**：leverages more object features, learns the relative feature importance, and performs fine-grained learning on each cached object, it has the highest potential for achieving high efficiency.
        - **Disadvantage**：predicting and ranking objects at each eviction incurs significant computation and storage overheads as we observe LRB suffers from a 775× slow down compared to LRU.

    - learning-from-distribution（LHD）：models request probability distributions to inform eviction decisions. For example, LHD measures object hit density using age and size, and evicts the object with the lowest hit density.
        - **Advantage**：a lower computation and storage overhead because it models request probability using fewer features at a coarser granularity.
        - **Disadvantage**：still has a lower throughput compared to simple heuristics (e.g., LRU) because it has to **randomly sample and compare many objects at each eviction**. Moreover, the existing design (e.g., LHD [7]) **does not leverage object features other than age and size, limiting its potential for high efficiency**.
    - learning-from-simple-experts（LeCaR, Cacheus）：performs evictions by choosing eviction candidates recommended by experts (e.g., LRU and LFU), and updates experts' weights based on their past performance on the workload.
        - **Disadvantage**：受强化学习中的 "delayed rewards" 限制，在不佳驱逐和该对象再次访问之间，专家权重没有得到更新；性能取决于专家的选择，现有系统使用简单的专家，无法利用到专家未考虑的特征。

5. 为了克服现有 learned caches 的缺点，提出以对象组为粒度进行学习。Group-level learning leverages multiple group-level features to learn object-group utility for evictions.

    - It reduces the computation and storage overheads of learning by hundreds of times through amortization compared to learning at the object level.
    - object groups accumulate more “signals” for learning and can leverage a variety of features for prediction, enabling better eviction decisions.

6. 组级别学习需要回答以下几个问题

    - How to group objects and perform evictions efficiently? 

        答：GL-Cache clusters similar objects into groups using write time (§3.3) and evicts the least useful groups using a merge-based eviction(§3.6).

    - How to measure the usefulness of object groups (termed “utility”) to determine the best eviction candidate? 

        答：GL-Cache introduces a group utility function (§3.4) to rank groups, which enables group-based eviction to achieve similar efficiency as object-based eviction (§4.2).

    - How to learn and predict the object-group utility online?

        答：(§3.5).

7. 本文贡献

    - 基于学习粒度将 learned caches 分为三类，并提出一个新的缓存学习方法—— group-level learning。
    - 设计并实现了 GL-Cache，解决了上面提到的组级别学习的挑战。同时也是第一次定义了 group-level utility function 用于缓存驱逐。
    - 使用生产跟踪评估 GL-Cache，展现了组级别学习的高效率和高吞吐量。

# Background and motivation

两个重要的缓存指标：**efficiency** measured using hit ratio, **performance** measured using throughput.

# GL-Cache: Group-level learned cache

The key idea behind group-level learning is to learn the usefulness of groups of objects (called “utility”).

## Overview

1. objects are clustered into fixed-size groups when writing to cache (§3.3).
2. The training module in GL-Cache collects training data online and periodically trains a model to learn the utility of object groups (§3.5).
3. The inference module predicts object-group utility and ranks object groups for eviction. When the cache is full, object groups are evicted using a merge-based eviction which merges multiple groups into one, evicts most objects, and retains a small portion of popular objects (§3.6).

## Group-level learning

1. 优势
    - **Grouping amortizes overheads**：these overheads are amortized over multiple objects in the group. The metadata overhead is only added for each group and the cost of inference computation is also amortized over objects.
    - **Grouping accumulates more signal**：大多缓存工作负载遵守 Zipf 分布，也就是说大部分的对象只会收到少量的请求，对象级的学习每个对象只能收到很少的信号；而将大量对象集群在一起，该集群就能收获更多的请求，从而更好地学习和预测。
2. 挑战
    - How to cluster objects into groups (§3.3)?
    - How to compare the usefulness of object groups (§3.4)?
    - How to learn the utility of object groups (§3.5)?
    - How to perform evictions at group level (§3.6)?

## Object groups

1. 当对象进入缓存时，对象的分组就被它的简单静态对象特征决定了，如时间、租户 ID、内容类型、对象大小等。本项工作专注于基于写入时间的分组，这在所有系统中都可用，具有通用性。
2. 相似时间写入的对象展现出相似的行为，它们的重用时间更接近。

## Utility of object groups

定义一个函数用于评估 group 的实用性。

1. 大对象占据更多空间，因此越大的对象有更低的实用性；（从另一个角度看，大对象驱逐后可以留下更多的空间存放更多的对象，因此更“值得”驱逐）
2. 前向重用距离越大的对象实用性更低；

但是对象的前向重用距离是无法实时获得的，因此作者提出使用 GL-Cache 学习一个模型从而基于特征来预测对象组的实用性。

## Learning object-group utility in GL-Cache

1. 静态特征：request rete, write rate, miss ratio and mean object size；
2. 动态特征：age, #requests and #requested objects；
3. 学习模型：
    - gradient boosting machines (GBM) because tree models do not require feature normalization.
    - 将学习任务指定为一个回归问题，最小化对象组效用的均方损失。

## Evictions of object groups

GL-Cache 在组级别使用重量级学习（均摊开销）从而识别出最佳驱逐组，再在驱逐组内充分利用轻量级对象指标（新近度、大小）保留一些非常有用的对象。两层驱逐使得 GL-Cache 能够在学习开销和缓存效率之间实现卓越的平衡。

**key insight：组级别驱逐的命中率竟接近 Belady，没有带来命中率的大幅下降，而这种一次性驱逐多个对象可以分摊推理开销**。

# Evaluation

1. Will group-based eviction limit the efficiency upper bound when compared to object-based eviction ? 

    **组驱逐并不会成为实现高效率的瓶颈**

    {% asset_img similar_hit_with_min.jpg similar_hit_with_min %}

2. Can GL-Cache improve hit ratio and efficiency over other learned caches ?

3. Can GL-Cache meet production-level throughput requirements and how much overhead does GL-Cache add ? 

4. How does GL-Cache improve efficiency without compromising throughput ?