---
title: NyxCache（FAST 22）
date: 2023-4-12 16:17:57
description: NyxCache（FAST 22）论文阅读
tags:
- Cache
- PM
- 论文阅读
categories:
- Cache

---

**Abstract**

1. 本文提出了一个用于多租户持久性内存（multi-tenant persistent memory）缓存的访问调节框架，**支持轻量级访问调节、每个缓存的资源使用估算和缓存间干扰分析**。
2. 有了以上机制和准入控制、容量分配逻辑，本文构建了重要的**共享策略**，例如资源限制、Qos-awareness、公平性和按比例共享：Nyx 的资源限制可以准确地限制每个缓存的 PM 使用量，提供比带宽限制方法 5 倍的性能隔离 ；Nyx QoS 可以为延迟敏感型缓存提供 QoS 保证，同时为不受干扰的 best-effort 缓存提供更高的吞吐量（与以前基于 DRAM 的方法相比，最高可达 6 倍）。
3. 最后，本文也展示了 Nyx 对于真实负载也是有用的，隔离了写入高峰，并确保重要的缓存不会因为增加的 best-effort 流量而减慢。

# Introduction

为了提高实用性和简化管理，多个缓存实例通常被合并成一个多租户服务器。然而，多租户服务器很难确保每个客户端缓存都能满足性能目标；目前，一系列内存多租户缓存提供了不同的共享策略，比如对使用的内存容量和带宽进行限制，保证一定的 quality-of-service（Qos），并按比例分配资源。

由于具有大容量、字节低开销、与 DRAM 性能接近的性质，PM 可以作为多租户的缓存。但是，与 DRAM 不同的是，PM 在读写性能上差别很大（最大读带宽 6.6GB/s，最大写带宽 2.3GB/s），读和写之间有着严重、不公平的干扰（以 1GB/s 的速度写入会导致共同运行的读取工作负载的吞吐量和 P99 延迟减慢，这种程度和 8GB/s 的速度读取相同）。

因此，现有多租户 DRAM 和缓存技术无法适配到 PM 上，于是本文就提出针对多租户 PM 键值缓存的访问调节框架，无需硬件支持就能优化 PM。

# Motivation and Challenges

评估先前的多租户缓存以及它们对于 PM 的限制。

# NyxCache Design

NyxCache 的设计。

# Evaluation

评估 Nyx 机制的开销和策略的有效性。

# Discussion

讨论潜在的扩展。

# Related Work

对比相关工作。