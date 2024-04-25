---
title: Flashield (NSDI 19)
date: 2023-6-26 18:52:57
description: Flashield (NSDI 19) 论文阅读
tags:
- Cache
- ML
- 论文阅读
categories:
- Cache
hidden: true
---

**Abstract**

1. SSD 价格下降，逐渐成为云数据库存储热数据的介质；
2. 但 KV 缓存频繁执行插入、驱逐和更新操作，将大大缩短 SSD 寿命；因此，需要想办法预测哪些对象会被频繁读而不会更新，只有这些对象才会被写入 SSD，然而对象第一次进入缓存时无法进行预测；
3. 使用内存作为过滤层，保存所有对象，同时，对于预测结果为将被频繁读而不更新的对象，才会写入 SSD；
4. 为对象在内存中设置索引，用于查找和驱逐。

# Problem

1. SSD 设备级写放大：无法覆盖写，更新只能将旧数据标记为失效，再写空白；擦除以块为单位，会导致块内一些有效数据页进行数据迁移，造成额外写入。

2. SSD 缓存级写放大：被缓存策略驱逐的对象，又被写入 SSD，造成缓存级写放大。

# Motivation

1. 并不是每个对象都有必要插入 SSD：一次性访问对象和频繁更新对象；
2. segment size 和 object size 差异很大，难以保证被驱逐策略选定为相似优先级的对象会被存储在 SSD 临近区域。

# Design

{% asset_img 架构图.png 架构图 %}

流程：

1. 对象先写入内存；
2. 内存可用空间小于 segment 大小时，将预测结果为闪存友好（分数超过某阈值）的对象按分数高低以 segment 粒度迁移至 SSD，预测结果为闪存不友好的对象直接逐出内存；
3. 闪存空间不够时，以 FIFO 顺序驱逐 segment，segment 内部具有较高优先级的对象可以重新插入到内存中。

DRAM 作用：

1. 作为过滤器来判断哪些对象应该被插入 SSD；
2. 充当对象移动到 SSD 以及无法从 SSD 中收益的对象的缓存层；
3. 存储用于在闪存中查找和逐出对象的元数据。

## SVM predicts Flashiness

flashiness 定义为对象能从 SSD 中受益的程度。

选用 SVM 作为模型，在对象即将从内存中驱逐时，使用对象在内存中被访问的次数、更新的次数作为特征，预测对象的 flashiness，从而决定是否插入 SSD。

## DRAM as an Index for Flash

将对象的索引保存在内存中，能够减少更新时对闪存的大量写入、降低索引的读取时延，但由于 DRAM 也作为数据的缓存层，因此需要尽可能将索引设计小。

Flash 的索引应该包含：key、value 在 Flash 中的位置、在驱逐队列中的位置。

1. Flash Index Hashtable："key" -> {segment_id, a hash function}；
    - 为了避免哈希碰撞导致的哈希表扩展，使用 multi-choice（多个哈希表） 代替 chain，如果全都冲突，则驱逐最后一个冲突对象；
2. Flash Bloom Filters：为了避免哈希碰撞导致的对 Flash 的过多读，为每个 segment 配置一个布隆过滤器，过滤 key 一定不在该 segment 中的情况，从而无需浪费一次 Flash 读，而是可以直接去查找下一个哈希函数；（文中讲到，布隆过滤器结果 true 但是不在 segment 的情况只占 1%，从而平均只需 1.03 次 Flash 读）

当对象在布隆过滤器上显示 true 时，得到一个 {segment_id, a hash function **h**} 的条目，h(key) 求得该对象在 segment 中的偏移量，读 Flash 中该 segment 的对应偏移，并进一步检查 key 值是否相等。

{% asset_img lookup_flash.jpg lookup_flash %}

{% asset_img flash_index.jpg flash_index %}

## Evict

全局 CLOCK 算法维护 DRAM 和 FLASH 中的所有缓存对象。如果下一个驱逐对象在 DRAM 中，则直接驱逐；如果在 FLASH 中，在 Flash Index Hashtable 将对应项标记为 ghost，它将随着所在 segment 从 Flash 删除而驱逐。