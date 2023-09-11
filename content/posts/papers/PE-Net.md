---
title: "PE Net"
date: 2023-08-20T17:55:49+08:00
draft: true
categories: [paper]
tags: []
author: pan
---
1. Title: Prototype-based Embedding Network for Scene Graph Generation
2. 作者: Chaofan Zheng, Xinyu Lyu, Lianli Gao†, Bo Dai, Jingkuan Son
3. 发表日期: 2023.3

## 一、Introduction

### 1.1 该论文试图解决什么问题？

许多subject-object对之间视觉外观存在多样性，导致类内方差大（intra-class variation）比如（"man-eating-pizza, giraffe-eating-leaf"）；类间相似（inter-class similarity）比如（"man-holding-plate, man-eating-pizza"）。导致当前的SGG方法无法捕获关系的compact and distinctive representations，无法学习到一个完美的决策边界（perfect decision boundaries）用于关系预测。
该文提出PE-Net（Prototype-based Embedding Network）网络，该网络用原型对齐的紧凑的有区分的表示（prototype-aligned compact and distinctive representations）来对实体和关系建模。最后关系的预测在常规的embedding空间进行。PE-Net还包含两个模块，作用如下：
Prototype-guided Learning (PL, 原型引导的学习): 帮助有效的学习谓词匹配
Prototype Regularization (PR)：缓解由语义重叠（semantic overlap）带来的二义性谓词匹配问题

解决思路

1. 类内（intra-class）: 紧凑性（compactness）
2. 类间（inter-class）: 区别性（distinctiveness）

关于prototype的理解：比如人eating，狗eating，马eating，对于具体的实例来讲，是不一样的，但是对于eating这个含义是一样，这个共性的含义就叫prototype

### 1.2 Key Contributions

1. 提出一个简单且有效的方法PE-Net，其生成compact and distinctive的实体|关系表征，然后建立实体对和关系的匹配用于关系识别。
2. 引入Prototype-guided Learning (PL)帮助PE-Net有效的学习，设计Prototype Regularization (PR)去缓解由语义重叠造成的二义性匹配问题
3. 在VG和Open Images上，显著提升关系识别能力，取得新的SOTA。

## Method

### Experiments

![PE-Net实验结果](/papers_PE-Net/PE-Net_1.png)