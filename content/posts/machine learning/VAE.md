---
layout: post
title: "VAE"
date: 2023-3-20
# menu: main
categories: [机器学习]
tags: [VAE]
# cover:
#     image : '/cover.jpeg'
#     alt: "<alt text>"
#     caption: "<text>"
#     relative: false
---

# Vanilla VAE(Variational Autoencoder)

## 一、AutoEncoder 回顾

![autoencoder](/VAE/8596800-8666af110c3d6d37.png)

## 生成模型

最理想的生成就是知道输入样本的分布$P(X)$, 然后我们并不知道该分布。那么可以近似求解。
$P(X) = \Sigma P(X|Z)*P(Z)$。但$P(X|Z), P(Z)$我们同样不知道。但是我们可以用神经网络去学习这两个分布。上图中的latent vector可以看成是$P(Z)$的一个采样，decoder可以看成条件概率$P(X|Z)$。但是我们真的可以采样一个z，然后用加一个decoder来作为我们的生成模型吗？

1. z是Encoder对应着样本X的输出，如果我们直接用Decoder对z还原，那么最终得到的$\hat{X}$是和X是差不多的，我们需要生成模型是生成一个和X类似的，而不是一模一样的
2. 如果对z做一些扰动，必然加一些噪声，那是不是就可以生成类似但是不一样的东西呢？理论上是可以，但是到目前为止，我们的模型并没有保证这一点（模型还没有学习） 

**加噪声是一个好的思路，如何加噪声？**

让z从一个分布采样（注意不是直接使用encoder的输出），就是噪声。
那不放让z从一个$N(u, \sigma^2)$中采样。那需要知道$u, \sigma^2$, 既然不知道那就用神经网络生成吧。
![image.png](/VAE/8596800-6c07e2d296d44414.png)
如果我们按照上述去训练我们的模型，生成的方差$\sigma^2$会倾向于变成0（因为容易学）。那如何加以限制？**使z倾向于一个标准正态分布**，即$\sigma^2$倾向于1。 如下图
![image.png](/VAE/8596800-c505c2a1e88756f5.png)

如何监督模型达到该目的，KL loss作为监督信号，KL loss如下
![image.png](/VAE/8596800-6b4357a33bd6974e.png)

## reparameterization trick

让z从$N(u, \sigma^2)$中采样，由于这个操作是不可导的，所以需要使用重采样技巧去解决不可导的问题。
$$z \sim N(u, \sigma^2) \iff z \sim u+\sigma^2 \times \epsilon$$, 其中$\epsilon \sim N(0,1)$
![image.png](/VAE/8596800-2a126d0c68120783.png)

## 思考？

1. 为什么要正态分布、其它分布可否？

## Variational Bayes

什么是变分贝叶斯推断？将贝叶斯后验概率计算问题转化为一个优化问题（计算->优化）。


## ..... 待补充完善

## 参考文献

1. Auto-Encoding Variational Bayes
2. Tutorial on Variational Autoencoders
3. [变分自编码器](https://spaces.ac.cn/archives/5253)
4. [李宏毅深度生成模型](https://www.bilibili.com/video/BV1Ux411S7Yk?p=18&vd_source=9ddbc59809ae2e80b717c2d15bbb9f69)