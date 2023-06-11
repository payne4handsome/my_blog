---
layout: post
title: "Transformer in pytorch"
date: 2022-09-17
# menu: main
categories: [机器学习]
tags: [Transformer]
---

# 一 Transformer overview

本文结合pytorch源码以尽可能简洁的方式把Transformer的工作流程讲解以及原理讲解清楚。全文分为三个部分

1. Transformer架构：这个模块的详细说明
2. pytorch中Transformer的api解读
3. 实际运用：虽然Transformer的api使用大大简化了打码量，但是还有需要自已实现一些代码的

## Transformer架构

Transformer结构如下：
![image.png](/Transformer in pytorch/8596800-4764969fee815f44.png)
Transformer的经典应用场景就是机器翻译。
整体分为Encoder、Decoder两大部分，具体实现细分为六块。

1. 输入编码、位置编码

    Encoder、Decoder都需要将输入字符进行编码送入网络训练。

    Input Embeding:将一个字符（或者汉字）进行编码，比如“我爱中国”四个汉字编码后会变成（4，d_model）的矩阵，Transformer中d_model等于512，那么输入就变成（4，512）的矩阵，*为了方便叙述，后面都用（4，512）来当成模型的输入*。

    positional encoding:**在Q、K、V的计算过程中，输入单词的位置信息会丢失掉**。所以需要额外用一个位置编码来表示输入单词的顺序。编码公式如下

    $PE_{pos,2i}=sin(pos/1000^{2i/d_{model}})$

    $PE_{pos,2i+1}=cos(pos/1000^{2i/d_{model}})$

    其中，pos:表示第几个单词，2i,2i+1表示Input Embeding编码维度（512）的偶数位、奇数位。
    **论文中作者也试过将positional encoding变成可以学习的，但是发现效果差不多；而且使用硬位置编码就不用考虑在推断环节中句子的实际长度超过训练环节中使用的位置编码长度的问题；为什么使用sin、cos呢？可以有效的考虑句中单词的相对位置信息**
2. 多头注意力机制（Multi-Head Attention）

    多头注意力机制是Transformer的核心，属于Self-Attention（自注意力）。**注意只要是可以注意到自身特征的注意力机制就叫Self-Attention，并不是只有Transformer有。** 示意图如下

    ![image.png](/Transformer in pytorch/8596800-993737ea7c1e190a.png)

    Multi-Head Attention的输入的Q、K、V就是输入的（4,512）维矩阵，Q=K=V。然后用全连接层对Q、K、V做一个变换，多头就是指的这里将输入映射到多个空间。公式描述如下：

    $MultiHead(Q,K,V)=Concat(head_1, head_2,..., head_n)W^o$

    其中
    $head_i=Attention(QW^Q_i, KW^K_i, VW^V_i)$

    其中
    $Attention(Q,K,V)=softmax(\frac{QK^T}{\sqrt{d_k}})V$

    其中$W^Q_i\in R^{d_{model}*d_k}, W^K_i\in R^{d_{model}*d_k}, W^V_i\in R^{d_{model}*d_v}, W^o\in R^{hd_v*d_{model}}$, 论文中h=8, $d_k=d_v=d_{model}/h=512/8=64$

    **$QK^T$称为注意力矩阵（attention）,表示两个句子中的任意两个单词的相关性。所以attention mask不一定是方阵。**

3. 前向传播模块

   Q、K、V经过Multi-Head Attention模块再加上一个残差跳链，维度不变，输入维度是（4，512），输出维度还是（4,512），只不过输出的矩阵的每一行已经融合了其他行的信息（根据attention mask）。
   这里前向传播模块是一个两层的全连接。公式如下：

    $FFN(x)=max(0, xW_1+b_1)W_2+b_2$, 其中输入输出维度为$d_model=512$, 中间维度$d_{ff}=2048$

4. 带Mask的多头注意力机制

    这里的Mask Multi-head Attention与步骤2中的稍有不同。“我爱中国”的英文翻译为“I love china”。 在翻译到“love”的时候，其实我们是不知道“china”的这个单词的，所以在**训练**的时候，就需要来模拟这个过程。即用一个mask来遮住后面信息。这个mask在实际实现中是一个三角矩阵（主对角线及下方为0，上方为-inf）， 定义为$attention\_mask$大概就长下面这个样子
    ![attention mask](/Transformer in pytorch/8596800-7aace427d17c31a9.png)

    $Attention(Q,K,V)=softmax(\frac{QK^T}{\sqrt{d_k}})V$, 加上$attention\_mask$后数学表达为

    $Masked\_Attention(Q,K,V)=Softmax(\frac{QK^T}{\sqrt{d_k}}+attention_{mask})V$。

    -inf在经过softmax后变成0，就相当于忽略对应的单词信息

5. Decoder中的多头注意力机制

    这里的Multi-head Attention和步骤2中的注意力就差不多是一个意思了，但是$Attention(Q,K,V)$中K，V是来自Encoder的输出，Q是Mask Multi-head Attention后的输出

6. 预测

    整个Transformer的输入维度是（4,512），输出维度是（4,512）。那么如果变成最终具体的单词呢（假如单词表大小为10000）。那么最后的输出output必须是（4，10000）。max(output, dim=-1)的下标就是单词的序号。

以上就是Transformer的核心流程以及解释，那么下面接下来看一下Transformer在pytorch中的具体实现。

## 二、Transformer在pytorch中具体实现

**pytorch version: 1.11**

### 2.1 Transformer类

要自已实现一个完整的Transformer，还是有点难度的，好在pytorch提供了官方实现。所有的核心细节都被封装**Transformer类**。**特别说明，下面的代码讲解中会删除非核心代码，只保留核心细节**

下面代码讲解会对矩阵的一些维度做一些说明，这里统一下符号

+ N: batch size
+ S: Encoder输入序列的长度
+ T: Decoder输入序列的长度
+ E: embeding的维度，就是上文中$d_{model}$

```python
class Transformer(Module):
"""   
Args:
        d_model: 单词维度(default=512).
        nhead: 多头注意力中的head数量 (default=8).
        num_encoder_layers: Encoder中子Encoder堆叠的数量(default=6).
        num_decoder_layers: Decoder中子Decoder堆叠的数量(default=6).
        dim_feedforward: 前向传播的中间维度 (default=2048).
        dropout: the dropout value (default=0.1).
        activation:  Default: relu
        custom_encoder: custom encoder (default=None).
        custom_decoder: custom decoder (default=None).
        layer_norm_eps:  (default=1e-5).
        batch_first: Default: ``False`` (seq, batch, feature).
        norm_first: 
"""
    def __init__(self, d_model: int = 512, nhead: int = 8, num_encoder_layers: int = 6,
                 num_decoder_layers: int = 6, dim_feedforward: int = 2048, dropout: float = 0.1,
                 activation: Union[str, Callable[[Tensor], Tensor]] = F.relu,
                 custom_encoder: Optional[Any] = None, custom_decoder: Optional[Any] = None,
                 layer_norm_eps: float = 1e-5, batch_first: bool = False, norm_first: bool = False,
                 device=None, dtype=None) -> None:
        factory_kwargs = {'device': device, 'dtype': dtype}
        super(Transformer, self).__init__()

        # 步骤2和3的实现
        encoder_layer = TransformerEncoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                activation, layer_norm_eps, batch_first, norm_first,
                                                **factory_kwargs)
        encoder_norm = LayerNorm(d_model, eps=layer_norm_eps, **factory_kwargs)

        # 将TransformerEncoderLayer执行6次，简单的堆叠而已
        self.encoder = TransformerEncoder(encoder_layer, num_encoder_layers, encoder_norm)

        # 步骤4、5的实现
        decoder_layer = TransformerDecoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                activation, layer_norm_eps, batch_first, norm_first,
                                                **factory_kwargs)
        decoder_norm = LayerNorm(d_model, eps=layer_norm_eps, **factory_kwargs)
        self.decoder = TransformerDecoder(decoder_layer, num_decoder_layers, decoder_norm)


        self.d_model = d_model
        self.nhead = nhead

        self.batch_first = batch_first

    def forward(self, src: Tensor, tgt: Tensor, src_mask: Optional[Tensor] = None, tgt_mask: Optional[Tensor] = None,
                memory_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None,
                tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        
        """
        src, tgt: 这里的src、tgt是已经经过input embeding和positional emdbeding后的输入。
        memory_mask：实际使用的过程中都是None
        memory_key_padding_mask：实际使用的过程中就是src_key_padding_mask
        """
        memory = self.encoder(src, mask=src_mask, src_key_padding_mask=src_key_padding_mask)

        # 这里的memory就是步骤5中K，V矩阵
        # memory_key_padding_mask实际使用过程就是传入的src_key_padding_mask，在步骤5中需要遮住Encoder中padding的位置
        output = self.decoder(tgt, memory, tgt_mask=tgt_mask, memory_mask=memory_mask，tgt_key_padding_mask=tgt_key_padding_mask，memory_key_padding_mask=memory_key_padding_mask)
        return output

    @staticmethod
    def generate_square_subsequent_mask(sz: int) -> Tensor:
        r"""用来生成步骤4中attention mask."""
        return torch.triu(torch.full((sz, sz), float('-inf')), diagonal=1)
```

#### 关键参数说明

+ batch_first：默认False，在pytorch中，rnn、Transformer层的输入维度一般是(seq, batch, feature)，第一个维度表示seq的长度，batch放在第二个维度
+ src、tgt：由于batch_first=False， 所以Encoder的输入src、Decoder的输入tgt的shape都是(seq, batch, feature)
+ src_mask： shape:(S, S), 含义就是上文讲的attenion_mask, Encoder的输入是不需要遮住后面的单词的，所以该参数一般是一个全为False的阵
+ tgt_mask: shape:(T, T), 含义同src_mask，但是Decoder的输入是需要遮住后面的单词的，所以这里的mask是一个三角矩阵（下三角是0，上三角是-inf。当然也可以用True， False表示）
+ src_key_padding_mask: shape:(N, S) 因为模型的输入一般是一个batch，实际场景中输入的句子或者序列是不等长的，那么就需要将不等长的多个序列通过padding的方式补齐成等长的。那么padding的位置无意义，不需要参与attention的计算，所以通src_key_padding_mask来标记padding的位置。后面计算attention的时候不计算该位置的权重
+ tgt_key_padding_mask: shape:(N, T) 含义同src_key_padding_mask

**src_mask，tgt_mask，src_key_padding_mask，tgt_key_padding_mask虽然在这里是分开的，但是在计算attention时候，实际是合并到同一个attention矩阵中的；维度不同，通过广播的方式合并**

### 2.2 TransformerEncoder、TransformerDecoder

TransformerEncoder、TransformerDecoder逻辑是类似的，就是执行TransformerEncoderLayer多次，默认是6次，以TransformerEncoder为例

```python
class TransformerEncoder(Module):
    r"""TransformerEncoderLayer 堆叠N次

    Args:
        encoder_layer: TransformerEncoderLayer（子模块）
        num_layers: 堆叠次数
        norm: 

    """

    def __init__(self, encoder_layer, num_layers, norm=None):
        super(TransformerEncoder, self).__init__()
        # 模块复制N次
        self.layers = _get_clones(encoder_layer, num_layers)
        self.num_layers = num_layers
        self.norm = norm

    def forward(self, src: Tensor, mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
     
        output = src
        # 执行N次TransformerEncoderLayer
        for mod in self.layers:
            output = mod(output, src_mask=mask, src_key_padding_mask=src_key_padding_mask)
        # 正则
        if self.norm is not None:
            output = self.norm(output)

        return output
```

### 2.3 TransformerEncoderLayer

```python
class TransformerEncoderLayer(Module):
    r"""步骤2和3的具体实现.

    Args:
        基本上见名知意

    """

    def __init__(self, d_model: int, nhead: int, dim_feedforward: int = 2048, dropout: float = 0.1,
                 activation: Union[str, Callable[[Tensor], Tensor]] = F.relu,
                 layer_norm_eps: float = 1e-5, batch_first: bool = False, norm_first: bool = False,
                 device=None, dtype=None) -> None:
        factory_kwargs = {'device': device, 'dtype': dtype}
        super(TransformerEncoderLayer, self).__init__()

        # 步骤2的实现
        self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first,
                                            **factory_kwargs)

        # 步骤3的实现，两个全连接层
        self.linear1 = Linear(d_model, dim_feedforward, **factory_kwargs)
        self.linear2 = Linear(dim_feedforward, d_model, **factory_kwargs)


    

    def forward(self, src: Tensor, src_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        r"""Pass the input through the encoder layer.

        Args:
            src: the sequence to the encoder layer (required).
            src_mask: the mask for the src sequence (optional).
            src_key_padding_mask: the mask for the src keys per batch (optional).

        """
        x = src
        if self.norm_first:  # norm操作放在哪里执行
            x = x + self._sa_block(self.norm1(x), src_mask, src_key_padding_mask)
            x = x + self._ff_block(self.norm2(x))
        else:
            x = self.norm1(x + self._sa_block(x, src_mask, src_key_padding_mask))
            x = self.norm2(x + self._ff_block(x))

        return x

    # 步骤2
    def _sa_block(self, x: Tensor,
                  attn_mask: Optional[Tensor], key_padding_mask: Optional[Tensor]) -> Tensor:
        x = self.self_attn(x, x, x,
                           attn_mask=attn_mask,
                           key_padding_mask=key_padding_mask,
                           need_weights=False)[0]
        return self.dropout1(x)

    # 步骤3
    def _ff_block(self, x: Tensor) -> Tensor:
        x = self.linear2(self.dropout(self.activation(self.linear1(x))))
        return self.dropout2(x)
```

上面除了MultiheadAttention的实现细节，其他逻辑是很清楚的

### 2.4 TransformerDecoderLayer

TransformerDecoderLayer的实现逻辑与TransformerEncoderLayer实现差不多，有两点不同

1. 第一个注意力（步骤4）的mask，需要遮住后续单词信息
2. 第二个注意力（不足5）的K，V来自Encoder的输出（代码中叫memory）

```python
class TransformerDecoderLayer(Module):
    r"""步骤4和5的具体实现.
    Args:
        见名知意

    """

    def __init__(self, d_model: int, nhead: int, dim_feedforward: int = 2048, dropout: float = 0.1,
                 activation: Union[str, Callable[[Tensor], Tensor]] = F.relu,
                 layer_norm_eps: float = 1e-5, batch_first: bool = False, norm_first: bool = False,
                 device=None, dtype=None) -> None:
        factory_kwargs = {'device': device, 'dtype': dtype}
        
        # 步骤4
        self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first,
                                            **factory_kwargs)
        # 步骤5                                    
        self.multihead_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first,
                                                 **factory_kwargs)
        # Implementation of Feedforward model
        self.linear1 = Linear(d_model, dim_feedforward, **factory_kwargs)
        self.dropout = Dropout(dropout)
        self.linear2 = Linear(dim_feedforward, d_model, **factory_kwargs)


    def forward(self, tgt: Tensor, memory: Tensor, tgt_mask: Optional[Tensor] = None, memory_mask: Optional[Tensor] = None,
                tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        r"""Pass the inputs (and mask) through the decoder layer.

       
        """
        
        x = tgt
        # 无论是_sa_block()还是_mha_block()的具体实现都在MultiheadAttention中
        if self.norm_first:
        # _sa_block对应步骤4，
            x = x + self._sa_block(self.norm1(x), tgt_mask, tgt_key_padding_mask)
        # _mha_block对应步骤5，只不过Q来自于Decoder本身，K，V来自于Encoder,就是这里memory
            x = x + self._mha_block(self.norm2(x), memory, memory_mask, memory_key_padding_mask)
            x = x + self._ff_block(self.norm3(x))
        else:
            x = self.norm1(x + self._sa_block(x, tgt_mask, tgt_key_padding_mask))
            x = self.norm2(x + self._mha_block(x, memory, memory_mask, memory_key_padding_mask))
            x = self.norm3(x + self._ff_block(x))

        return x

    # # 步骤4
    def _sa_block(self, x: Tensor,
                  attn_mask: Optional[Tensor], key_padding_mask: Optional[Tensor]) -> Tensor:

        # Q=K=V=x, 
        x = self.self_attn(x, x, x,
                           attn_mask=attn_mask,
                           key_padding_mask=key_padding_mask,
                           need_weights=False)[0]
        return self.dropout1(x)

    # # 步骤5
    def _mha_block(self, x: Tensor, mem: Tensor,
                   attn_mask: Optional[Tensor], key_padding_mask: Optional[Tensor]) -> Tensor:
        # Q=x(来自于_sa_block（）的输出)， K=V=mem(来自Encoder的输出)，
        # attn_mask=None, key_padding_mask等于src_key_padding_mask
        x = self.multihead_attn(x, mem, mem,
                                attn_mask=attn_mask,
                                key_padding_mask=key_padding_mask,
                                need_weights=False)[0]
        return self.dropout2(x)

    # feed forward block
    def _ff_block(self, x: Tensor) -> Tensor:
        x = self.linear2(self.dropout(self.activation(self.linear1(x))))
        return self.dropout3(x)
```

整个流程以及参数已经说明了，TransformerEncoderLayer、TransformerDecoderLayer的代码实现中就剩MultiheadAttention实现没有讲了

### 2.5 MultiheadAttention

MultiheadAttention的核心实现在multi_head_attention_forward方法中，我们直接看multi_head_attention_forward方法。
**以下代码删除了非核心参数和非核心代码**

```python
def multi_head_attention_forward(
    query: Tensor,
    key: Tensor,
    value: Tensor,
    key_padding_mask: Optional[Tensor] = None,
    attn_mask: Optional[Tensor] = None,
) -> Tuple[Tensor, Optional[Tensor]]:
    r"""
    Args:
        multi_head_attention 实现
    Shape:
       
    """
    # set up shape vars
    tgt_len, bsz, embed_dim = query.shape
    src_len, _, _ = key.shape
    match value shape {value.shape}"

    # prep attention mask
    # 将attn_mask变成3维的，方便后面与key_padding_mask合并
    if attn_mask is not None:
        if attn_mask.dtype == torch.uint8:
            warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
            attn_mask = attn_mask.to(torch.bool)
        # ensure attn_mask's dim is 3
        if attn_mask.dim() == 2:
            correct_2d_size = (tgt_len, src_len)
            attn_mask = attn_mask.unsqueeze(0)
        elif attn_mask.dim() == 3:
            correct_3d_size = (bsz * num_heads, tgt_len, src_len)
        else:
            raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")

    # prep key padding mask
    if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
        key_padding_mask = key_padding_mask.to(torch.bool)

    
    #
    # reshape q, k, v for multihead attention and make em batch first
    #
    q = q.contiguous().view(tgt_len, bsz * num_heads, head_dim).transpose(0, 1)
    k = k.contiguous().view(k.shape[0], bsz * num_heads, head_dim).transpose(0, 1)
    v = v.contiguous().view(v.shape[0], bsz * num_heads, head_dim).transpose(0, 1)
   

   
    # update source sequence length after adjustments
    src_len = k.size(1)

    # merge key padding and attention masks
    # key_padding_mask 和 attn_mask合并，
    if key_padding_mask is not None:
        # key_padding_mask 做一些维度的变换
        key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
            expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
        if attn_mask is None:
            attn_mask = key_padding_mask
        elif attn_mask.dtype == torch.bool:
            # 合并attention mask
            attn_mask = attn_mask.logical_or(key_padding_mask)
        else:
            attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))

    # 将attn_mask变成float类型，True变成负无穷，False变成0
    if attn_mask is not None and attn_mask.dtype == torch.bool:
        new_attn_mask = torch.zeros_like(attn_mask, dtype=q.dtype)
        new_attn_mask.masked_fill_(attn_mask, float("-inf"))
        attn_mask = new_attn_mask

    #
    # (deep breath) calculate attention and out projection
    # QKV计算
    attn_output, attn_output_weights = _scaled_dot_product_attention(q, k, v, attn_mask, dropout_p)

    attn_output = attn_output.transpose(0, 1).contiguous().view(tgt_len, bsz, embed_dim)
    attn_output = linear(attn_output, out_proj_weight, out_proj_bias)

    return attn_output, None
```

**上面的代码实现的核心逻辑是Attention（Q，K，V）的计算，还有一个就是Transformer的输入参数\*_mask, \*_key_padding_mask是这么影响最终的注意力权重的；就是将两个mask合并为attn_mask,最后加到$QK^T$上**

三、实际应用

实际应用其实官方有一个翻译的例子[LANGUAGE TRANSLATION WITH NN.TRANSFORMER AND TORCHTEXT](https://pytorch.org/tutorials/beginner/translation_transformer.html)，说的还是很清楚的。可以参考。

通过上面Transformer类的详细说明，我们是否可以训练自已的seq2seq模型了呢？其实没有，还有几件事要做

1. 将输入字符变成一个个数字，需要自已按照使用场景实现一个类。以中文为例，是将一个汉字mapping到一个索引，还是将一个词mapping到一个索引。可以仿照pytorch的torchtext.vocab.build_vocab_from_iterator去实现

2. 将上面转化后的索引list转化为，Transformer类的输入。就需要实现一个映射词向量表和位置编码。比较“我爱中国”转化为索引后可能是[300, 250, 10, 888],词向量表需要将这个list变成（4， 512）的向量，即用一个512位的向量来表示一个单词
这里给出一个实现, 其实就将pytorch的Transformer类加上输入编码和位置编码部分。

```python
class Seq2SeqTransformer(nn.Module):
    def __init__(self,
                 num_encoder_layers: int,
                 num_decoder_layers: int,
                 emb_size: int,
                 nhead: int,
                 src_vocab_size: int,
                 tgt_vocab_size: int,
                 dim_feedforward: int = 512,
                 dropout: float = 0.1):
        super(Seq2SeqTransformer, self).__init__()
        self.transformer = Transformer(d_model=emb_size,
                                       nhead=nhead,
                                       num_encoder_layers=num_encoder_layers,
                                       num_decoder_layers=num_decoder_layers,
                                       dim_feedforward=dim_feedforward,
                                       dropout=dropout)
        self.generator = nn.Linear(emb_size, tgt_vocab_size)
        self.src_tok_emb = TokenEmbedding(src_vocab_size, emb_size)
        self.tgt_tok_emb = TokenEmbedding(tgt_vocab_size, emb_size)
        self.positional_encoding = PositionalEncoding(
            emb_size, dropout=dropout)

    def forward(self,
                src: Tensor,
                trg: Tensor,
                src_mask: Tensor,
                tgt_mask: Tensor,
                src_padding_mask: Tensor,
                tgt_padding_mask: Tensor,
                memory_key_padding_mask: Tensor):
        """

        :param src:
        :param trg:
        :param src_mask: 用于遮挡句子的下文，shape(S, S)
        :param tgt_mask:
        :param src_padding_mask: 用于指定pad位置，shape(B, S)
        :param tgt_padding_mask:
        :param memory_key_padding_mask:
        :return:
        """
        src_emb = self.positional_encoding(self.src_tok_emb(src))
        tgt_emb = self.positional_encoding(self.tgt_tok_emb(trg))
        outs = self.transformer(src_emb, tgt_emb, src_mask, tgt_mask, None,
                                src_padding_mask, tgt_padding_mask, memory_key_padding_mask)
        return self.generator(outs)

    def encode(self, src: Tensor, src_mask: Tensor):
        return self.transformer.encoder(self.positional_encoding(
                            self.src_tok_emb(src)), src_mask)

    def decode(self, tgt: Tensor, memory: Tensor, tgt_mask: Tensor):
        return self.transformer.decoder(self.positional_encoding(
                          self.tgt_tok_emb(tgt)), memory,
                          tgt_mask)

# 单词编码
class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size: int, emb_size):
        super(TokenEmbedding, self).__init__()
        self.embedding = nn.Embedding(vocab_size, emb_size)
        self.emb_size = emb_size

    def forward(self, tokens: Tensor):
        return self.embedding(tokens.long()) * math.sqrt(self.emb_size)

# 位置编码
class PositionalEncoding(nn.Module):
    def __init__(self,
                 emb_size: int,
                 dropout: float,
                 maxlen: int = 5000):
        super(PositionalEncoding, self).__init__()
        den = torch.exp(- torch.arange(0, emb_size, 2)* math.log(10000) / emb_size)
        pos = torch.arange(0, maxlen).reshape(maxlen, 1)
        pos_embedding = torch.zeros((maxlen, emb_size))
        pos_embedding[:, 0::2] = torch.sin(pos * den)
        pos_embedding[:, 1::2] = torch.cos(pos * den)
        pos_embedding = pos_embedding.unsqueeze(-2)

        self.dropout = nn.Dropout(dropout)
        self.register_buffer('pos_embedding', pos_embedding)

    def forward(self, token_embedding: Tensor):
        return self.dropout(token_embedding + self.pos_embedding[:token_embedding.size(0), :])
```

3. 实际的翻译过程

**这一部分等我有空且想完善的时候，会继续完善**，在这之前强烈建议大家直接参考官方的例子，见参考文献一


## 参考文献
1. [LANGUAGE TRANSLATION WITH NN.TRANSFORMER AND TORCHTEXT](https://pytorch.org/tutorials/beginner/translation_transformer.html)
