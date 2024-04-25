---
title: SLAP (IPDPS 23)
date: 2023-7-11 12:52:57
description: SLAP (IPDPS 23) 论文阅读
tags:
- Cache
- ML
- 论文阅读
categories:
- Cache
hidden: true
---

使用 LSTM 进行缓存准入决策。

# Design

## Feature

1. 用户访问特征：新近度、访问时间、频率、操作系统；
2. 对象信息：对象大小、对象类型；
3. 工作负载信息：最近命中率、访问 URL；

## Architecture

{% asset_img module.jpg module %}

1. Request Information Collector：收集最近请求的特征以及真实的重用时间信息；

    - 收集最近请求的特征，并计算出最近一小时的重用率 R；当缓存缺失时，最近访问特征被输入到 LSTM 计算对应状态并写入到 Historical Information Database；

        {% asset_img 重用率.jpg 重用率 %}

    - 当先前缺失的对象再次被访问，可以得到真实的重用距离，和预测结果进行对比，

2. Labeling Model：模型训练和预测；

    - 预测准入率低于 75% 则重新训练，从 Historical Information Database 中提取访问信息离线训练；
    - 当对象缺失时，状态值（最近访问信息）和当前对象特征传递到预测器输入层的最后一个单元格以预测重用时间标签；

    