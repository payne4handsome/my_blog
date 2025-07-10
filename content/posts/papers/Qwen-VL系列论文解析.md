---
title: "Qwen-VL系列论文解析"
date: 2025-03-12T19:11:51+08:00
draft: False
categories: [paper]
tags: [VLM]
author: pan
---

目前在多模大模型领域，学术界、工业界一般使用LLaVA系列模型或者Qwen-VL系列模型作为基底模型，然后再根据自已的研究方向或者自已公司业务做SFT。如果需要中文的支持，那用Qwen作为基底模型是更合适的。Qwen-VL，也就是Qwen的第一个版本，在2023.10月就发布了。我特地查了一下BLIP模型早在2022.2月就发布了，我大概在2023年8、9月开始基于InstructBLIP(发表于2023.5)和LLaVA（发表于2023.4），基于公司的业务需要做了一些探索。虽然在一些场景下，可以满足公司业务一定的需要，但是里真正的商用还是有一定的距离。现在，眼看着AGI的临近（可能有点乐观了，但是在很多任务上超过传统的模型，还是可以的），QWen也更新到2.5版本，国内再加上DeepSeek的加持，多模领域在未来两年一定会是大家关注的热点，所以我最近把Qwen-VL、Qwen2-VL、Qwen2.5-VL系列工作重新梳理了一下，以供参考。整体脉络如下。**详细的论文阅读笔记见我的飞书文档：**
**[Qwen-VL系列论文解析](https://nw821o5xhc.feishu.cn/wiki/Qlx8wqYtjiH1jqkt6UucevSTn9e?fromScene=spaceOverview)**

![Qwen系列概览](/Qwen系列论文解析/Xnip2025-03-30_22-02-40.jpg)
LLaVA系列的工作我也在整理，不过还没有整理完，先放个链接吧。[【更新中】LLaVA系列论文整理](https://nw821o5xhc.feishu.cn/wiki/E0C6wHrcOiwdNBkUjqJcHIDen6J?fromScene=spaceOverview)