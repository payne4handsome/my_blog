---
title: "QWen2 VL"
date: 2024-10-05T13:35:49+08:00
draft: true
categories: [paper]
tags: []
author: pan
---
1. Title: Qwen2-VL: Enhancing Vision-Language Model's Perception of the World at Any Resolution
2. 作者: Qwen Team Alibaba Group
3. 发表日期: 2024.9

## 一、Introduction

### 1.1 该论文试图解决什么问题？

Qwen-VL升级版本，QWen-VL介绍见：[QWen-VL](https://nw821o5xhc.feishu.cn/wiki/JD66wDn9UixQ8Ckm2izcswjLnTc?fromScene=spaceOverview)。QWen2-VL重新定义了预处理时候需要事先决定图片分辨率的惯例。QWen2-VL引入了动态原生的动态分辨率处理机制，允许模型处理不同的分别率图片成不同数量的viusal tokens。

### 1.2 Key Contributions

## Method

训练方法与QWen-VL一致，三阶段。主要的创新点就是引入了NaViT的可以动态处理不同分辨率、长宽比的图片还是多模的旋转位置编码（M-RoPE）. 其它方面透露的信息不多。

### Experiments
