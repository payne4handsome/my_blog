---
title: "GLM-VL系列论文解析"
date: 2025-08-24T21:11:02+08:00
draft: false
categories: [paper]
tags: [VLM]
author: pan
---

发布GLM-4.1V-9B-Thinking和GLM-4.5V两个模型，其中GLM-4.5V是一个参数量106B，激活12B的MOE结构的模型，且包含thinking和non-thinking两个。其中比较有亮点的是GLM-4.1V-9B-Thinking一个9B的模型在29个Benchmark上超过了Qwen2.5- VL -72B（non-thinking模型）。大模型的发展方向有两个，一个往大的方向发展：不断的探索scaling law。一个是往小的方向发展：参数量比较你小，但是性能比你好。所以GLM-4.1V-9B-Thinking一个9B参数的模型在多个方面超过一个72B的模型，还是很令人吃惊的。
GLM-4.1V-9B-Thinking成功的关键我觉得有两个：1. 高质量数据的构建（具体数量位置，数据集也没有开源）2. ReinforcementLearning with Curriculum Sampling (RLCS) ，RLCS在训练的过程通过样本的困难程度动态的去采样合适难度的样本（不要太难、也不要太简单，seed-1.5VL中有同样的思想），这个不是超过Qwen-72B的关键，关键其实还是在RL阶段构建的任务是综合的，包含各种任务，在训练方法上即包括RLVF也包含RLHF，并且两者结合。
对于大规模的RL，很容易训练不稳定，智普团队在这篇论文也给出一些发现和洞察，比如针对各个任务设计合适的奖励系统，要不然很容易遇到reward hacking等问题。

**详细的论文阅读笔记见我的飞书文档：**
**[GLM系列论文阅读](https://nw821o5xhc.feishu.cn/wiki/RYOQw7pOEi4bKvksiijcwfKxnng)**

