---
title: "Deepseek系列论文解析"
date: 2025-03-01T22:02:24+08:00
draft: False
categories: [paper]
tags: []
author: pan
---
1. Title: Deepseek 系列论文解析
2. 作者: DeepSeek AI

2025春节期间，Deepseek爆火，而且还是先从外网火到内网。DeepSeek在各大专业评价基准上与open AI的O1不相上下。本来这应该是国内最大几个公司应该干的事情，竟然被一个做量化的公司干了。
最近抽空把DeepSeek的几篇论文都读了一些，其中DeepSeek V2、V3、R1三篇论文我详细读了，并详细整理了阅读笔记，以供大家参考。DeepSeek V1、V2、V3、R1 四篇论文的发布时间跨度在一年左右，所以DeepSeek团队的节奏是很快的。而且四篇论文结构都很清晰，基本每篇都是从Architecture、Pre-Traing、Post-Training几个角度阐释，而且几篇论文衔接的都很紧密。以下大体梳理一下几篇文章的重点，有了这些先验，再去读者几篇文章会更容易抓住重点。

- DeepSeek v1: 主要探究了大模型时代下Scaling law, 比如在算力预算下，什么样超参数是最优的、数据缩放策略、如何估计模型最终的性能。所以DeepSeek v1是为后面做更大的模型准备的。
- DeepSeek v2: 主打省钱（economical training）、快（efficient inference）、好（优于更大规模的模型）。总236B参数，但是每个token只激活21B参数。相对于DeepSeek 67B，DeepSeek-V2效果更好，节省了42.5%的训练成本，减少了93.3%的KV cache，提升生成吞吐量5.76倍。Transformer主要就两个模块，一个MHA、一个FFN，DeepSeek v2都对其做了修改，对与MHA部分，提出MLA(Multi-head Latent Attention),大大减少了KV cache，极大的提升了推理的性能。对于FFN，引入MOE架构，再次提升推理性能。
- DeepSeek v3：671B总参数量，37B激活参数量。延用了deepseek v2中的MLA、MOE架构。DeepSeek-V3在moe的专家路由上做了一些改进，提成auxiliary-loss-free strategy。除此之外，deepseek-v3提出了MTP(multi-token prediction), 进一步提升了性能。
- DeepSeek R1: 介绍了deepseek团队第一代的两个reasoning模型：DeepSeek-R1-Zero and DeepSeek-R1。
  - DeepSeek-R1-Zero ：无SFT,直接使用大规模强化学习得到的模型，其展示了强大的推理能力，但是存在差的可读性和语言混乱问题（即模型答复不符合人的阅读习惯，存在多种语言混合输出的问题）。
  - DeepSeek-R1：为了解决DeepSeek-R1-Zero的缺点和进一步提升推理能力，训练了DeepSeek-R1，其在强化学习之前包含了multi-stage training and cold-start data。
在推理任务上，DeepSeek-R1取得了和openai-o1 comparable的结果。DeepSeek-AI开源了DeepSeek-R1-Zero 、 DeepSeek-R1以及6个蒸馏得到的小模型(1.5B, 7B, 8B, 14B, 32B, 70B)。

关于这4篇论文详细的演变过程，见下表。DeepSeek V2、V3、R1三篇论文详细的阅读笔记见我的飞书文档
**[deepseek系列论文解析](https://nw821o5xhc.feishu.cn/wiki/PGSRwWYWaieVDkkXksac4yoXnxd)** 。

![DeepSeek系列概览](/deepseek系列论文解析/Xnip2025-03-23_23-57-05.jpg)