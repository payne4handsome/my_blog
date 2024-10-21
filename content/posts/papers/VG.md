---
title: "Visual Genome:Connecting Language and Vision Using Crowdsourced Dense Image Annotations"
date: 2023-07-14T13:55:24+08:00
draft: true
categories: ["paper"]
tags: ["VG", "Visual Genome", "Dataset"]
author: pan
---

1. Title: Visual Genome: 用众包稠密图片标注去连接语言和视觉
2. 作者: Ranjay Krishna  Yuke Zhu  Oliver Groth  Justin Johnson Kenji Hata  Joshua Kravitz  Stephanie Chen  Yannis Kalantidis Li-Jia Li  David A. Shamma  Michael S. Bernstein  Li Fei-Fei
3. 发表日期: 2016.2

一、介绍

1. 官网地址：https://homes.cs.washington.edu/~ranjay/visualgenome/index.html

2. 数据统计信息

- 108,077 Images
- 5.4 Million Region Descriptions
- 1.7 Million Visual Question Answers
- 3.8 Million Object Instances
- 2.8 Million Attributes
- 2.3 Million Relationships

1. 来源

  The first release of the Visual Genome dataset uses 108, 249 images from the intersection of the YFCC100M and MS-COCO.


1. 备注说明
  1. 该数据集有v1.0, v1.2, v1.4三个版本，最新的v1.4版本清洗了object标注、新增了关系等，但是下载提示403 forbidden错误，应该是没有放开权限。所以目前下载的数据集为v1.2版本。
    v1.4的官方readme


二、VG数据集组成部分
VG数据集的构建是从收集描述和问答开始的。目标、属性、关系来自于描述（To enable research on comprehensive understanding of images, we begin by collecting descriptions and question answers. These are raw texts without any restrictions on length or vocabulary. Next, we extract objects, attributes and relationships from our descriptions.）

- region descriptions
  -  一个简单的总结句子不足以描述所有的内容；所以可以分区域描述
  - 每个区域都有一个bounding box
  - 每张图片平均有42个bounding box和区域描述
- Objects
  - 每个object映射到WordNet的同义词集合，通过synset ID
  - 每种图片大约21个目标
- Attributes
  - 颜色、状态（standing）等
  - 每张图片平均包含21个属性
- Relationships
  - actions (jumping over), spatial (is behind), verbs (wear), prepositions (with)，comparative (taller than), or prepositional phrases (drive on)等
  - 每张图片平均19个关系
- region graphs
  - 从区域描述中组合目标、属性、关系得到
- scene graphs
  - 包含一整张图的所有信息，目标、属性、关系
- questionanswer pairs
  - 问答类型
    - 自由问答，基于整张图片
    - 基于区域的问答
  - 问题类型：what, where, how, when, who, and why
  - 每张图片大约17个QA

三、标注策略

3.1  区域描述标注
使用AMT(Amazon Mechanical Turk, 亚马逊众包标注平台)标注，超过33000人标注过该数据集
1. 对于一张新的图片，标注人员画3个bounding box和写3个bounding box中内容的描述
2. 将步骤1后的图片和描述发送下一个标注人员，鼓励标注人员写没有被写过的描述
3. 重复2， 直到每张图片收集超过50个区域描述
通过BLEU去计算相似描述
[图片]
3.2 目标标注
当每张图片收集到50个描述后开始目标标注，每个描述发给一个标注人员。两个要求
1. 覆盖范围（coverage）：必须完全覆盖着目标
2. 质量（quality）：尽可能的紧凑，要求是4个像素
3.3 属性、关系、区域图
当有了区域描述和目标信息后，开始标注属性、关系。
Eg: "woman in shorts is standing behind the man" 将会标注会woman标上standing的属性。产生关系，in(woman, shorts) ， behind (woman, man).
3.4 问题标注
标注要求如下：
1. 问题的开始是如下的一个，who, what, where, when, why, how , which
2. 避免模棱两可和推测性的问题（avoid ambiguous and speculative questions）
3. 精确和唯一：当给出图片和问题，回答必须是确定的、清楚的
标注流程：
包含两种类型问答：freeform QAs and region-based QAs
对于freeform类型问答，每个标注人员写8个，其中必要3个是不同类型的（即who, what, where, when, why, how , which的三个）
3.5 核查
标注完以后需要核查，剔除不正确的标注的目标、属性、关系。删除不清楚的、主观的、武断的标注。
Eg: This person seems to enjoy the sun；room looks dirty；Being exposed to hot sun like this may cause cancer
3.6 规划化（Canonicalization）
由于同一个目标，不同的人可能打的标签不一样，比如对于一个人，有的打标“man”，有的打标“person”，“boy”等。
将打标信息关联到WordNet（WordNet中每个词的含义是确定的），然后将这些词的统一到一个词。这个部分论文中化了比较多的篇幅说明，实际操作应该是比较复杂的。该部分待完善......（大致是这样操作，不是很确定详细的操作流程）
四、统计信息
1. 宽高分布
[图片]
2. 区域描述单词数量的分布
[图片]
3. 区域描述中，使用频次最多的短语和单词
[图片]
4. 区域描述中目标的数量、每张图片中目标的数量
[图片]

[图片]

5. 目标、类别与其他数据集比较
[图片]
6. 经常出现的目标类别
[图片]
7. 属性统计
每张图片包含的属性分布
每个区域包含的属性分布
每个目标包含的属性分布
[图片]

[图片]

[图片]

8. 常见的属性
[图片]
9. 常见的描述人的属性
[图片]
10. 关系统计
每张图片包含的关系分布
每个区域包含的关系分布
每个目标包含的关系分布
[图片]

[图片]

[图片]

11. 常见的关系
[图片]
12. 涉及人的常见关系
[图片]
五、实验
5.1 属性预测
两个实验，（1）给定类别，预测属性；（2）同时预测类别和属性
[图片]
5.2 关系预测
两个实验，（1）只预测两个目标的关系；（2）同时预测目标和类别
[图片]
5.3 区域描述
[图片]
5.4 问答
[图片]


补充

1. wordnet 基本用法
   ![wordnet](/papers_VG/VG_WN_1.png)


参考资料
1. [NLTK Documentation](https://www.nltk.org/howto/wordnet.html)
2. [关于WordNet，我的一些用法和思路(一)](https://zhuanlan.zhihu.com/p/26461511)
3. [关于WordNet，我的一些用法和思路（二）](https://zhuanlan.zhihu.com/p/26527203)