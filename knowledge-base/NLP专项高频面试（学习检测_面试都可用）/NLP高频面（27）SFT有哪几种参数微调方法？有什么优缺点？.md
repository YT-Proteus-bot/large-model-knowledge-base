---
title: NLP高频面（27）SFT有哪几种参数微调方法？有什么优缺点？
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（27）SFT有哪几种参数微调方法？有什么优缺点？.pdf
questions: 2
chars: 1125
build_date: '2026-11-20'
pages: 2
---

# NLP高频面（27）SFT有哪几种参数微调方法？有什么优缺点？

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（27）SFT有哪几种参数微调方法？有什么优缺点？.pdf

## 题1：NLP高频面（27）SFT有哪几种参数微调方法？

## 题2：有什么优缺点？

本文讨论了随着大语言模型的普及，SFT中常用的几种参数微调方法及各自的优缺点。关键要点包括：

参数微调方法分类：常见分类为全参微调和PEFT方法，全参微调让所有模型参数参与微调，能取得最佳性能，但参数量巨大、计算成本高且易过拟合。

Adapter Tuning方法：通过在Transformer层插入额外串行结构（Adapter模块），仅更新其参数，优点是参数量小、更新成本低且可插拔；缺点是串行结构在特定情况下会降低推理效率。

Prefix Tuning方法：在Transformer每层前增加虚拟token作为前缀，仅更新Prefix参数，优点为并行结构不影响计算效率、参数可学习；缺点是占用输入序列长度、训练不稳定。

Prompt Tuning方法：仅在输入层加入少量可学习Prompt tokens，优点是参数量极小、显存需求少；

缺点是适合简单任务，在小规模模型上效果有限。

LoRA方法：利用低秩矩阵模拟权重更新，仅更新少量参数，优点为并行结构不影响推理效率、参数少且训练稳定、可插拔；缺点是对复杂或多模态任务表达能力可能有限。

随着大 语言模型 的普及，高效微调方法逐渐成为研究热点。尤其在监督微调过程中，如何选择合适的参数高效微调（Parameter-Efficient Fine-Tuning，PEFT）方法成为关注重点。本文将梳理SFT中常用的几种参数微调方法及各自的优缺点。

### 一、SFT中的参数 微调 方法

常见的参数微调方法主要分为两类：全参微调和PEFT方法。

全参微调：所有模型参数都参与微调，能够取得最佳性能，但参数量巨大、计算成本高，且易过拟合。

PEFT方法：仅更新部分参数，具体包括：

### 二、几种PEFT方法的优缺点

Adapter Tuning

Adapter Tuning方法通过在Transformer层中插入额外的串行结构（Adapter模块），固定主模型参数，仅更新Adapter模块参数。

优点：

缺点：

Prefix Tuning

Prefix Tuning是在Transformer的每一层前面增加一段任务相关的虚拟token作为前缀，仅更新Prefix参数。

优点：

缺点：

Prompt Tuning

Prompt Tuning仅在输入层加入少量的可学习Prompt tokens。

优点：

缺点：

LoRA（Low-Rank Adaptation）

LoRA利用低秩矩阵对模型的权重更新进行模拟，仅更新少量参数。

优点：

缺点：
