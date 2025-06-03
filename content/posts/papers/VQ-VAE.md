---
title: "VQ VAE"
date: 2024-10-26T15:04:02+08:00
draft: true
categories: [paper]
tags: []
author: pan
---
1. Title: Neural Discrete Representation Learning
2. 作者: DeepMind
3. 发表日期: 2018.3

## 一、Introduction

### 1.1 该论文试图解决什么问题？

提出一个从**离散表示**中学习到的简单强大的生成模型，改模型没有VAE模型中的后验坍塌问题（posterior collapse）。将这个模型和自回归先验结合，可以生成高质量的图片、语音，并可以进行高质量的说话人转换（speaker conversion）和无监督的音素学习（unsupervised learning of phonemes）。

图片为什么需要离散化表示？有什么好处？

1. 语言天然就是离散的，图片可以准确的被语言表示
2. 离散的表示天然的适合复杂推理、规划、预测（e.g., if it rains, I will use an umbrella）
3. 训练简单（Decoder依赖于codebook， 不会有大variance），因此可以加大数据集，训练更大的模型（个人理解），这一点在大模型时代，符合scale law
4. 可以避免VAE模型中的后验坍塌问题（posterior collapse）。

### posterior collapse

VAE模型训练过程会使后验分布逼近先验分布，即使后验分布和先验分布的KL散度越小越好。记为$D_{KL}(q_\phi(z|x)||p_\theta(z))$。如果万一这里的KL散度针对为0，即Decoder的输入将不在含有x的信息，变成Decoder直接从p(z)中生成了，那VAE模型失效（要么模型不收敛、要不模型趋向于一个常量（x的均值））

### 1.2 Key Contributions

1. 引入简单、离散latent的VQ-VAE模型， 没有后验坍塌问题和方差问题（does not suffer from "posterior collapse" and has no variance issues）。
2. 展示了VQ-VAE模型可以和连续模型变现一样。
3. 当与强大的先验配对时，我们的样本在各种应用（如语音和视频生成）上是连贯和高质量的
4. 我们展示了在没有任何监督的情况下通过原始语音学习语言的证据，并展示了无监督说话人转换的应用。

## Method

### Experiments

