---
title: TinyLFU (TOS 17)
date: 2023-7-12 12:52:57
description: TinyLFU (TOS17) 论文阅读
tags:
- Cache
- 论文阅读
categories:
- Cache
---

SOTA 缓存准入（高效）。

# TinyLFU Architecture

## Overview

缓存驱逐策略选择驱逐对象，TinyLFU 将驱逐对象和缺失对象进行比较，以决定是否以驱逐对象为代价准入缺失对象。

{% asset_img Overview.jpg Overview %}

## Challenges

1. 如何维护最近请求对象的历史信息，删除久远的历史信息；
2. 减少内存开销。

## Freshness Mechanism

每次请求发往近似 sketch 时，都会增加一个全局计数器 S。一旦该计数器 S 的值达到样本大小（W），就会触发 reset，将 S 和近似 sketch 中的所有其他计数器除以 2。

**优势**

1. 只会额外增加 Log(W) 个 bits 的空间开销；
2. 久远的请求信息会指数衰减。

## Counting Bloom Filter

{% asset_img count_bloom_filter.jpg count_bloom_filter %}

增加计数：当请求到来时，通过每个哈希函数映射到各自的索引，将最小计数的对应计数器值+1，如 {2, 2, 5} -> {3, 3, 5}（实际代码还是倾向增加所有计数器）；

估算计数：通过每个哈希函数映射到各自的索引，取对应计数器中的最小值。

### Space Reduction

1. 总请求量每到 W，则会对全局计数器和近似 sketch 中的所有计数器除以 2，那么理论上，每个 sketch 中的计数器需要 Log(W) 个 bits，但如果假设请求均匀，对于容量为 C 的缓存，通过牺牲部分准确性（频率最高到达 W/C），则只需 Log(W/C) 个 bits。；

2. 如果很多对象只访问一次，则它们会占据大量的计数器。因此，放置一个常规布隆计数器 DoorKeeper 在 Counting Bloom Filter 之前，对象到来时，先检查是否在 DoorKeeper 中，如果不在，则插入到 DoorKeeper 中，否则，请求 Counting Bloom Filter。每次 reset 操作也会清空 DoorKeeper。

    （门卫，很形象 (:

# Extension

TinyLFU 无法适应对象大小可变的缓存环境，因此，作者在 TOS 22 上提出了一种大小感知的 TinyLFU 策略（AV-TinyLFU，aggregated victims）。

当准入一个对象需要驱逐多个对象时，使用分离的替换策略选出多个驱逐对象，计算所有驱逐对象的频率和，将之与请求对象的频率进行比较，如果 freq(req) > freq(victims)，则准入；否则，不准入，且将所有 victims 进行 promote。（**考虑 freq/size 是否更好**）