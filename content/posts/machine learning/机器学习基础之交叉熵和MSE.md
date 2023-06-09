---
layout: post
title: "机器学习基础之交叉熵和MSE"
date: 2023-9-25
# menu: main
categories: [机器学习]
tags: [损失函数]
---

# 机器学习基础之交叉熵与均方误差

我们都知道，对于分类任务，可以选用交叉熵做为模型的损失函数；对于回归任务，可以选用MSE来作为模型的损失函数。那么分类任务能否选用MSE来做为损失函数；回归任务能否选用交叉熵来作为损失函数呢？
**本文只能尽可能尝试回答这个问题，帮助大家有个大概的认识，目前尚无法对其做严格的数学证明。如果大家看到对这个问题有很好的数学证明，欢迎讨论**

符号定义：

$N$: 类别数量

$y_i$: 样本onehot编码后label

$p_i$: 模型预测第i个类别的输出

那么可以用交叉熵和MSE来衡量真值和模型预测结果的偏差。公式如下：

交叉熵：$loss_{cross\_entropy}=-\sum_i^N y_ilog(p_i)$

MSE: $\frac{1}{N}\sum_i^N (y_i-p_i)^2$

CE是多项式分布的最大似然；

## 一、为什么分类任务用交叉熵，不能用MSE

### 1.1 直观感受

   假设真实标签为（1,0,0），预测结果一是（0.8,0.1,0.1）, 二是（0.8,0.15,0.05）。那么这两个预测哪个更好一点呢？
   两个预测结果的交叉熵都是$-log0.8=0.223$, 预测一的MSE=0.02, 预测二MSE=0.025。 即MSE任务预测一的结果要好于预测二。**MSE潜在的会让预测结果除真实标签以后的标签趋于平均分布**。但是实际上我们不能主观的认为预测结果一好于二。

### 1.2 凹凸性角度

### 1.2.1 使用sigmod激活、或者softmax，MSE是非凸的

我们知道，如果一个优化问题是凸优化，那么我们是可以找到全局最优解的。但是如果问题是非凸的，那么很有可能找的解是sub-optimal的。
我们用desmos（一个非常好的画图工具）画一个图来说明，对于分类问题，如果用MSE来作为损失函数，它的函数图像是非凸的。
![image.png](/机器学习基础之交叉熵和MSE/8596800-e2e2414ce66ac01a.png)
这个例子使用了7个样本，每个样本只具有单个特征。我们可以看到函数图像是**非凸**的。

在参考文献3中，作者也给出了简单的数学证明，过程如下：
![image.png](/机器学习基础之交叉熵和MSE/8596800-f65e6800570369d1.png)
![image.png](/机器学习基础之交叉熵和MSE/8596800-c2ba4736a000f0c7.png)
**但是以上证明只是证明了最简单的情况（逻辑回归），且只有一个参数$\theta$的情况，如果要证明多元函数是凸的，需要证明黑塞矩阵的正定的，这个很难证明**

### 1.2.2 交叉熵是凸的

还以逻辑回归为例。

$z = wx+b\\
a=\sigma(z)\\
P(Y=1;w)=a, P(Y=0;w)=1-a$

$\sigma=\frac{1}{1+e^(-x)}$是激活函数

交叉熵为$J(w)=-[y_ilog(a)+(1-y_i)log(1-a)]$

$\frac{\partial J(w)}{\partial w}=-x(y_i-a)$

$\frac{\partial^2 J(w)}{\partial w^2}=-x[-a*(1-a)*x]=x^2*a*(1-a)$, 其中$a \in (0,1)$, 所以交叉熵的二阶导是大于等于0的。所以交叉熵是凸的。 **注意上述证明是特例证明，非严格证明**

### 1.3 参数估计角度

**交叉熵多项式分布的极大似然估计**

对于样本${(x_1,y_1), (x_2, y_2), ...,(x_N, y_N)}$，使用逻辑回归来分类，那么这批样本的极大似然估计可以用如下式子表达，其中a(x)是sigmod激活

$L(w)=\prod_{i=1}^N(a_w(x_i))^{y_i}(1-a_w(x_i))^{1-y_i}$$

对数似然如下：

$ln(L(w))=\sum_{i=1}^N[y_iln(a(x_i))+(1-y_i)ln(1-a(x_i))]$

上述式子是不是很眼熟，**其实就是交叉熵**。

**其实，对于分类任务不能用MSE的原因是分类需要用sigmod或者softmax来作为激活函数，导致了MSE变成了非凸的函数**

## 二、回归任务用MSE，可以用交叉熵吗

对于回归问题使用MSE应该是没有问题的（其实有问题，你能证明此时的MSE是凸函数吗？），那么对于回归问题可以使用交叉熵吗？**我觉得对于合适的回归问题使用交叉熵是完全可以的，因为很多模型就是这么用的，比如训练GAN的是会把真实的label加上一个随机噪声来提高GAN的性能，但是损失就是交叉熵；在知识蒸馏中，Student学习Teacher的soft label也是用的交叉熵**。所以只要回归任务的真实值是属于（0，1）之间的，或者可以转换为（0,1）之间，那么就可以用交叉熵来学回归任务。

知乎上有一个高赞回答（见参考文献一），从参数估计的角度上讲

**MSE是高斯分布的最大似然**

![image.png](/机器学习基础之交叉熵和MSE/8596800-c55cfbf94b1bc45d.png)

**注意，网上很多文章都回归任务不能用交叉熵，也是有一定的道理的**

总结：总的来说，上面的一些表述都是不那么严格的，水平有限只能解释到这种程度，如果有更好的解释或者更权威的证明，请留言，万分感激。

参考文献：

1. [不理解为什么分类问题的代价函数是交叉熵而不是误差平方，为什么逻辑回归要配一个sigmod函数](https://www.zhihu.com/question/314185485)
2. [deamos画函数图像](https://www.desmos.com/calculator?lang=zh-CN)
3. [逻辑回归损失函数不使用MSE的原因](https://www.jianshu.com/p/af1e5cff21b9)
4. [分类必然交叉熵，回归无脑MSE？未必](https://zhuanlan.zhihu.com/p/362496849)