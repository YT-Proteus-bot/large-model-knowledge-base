# 序列到序列 (Seq2Seq) 与注意力

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/HDMNwW1rYiwDBzkPnxacJdqFnKc

#### 1. Seq2Seq 模型的基本架构是怎样的？Encoder 和 Decoder 分别起什么作用？

![image](../../../assets/FZCfd9SiJoz7FKxH8v1c8m2znCd/U6RMbI6zOoJSKrx4AN4cORktnJh.png)

Seq2Seq（Sequence-to-Sequence）模型 是一种通用的端到端序列变换框架，用于将一个输入序列映射为一个输出序列，输入和输出长度可以不同。其最经典的实现基于 RNN（LSTM/GRU）。

基本架构

由两部分组成：编码器（Encoder） 和 解码器（Decoder）。

![image](../../../assets/FZCfd9SiJoz7FKxH8v1c8m2znCd/CvZhbPFvYofGi1x2xgLcTPhhngb.png)

作用总结

- 编码器：负责理解输入序列，将其信息压缩到上下文向量中。
- 解码器：负责生成输出序列，从上下文向量解码出目标序列，逐步预测下一个词。

---

#### 2. 在 Seq2Seq 中，上下文向量（Context Vector）是如何产生的？存在什么瓶颈？

上下文向量的产生

在最原始的 Seq2Seq 模型（Cho et al., 2014; Sutskever et al., 2014）中，编码器最后一个时间步的隐藏状态直接被作为上下文向量：

![image](../../../assets/FZCfd9SiJoz7FKxH8v1c8m2znCd/RgZ3b9uhKoAKRLxBVeXcbWnynub.png)

或者通过一个简单的变换（如 tanh⁡tanh 层）。无论输入序列多长，所有信息都必须压缩到这个固定维度的向量中。

瓶颈

1. 容量限制：固定大小的向量难以完整保留长序列的全部语义和细节信息，尤其是序列很长时。这会导致信息丢失，对长句输入表现较差。
1. 梯度问题：长序列中，编码器后部的信号需要通过很多时间步反向传播到前端，导致梯度消失/爆炸，使得模型难以学习长距离依赖。
1. 对齐困难：解码器在每个生成时刻都只看到同一个全局向量，无法有选择地关注输入的不同部分。在翻译、摘要等任务中，输出词往往只与输入的局部片段相关，全局上下文向量无法提供这种细粒度的对齐。
1. 性能退化：随着输入长度增加，模型性能（如 BLEU）明显下降，尤其在机器翻译中，超过一定长度后准确率骤降。

解决方案

注意力机制（Bahdanau et al., 2015）的引入打破了这一瓶颈：解码器不再依赖单一的固定上下文向量，而是在每个时间步动态地生成一个加权上下文向量，有重点地读取编码器的各个隐藏状态。

---

#### 3. Teacher Forcing 是什么？它在训练 Seq2Seq 时如何工作？有什么优缺点？

Teacher Forcing（教师强制）

Teacher Forcing 是一种训练自回归模型的标准策略。在训练 Seq2Seq 解码器时，不使用模型自己上一步预测的词作为下一步的输入，而是强制使用真实标签（目标序列中的词） 作为下一时间步的输入。

工作机制

![image](../../../assets/FZCfd9SiJoz7FKxH8v1c8m2znCd/Z1TFbWcj2oOannxR602ceaHPnlh.png)

这样，损失的梯度可以从每个时间步直接回传，因为输入不依赖于模型的采样，计算图是确定性的，且可以高效地并行训练（通过右移目标序列作为输入，一次性计算所有时间步的预测）。

优点

- 加速收敛：模型始终看到正确的上下文，避免错误累积，梯度更稳定。
- 训练高效：可以通过矩阵运算同时计算整个序列的损失，而不必逐个时间步串行。
