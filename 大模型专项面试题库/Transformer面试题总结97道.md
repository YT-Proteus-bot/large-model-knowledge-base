# Transformer面试题总结97道

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/IRWfwaMWHiy8VLkBWyQc9Sc3njn

## **Transformer面试题总结97道**

#### **1、请简述Transformer的整体架构，并说明其与RNN/LSTM的核心区别。**

答案：

Transformer由Encoder和Decoder两部分组成（若仅用于BERT等预训练模型，可单独使用Encoder）。

- Encoder：由N个相同的层堆叠而成，每层包含两个子层：
  1. 多头自注意力：捕捉输入序列中任意两个位置间的依赖关系。
  1. 前馈神经网络：对每个位置的表示进行非线性变换。
  - 每个子层后都接有残差连接和层归一化。
- Decoder：同样由N个相同的层堆叠而成，每层包含三个子层：
  1. 掩码多头自注意力：防止在预测当前位置时看到未来位置的信息。
  1. 交叉注意力：将Encoder的输出作为K、V，Decoder的当前状态作为Q，实现信息交互。
  1. 前馈神经网络。

核心区别：

- 并行计算：RNN/LSTM按时间步顺序计算，无法并行；Transformer通过自注意力一次性处理整个序列，可并行计算，训练速度更快。
- 长程依赖：RNN/LSTM存在梯度消失/爆炸问题，难以捕捉长距离依赖；Transformer中任意两个位置的距离均为1，能有效建模长程依赖。
- 架构组成：RNN/LSTM依赖循环单元；Transformer完全基于注意力机制，无循环结构。

---

#### **2、详细解释Self-Attention的计算过程，并写出其公式。**

答案：

Self-Attention的核心是将输入序列的每个元素与序列中所有元素进行交互，通过加权求和得到新的表示。计算步骤如下：

1. 线性变换：对输入矩阵 X∈Rn×d*X*∈R*n*×*d*（n*n*为序列长度，d*d*为嵌入维度），通过三个权重矩阵

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/Rtgqbv7dtoli1dxje22ccJNvnWC.png)

1. 计算注意力分数：通过 Q*Q* 与 K*K* 的点积计算相似度，并除以 dk*dk*（dk*dk* 是每个头的维度，用于防止点积结果过大导致梯度消失）：

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/BDOJbtgHnoVMNUxLzJxcw9bFnKf.png)

1. Softmax归一化：对每一行（即每个查询位置）应用Softmax，得到注意力权重：

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/MV1wb8zzIofQynxHXHOcXdzhnjf.png)

1. 加权求和：将注意力权重与 V*V* 相乘，得到最终输出：

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/VNH3bZROBoZxuYx1ncvchNhvnGf.png)

---

#### **3、为什么要使用多头注意力（Multi-Head Attention）？它的作用是什么？**

答案：
多头注意力是将 Q,K,V*Q*,*K*,*V* 分别线性投影到 h*h* 个不同的低维空间，分别进行注意力计算，然后将结果拼接并再次投影。

作用：

- 捕捉多种语义关系：不同的头可以关注不同的信息（例如，有的头关注句法结构，有的头关注长程依赖，有的头关注局部信息），使模型能从多个角度理解序列。
- 增强表示能力：通过多个头，模型可以学习到更丰富的特征表示，提升泛化能力。
- 稳定训练：将高维注意力分解为多个低维头，避免了单头注意力在高维空间可能出现的梯度问题。

公式：

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/EGwAbybMRoJE1IxToHzclosRnkh.png)

---

#### **4、为什么Transformer需要位置编码？有哪些常见的位置编码方式？**

答案：

Transformer的注意力机制是置换不变的，即交换输入序列中两个token的位置，输出也会相应交换，这导致模型无法感知token的顺序信息。因此需要显式注入位置信息。

常见位置编码方式：

1. 正弦/余弦位置编码（原始Transformer）：使用不同频率的三角函数：

![image](../../assets/UDyMdDE4fohQTjxDQAvcMvyYnwh/LfVybXRQcoBo4gxuP1CcfNK5n5f.png)

1. 优点：无需学习，能外推到更长的序列。
1. 可学习位置编码：将位置编码作为可训练参数，如BERT、GPT中使用。优点：灵活，适合固定长度序列。
1. 相对位置编码：编码token之间的相对距离，如Transformer-XL、T5、RoPE（旋转位置编码）等。能更好地处理长序列和泛化。

---
