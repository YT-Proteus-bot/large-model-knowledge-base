---
title: 【字节】大厂LLM面试必考题
topic: 大模型大厂面试题，持续更新
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: 大模型大厂面试题，持续更新/【字节】大厂LLM面试必考题.pdf
questions: 9
chars: 3103
build_date: '2026-11-20'
pages: 4
---

# 【字节】大厂LLM面试必考题

> 主题：大模型大厂面试题，持续更新　｜　原始文件：大模型大厂面试题，持续更新/【字节】大厂LLM面试必考题.pdf

## 大厂LLM面试必考题

本文讨论了大厂LLM面试相关内容，涵盖AI大模型教程、就业现状以及Transformer相关技术问题解答等。关键要点包括：

教程介绍：有AI大模型全套教程、企业级应用开发教程和MCP Server开发零基础教程。

就业思考：探讨AI大模型工程师就业现状，如开发与算法方向选择、岗位技术门槛和学历要求、学习方法等。

Transformer优缺点：优点是位置关联操作不受限、建模和通用扩展性强、利于并行运算；缺点是局部信息获取弱、有位置信息编码问题和顶层梯度消失。

交互机制：Encoder端和Decoder端通过Cross Self - attention交互，Decoder提供Q，Encoder提供K、V。

多头注意力作用：将模型分多个头形成子空间，关注不同信息，解决注意力集中自身问题，h越大模型表达和注意力权重分配能力越强。

QK缩放原因：大的dk使QK点积值大，将softmax推向梯度平缓区，缩放可获更平缓softmax，利于网络训练。

K和Q使用不同值原因：K和Q点乘得attention score矩阵提纯V，不同投影增强表达和泛化能力，用相同值泛化能力差。

## Transformer的优缺点

优点： 位置关联操作不受限，建模能力强 ，通用性和可扩展性强，能更好的进行并行运算

缺点： 局部信息获取不强，位置信息编码问题，顶层梯度消失

## 题1：Encoder端和Decoder端是如何进行交互的？

Cross Self-attention、Decoder提供Q、Encoder提供K，V

## 题2：为什么需要multi-head attention？

将模型分为多个头，形成多个子空间，可以关注不同方面的信息，类似于 cnn 的多个卷积核

为了解决模型在对当前位置的信息进行编码时，会过度的将注意力集中于自身位置的问题

一定程度上h越大整个模型的表达能力越强，越能提高模型对于注意力权重的合理分配

## 题3：QK为什么要除以dk来进行Scale？

对于较大的dk来说，QK会得到极大的点积值，将整个 softmax 推向梯度平缓区，收敛困难，不利于网络的训练

假设Q和K的均值为0，方差为1。它们的矩阵乘积将有均值为0，方差为dk，因此用dk的平方根被用于缩放， 因为，Q和K的矩阵乘积的均值本应该为0，方差本应该为1，这样可以获得更平缓的softmax 。

当维度很大时，点积结果会很大，会导致softmax的梯度很小； 为了减轻这个影响，对点积进行缩放

## 题4：为什么使用不同的K和Q，为什么不能使用同一个值？

K和Q的点乘是为了得到一个 attention score 矩阵，用来对V进行提纯

K和Q使用了不同的 , ， 是在不同空间上的投影，增强了表达能力，提高泛化能力

如果不用Q，直接拿K和K点乘的话， attention score 是一个对称矩阵。 因为同一个矩阵都投影到了同样一个空间，所以泛化能力很差

## 题5：在计算attention score的时候如何对padding做mask操作？

padding mask： 所有 scaled dot-productattention 都用到，因为一个batch输入中，所有序列的长度使不同的。为了符合模型的输入方式，会用padding的方式来填充（比如填0），使所有序列的长度一致。但填充部分是没有意义的，所以在计算注意力的时候，不需要也不应该有注意力分配到这些填充的值上面。padding位置置为负无穷（一般来说-1000就可以）

sequence mask： 在 decoder 的 self-attention 里面用到 。在decoder部分，因为不能见到下文信息（防止泄漏），所以用mask的方式掩盖掉当前时刻t及之后的下文信息。具体，可产生一个对角线为0的上三角矩阵，将其作用到每个decoder的输入列上

## 题6：self-attention是什么？

self-attention 的特点在于无视token之间的距离直接计算依赖关系，从而学习序列关系

## 题7：关于 self-attention 为什么它能发挥如此大的作用？

强大的特征抽取能力，更好的 representation 来表达自身，并行计算

## 题8：Transformer 中的Add & Norm 模块，具体是怎么做的？

Layer Normalization ， 公式其实是常见的归一化方式： （x-mu）/sigma

## 简单介绍一下Transformer的位置编码？有什么意义和优缺点？

因为 self-attention 是位置无关的，无论句子的顺序是什么样的，通过 self-attention 计算的 token 的hidden embedding 都是一样的，这显然不符合人类的思维

因此要有一个办法能够在模型中表达出一个token的位置信息， transformer 使用了固定的 positionalencoding 来表示 token 在句子中的绝对位置信息

## 题9：为什么transformer块使用LayerNorm而不是BatchNorm？

正则化的本质是通过把一部分不重要的复杂信息损失掉，以此来降低拟合难度以及过拟合的风险，Normalization的目的是降低各维度数据的方差

CV用BN是认为不同卷积核feature map（channel维）之间的差异性很重要，LN会损失channel的差异性，对于batch内的不同样本，同一卷积核提取特征的目的性是一致的，所以使用BN仅是为了进一步保证同一个卷积核在不同样本上提取特征的稳定性。

NLP用LN是认为batch内不同样本同一位置token之间的差异性更重要，而embedding维，网络对于不同token提取的特征目的性是一致的，使用LN是为了进一步保证在不同 token 上提取的稳定性。NLP每个序列的长度是不一致的，BN不适用

## Decoder阶段的多头自注意力和encoder的多头自注意力有什么区别？

Decoder有两层mha, encoder有一层mha， Decoder的第二层mha是为了转化输入与输出句长，Decoder的请求q与键k和数值v的倒数第二个维度可以不一样，但是encoder的qkv维度一样

## Transformer attention的注意力矩阵的计算为什么用乘法而不是加法？

为了计算更快 。加法形式是先加、后tanh、再和V矩阵相乘，相当于一个完整的隐层。在计算复杂度上，乘法和加法理论上的复杂度相似，但在实践中，乘法可以利用高度优化的矩阵乘法代码（有成熟的加速实现）使得点乘速度更快，空间利用率更高

在dk较小的时候，加法和乘法形式效果相近 。但是随着dk增大，加法开始显著优于乘法。作者认为，dk增大导致乘法性能不佳的原因，是极大的点乘值将整个softmax推向梯度平缓区，使得收敛困难。

于是选择scale

## Transformer的残差结构及意义

解决梯度消失，防止过拟合

## Transformer的并行化体现在哪里，Decoder可以做并行化嘛？

Encoder的模块是串行的，子模块Multi-Head和FFN是并行，因为单词之间没有依赖关系

Decode引入sequence mask就是为了并行化训练，推理过程不并行
