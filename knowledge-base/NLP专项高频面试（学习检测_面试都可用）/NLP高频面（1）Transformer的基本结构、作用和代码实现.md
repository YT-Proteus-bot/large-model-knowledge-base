---
title: NLP高频面（1）Transformer的基本结构、作用和代码实现
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（1）Transformer的基本结构、作用和代码实现.pdf
questions: 0
chars: 2795
build_date: '2026-11-20'
pages: 4
---

# NLP高频面（1）Transformer的基本结构、作用和代码实现

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（1）Transformer的基本结构、作用和代码实现.pdf

本文讨论了NLP高频面试中Transformer的基本结构、作用和代码实现，涵盖简洁版和详细版面试回答及PyTorch代码示例。关键要点包括：

模型组件：Transformer模型主要由嵌入层、编码器、解码器（若存在）和输出层组成。嵌入层包括Token、Positional和（可选）Segment Embedding；编码器由多头自注意力和前馈神经网络子层组成；解码器有掩码多头自注意力、编码 - 解码多头注意力和前馈神经网络；输出层根据任务设计。

Attention机制：用于动态聚焦序列中重要信息，计算步骤为计算Query、Key和Value，计算注意力权重，加权求和，克服了传统RNN处理长序列时的问题。

常见结构：有Encoder - only（如BERT类）、Decoder - only（如GPT类）和Encoder - Decoder（如原始Transformer、T5等）三种。

前馈神经网络：由两个线性变换层和非线性激活函数组成，用于增强特征表示能力。

代码实现：给出了Transformer模型的PyTorch代码，包括PositionalEncoding、MultiHeadAttention、FeedForward、EncoderLayer、DecoderLayer和Transformer类。

测试示例：代码中包含测试示例，定义了源和目标词汇表大小、序列长度，创建模型并输入随机数据进行测试。

残差与归一化：编码器和解码器的每个子层都配备残差连接和层归一化，有助于信息传递和训练稳定性。

## 面试回答简洁版

Transformer模型 主要由以下组件组成：

嵌入层（Embedding Layer）

编码器（Encoder）

解码器（Decoder，若存在）

输出层（Output Layer）

Attention机制 ：动态聚焦序列中重要信息，通过计算Query、Key、Value实现：

常见Transformer结构 ：

Encoder-only (BERT类)：Embedding → Encoder → 输出

Decoder-only (GPT类)：Embedding → Decoder → 输出

Encoder-Decoder (原始Transformer, T5等)：Embedding → Encoder → Decoder → 输出

## 面试准备细节版

## 基本结构

Transformers的模型结构主要由以下几个部分组成：

## 1.1 嵌入层（Embedding Layer）

Token Embedding：将词或子词(token)转换为对应的词向量（embedding）。

Positional Embedding：为输入的token添加位置信息，以帮助模型理解序列顺序。

（可选）Segment Embedding（如BERT）：区分不同的句子或段落。

## 1.2 编码器 （Encoder）

通常由多层相同的模块堆叠而成，每一层包含两个关键子层：

每个子层都伴随：

1.2.1 什么是Attention？（可以用于回答attention or transformer结构问题的前面用于综述性介绍）

Attention机制最初用于序列建模任务，尤其适合处理输入序列长度较长时的信息提取与整合问题。它允许模型动态地关注输入序列中与当前预测目标最相关的信息。

Attention克服了传统循环神经网络（RNN）处理长序列时遗忘和梯度消失的问题，通过显式建模序列内部的依赖关系，实现了信息的高效传递。

1.2.2 Attention的计算过程Attention的计算步骤如下：

计算Query、Key和Value：通过三个不同的线性变换得到查询（Query）、键（Key）和值（Value）。【三个线性变换可以增加模型的表达能力】

计算注意力权重：通过Query与Key的内积计算相似度，并经过softmax函数归一化，得到各位置的权重。【这里有一个缩放的操作，目的也是为了归一化，防止梯度爆炸】

加权求和：利用注意力权重对Value进行加权求和，得到注意力的输出。

Attention的具体计算公式为：

1.2.3 前馈神经网络（FFN）

FFN由两个线性变换层和一个非线性激活函数组成，通常用于进一步增强Transformer的特征表示能力。

这里的activation为激活函数，后续很多模型对这个激活函数有一定的改进。

## 1.3 解码器（Decoder，若存在）

同样由多个相同的模块堆叠，每层包括三个关键子层：

同样包括残差连接与层归一化。

注意：

## 1.4 输出层（Output Layer）

任务相关的输出，如分类任务的全连接分类层、生成任务中的线性层+softmax层、命名实体识别中的序列标注层等。

综上所述，一个完整的Transformer模型结构可以概括为：

代码块Input → Embedding Layer → Encoder → (Decoder) → Output Layer → Prediction

Encoder-only (如BERT)：

代码块Input → Embedding Layer → Encoder → Output Layer

Decoder-only (如GPT)：

代码块Input → Embedding Layer → Decoder → Output Layer

Encoder-Decoder (如T5、Transformer原论文模型)：

代码块Input → Embedding Layer → Encoder → Decoder → Output Layer

下面提供一个清晰易懂的Transformer模型PyTorch代码实现示例，包含了关键组件，如嵌入层、多头自注意力、位置编码、前馈网络、编码器和解码器：

Transformer PyTorch 实现

目录结构：

代码块Transformer├── PositionalEncoding├── MultiHeadAttention├── FeedForward├── EncoderLayer├── DecoderLayer├── Transformer

完整代码示例

代码块import torchimport torch.nn as nnimport math

# Positional Encodingclass PositionalEncoding(nn.Module):

def __init__(self, d_model, max_len=5000):
