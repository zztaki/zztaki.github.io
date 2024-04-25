---
title: S3fifo (SOSP 23)
date: 2024-4-23 17:52:57
description: S3fifo (SOSP 23) 论文阅读
tags:
- Cache
- 论文阅读
categories:
- Cache
---

# Abstract

FIFO 优势：简单，速度快，可扩展性高，闪存友好；

FIFO 劣势：缓存缺失率高。

Insight：大多数对象在短时间内只会被访问一次，将这些对象尽早地从缓存中驱逐很关键。

# Introduction

1. 缓存指标
    - 高效率：命中率高；
    - 高性能：缓存命中或驱逐的操作尽可能少，保证系统高吞吐；
    - 高可扩展性：随着 CPU 核心数量的增长，系统吞吐尽可能线性增长；
2. 