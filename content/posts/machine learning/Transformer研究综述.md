---
layout: post
title: "Transformer研究综述"
date: 2022-4-27
# menu: main
categories: [机器学习]
tags: [Transformer]
---

# 一、基础部分

&emsp;&emsp;2017年google发表了一篇All Attention Is All You Need论文, 在机器翻译任务中取得了SOTA
的成绩。论文中提出的Transformer结构取消了传统的Seg2Seg模型中RNN和CNN传统神经网络单元，取而代之代之的Self-Attention（自注意力机制）的计算单元。该计算单元并行化程度高，训练时间短。所以Transformer引起了学术界和工业界的广泛注意。目前已经是NLP领域的标配。随后在2019年google提出基本Transformer的**bert**模型, 该模型在大量的数据集中通过自监督的训练，然后在特定任务上只需要做少量改动和训练就可以得到非常好的效果，开源的bert模型在11个NLP任务取得SOTA结果。

&emsp;&emsp;Transformer除了在NLP任务上表现优异，在CV领域也取得了很多突破。2020年google又发表了一篇
VIT Transformer的论文，实验证明Transformer在imagenet分类任务上取得了SOTA结果。后来CV领域中的各种基本问题，比如目标检测、语义分割、物体追踪、视频等各种任务都用Transformer方法又搞了一遍，基本上也取得了一些不错的结果。

&emsp;&emsp;鉴于Transformer在NLP和CV上的巨大成功，本文竟可能详细描述Transformer的基本原理；特定的一些应用，主要是一些经典论文的方法；以及目前Transformer在效率问题上的一些改进的方案。

## 1.1 Attention
&emsp;在学习Transformer之前，了解一些基础问题是很有必要。毕竟在没有Transformer之前，学术上在NLP领域也做了大量的研究和成果。我们先从Encoder Decoder和Seq2Seq开始说起。我想大家肯定都听过这两个名称，简单来说就是如下图。
![image.png](/Transformer研究综述/8596800-617fce8b48bfae84.png)
Encoder输入（可以是文字也可以是图像）编程成一个固定长度的向量（content）,那么这个content向量肯定是包含输入的信息的（至于包含多少那就看这个编码了），Decoder根据content解码出我们需要的结果。Encoder Decoder可以是机器翻译问题、语义分割问题等等。那么Seq2Seq（Sequence-to-sequence ）是什么？输入一个序列，输出另一个序列。这种结构最重要的地方在于输入序列和输出序列的长度是可变的。如下图所示。

![v2-db740610d0fd35452eeadde26b889172_b.gif](/Transformer研究综述/8596800-ce9b2e0fb5e318df.gif)
从本质上看，Encoder-Decoder和seq2seq好像差不多，但是又有一点区别。

+ Seq2Seq 属于 Encoder-Decoder 的大范畴
+ Seq2Seq 更强调目的，Encoder-Decoder 更强调方法

**那么这个Encoder-Decoder有什么缺陷呢？**

&emsp;&emsp;从上面的示意图我们看到，无论输入的信息又多少，Encoder后就剩下一个content向量了，那么这里面有一个缺陷就是这个content向量会丢掉一些信息，特别是输入很大（文本很长图像分辨率很高）的情况下。尽管后面出现的LSTM、GRU等通过**门**设计的循环神经网络单元，可以一定程度上缓解**长距离**问题，但是效果有限。

&emsp;&emsp;从这里开始，我们要进入文章的正题了，Transformer的核心是Self-Attention，那么在这之前，我们最起码要了解什么是Attention，然后再看是这么在  `Attention`的基础上加上`self`的。

### 1.1.1 NLP中的Attention

&emsp;&emsp; 由于传统的Encoder-Decoder模型将所有的输入信息编码成一个固定长度的content向量存在长距离问题。那么随之而然的一个做法就是我们在decoder阶段解码$h_t$不仅依赖前一个节点的隐藏状态$h_{t-1}$, 同时依赖Encoder阶段所有的状态，就和我们自已翻译的时候一样。这里有两个经典注意力机制，Bahdanau Attention （2014年提出）和 Luong Attention（2015年）。

![2019-10-28-nmt-model-fast.gif](/Transformer研究综述/8596800-6d041f7ac22983ee.gif)

#### 1.1.1.1 **Bahdanau Attention 注意力机制**

&emsp;&emsp; 示意图如下：
![image.png](/Transformer研究综述/8596800-47bc406c6b448748.png)
假设现在我们Decoder t时刻。 那么$h_t$隐状态计算过程如下：

1. 计算对齐向量$a_t$  
   $a_t$的长度与Encoder输出向量的个数相同。$a_t(s)$表示Decoder阶段的转态$h_{t-1}$与Encoder阶段第s个隐状态，通过align对齐函数计算出的一个权重。$a_t$就是$h_{t-1}与每一个Encoder隐状态计算权重后组成的一个向量。

2. 计算$c_t$即content vector  
   将上一步计算出的$a_t$向量乘以Encoder所有的隐向量。即Encoder所有的隐向量的加权和。

3. 计算Decoder阶段t时刻的输出，$h_t$  
   将$h^{l-1}_{t-1}$与concat（$c_t$， $h_{t-1}$）送入多层RNN（最后一层）。其中$h^{l-1}_{t-1}$为上一阶段的预测输出。concat（$c_t$， $h_{t-1}$）相当于RNN的隐状态。最终将$h_t$过一个softmax就可以预测最终的输出（$y^t$）了。

#### 1.1.1.2 **Luong Attention 注意力机制**  
![image.png](/Transformer研究综述/8596800-80e0ae3a4dce29ef.png)
&emsp;&emsp;Luong Attention是在Bahdanau Attention之后提出的。结构更加简洁，效果也更加好一点。
假设现在我们Decoder t时刻。 那么$h_t$隐状态计算过程如下：

1. 计算对齐向量$a_t$  
   $a_t$的长度与Encoder输出向量的个数相同。$a_t(s)$表示Decoder阶段的状态$h_t$与Encoder阶段第s个隐状态，通过align对齐函数计算出的一个权重。$a_t$就是$h_t$与每一个Encoder隐状态计算权重后组成的一个向量。

2. 计算$c_t$即content vector  
   将上一步计算出的$a_t$向量乘以Encoder所有的隐向量。即Encoder所有的隐向量的加权和。

3. 计算Decoder阶段t时刻的输出，$\hat{h_t}$  
   将concat（$c_t$， $h_t$）送入一个前向更加网络（一个全连接）。最终将$\hat{h_t}$ 过一个softmax就可以预测最终的输出（$y^t$）了。

**注意，上面两种注意力机制的对齐函数align公式还没有给出，如下**

![image.png](/Transformer研究综述/8596800-2dbba89a384ce30a.png)
其中score计算方式有如下三种
![image.png](/Transformer研究综述/8596800-0ec699032599a691.png)

#### 1.1.1.3 Bahdanau Attention与Luong Attention 区别

1. 计算$a_t$时，Bahdanau Attention是拿$h_{t-1}$与Encoder所有隐向量计算；Luong Attention是拿$h_t$与Encoder所有隐向量计算

2. 对于content向量（$c_t$），Bahdanau Attention是将$c_t$当成rnn的输入了；Luong Attention将$c_t$用在最后预测$\hat{y_t}$的时候。

对于score函数，Bahdanau Attention与Luong Attention使用的公式不同
![image.png](/Transformer研究综述/8596800-c0a9956542de813f.png)

### 1.1.2 CV中的Attention

&emsp;&emsp;视觉中的Attention比较简单。一言以蔽之就是生成一个mask作用于特征图上。作用域的不同又分为三个类别。

+ 通道注意力机制， 对通道生成掩码mask, Channel Attention Module
+ 空间注意力机制, 对空间进行掩码的生成, Spatial Attention Module
+ 混合域注意力机制, 时对通道注意力和空间注意力进行评价打分, Convolutional Block Attention Module

#### 1.1.2.1 通道注意力机制(CAM)

示意图如下：
![image.png](/Transformer研究综述/8596800-ee77da51c297ed43.png)
代码如下（pytorch）

```python
class ChannelAttention(nn.Module):
    def __init__(self, in_planes, rotio=16):
        super(ChannelAttention, self).__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.max_pool = nn.AdaptiveMaxPool2d(1)

        self.sharedMLP = nn.Sequential(
            nn.Conv2d(in_planes, in_planes // ratio, 1, bias=False), nn.ReLU(),
            nn.Conv2d(in_planes // rotio, in_planes, 1, bias=False))
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        avgout = self.sharedMLP(self.avg_pool(x))
        maxout = self.sharedMLP(self.max_pool(x))
        return self.sigmoid(avgout + maxout)
```

#### 1.1.2.2 空间注意力机制(SAM)

示意图如下：
![image.png](/Transformer研究综述/8596800-be3e420d24d30b3c.png)

代码如下（pytorch）

```python
class SpatialAttention(nn.Module):
    def __init__(self, kernel_size=7):
        super(SpatialAttention, self).__init__()
        assert kernel_size in (3,7), "kernel size must be 3 or 7"
        padding = 3 if kernel_size == 7 else 1

        self.conv = nn.Conv2d(2,1,kernel_size, padding=padding, bias=False)
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        avgout = torch.mean(x, dim=1, keepdim=True)
        maxout, _ = torch.max(x, dim=1, keepdim=True)
        x = torch.cat([avgout, maxout], dim=1)
        x = self.conv(x)
        return self.sigmoid(x)
```

##### 1.1.2.3 混合域注意力机制(CBAM)

示意图如下：
![image.png](/Transformer研究综述/8596800-51496a3a2a1c31b4.png)

代码如下（pytorch）

```python
class BasicBlock(nn.Module):
    expansion = 1
    def __init__(self, inplanes, planes, stride=1, downsample=None):
        super(BasicBlock, self).__init__()
        self.conv1 = conv3x3(inplanes, planes, stride)
        self.bn1 = nn.BatchNorm2d(planes)
        self.relu = nn.ReLU(inplace=True)
        self.conv2 = conv3x3(planes, planes)
        self.bn2 = nn.BatchNorm2d(planes)
        self.ca = ChannelAttention(planes)
        self.sa = SpatialAttention()
        self.downsample = downsample
        self.stride = stride
    def forward(self, x):
        residual = x
        out = self.conv1(x)
        out = self.bn1(out)
        out = self.relu(out)
        out = self.conv2(out)
        out = self.bn2(out)
        out = self.ca(out) * out  # 广播机制
        out = self.sa(out) * out  # 广播机制
        if self.downsample is not None:
            residual = self.downsample(x)
        out += residual
        out = self.relu(out)
        return out
```

## 1.2 Self-Attention

Self-Attention是Transformer中核心模板，搞清楚Self-Attention后，Transformer就容易搞清楚了。首先任何新的模型或者方案的提出肯定是因为之前的模型有所缺点。那我们想一下在NLP中的Attention机制中有什么缺点呢？

+ 在传统的Attention中，已经考虑到了Decoder阶段的词（token）和Encoder阶段的每一个词（token）之前的关联了（通过对每一个Encoder阶段的隐向量加权求和）。但是Encoder中的每一个词之前有没有关联呢？如果有关联，我们把这中关联考虑到我们的编码中是不是效果要好一点？我觉得Self-Attention中的`self`就是指的是Encoder阶段所有的词（token）之前自已的关联（当然Decoder阶段也可以）
+ 传统的Attention是利用的RNN网络来实现的，每一个输入都依赖于上一个阶段的输出，训练的时候模型并行程度不高，也就意味着效率低下。

基于这样的考虑, google在2017年提出了一个完全全新的注意力机制`Self-Attention`。

我们看一下Self-Attention的结构。
Self-Attention的核心结果是Multi-Head Attention。 Multi-Head Attention的核心结构是Scaled Dot-Product Attention。
![image.png](/Transformer研究综述/8596800-111433262caab26a.png)
Scaled Dot-Product Attention的数学表达如下：
![image.png](/Transformer研究综述/8596800-59de1cb6ec3608ef.png)

![image.png](/Transformer研究综述/8596800-1661f5583f14b7af.png)

举一个更详细的例子如下，假设输入是Thinking Machines
![image.png](/Transformer研究综述/8596800-f9eb616a9277a8dd.png)

1. 将输入Thinking Machines进行编码（word embeding），embeding结果为X（假设每个单词编码为一个512的向量，那么X维度2*512）
2. 将X垂直划分为多头，Transformer为8头，那么X的维度变为（2\*8\*64转置为8\*2\*64）。X变成8个矩阵$X_0, X_1, ...,X_7$(每一个维度为2\*64),将每个头投影到其他空间（以$X_0$为例，乘以权重矩阵$W_0^Q, W_0^K, W_0^V$）,得到$Q_0, K_0, V_0$，$X_1, ...,X_7$同理
3. **计算Attention，计算的公式就是上面的Scaled Dot-Product Attention**；8个头的结果分别是$Z_0, Z_1, ...,Z_7$
4. 将$Z_0, Z_1, ...,Z_7$ 拼接， 然后乘以$W_0$, 得到最终的结果Z(Z的维度和X的维度是一样的)

## 1.3 Transformer
Transformer结构就是Multi-Head Attention结构的嵌套。整体结构如下。
![Transformer结构](/Transformer研究综述/8596800-19e183643f3f8a85.png)
论文Encoder、Decoder都是6个固定结构的嵌套。每个Encoder包含两个部分，（1）Multi-Head Attention加残差链接；（2）前向网络加残差链接。

### 1.3.1 Transformer常见问题的思考

+ 为什么一定要位置编码？输入矩阵是将单词按顺序排列的，这个难道不是包含了顺序信息

在翻译阶段，Q来自Decoder，K，V来之Encoder。Attention的计算公式为$softmax(\frac{Q.K^T}{\sqrt{d_k}})V$。交换K,V中任意两行的顺序，这个计算的结果是不变的，也就是说尽管输入矩阵K，V本身是包含顺序的，但是经过计算这个顺序丢失了。所以需要额外的加入位置编码

+ Self-Attention为什么dot-product attention采用而不用additive attention  

additive attention与dot-product attention的区别主要是Encoder的隐向量和Decoder的隐向量是如何交互的，如果是Bahdanau Attention的计算方式（一个全连接网络）那就是additive attention；如果通过Self-Attention计算公式那就是dot-product attention
![image.png](/Transformer研究综述/8596800-8ee8999a6f69d397.png)
至于是用additive attention还是dot-product attention其实都是可以。按照论文中的表述主要是因为dot-product attention可以更快

![image.png](/Transformer研究综述/8596800-21ad42bfa0c85f0d.png)

+ 为什么需要Q、K、V三个矩阵？只要Q、V行不行？

对于这个问题我一开始想的很久，也查看了很多资料，但是都没有令人满意的回答。很多人说增加的模型特征表达能力，这不是废话吗？增加了参数肯定增加模型的特征表达能力，那这里可以100个矩阵啊。我们举一个例子。
比如翻译句子`She crept upstairs, quiet as a mouse`。这里的单词`mouse`其实和单词`she`的关联更大一点的。如果只用两个矩阵Q、V，即K=V, 那么理论上`mouse`和自身的关联是比较大的，因为向量夹角为0嘛。

+ Scaled Dot-Product Attention计算公式中为什么要除以$\sqrt{d_k}$

因为Q、K、V初始化的时候一般都是服从0,1正太分布的，那么Q.K^T后就服从0，$\sqrt{d_k}$的正太分布了。那么经过softmax后，会造成梯度的过多或者过小。不利于训练

+ Transformer是如何解决长距离依赖问题的？

经过公式$softmax(\frac{Q.K^T}{\sqrt{d_k}})V$后，每个token都是包含了其他token的信息。所以无论句子多长，都没有问题

#  二、Transformer 应用

本章节主要记录一些使用Transformer解决具体问题的论文。这里给出我看过的几篇相关的综述供大家参考（论文地址见参考文献17、18、19）。
1. A Survey on Vision Transformer。 来自华为诺亚方舟实验室，是一篇关于Transformer在视觉应用上的综述
2. Transformers Meet Visual Learning Understanding: A Comprehensive Review。 来自西安电子科技大学，关于Transformer在图像和视频上的综述
3. Efficient Transformers: A Survey。 来自google。Transformer的self-attention的时间复杂度、空间复杂度都是O(n^2), 对于长文本，高分辨率图像是不友好的。这篇综述把相关解决这个问题的论文基本囊括了

![Transformer在CV中的应用的各种方法](/Transformer研究综述/8596800-f7b129a9450cfd05.png)

## 2.1 VIT

VIT是google首次发表在2020的一篇论文，该篇论文是首次将Transformer应用于CV领域。并且在ImageNet图像分类任务上取得的SOTA的结果。
VIT应用示意图。
![image.png](/Transformer研究综述/8596800-3bf89b3008c21462.png)
该结构只用到了Transformer Encoder，没有用TRansformer Decoder。唯一的不同在所有的输入前面加入一个`classification token`的向量最终用于分类。**这种处理方式在后续的很多地方都有类似的应用，比如Bert， Big Bird**

VIT这篇论文比较重要的作者做的一些实验的结果。

**VIT在大数据集上的表现**
![image.png](/Transformer研究综述/8596800-6971e11939478896.png)
VIT在JFT（大约3亿张图片）数据集上训练后，表现是比基于RetNet的BiT-L模型要好的。这个证明了**在大量数据集下TRansformer结构是可以取得比传统基于CNN结构的模型更好的结果**

**VIT在不同数据集上的表现**
![image.png](/Transformer研究综述/8596800-fa86092dc55c1afb.png)
上图表明了ViT在不同数量级上的表现，ImageNet（百万级），ImageNet-21k(千万级)，JFT-300M(亿级)
结论如下：

1. 在百万级数据下，BiT取得的结果最好，说明在小量数据集下，Transformer不适用；在千万数据集下，ViT可以取得和BiT差不多的效果；在亿级别数据下，ViT才表现比BiT效果好。
2. ViT的性能随着数据量的增加，模型的大小的增加可以取得更好的结果。而且论文表明JFT数据集并没有达到模型是上限

**VIT的混合模型效果**
![image.png](/Transformer研究综述/8596800-93be9dd96a73c42c.png)
很多时候我们是没有那么多数据集的，比如在检测、分割等任务上，别说亿级别数据了，百万级数据集的获取都比较困难。那这个时候可以使用VIT的混合结构（Transformer加CNN混在一起使用）。如上图所示也是可以获得比较不错的结果的。

## 2.2 DPT

Vision Transformers for Dense Prediction
![image.png](/Transformer研究综述/8596800-eaba74096b817e73.png)

## 2.3 T2T-ViT

Tokens-to-Token ViT: Training Vision Transformers from Scratch on ImageNet
该篇论文主要解决两个问题

1. ViT要想取得好的效果是需要在大量数据集上训练的。之前基于CNN结构的模型都是直接在ImageNet上训练。那么基于ViT结构能不能也直接在ImageNet训练然后取得比RestNet更好的结果呢？答案就是利用这篇论文提出的T2T-ViT(Note: 该方法是没有CNN的)
2. ViT的模型是比较大的，而且时间复杂度和空间复杂度都比较高（$O(n^2)$）,输入越长，那么耗时越久。T2T-ViT的结构是可以大幅减少这个复杂度的

![T2T-ViT整体结构](/Transformer研究综述/8596800-c802041bdf2dc0c1.png)

![T2T-Process](/Transformer研究综述/8596800-385e821d929dc223.png)

![从头训练，T2T-ViT的性能对比](/Transformer研究综述/8596800-4053967e2aa62d78.png)

**上图中ViT是直接在ImageNet上训练的结果**

1. train from scratch on ImageNet, T2T-ViT的效果是好于ResNet和ViT（MACs指标表示计算量）
2. T2T-ViT的模型大小是和MobileNet一个量级的，但是效果是好于MobileNet

# 三、高效率Transformer——————X-transformer

在前面的讲解中，提到Transformer是有一个问题的，Transformer需要的资源是正比于$N^2$的。当输入是一个比较长的序列时候，是没有办法满足需要的，GPU的显存很有可能就不允许。于是，最近出现了各种着力于解决这个问题的论文出现了，可以统称为（X-former）
下图是按照模型的特点做的一个分类（详细见参考文献19）

**图中的有些方法是降低内存的使用的，有些是降低计算复杂度的**

![image.png](/Transformer研究综述/8596800-f9f743446f1514a4.png)

![image.png](/Transformer研究综述/8596800-5307968d9940424b.png)

**X-transformer的本质就是在注意力矩阵上做文章**

分类思想简述：

1. Fixed patterns(FP)
   Transformer是具有全局注意力机制的，那就意味这个每个词之前都要计算注意力。但是这种方式有些浪费。那就把**全局变成局部**（好像要走老路了）
   局部注意力分为三种blockwise pattern，strided pattern，compressed pattern。
   blockwise pattern： 将输入序列切成多个block，在block中计算注意力机制
   Strided pattern： 采用滑动窗口的形式，每个token与周围相邻的几个token作attention，相邻的token范围就是window size
   Compressed pattern：通过卷积池化对序列进行降采样

2. Learnable patterns(LP)
   learnable patterns是对上文提到的fixed patterns的扩展，简单来说fixed pattern是认为规定好一些区域，让该区域的token进行注意力计算，而learnable patterns则是通过引入可学习参数，让模型自己找到划分区域。比如reformer

3. Memory
   这个不知道是啥，个人感觉类似ViT和Bert中cls token，即引入一些全局的记忆，但是其他输入之前用局部注意力，比如set transformer

4. Low rank methods
   将注意力矩阵降维，self-attention的计算公司中Q,K,V， 其中K，V的维度是要一样的，Q的维度可以和K，V不一样。假设之前的Q,K,V 维度都是n\*d, 那么现在将K，V的维度降为k\*d。比如linformer
   ![image.png](/Transformer研究综述/8596800-024a0cb00314ce91.png)

5. Kernels
   以核函数变换的新形式取代原有的softmax注意力矩阵计算，将计算复杂度降至 [公式] 范围内，比较代表的有Linear Transformers

6. Recurrence
   recurrence实际上也是上文提到的fixed patterns中blockwise的一种延伸。本质上仍是对输入序列进行区域划分，不过它进一步的对划分后的block做了一层循环连接，通过这样的层级关系就可以把一个长序列的输入更好的表征。以recurrence为代表的就是Transformer-XL。

这么多变种Transformer中，我觉得基于稀疏注意力（局部attention+全局attention）是比较好的方案，主要是容易想到。
代表的模型有Sparse Transformer，Longformer，Big Bird等

+ Sparse Transformer

![image.png](/Transformer研究综述/8596800-fc7a0e8a2f37cdbc.png)

**缺点: 由于模型需要一个块稀疏变量，因此这个方法需要自定义GPU内核，所以不能很容易地用于诸如TPU等其他硬件中**

+ Transformer，Longformer

![image.png](/Transformer研究综述/8596800-fbf9b4fb1928643a.png)

+ Big Bird

![image.png](/Transformer研究综述/8596800-291bf2426ab33695.png)

最后我们看下google团队这么多变种的评价。

![image.png](/Transformer研究综述/8596800-dd3ad154522eaf17.png)

总结下来就是

1. 需要很好的解决quadratic memory problem， 同时适用范围广，不仅仅是应用在长范围的任务中
2. 竟可能的兼容不同硬件，比如TPU
3. 思想要简单，容易实现（个人特别不喜欢那些把网络结构搞的很复杂，花式变体）
4. 简单来说就是要**优雅**

# 参考文献
1. [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)
2. [The Illustrated Transformer 中文版](https://blog.csdn.net/yujianmin1990/article/details/85221271)
3. [The Annotated Transformer](https://nlp.seas.harvard.edu/2018/04/03/attention.html)
4.  [Visualizing A Neural Machine Translation Model](https://jalammar.github.io/visualizing-neural-machine-translation-mechanics-of-seq2seq-models-with-attention/)
5. [The Positional Encoding](https://kazemnejad.com/blog/transformer_architecture_positional_encoding/)
6. [Mechanics of Seq2seq Models With Attention](https://jalammar.github.io/visualizing-neural-machine-translation-mechanics-of-seq2seq-models-with-attention/)
7. [Transformers Explained Visually (Part 1): Overview of Functionality](https://towardsdatascience.com/transformers-explained-visually-part-1-overview-of-functionality-95a6dd460452)
8. [Transformers Explained Visually (Part 2): How it works, step-by-step](https://towardsdatascience.com/transformers-explained-visually-part-2-how-it-works-step-by-step-b49fa4a64f34)
9. [Transformers Explained Visually (Part 3): Multi-head Attention, deep dive](https://towardsdatascience.com/transformers-explained-visually-part-3-multi-head-attention-deep-dive-1c1ff1024853)
10. [Attention in computer vision](https://towardsdatascience.com/attention-in-computer-vision-fd289a5bd7ad)
11. [VIT 三部曲 -1 Transformer](https://zhuanlan.zhihu.com/p/326892493)
12. [transformer直观理解](https://lisijian.cn/2020/09/04/2020-09-04-transformer%E7%9B%B4%E8%A7%82%E7%90%86%E8%A7%A3/)
13. [【关于Transformer】那些你不知道的事](https://github.com/km1994/NLP-Interview-Notes/tree/main/DeepLearningAlgorithm/transformer)
14. [一文看懂 Bahdanau 和 Luong 两种 Attention 机制的区别](https://zhuanlan.zhihu.com/p/129316415)
15. [Generating Long Sequences with Sparse Transformers](https://arxiv.org/pdf/1904.10509.pdf)
16. [SPARSE TRANSFORMER浅析](https://zhuanlan.zhihu.com/p/259591644)
17. [A Survey on Vision Transformer](https://arxiv.org/pdf/2012.12556.pdf)
18. [Transformers Meet Visual Learning Understanding: A Comprehensive Review](https://arxiv.org/pdf/2203.12944.pdf)
19. [Efficient Transformers: A Survey](https://arxiv.org/pdf/2009.06732.pdf)
20. [Transformers大家族——Efficient Transformers: A Survey](https://zhuanlan.zhihu.com/p/263031249)