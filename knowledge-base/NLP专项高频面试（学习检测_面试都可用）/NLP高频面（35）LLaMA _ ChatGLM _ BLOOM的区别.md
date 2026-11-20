---
title: NLP高频面（35）LLaMA _ ChatGLM _ BLOOM的区别
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（35）LLaMA _ ChatGLM _ BLOOM的区别.pdf
questions: 0
chars: 2821
build_date: '2026-11-20'
pages: 3
---

# NLP高频面（35）LLaMA _ ChatGLM _ BLOOM的区别

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（35）LLaMA _ ChatGLM _ BLOOM的区别.pdf

本文讨论了NLP高频面试中LLaMA、ChatGLM - 6B和BLOOM的区别，从训练数据、模型结构特点、Tokenizer特性、训练目标与效率等方面进行了分析。关键要点包括：

LLaMA ：由Meta开发，有7B、13B、33B和65B多个版本，小版本用约1万亿tokens训练，大版本用约1.4万亿tokens训练；采用causal decoder - only Transformer结构，有前置RMS Norm、SwiGLU激活函数、旋转位置编码等改进；使用SentencePiece tokenizer，词表32,000个tokens，对中文支持有限。

ChatGLM - 6B ：由智谱AI开发，训练语料约1万亿tokens，中英文各占50%；采用prefix decoder -only结构的Transformer，输入双向、输出单向注意力；有Embedding梯度缩减、后置Layer Norm、GeGLU激活函数、旋转位置编码等改进；训练目标为自回归文本填空，训练效率略低于causal结构。

BLOOM ：由BigScience项目开发，有多个参数规模版本，BLOOMZ系列针对英文场景，BLOOMZ -MT用于非英语场景；采用causal decoder - only Transformer结构，有Embedding层Layer Norm、pre layer norm、GeLU激活函数、相对位置编码ALiBi等改进；训练目标是语言模型任务，预测下一个词。

本文讨论了NLP高频面试中LLaMA、ChatGLM - 6B和BLOOM的区别，从训练数据、模型结构特点、Tokenizer特性、训练目标与效率等方面进行了分析。关键要点包括：

LLaMA ：由Meta开发，有7B、13B、33B和65B多个版本，小版本用约1万亿tokens训练，大版本用约1.4万亿tokens训练；采用causal decoder - only Transformer结构，有前置RMS Norm、SwiGLU激活函数、旋转位置编码等改进；使用SentencePiece tokenizer，词表32,000个tokens，对中文支持有限。

ChatGLM - 6B ：由智谱AI开发，训练语料约1万亿tokens，中英文各占50%；采用prefix decoder -only结构的Transformer，输入双向、输出单向注意力；有Embedding梯度缩减、后置Layer Norm、GeGLU激活函数、旋转位置编码等改进；训练目标为自回归文本填空，训练效率略低于causal结构。

BLOOM ：由BigScience项目开发，有多个参数规模版本，BLOOMZ系列针对英文场景，BLOOMZ -MT用于非英语场景；采用causal decoder - only Transformer结构，有Embedding层Layer Norm、pre layer norm、GeLU激活函数、相对位置编码ALiBi等改进；训练目标是语言模型任务，预测下一个词。

## 一、LLaMA

## 训练数据

LLaMA由Meta开发，拥有多个参数规模的版本：7B、13B、33B和65B。其中，较小的7B和13B版本采用了约1万亿tokens进行训练，而更大的33B和65B版本使用了约1.4万亿tokens进行训练。

## 模型结构特点

LLaMA采用与 GPT 类似的causal decoder-only Transformer结构，在细节上进行了一些重要改进：

Layer Normalization：采用了前置的RMS Norm（均方根Norm）以提高训练稳定性，并去除了传统的layer norm中的偏置项。

激活函数：选用了SwiGLU激活函数，相比传统FFN多出一个权重矩阵，提升了模型的表达能力。

位置编码：使用了旋转位置编码（RoPE），去除了传统的绝对位置编码，更好地处理序列长度变化。

## Tokenizer 特性

LLaMA主要以英文数据训练，使用SentencePiece tokenizer，词表仅有32,000个tokens，对中文支持有限， 中文编码 效率较低。

## 二、ChatGLM-6B

## 训练数据

ChatGLM-6B由智谱AI开发，训练语料包含约1万亿tokens，中文与英文数据比例各占50%，相比GLM-130B（4000亿tokens），数据量更为丰富。

## 模型结构特点

ChatGLM采用了prefix decoder-only结构的Transformer，与GPT或LLaMA的纯causal结构不同，输入部分为双向注意力，输出部分则采用单向注意力。此外还包括以下具体改进：

Embedding梯度缩减：为了提升训练稳定性，将Embedding层的梯度缩小约10倍。

Layer Normalization：采用了基于Deep Norm的后置Layer Norm，有效提升了训练稳定性。

激活函数：使用了GeGLU激活函数，这种门控结构增加了权重矩阵，提升了模型的表达能力。

位置编码：与LLaMA一样，ChatGLM采用了旋转位置编码RoPE。

## 训练目标与效率

ChatGLM的训练目标为自回归文本填空。由于prefix decoder结构只在输出端计算损失，相比causaldecoder-only结构，实际利用的token数量较少，因此在相同token数下训练效率略低于causal结构。

## 三、BLOOM

## 训练数据

BLOOM由BigScience项目开发，有多个参数规模版本，包括560M、1.1B、1.7B、3B、7.1B及176B。

其中，BLOOMZ系列主要针对英文场景微调，而BLOOMZ-MT则用于非英语场景。

## 模型结构特点

BLOOM同样采用了causal decoder-only Transformer结构，与GPT风格类似，但在细节上也有所创新：

Embedding层Layer Norm：在Embedding层后增加了Layer Norm，有效提升训练的稳定性。

Layer Normalization：类似LLaMA，使用了pre layer norm而非传统的后置方法。

激活函数：采用了GeLU激活函数，这是一种广泛应用且表现稳定的激活方式。

位置编码：使用相对位置编码ALiBi（Attention with Linear Biases），相比绝对位置编码，ALiBi更适合长序列推理，能够处理超过训练时长度的输入序列。

## 训练目标

BLOOM的训练目标与LLaMA类似，即语言模型任务，通过给定上文预测下一个词。

## 总结与对比
