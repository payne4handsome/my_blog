---
layout: post
title: "matplotlib画图"
date: 2021-12-19
categories: 
    - "python"
tags: [matplotlib]
author: "pan"
---

# matplotlib基本使用

+ 画板figure，画纸Sublpot画质，可多图绘画
+ 画纸上最上方是标题title，用来给图形起名字
+ 坐标轴Axis,横轴叫x坐标轴label，纵轴叫y坐标轴ylabel
+ 图例Legend 代表图形里的内容
+ 网格Grid，图形中的虚线，True显示网格
+ Markers：表示点的形状

## 常用图形

+ scatter：散点图
+ plot: 折线图
+ bar： 柱状图
+ heat：热力图
+ box：箱线图
+ hist：直方图
+ pie：饼图
+ area：面积图
  
## 基本步骤

![image.png](https://upload-images.jianshu.io/upload_images/8596800-cea42c8cc4f0f71c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```python
def base_plot():
    x = np.arange(-1000, 1000, 1)
    y = x * x
    # plt.plot(x, y, color='r', marker='o', linestyle='dashed')
    plt.plot(x, y, '.r', linestyle='dashed')  # 使用format格式，与上面含义相同
    plt.xlabel('X')
    plt.ylabel('y = x^2')
    plt.axis([-100, 100, 0, 100])  # 设置坐标轴范围
    plt.show()
```

## 多图绘制

使用Python创建多个`画板`和`画纸`来绘制多幅图，如果事先不声明画板画板，默认是创建一个画板一个画纸

+ 使用figure()方法创建画板1
+ 使用subplot()方法创建画纸，并选择当前画纸并绘图
+ 同样用subplot()方法选择画纸并绘图

```python
def base_subplot():
    fig = plt.figure(1)  # 创建画板
    ax1 = plt.subplot(2, 1, 1)  # 创建画纸（子图）， 三个数字，前两个表示创建1*1的画纸，第三个表示选择第一个画纸
    plt.plot([1, 2, 3])
    ax2 = plt.subplot(2, 1, 2)  # 选择第二个画纸
    plt.plot([4, 5, 6])
    plt.show()
```


# 参考文献
[matplot 官方文档](https://matplotlib.org/stable/plot_types/index.html)
[python如何使用Matplotlib画图](https://zhuanlan.zhihu.com/p/109245779)