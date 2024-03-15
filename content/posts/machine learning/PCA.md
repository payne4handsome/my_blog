---
title: "PCA"
date: 2024-03-10T14:51:51+08:00
draft: true
categories: []
tags: [PCA]
author: pan
---

# preliminary

## 特征值与特征向量

设A为n阶实**方阵**，如果存在某个数$\lambda$及某个n维**非零**列向量$x$，使得$Ax=\lambda x$，则称$\lambda$是方阵A的一个特征值，$x$是方阵A的属于特征值的一个特征向量。

### 特征值与特征向量求解

$$Ax=\lambda x, x\not ={0}
\iff (A-\lambda E)x=0, x\not ={0} \\
\iff |A-\lambda E|=0
$$
其中$|A-\lambda E|$称为特征多项式。
> 注：n阶方阵一定存在n个特征根（可能存在复根和重根）

## 协方差矩阵

假设存在一个m大小数据集，每个样本的特征维度为n。那么这个数据集可以表示为$X_{n*m}$, 其中每一列表示一个样本，每一行表示随机变量x的m个观察值。n行表示有n个随机变量。我们用
$K=(x_1, x_2, ...,x_n)$表示这个随机变量序列，则这个变量序列的协方差矩阵为：
$$C=(c_{ij})_{n*n} = \left[\begin{matrix}
    cov(x_1, x_1) & cov(x_1, x_2) & ... &cov(x_1, x_n)\\
    cov(x_2, x_1) & cov(x_2, x_2) & ... &cov(x_2, x_n)\\
    .& .& ...& .\\
    .& .& ...& .\\
    cov(x_n, x_1) & cov(x_n, x_2) & ... &cov(x_n, x_n)\\
\end{matrix}\right]$$
其中$cov(x_i,x_j)=\frac{1}{m-1}\sum_{k=1}^m(x_{ik}-\overline{x_{i}})(x_{jk}-\overline{x_j})$, 表示两个随机变量的协方差。
协方差矩阵C是一个对称矩阵，对角线实际为随机变量$x_i$的方差。当随机变量序列K中每个**随机变量均值为0**时，协方差矩阵C有一个极简表达。
$$C = \frac{1}{m-1}X*X^T$$

**协方差矩阵正对角线表示随件变量方差，其它位置表示两个随机变量的程度**, 大家记住这个特性，因为后面我们会用到。

# Principal Component Analysis(PCA)
PCA是这一种降维算法，即将原始数据集$X_{n*m}$(m表示样本数量，n表示一个样本的特征维度)，通过变换，得到$Y_{r*m}$, 其中r<n. 数学表示如下：
$$Y_{r*m} = P_{r*n}X_{n*m}$$
其中$P_{r*n}$为变化矩阵。

## 优化目标
任意$P_{r*n}$都会讲X的维度降为Y, 那么我们需要什么样的Y呢？
我们希望
1. Y在空间上尽可能的分散，
2. Y在的各个维度尽可能的不相关。
对于优化目标1，可以用方差来衡量；对于优化目标2，可以用协方差来衡量。preliminary中我们说过协方差矩阵恰好就包含了这两种度量。当**Y的各个特征均值为0**(X的各个特征均值为0时，PX的特征均值也为0)时，Y的协方差矩阵D为：
$$D_{r*r} = \frac{1}{m-1}YY^T$$
因为改变数据集的均值，并不会改变数据集的分布，且在数学上表示、推导简单。所以PCA降为的第一部就是将X的每一行进行零均值化，即减去这一行的均值。

**我们希望D的对角线元素尽可能的大（越分散），非对角线元素为0（不相关）。即找到P, 使得Y的协方差矩阵D对角化**。
那么P和D的关系如何？
$$
D_{r*r} = \frac{1}{m-1}YY^T\\
= \frac{1}{m-1}(PX)(PX)^T\\
= \frac{1}{m-1}(PXX^TP^T)\\
= P(\frac{1}{m-1}XX^T)P^T\\
= PCP^T
$$
其中$\frac{1}{m-1}XX^T$为原始数据集X的协方差矩阵。
现在的优化目标为找到P，使得$PCP^T$为一个对角矩阵。
对于对称矩阵$C_{n*n}$, 一定可以找到一组单位特征向量$e_{1}, e_{2}, ...,e_{n}$，对应矩阵$E=(e_{1}, e_{2}, ...,e_{n})$，满足
$$
E^TCE = \lambda = 
\left(\begin{matrix}
    \lambda_{1}\\
    &\lambda_{2}\\
    &&...\\
    &&&\lambda_{n}\\
\end{matrix}\right)
$$


# 参考文献

1. [特征值与特征向量](https://www2.edu-edu.com.cn/lesson_crs78/self/j_4184/soft/ch0501.html)
2. [图解机器学习 | 降维算法详解](https://www.showmeai.tech/article-detail/198)
3. [【机器学习】降维——PCA（非常详细）](https://zhuanlan.zhihu.com/p/77151308)
