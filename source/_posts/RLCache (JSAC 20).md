---
title: RLCache (JSAC 20)
date: 2023-7-10 17:52:57
description: RLCache (JSAC 20) 论文阅读
tags:
- Cache
- ML
- 论文阅读
categories:
- Cache
hidden: true
---

使用强化学习进行缓存准入决策。

# RL-Cache

当前请求对象的特征向量为**状态**，缓存准入为**决策**，缓存命中为**奖励**；

## 特征选择

{% asset_img feature.jpg feature %}

## 网络

有着指数线性单元激活函数的全连接神经网络，具有**5**个隐藏层，输出层使用 softmax 激活函数，并包含两个神经元，它们生成允许或不允许请求的对象进入缓存的两个相应结果的概率。

## 训练

1. 使用 MC 采样生成 m 个**缓存准入决策序列**，每个序列包含 k 次**缓存准入决策**；
2. 从额外带有 L 个后续请求的 m 个缓存准入决策序列中，选择奖励值 Top p% 的序列，以引导策略搜索朝向最大化命中率方向；
    - 奖励函数：{% asset_img reward.jpg reward %}
3. 利用上一步选择的序列，通过使用二元交叉损失函数进行反向传播，更新权重。

当权重 w 收敛或者迭代次数达到阈值，将滑动到下一个 **k** 请求窗口，重复以上步骤。

{% asset_img train.jpg train %}