---
title: Large-scale Analysis at Twitter Cache Clusters（TOS 21）
date: 2023-4-17 12:52:57
description: Large-scale Analysis at Twitter Cache Clusters（TOS 21）论文阅读，Special Section on OSDI 2020
tags:
- Cache
- Plot
- Cache Trace Analysis
- 论文阅读
categories:
- Cache

---

**Abstract**

1. 现代 Web 服务广泛使用内存缓存来提高吞吐量并减少访问延迟；
2. 对生产系统进行工作负载分析，可以推动提高内存缓存系统有效性的研究；
3. 本文对 Twitter 的多个内存缓存集群进行分析，比如生命周期、流行度分布、大小分布等等。

# Production Stats and Workload Analysis

## Miss Ratio

{% asset_img 缺失率变化.jpg 缺失率变化 %}

## Request Rate

请求率呈现昼夜特征，同时，请求率的尖峰可能是由于热键，也可能是请求对象增多了。

{% asset_img 请求率变化.jpg 请求率变化 %}

## Types of Operations

对于多 trace 的数据集分析和评估，图 a 挺棒的。

{% asset_img 操作类型.jpg 操作类型 %}

## Time-To-Living（TTL）

TTL 相关的累计分布曲线。

{% asset_img TTL的累计分布.jpg TTL的累计分布 %}

不考虑 TTL，工作集大小是当前时间之前所有不同对象大小之和，对于 Web 场景，是无限递增的；但若是考虑了 TTL，工作集大小是当前时间之前所有未过期的不同对象大小之和，随着时间的推移，会趋向于平稳，因此，无限大的缓存容量是没有意义的。

{% asset_img TTL对工作集大小的影响.jpg TTL对工作集大小的影响 %}

## Popularity Distribution

一些负载在最流行部分和最不流行部分展现出和齐夫分布的差异。

{% asset_img 流行度.jpg 流行度 %}

## Object Size

推特数据集中的对象大多很小。

{% asset_img 对象大小.jpg 对象大小 %}

请求大小随时间变化的热力图。（相比将几个 CDF 曲线画在一个图里，用热力图可以看到更细粒度的差距）

{% asset_img 请求大小分布随时间变化.jpg 请求大小分布随时间变化 %}

## Reuse Distance

对于图 a，是一个经典适用 LRU 替换算法的重用距离累计分布，展现出最近访问的对象，有更高的概率被再次访问；而对于图 b，10^6^ 附近的重用距离占比很高，小于这附近值的缓存大小使用 LRU 效果会很差，因为 LRU 会驱逐重用距离为 10^6^ 附近的对象，而选择保留重用距离为 10^2^ ~ 10^5^ 的对象，FIFO 则平等对待所有对象，不会牺牲全部的较大重用距离对象而选择保留全部重用距离较小的对象。

{% asset_img 重用距离的CDF.jpg 重用距离的CDF %}