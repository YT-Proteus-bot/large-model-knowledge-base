# 感知机与多层感知机 (MLP)

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/KW9MwdBiKi5SJckoo2qchSqynvd

#### 1. 感知机模型的基本结构是什么？它的数学表达式是怎样的？

感知机是模拟单个神经元工作方式的二分类线性模型，由Frank Rosenblatt在1957年提出。它的基本结构包含三个关键部分：

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/NfNcbE3JLoQUBWxj7k7cpTUdnxe.png)

数学表达式为：

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/VzvPbW1ZVoIrPXxNtNGclsgVnzg.png)

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/OEYDbIvhyo1s3jxfSMicjIk9nol.png)

两类样本。训练时，感知机采用迭代更新策略：对于每个被误分类的样本，按照规则 **w**←**w**+*η*⋅(*ytrue*−*ypred*)⋅**x** 调整权重，其中 *η* 是学习率。如果数据是线性可分的，感知机可以在有限步内收敛到正确分类所有样本的解。

然而，感知机的局限性非常明显：它只能处理线性可分的问题。当数据分布无法用一条直线（或超平面）完全分开时，感知机便束手无策，最经典的例子就是异或问题。

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/MyzobL7qtoPa1PxdUZ0cqt3bnpc.png)

#### 2. 为什么单层感知机无法解决异或（XOR）问题？请从线性可分性角度解释。

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/IuVdbnghzo72TmxmGBmcPOq1nrf.png)

异或逻辑的真值表为：当两个输入相同时输出0，不同时输出1。在二维平面上，四个点分别是 (0,0) 输出0、(0,1) 输出1、(1,0) 输出1、(1,1) 输出0。

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/CW4ObAxq8opSsdxnzHuc9RiqnFf.png)

对于异或，情况并非如此。正类点 (0,1) 和 (1,0) 与负类点 (0,0) 和 (1,1) 在平面上交错分布，没有任何一条直线可以将它们清晰地分为两类。无论权重如何调整，感知机的决策边界永远是一条直线，它无法在异或这种非线性可分的数据上收敛。这个发现是人工智能早期的一次重大挫折，直接导致了神经网络研究的第一次寒冬。

#### 3. 多层感知机（MLP）是如何解决异或问题的？需要至少几个隐藏层？

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/AMssbs703oCwgBxqwqncB33Cnpc.png)

解决异或的关键在于引入隐藏层，将原始输入空间映射到新的特征空间，使数据在新空间中变得线性可分。

多层感知机仅需一个隐藏层，其中包含两个神经元，就能表示异或函数。具体来说，两个隐藏神经元分别学习输入的不同线性组合，经过非线性激活函数后，产生新的特征表示。输出层再将这两个新特征进行线性组合，即可构造出异或的决策边界。

例如，使用ReLU激活函数：

![image](../../../assets/DpUZdErVtoMzlWxayNYcdtHYnne/Abdtb6yAZowQWXx1qsecUxhjnjd.png)
