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

