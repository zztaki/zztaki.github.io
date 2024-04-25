---
title: CacheDedup（FAST 16）
date: 2023-5-21 17:43:57
description: CacheDedup（FAST 16）论文阅读
tags:
- Cache
- 论文阅读
- Deduplication
categories:
- Cache
---

**Abstract**

1. 使用闪存作为较慢主存的缓存层可以解决存储系统的扩展性问题，但是，闪存容量有限且耐用性较差；
2. 本文提出一种新的架构，将数据缓存和重删元数据（source addresses and fingerprints of the data）结合起来，有效管理两个组件；
3. 进一步，提出重复感知的 D-LRU 和 D-ARC 缓存替换算法，优化缓存性能和提高闪存寿命。

**创新性：本文强调，虽然已有将重删和闪存结合起来的研究，但本文谈到还可以将压缩感知结合起来，同时重点介绍了整合重删和缓存过程中的挑战和解决办法。**

## key idea

将闪存划分成两个部分：数据缓存和元数据缓存。

元数据缓存中包含两个重要数据结构：

1. source address index map：将后端存储的源地址映射到元数据缓存保存的某一指纹。由于 chunk 内容重复的存在，多个源地址可能映射到同一指纹。（多对一）
2.  fingerprint store：将指纹映射到数据缓存中的块地址。（一对一）

因此，对于相同内容的 chunk，它们在数据缓存中只会有一份副本。

