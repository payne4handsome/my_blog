---
title: "Qwen-VL系列论文解析"
date: 2025-12-14T16:11:51+08:00
draft: False
categories: [paper]
tags: [VLM]
author: pan
---

# Updates

- 2025.12.14: 添加Qwen3-VL工作整理
- 2025.03.12: 完成Qwen-VL、Qwen2-VL、Qwen2.5-VL系列工作梳理

## Qwen3-VL

随着2025.5 qwen3发布后，社区就一直期待Qwen3-VL的发布，直到9月底Qwen3-VL终于发布且开源。technical report 到11月底才发布。这一年大家都在做推理模型，qwen3也不例外，一开始发布的模型是一个模型同时支持thinking和non-thinking，后来7月份又拆开分为两个模型。可见大公司在技术方向的选择也出现的分歧。
Qwen3-VL相对于Qwen2.5-VL改变还是比较大的。首先在模型尺寸上就有最大的Qwen2.5-VL-72B来到了Qwen3-VL-235B-A22B。Qwen2.5-VL-72B是不支持thinking的（指的是没有使用RL）。Qwen3-VL-235B-A22B包含thinking和non-thinking两个版本。在模型架构上位置编码由MRoPE改进到interleaved-MRoPE，解决位置编码中宽、高、时间三个维度频率不平衡问题（时间在高维度，低频段，导致长视频任务效果变差）；虽然还是经典的Vision-LLM-Adapter架构，但是vision特征和LLM特征不是简单的拼接了，而是做了提出vison不同层级的特征和LLM做特征融合；与绝对时间对齐的position id也变成了文本方式去表示时间（同seed1.5-VL）。在任务数量上也做了不少的扩充，如agent能力等。上下文长度上也由32k增加到256k（原生支持，不是通过YaRN等扩展位置编码， 可扩展至1M）。

## Qwen-VL、Qwen2-VL、Qwen2.5-VL

目前在多模大模型领域，学术界、工业界一般使用LLaVA系列模型或者Qwen-VL系列模型作为基底模型，然后再根据自已的研究方向或者自已公司业务做SFT。如果需要中文的支持，那用Qwen作为基底模型是更合适的。Qwen-VL，也就是Qwen的第一个版本，在2023.10月就发布了。我特地查了一下BLIP模型早在2022.2月就发布了，我大概在2023年8、9月开始基于InstructBLIP(发表于2023.5)和LLaVA（发表于2023.4），基于公司的业务需要做了一些探索。虽然在一些场景下，可以满足公司业务一定的需要，但是里真正的商用还是有一定的距离。现在，眼看着AGI的临近（可能有点乐观了，但是在很多任务上超过传统的模型，还是可以的），QWen也更新到2.5版本，国内再加上DeepSeek的加持，多模领域在未来两年一定会是大家关注的热点，所以我最近把Qwen-VL、Qwen2-VL、Qwen2.5-VL系列工作重新梳理了一下，以供参考。整体脉络如下。**详细的论文阅读笔记见我的飞书文档：**
**[Qwen-VL系列论文解析](https://nw821o5xhc.feishu.cn/wiki/Qlx8wqYtjiH1jqkt6UucevSTn9e?fromScene=spaceOverview)**

![Qwen系列概览](/Qwen系列论文解析/Xnip2025-03-30_22-02-40.jpg)