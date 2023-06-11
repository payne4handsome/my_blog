---
layout: post
title: "YOLOv4学习摘要"
date: 2022-09-27
# menu: main
categories: [机器学习]
tags: [yolov4, yolo, '目标检测']
---

本文主要是学习yolov4论文的一些学习笔记，可能不会那么详细。所以在文章最后列出了一些参考文献，供大家参考
![image.png](/YOLOv4学习摘要/8596800-9faebc33a9e7e4ac.png)

**yolo系列发展历程**

| yolo系列       | 发表时间   |  作者  |   mAP  |  FPS|  mark|
| --------   | :-----:  | :----:  |:----:  |:----:  |:----:  |
| yolov1     | 2016|   Joseph Redmon     | 63.4<br>(VOC 2007+2012)     | 45 (448*448)    |      |
| yolov2     | 2016 |  Joseph Redmon     | 21.6<br>(coco test-dev2015)     | 59(480*480)     |      |
| yolov3     | 2018 |  Joseph Redmon     | 33<br>(coco)     | 20(608*608), 35(416*416)     |      |
| yolov4     | 2020 |   Alexey Bochkovskiy、Chien-Yao Wang     | 43.5<br>(coco)     | 65(608*608)     |     |
| yolov5     | 2020 |   Glenn Jocher （Ultralytics  CEO）     | 48.2 <br>(coco)     |    没有实测数据  | 没有论文，学术界不认可；该模型有不同大小的模型     |
| scaled-yolov4     | 2021 |   Chien-Yao Wang、 Alexey Bochkovskiy    | 47.8<br>(coco)     |    62（608*608）  | 该模型有不同大小的模型     |

**插曲，与中心思想无关**
我们看到yolov1到v4 是由不同作者完成的，因为yolo 原作者已经在2018年推出CV界了，在yolov3论文的最后作者是这么写的
![yolov3作者原话](/YOLOv4学习摘要/8596800-0f583a3c6c3ab0bd.png)
他发现他的研究成果被应用于战争，还有一些隐私的问题，所以他决定推出了......
yolo系统v1-v3由原作者Joseph Redmon完成，v4由AB(Alexey Bochkovskiy)完成，AB是参与前几个系列yolo代码的开发的，并且yolov4是得到原作者的认可的，在作者的原网页上有引用。

**代码实现**

| coda       | mark   | 
| --------   | :-----:  | 
| https://github.com/pjreddie/darknet.git     | darknet实现，现在由AB维护| 
| https://github.com/AlexeyAB/darknet.git     | 作者AB，是从上面的代码fork来的,目前领先上面的代码很多，所以大家可以参考这个 | 
|  https://github.com/WongKinYiu/PyTorch_YOLOv4.git     | 官方推荐pytorch实现，不过我复现的mAP只有38%，不能达到作者说的47%，原因不知道| 
| https://github.com/WongKinYiu/ScaledYOLOv4.git      |和PyTorch_YOLOv4实现有重叠部分，现在正在复现，看看能不能达到作者说的mAP | 
| https://github.com/ultralytics/yolov5.git     | 还有一个yolov3的版本，很多代码的实现都是参考这个代码实现的，虽然叫yolov5这个名字不厚道，但是代码实现的还是有很大贡献的| 
经验证，在coco 2017 val上第四份代码是可以达到47.31%的精度。300个epoch，Ti 2080 上耗时11天左右
![image.png](/YOLOv4学习摘要/8596800-8defacad36cfe6cd.png)


## 正文部分
从yolov3开始，我才觉得它可以和其他模型一较高低。yolov1和yolov2虽然快、简洁，但是mAP比其他两阶段的模型差很多，所以在实际项目中我肯定不会选择v1和v2。但是yolov3的性能已经和其他模型差不多了，但是具有一个明显的优势**快**。与Faster R-CNN系列虽然差一点点，但是可以接受。
![image.png](/YOLOv4学习摘要/8596800-e052e136ac586000.png)
### yolov3 简述
yolov3的创新点有两个
+ 选用darknet53作为backbone, darknet53在分类问题上表现与resnet差不到，但是参数量较小，所以速度更快
+ 添加FPN（特征金子塔）结构，高分辨率特征图预测小一点的框（anchor），低分率特征图预测大一点的框

关于darknet53和yolov3的网络结构如下：
![darknet53网络结构](/YOLOv4学习摘要/8596800-f243f81d9645bd87.png)


![yolov3网络结构](/YOLOv4学习摘要/8596800-b73dfe21e89cd6f8.png)

### yolov4 详解
> yolov4的论文值得所有做目标检测的同学一读。yolov4其实并没有什么创新点，它完全是把其他论文的创新点拿过来做了一个排列组合。选择能提高性能的那些然后组合起来。

yolov4罗列了目前目标检测常见的tricks，分为两个大类Bag of freebies 和Bag of specials。
+ Bag of freebies: 仅通过改变训练方法或者技巧，但是不增加推断耗时叫做Bag of freebies（BoF）
**一般的BoF指数据增强（常见的一些变换、CutOut、MixUp、CutMix）、数据不平衡（权重、focal loss）、smooth label、IOU(GOU、DIOU、CIOU)等**
+ Bag of specials: 增加少量的推断时间，但是显著的提高目标检测的准确率叫做Bag of specials（BoS）
**包含SPP、ASPP(不降低分辨率但增大了感受野)、RFB、注意力模块（SE、SAM）、特征融合（FPN、ASFF、BIFPN、ASFF）、激活函数（RELU、Swish、hard-Swish、Mish）、NMS(soft-NMS, DIoU-NMS)**
完整的清单如下（这些方法是已有的）：
![image.png](/YOLOv4学习摘要/8596800-8e3580892aad6f41.png)
稍微有一点创新或者说作者改进的：
![image.png](/YOLOv4学习摘要/8596800-fc6d53c2850a0f0f.png)
+ Mosaic： 将4张图片合成一张，相当于增加了batch size
+ SAT: 不太了解，摘抄自【参考文献2】
自对抗训练也是一种新的数据增强方法，可以一定程度上抵抗对抗攻击。其包括两个阶段，每个阶段进行一次前向传播和一次反向传播。
    1. 第一阶段，CNN通过反向传播改变图片信息，而不是改变网络权值。通过这种方式，CNN可以进行对抗性攻击，改变原始图像，造成图像上没有目标的假象。
    2. 第二阶段，对修改后的图像进行正常的目标检测。
+ 修改的SAM: channel级别的权重改为像素级别的权重。**但是代码中根本没有使用该模块**
![modified SAM](/YOLOv4学习摘要/8596800-a4979d2f184a8c41.png)
+ 修改的PAN：加改成concat操作
![modified PAN](/YOLOv4学习摘要/8596800-4a3de57ee8c8eef7.png)
+ CmBN：通过过去4个step的BN信息来更新现在的BN信息，具体的不太了解，代码中也没有使用这个东西，所以先跳过
![CmBN](/YOLOv4学习摘要/8596800-2631e2dc2aeffcf1.png)
## 实验

实验部分是yolov4论文的精华，也是所有做学术研究的同学需要学习的地方。yolov4的实验做的可以说是非常详尽。仅通过叠加前人的tricks就能把mAP提升10个点（yolov3:mAP=33%, yolov4:mAP=42.4%）。
### BoF 实验
我们来看一下用BoF tricks所做的对比实验。
![BoF 消融实验](/YOLOv4学习摘要/8596800-faaf5917672da93a.png)
从实验结果看，Mosaic、CIoU是明显可以提高mAP一到两个点的。没有加任何tricks模型（CSPResNeXt50-PANet-SPP, 512*512）是比yolov3高5个点，我觉得主要的贡献应该是CSP结构。通过tricks的叠加又提高了5个点。所以导致yolov4的性能巨大提升，其实从现在开源的代码来看，作者复现的mAP是高于42.4%，大概在48%左右。Scaled-yolov4通过增大输入尺寸、优化模型结构，甚至可以把mAP提到到55.5%（输入尺寸1536，肯定也增加了耗时）。

实验中这个`Eliminate grid sensitivity`可能一开始不知道什么意思，所以单独说一下。意思就是说如果目标靠近feature map的grid的边界，既坐标靠近$c_x$或者$c_x+1$,那么就需要预测的$t_x$的值比较大。如果你使用sigmod激活函数，那输入值的很大，才能使输出靠近1。所以作者认为这个需要优化下，既在激活函数前面乘以一个大于1的系数。但是**实验证明该trick并没有提高性能**
像smooth label 等在其他任务中一般是可以提高性能的，但是在检测任务中还是性能下降了，所以**实验才是最好的证明**
### BoS 实验
使用SAM注意力机制确实使得性能提升一点点，但是不明显。所以就用CSP+PAN+SPP就够了，在开源代码实现中也并没有使用SAM（因为只提高了0.3个点）。
![image.png](/YOLOv4学习摘要/8596800-81af22497102c636.png)

### backbone对性能的影响
这个部门作者实验了证明了在分类任务中表现更好的模型CSPResNeXt50在检测任务中并没有CSPDarknet53好(相差也不多)。

![image.png](/YOLOv4学习摘要/8596800-0815c6dbe625929f.png)

### 网络结构
### CSP网络结构
在CSP论文中作者实验了如下图中b、c、d三种网络结构。证明b是性能最好的。三个结构中Part 1 , Part 2， Transition（Conv或者Pool）是自已添加的，剩下是原始网络的，比如你可以使用Dense Net，Restnet Net，Darknet。所以CSP结构是一种**通用的结构**，你可以使用在任何其他网络中。**注意，在下面图中concat操作没有画出来，两个箭头相交的地方就是concat操作。所以b示意图中CSP添加的操作的顺序是（1）Part 1, Part 2 ;(2) Transition (3)Concat (4) Transition**
![CSP网络结构](/YOLOv4学习摘要/8596800-ba8e69bbb87d6145.png)
以darknet第二个block为例，它的CSP结构示意图如下：
![csp结构应用于darknet53](/YOLOv4学习摘要/8596800-f109304a88f0bf33.png)
### yolov4结果图
一下两幅图摘录自知乎，参考文章已在参考文献中列出
![yolov4简化图1](/YOLOv4学习摘要/8596800-a779296f54d2b87b.png)

![yolov4简化图2](/YOLOv4学习摘要/8596800-ebccf9d40a869bd6.png)

### 损失函数
![image.png](/YOLOv4学习摘要/8596800-1330dd827527c924.png)
其中$t_x, t_y, t_w, t_h$是网络预测输出，$p_w, p_h$是通过聚类出来的anchor宽和高，$c_x, c_y$是特征图中的坐标点（整数）

先看一下yolov3中损失函数
![image.png](/YOLOv4学习摘要/8596800-00d23f9f669c9c11.png)

+ 边框回归损失中为什么要乘以$2-w_i*h_i$?
同样的标准MSE损失，对不同的框感知是不一样的。小的框的权重应该大一点
+ 我看到的代码实现中置信度loss中的$\lambda_noobj$=1，也是有无目标的权重是一样的
+ 分类是用的二进制交叉熵，不是一般的交叉熵

**yolov4的损失函数把上面的边框回归损失从MSE变成CIOU损失**，关于CIOU见参考文献5

### IOU
在yolov4中使用CIOU损失来作为边框回归损失的。关于IOU在参考文献5中已经说明的很详细了，这里不再赘述。下面这个给出公式。
+ 一般的IOU
![image.png](/YOLOv4学习摘要/8596800-38962647f77c801d.png)
+ GIoU(Generalized Intersection over Union)
![image.png](/YOLOv4学习摘要/8596800-6c599560b8a9d93c.png)
+ DIOU(Distance-IoU)
![image.png](/YOLOv4学习摘要/8596800-5aed11067b171c7a.png)
+ CIOU（Complete-IoU）
![image.png](/YOLOv4学习摘要/8596800-f5bbb8e10ec84986.png)


### 训练技巧
#### warmup
+ 什么是warmup
warmup是训练时候一开始以一个特别小的学习学习，然后慢慢的逼近初始化学习率
```
if warmup:
    warmup_steps = int(batches_per_epoch * 3)
    warmup_lr = (initial_learning_rate * tf.cast(current_step, tf.float32) / tf.cast(warmup_steps, tf.float32))
    return tf.cond(global_step < warmup_steps, lambda: warmup_lr, lambda: lr)
```
实现如上代码所示，比如说我们warmup设置为3 epoch，那么前3 ecoch的学习率=初始学习率*（当前step/warmup总的step）。也就是在前3个epoch，学习是慢慢变大的
+ 为什么需要warmup
一般的我们的学习率通常设置为0.01或者0.001。那么到底是设置0.01合适还是设置0.001合适呢？通常来讲模型一开始在没有见过所有的数据，既还没有训练到1个epoch时候，如果训练率比较大，那么造成一开始就过拟合了，后面可能需要很多个epoch才能拉回来。**从loss曲线上表现为一开始训练loss下降，后来上升**。所以一开始通常我们需要用较小的学习率让模型去适应数据，当训练几个epoch后，模型已经见过所有的数据了，这个时候学习率大一点，也不会造成模型的剧烈波动。**还有一个好处，通过warmup观察loss曲线，我们可以观察到那个学习率是合适的**
### 前向传播多次，反向传播一次
在YOLOv4的实现中，有一个名义batch size（64），还有一个subdivisions配置。表示的意思是前向传播k=（batch size / subdivisions）次后，再执行一次反向传播。相当于变相的扩大了batch size。不知道这个对实际的性能可以提示多大，因为batch normal并没有对应的扩大k倍的。在多GPU环境可以用 sync batch normal 来解决这个问题。单GPU下可以用CBN来解决当batch size 太小，BN性能下降的问题

# 代码
**补充于2021年7月5号**
在计算损失的方法中有一个build_targets方法，该方法的主要作用是建立预测的框和真实的框的对应关系。就是用那个特征图上的点去预测真实的框。在阅读代码的时候发现如下奇怪的部分，一开始没有明白。
```
            gxy = t[:, 2:4]  # grid xy
            z = torch.zeros_like(gxy)
            j, k = ((gxy % 1. < g) & (gxy > 1.)).T
            l, m = ((gxy % 1. > (1 - g)) & (gxy < (gain[[2, 3]] - 1.))).T
            a, t = torch.cat((a, a[j], a[k], a[l], a[m]), 0), torch.cat((t, t[j], t[k], t[l], t[m]), 0)
            offsets = torch.cat((z, z[j] + off[0], z[k] + off[1], z[l] + off[2], z[m] + off[3]), 0) * g
```
直到找到了下面这张图，[来自这里](https://zhuanlan.zhihu.com/p/159371985)
![image.png](/YOLOv4学习摘要/8596800-bb3c9a9110ca5a6b.png)

我又在上面做了一点标记，方便大家理解。
![image.png](/YOLOv4学习摘要/8596800-202f61cf38adc35e.png)

+ 红色三角形标记的框：表示目标落在特征图上的位置，分别可能落在一个grid cell的左上、左下、右上、右下4个位置。
+ 黄色矩形标记的框：yolo系列中是利用目前落在特征图网格的左上角来预测真实位置的，就是4个黄色标记框的位置。
+ 除了上面黄色的框，我们发现还多了两个框，这两个框是yolov4的代码中才加入的。**相当于给每个目标增加了两个正样本**。实现就是上面的代码。
上面中存在如下代码：
```
offsets = torch.cat((z, z[j] + off[0], z[k] + off[1], z[l] + off[2], z[m] + off[3]), 0) * g
```
本来grid cell的大小是1，既网络预测的边框的中心点的大小在[0, 1]之间。off的大小在[-1,1] , g = 0.5. 那么上面的offsets（偏移）大小在[-0.5, 0.5]。 那么预测的边框的中心点坐标范围就变成了[-0,5, 1.5]之间。
现在再来看一下计算边框回归损失时候的代码。
```
pxy = ps[:, :2].sigmoid() * 2. - 0.5
```
为什么激活后要乘以2，再减去0.5？ 就是上面的原因




# 参考文献

1. [YOLO v3网络结构分析](https://blog.csdn.net/qq_37541097/article/details/81214953)
2. [一张图梳理YOLOv4论文](https://zhuanlan.zhihu.com/p/136115652)
3. [YOLO V4 — 网络结构和损失函数解析（超级详细！)](https://zhuanlan.zhihu.com/p/150127712)
4. [YOLOv4 介绍及其模型优化方法](https://zhuanlan.zhihu.com/p/342570549)
5. [IoU、GIoU、DIoU、CIoU损失函数的那点事儿](https://zhuanlan.zhihu.com/p/94799295)
6. [CVPR2019: 使用GIoU作为检测任务的Loss](https://zhuanlan.zhihu.com/p/57992040)

# 附录

![yolov4-csp.png](/YOLOv4学习摘要/8596800-f9ec26b79cec5de8.png)