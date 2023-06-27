---
title: "IMAGEBIND: One Embedding Space To Bind Them All"
date: 2023-06-26T23:15:33+08:00
draft: False
categories: [paper]
tags: ["IMAGEBIND", "Vision-Language model", "pre-training", "multimodality"]
author: pan
---


## 一、Introduction

### 1.1 该论文试图解决什么问题？

该论文主要解决的多模态对齐的问题，该论文将图片（视频）、文本、音频、深度图、热力图（thermal）、IMU六种模态的特征对齐在一个空间。
所以IMAGEBIND可以做跨模态召回（cross-modal retrieval）、简单相加融合模态信息（composing modalities with arithmetic）、跨模态检测和生成（cross-modal detection and generation）等任务。另外IMAGEBIND的few-shot能力也不错

### 补充说明

1. 目前主流的方法还是将图片和文本（或者声音）对齐，比如CLIP（Audio-CLIP）。但是没有像IMAGEBIND方法这样讲6种模态的特征对齐，本质原因是没有6种模态对齐的训练数据（指一条样本对包含的6种模态数据完成对应）。但是每一种模态和图片成对的数量是够的，就是（图片-文本）、（图片-音频）、（图片-深度图）、（图片-热力图）、（图片-IMU）这种成对的数据是够的。IMAGEBIND就是把所有模态的数据都和图片这个模态的数据进行对齐。那么比如（文本-音频）、（文本-深度图）等跨模态的数据就也对齐的。这种在数学上叫做传递性，因为所有模态的相似度量是用的cosine距离，这个度量方式就是可传递的，所以IMAGEBIND能把这么多模态对齐是显然的。
2. emergent zero-shot：由于IMAGEBIND是将其他模态和图片模态配对然后训练，其它的模态对是没有进行训练的，比如（文本-音频）、（文本-深度图）。所以（文本-音频）的召回或者分类能力，IMAGEBIND叫做涌现的zero-shot能力。
3. 至于网络结构损失函数等，并没有新的东西。甚至图像-文本的模态对齐就是用的**CLIP**（文中用的OPEN-CLIP），直接**frozen掉没有训练**

## Method

ImageBind的网络结构没有什么新的架构，无非就是不同规模的VIT结构。损失与CLIP的对比损失不同，用的是InfoNCE loss。公式如下：

![InfoNce](/papers_ImageBind/ImageBind_3.png)
其中$q_i$, $k_i$分别表示图片、其它模态数据经过encoder后的embedding。$\tau$表示温度，用于控制softmax后的平滑程度。

### Experiments

1. ImageBind的应用
![InfoNce](/papers_ImageBind/ImageBind_1.png)
   + 1. 跨模态召回
   + 2. embeding相加就等价于语义的相加
   + 3. 声音生产图片
2. ImageBind使用的数据样例
   ![InfoNce](/papers_ImageBind/ImageBind_2.png)
   都是自然与图片配对的数据
3. ImageBind使用的评测数据集
   ![InfoNce](/papers_ImageBind/ImageBind_4.png)
   可以看到都是分类、召回类的任务
4. Emergent zero-shot分类能力
   ![InfoNce](/papers_ImageBind/ImageBind_5.png)
   音频的分类任务重ImageBind与AudioCLIP对比，但是AudioCLIP是直接在（text, audio）成对的数据上训练的，且AudioCLIP用到了AS类别信息，所以ImageBind提到AudioCLIP的指标不能算zero-shot，所以AudioCLIP的指标对ImageBind的高一点
5. 文本召回视频
   ![InfoNce](/papers_ImageBind/ImageBind_6.png)
   A: Audio, V:Video。 可以看到用音频和图片的联合embedding取得了最好的效果。
6. Few-shot能力
   ![InfoNce](/papers_ImageBind/ImageBind_7.png)
7. 使用不同规模的Image Encoder
   ![InfoNce](/papers_ImageBind/ImageBind_8.png)
8. 关于温度（损失函数中用于控制平衡的参数，见损失公式）$\tau$的影响
   ![InfoNce](/papers_ImageBind/ImageBind_9.png)