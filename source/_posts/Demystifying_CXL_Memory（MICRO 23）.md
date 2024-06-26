---
title: Demystifying CXL Memory（MICRO 23）
date: 2023-12-27 12:52:57
description: Demystifying CXL Memory（MICRO 23）论文阅读
tags:
- CXL
- Memory
- 论文阅读
categories:
- CXL
---

对容量更大，带宽更高的内存需求推动了基于 CXL 的内存扩展和分离的创新。基于 CXL 的内存扩展，不仅能够扩展内存容量和带宽，而且能够将内存从与 CPU 的紧耦合中分离出来。然而，由于 CXL 设备并没有大规模生产，因此大多数研究者都是使用远程 NUMA 节点进行模拟。

作者则是首次在一个真正的 CXL 系统中，使用来自不同制造商的三个 CXL 内存设备进行评估，并比较了真实 CXL 内存和模拟 CXL 内存的性能，此外，还分析了 CPU 和 CXL 内存之间的复杂相互作用，解释了模拟的 CXL 内存和真实 CXL 内存的一些差异；接下来，作者探讨了内存带宽密集型应用程序从 CXL 内存中获益的机会；最后，提出一个 CXL 内存感知的动态页面分配机制.

# Introduction

新兴应用程序对内存有着更大容量和更高带宽的需求，然而内存容量和带宽是每个 CPU 通道数，每个通道的 DIMM 数以及每个通道的比特传输率的函数，因此 DDR 的带宽和容量扩展能力有限，此时需要替代的内存接口技术和内存子系统架构。

CXL 构建在 PCIe 之上，相比 DDR4 有着更高的 bit 传输速度，更低的 bit 传输能耗，但连接时延相对较高；同时，CXL 内存设备相比于 DDR5，减少了 3 倍引脚数量的消耗，可以经济高效地扩展系统的存储容量和带宽；此外，通过 CPU 和内存设备之间的 CXL 控制器，CXL 将内存技术与 CPU 支持的特定内存接口技术解耦。最后，通过采用交换机，支持 CXL 的 CPU 可以轻松访问远程节点中的内存，并且比 RDMA 等网络接口技术的延迟更低，从而有效促进内存分解。

现有对 CXL 的模拟主要是使用远程 NUMA 节点，但这种仿真方式可能带来误导性的性能表征结果和次优设计决策。于是作者在本文比较了真实 CXL 和模拟 CXL 的性能，并对 CPU 和 CXL 内存之间的复杂相互作用进行了深入分析，基于这些分析，做出了以下贡献：

1. CXL 内存 != 远程 NUMA 内存。
    - 不同的 CXL 控制器设计或内存技术，将使得真正的 CXL 访问延迟和带宽具有很大不同；
    - 与模拟的 CXL 内存相比，真正的 CXL 内存可降低高达 26% 的延迟，并提高 3%-66% 的带宽效率；
    - 使用真正的 CXL 内存后，多个节点上的 CPU 可以将它们的 L2 缓存行驱逐到所有的 LLC 缓存，因此，CPU 访问 CXL 内存时，可以从超大容量的 LLC 缓存中受益，特别是缓存友好型工作负载。
2. 盲目使用 CXL 内存可能是有害的。
    - 要求 us 级延迟的简单应用程序（比如内存键值存储）对内存访问延迟高度敏感，与本地 DDR 内存相比，将页面分配给 CXL 内存会使得尾延迟增加 1082%；
    - 现有的 CXL 内存感知的多级页面放置策略 TPP，相比于在 DDR 内存和 CXL 内存之间静态分区页面，可能会增加尾延迟，这是因为存在页面迁移开销；
    - ......
3. CXL 内存感知的动态页面分配策略 Caption。
    - Caption 首先确定制造商特定的 CXL 内存设备的带宽，随后，定期监控各种 CPU 计数器（比如内存访问延迟），并评估它们在运行时消耗的带宽；
    - 根据上一步监控的计数器值，估计一段时间内的内存子系统性能。当给定应用程序需要分配新页面时，Caption 考虑内存子系统性能的历史记录以及过去分配给 CXL 内存的页面百分比；
    - 然后，使用简单的贪心算法调整分配给 CXL 内存的页面百分比，以提高整体系统吞吐量。

# 内存延迟和带宽特征

本节评估不同 CXL 内存设备、基于 NUMA 节点模拟的 CXL 内存的延迟和带宽，并研究 CPU 和 CXL 内存设备之间的交互。

## 延迟

1. 全双工 CXL 和 UPI 接口可减少内存访问延迟；

2. 访问真实 CXL 内存设备的延迟高度依赖给定的 CXL 控制器设计；

3. 模拟 CXL 内存的延迟更高；

    这是因为向模拟 CXL 内存发出请求时，本地 CPU 必须首先检查通过芯片间 UPI 接口连接的远程 CPU 的缓存一致性；此外，内存请求必须通过远程 CPU 内的长芯片内互联才能到达其内存控制器，这些开销也随着 CPU 内核的增加而增加。

## 带宽

以带宽效率（实际最大吞吐量/理论最大带宽）为标准。

1. 带宽效率很大程度上取决于 CXL 控制器的效率；
2. 真实的 CXL 内存写入操作带宽效率比模拟 CXL 内存高。

## 缓存层次交互

与访问本地内存相比，访问 CXL 内存时，更大的有效 LLC 容量会暴漏给 CPU 内核（可以利用其它 SNC 节点的 LLC 缓存）。

# 使用 CXL 对应用程序的性能影响

## 延迟

内存密集型应用程序分配给页面不同比例的 CXL 内存时，应用程序的 P99 时延。看起来，CXL 内存时延大致是本地 DRAM 的 2 倍。

对于 CXL 内存的分配方式，由于现有 SOTA 策略 TPP 的页面迁移开销，导致使用此页面分配方法时，延迟比静态分配（固定以 75% 概率分配给本地 DRAM，25% 概率分配给 CXL）还高。

此外，端到端应用程序存在很多前端逻辑开销，因此访问 CXL 内存的长延迟对此类应用程序的影响较小。  

## 吞吐量

**将 100% 的页面分配给 DDR 内存时，当线程数到达 20 个，吞吐量达到饱和，而如果此时适当地分配一些页面给 CXL 内存，吞吐量可以进一步提高。**

然而，如果本地 DDR 内存带宽没有跑满，此时将一些页面分配给 CXL，虽然总带宽增加了，但吞吐量会下降，因此，作者也就提出了一个动态页面分配策略，**根据给定 CXL 内存设备的带宽能力和当前所有应用程序消耗的带宽，自动配置运行时分配给 CXL 内存的页面百分比。**