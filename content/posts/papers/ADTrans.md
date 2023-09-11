---
title: "ADTrans"
date: 2023-08-13T17:39:50+08:00
draft: true
categories: []
tags: [SSG, PSG, ]
author: pan
---


## 一、Introduction

### 1.1 该论文试图解决什么问题？

由于标注者的语言偏好和关系之间存在语义重叠导致有偏的（biased）数据标注。该论文提出ADTrans框架可以自适应的迁移有偏的关系标注（biased predicate）到更有信息量（informative）和统一的（unified）标注。

具体的，需要修正两种关系标注，（1）有语义重叠的难以区分的三语组，（2）被标注者丢弃的潜在的正样本

### 1.2 创新点

1. 提出即插即用的框架ADTrans, 可以自适应的、更准确的将数据迁移到一个更informative和统一标准标签的数据。
2. 提出一个基于原型的关系表示学习方法（prototype-based predicate representation learning method），在文本域（textual domain）和关系域（relationship domain）之间进行更合理的对齐处理。
3. 全面综合实验表明ADTrans可以提升之前方法的性能，达到新的SOTA.

## 二、Method

### Relation Representation Extraction

通过对比学习，获取关系的表示

### Semantics-prototype Learning

将数据集中的每个关系都映射到一个语义的原型空间（取均值）。

### Multistage Data Filtration

偏离方差过大

### Data Transfer

看样本离Semantics-prototype空间谁近

## Experiments