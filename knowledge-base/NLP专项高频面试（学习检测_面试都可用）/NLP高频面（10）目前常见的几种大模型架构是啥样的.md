---
title: NLP高频面（10）目前常见的几种大模型架构是啥样的
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（10）目前常见的几种大模型架构是啥样的.pdf
questions: 3
chars: 2912
build_date: '2026-11-20'
pages: 3
---

# NLP高频面（10）目前常见的几种大模型架构是啥样的

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（10）目前常见的几种大模型架构是啥样的.pdf

本文讨论了随着 Transformer 模型的发展，语言大模型成为人工智能领域热点的背景下，对几种常见大模型架构进行解析的内容。关键要点包括：

LLM 定义：通常指参数量巨大、能捕捉丰富语义信息的 Transformer 模型，可实现文本生成、语言理解任务。

主流开源大模型架构：GPT 系列（Decoder - only）、BERT 系列（Encoder - only）、T5 系列（Encoder - Decoder）、GLM 系列（Prefix LM）、LLama 系列（Decoder - only）。

GPT 架构特点及场景：采用自回归方式，Decoder - only 结构，单向因果注意力，擅长生成任务，适用于文本生成等场景，代表有 GPT - 3、GPT - 4 等。

BERT 架构特点及场景：基于编码器，双向注意力预训练，适合自然语言理解任务，代表有 BERT、RoBERTa 等。

T5 架构特点及场景：Encoder - Decoder 架构，Text - to - Text 预训练，适合翻译、摘要生成等任务，代表有 T5、BART。

Decoder - only 成主流原因：Zero - shot 表现优，无需大量标注数据；效率高，适合长文本和多轮对话；避免 Encoder 低秩问题。

模型选择建议：自然语言理解选 BERT；条件生成或翻译选 T5 或 BART；高效文本生成选 GPT 或LLaMA；混合场景选 GLM 或 ChatGLM。

### 深入浅出：目前常见的几种大模型架构解析

随着 Transformer模型 的提出与发展，语言大模型迅速崛起，已经成为人工智能领域最为关注的热点之一。本文将为大家详细解析几种目前常见的大模型架构，帮助读者理解其核心差异及适用场景。

## 题1：什么是LLM（大语言模型）？

LLM通常指参数量巨大、能够捕捉丰富语义信息的Transformer模型，它们通过海量的文本数据训练而成，能够实现高度逼真的 文本生成 、复杂的语言理解任务。

目前主流的开源大模型架构主要有以下几种：

GPT系列：Decoder-only

BERT系列：Encoder-only

T5系列：Encoder-Decoder

GLM系列：Prefix LM（部分双向注意力+单向生成）

LLama系列：Decoder-only

下面我们将逐一分析这些模型架构的具体特点与使用场景。

### 常见的LLM架构详解

2.1 GPT架构（Decoder-only）

GPT（Generative Pre-trained Transformer）采用了自回归（Autoregressive）方式，使用Decoder-only结构。每次生成的token仅依赖于此前的token序列，而无法看到未来的信息。

特点：

代表模型：GPT-3、GPT-4、LLaMA系列

适用场景：文本生成、聊天机器人、创意内容生成

2.2 BERT架构（Encoder-only）

BERT（Bidirectional Encoder Representations from Transformers）是完全基于编码器（Encoder）的架构，采用双向注意力机制进行预训练。

特点：

代表模型：BERT、RoBERTa、ERNIE

适用场景：自然语言理解（NLU）任务，如分类、实体识别、问答系统

2.3 T5架构（Encoder-Decoder）

T5（Text-to-Text Transfer Transformer）采用标准的Encoder-Decoder架构，进行Text-to-Text的预训练。模型的输入和输出均为文本形式，训练目标灵活。

特点：

代表模型：T5、BART

适用场景：翻译、摘要生成、条件文本生成

2.4 GLM架构（Prefix LM）

GLM（General Language Model）是一种Prefix LM结构，它结合了自编码（AE）和自回归（AR）的优点，在输入部分采用双向注意力，生成预测部分则采用单向因果注意力。

特点：

代表模型：GLM、ChatGLM

适用场景：问答系统、对话机器人、条件生成任务

在您的博客中，您可以添加对 LLaMA 系列模型和 DeepSeek 模型架构的介绍，以丰富内容并提供读者对最新 AI 模型的了解。

2.5 LLaMA 系列模型

LLaMA（Large Language Model Meta AI）是由 Meta（前 Facebook）开发的一系列大型语言模型，旨在提高自然语言处理任务的性能。自 2023 年 2 月发布以来，LLaMA 系列经历了多次迭代，最新版本为 LLaMA 3。这些模型的架构主要基于 Transformer，但在以下方面进行了改进：

前置归一化（Pre-normalization）：采用 RMSNorm 归一化函数，提升训练稳定性。

激活函数：使用 SwiGLU 激活函数，替代传统的 ReLU，以提高模型性能。

位置编码：采用旋转位置嵌入（RoPE），增强模型对序列数据的处理能力。

LLaMA 3 在上下文长度、参数规模和训练数据量等方面均有显著提升，支持多达 30 种语言，展示了卓越的性能和广泛的适用性。

2.6 DeepSeek 模型架构DeepSeek 是一家 AI 初创公司，因其高效且性能卓越的模型而备受关注。其最新版本 DeepSeek-V3 采用了混合专家（Mixture-of-Experts，MoE）架构，拥有 6710 亿个参数，每个标记激活 370 亿个参数。这种架构通过稀疏激活和动态路由机制，实现了模型容量的高效扩展，同时平衡了计算效率与模型性能。此外，DeepSeek-R1 模型在数学和推理基准测试中表现出色，展示了其在复杂任务中的强大能力。

将这些内容添加到您的博客中，可以为读者提供对当前领先 AI 模型架构的深入了解，展示这些模型在技术上的创新和实际应用中的表现。

## 题2：为什么Decoder-only架构成为主流？

目前，以GPT为代表的Decoder-only结构在LLM领域占据主导地位，主要原因如下：

更优的Zero-shot表现：无需大量标注数据或任务特定微调即可表现出很强的泛化能力。

效率更高：Decoder-only结构易于支持KV-cache复用，尤其适合处理长文本和多轮对话场景。

避免Encoder的低秩问题：Encoder的双向注意力可能带来低秩问题，限制了模型的生成表现。

## 题3：各类模型如何选择？

自然语言理解（NLU）任务：选择BERT（Encoder-only）

条件生成或翻译任务：选择T5或BART（Encoder-Decoder）

高效文本生成、聊天机器人：选择GPT或LLaMA（Decoder-only）

混合场景或对话任务：选择GLM或ChatGLM（Prefix LM）
