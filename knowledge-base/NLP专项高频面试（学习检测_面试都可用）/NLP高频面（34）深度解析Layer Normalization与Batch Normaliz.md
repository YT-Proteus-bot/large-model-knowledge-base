---
title: NLP高频面（34）深度解析Layer Normalization与Batch Normaliz
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（34）深度解析Layer Normalization与Batch Normaliz.pdf
questions: 1
chars: 1781
build_date: '2026-11-20'
pages: 3
---

# NLP高频面（34）深度解析Layer Normalization与Batch Normaliz

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（34）深度解析Layer Normalization与Batch Normaliz.pdf

本文讨论了深度神经网络中常用的归一化方法 Batch Normalization（BN）与 Layer Normalization（LN）的区别、联系，以及 Transformer 架构偏爱使用 LN 而非 BN 的原因。关键要点包括：

Batch Normalization（BN）：核心是对每个神经元输入激活函数前的数据进行批量归一化，沿批量维度计算均值和方差，依赖批次大小和数据分布，适用于 CNN，大 batch 效果好。

Layer Normalization（LN）：在单个样本内部进行归一化，不受批量大小和数据分布变化影响，计算公式为对单个样本所有特征集合进行处理，适合处理序列数据。

联系：都旨在缓解梯度消失问题、加速网络训练，都引入可学习的缩放和平移参数。

区别：BN 沿批量维度、对批量大小敏感、适用于图像；LN 沿特征维度、不依赖批量大小、适用于序列数据。

Transformer 选用 LN 的原因：序列位置重要且不确定，BN 无法有效捕捉序列内部特征关系；

Transformer 关注单个样本内部特征相对关系，LN 能保证其分布稳定；Transformer 处理序列长度动态变化，LN 不受影响，而 BN 受限制。

深度神经网络 中常用的归一化、方法包括Batch Normalization（BN）与Layer Normalization（LN），它们在提高模型稳定性、加速收敛等方面效果显著。然而，不同类型的模型往往会选择不同的归一化策略，本文详细讨论BN与LN的区别和联系，并解释为什么Transformer架构偏爱使用LN而非BN。

### 一、Batch Normalization（BN）

核心思想与特点：

Batch Norm的核心思想是对每个神经元输入激活函数之前的数据进行 批量归一化 操作，通过计算每个mini-batch中数据的均值和方差，实现数据分布稳定化。BN针对的是一个批次中的同一特征维度进行归一化操作，因此极其依赖于批次大小和数据分布。

BN的计算方式：

沿批量维度计算每个特征的均值与方差；

归一化特征，使其具有0均值和单位方差；

引入可学习的参数γ（缩放）和β（平移），使模型有能力恢复特征原本的表达能力。

适用场景：

BN广泛应用于卷积神经网络，因为CNN中卷积核的权重在不同空间位置共享，批量维度的信息能够较好地代表整体数据分布。

BN对大batch效果较好，但当batch size很小时效果显著下降。

### 二、Layer Normalization（LN）

核心思想与特点：

Layer Norm与Batch Norm不同，它是在单个样本内部进行归一化，不依赖于其他样本的统计信息。

具体地说，LN会对单个样本中的所有特征维度进行归一化，因此其不受批量大小限制，也不受数据分布变化的显著影响。

LN的计算方式：

Layer Norm的计算公式为：

其中，x代表单个样本中所有特征的集合，E(x)与Var(x)分别为这些特征的均值和方差，γ和β同样为可学习参数。

适用场景：

LN适合于处理序列数据，比如Transformer和RNN模型中，因为LN独立于batch size，天然适合动态长度或小batch场景。

在CNN中，LN效果往往不如BN，因为CNN依赖于空间维度上的统计稳定性，而非单个样本内部。

### 三、BN与LN的联系与区别

联系：

都旨在通过数据的归一化，缓解梯度消失问题，加速网络训练。

都引入了可学习的缩放和平移参数（γ与β），使网络能够保留并学习原始数据分布中的有效信息。

区别：

归一化维度不同： BN沿批量维度（跨样本），LN沿特征维度（单样本内）。

对批量大小的敏感性不同： BN对批量大小敏感，小批量性能大幅降低；LN不依赖批量大小。

适用的数据类型不同： BN适用于图像（CNN），LN适用于序列数据（Transformer、RNN）。

## 题1：四、为什么Transformer使用LN而不是BN？

Transformer选择LN而非BN的根本原因在于序列数据（如文本）本身的特点：

序列位置的重要性与不确定性：

单个样本内部的稳定性需求：

序列长度动态变化的场景：
