---
title: NLP高频面（7）GPT和Bert的mask有什么区别？ (2)
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（7）GPT和Bert的mask有什么区别？ (2).pdf
questions: 0
chars: 1821
build_date: '2026-11-20'
pages: 2
---

# NLP高频面（7）GPT和Bert的mask有什么区别？ (2)

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（7）GPT和Bert的mask有什么区别？ (2).pdf

本文讨论了NLP领域中GPT和BERT两种预训练语言模型的Mask机制，深入探讨了二者的核心区别及优化策略。关键要点包括：

BERT的Mask机制基于MLM：BERT采用Masked Language Model进行训练，随机Mask部分token，让模型结合上下文预测。

BERT的Token级Mask：随机选择一定比例的token用[MASK]替换，如 “I love [MASK] dogs.” ，模型需结合上下文预测原词。

BERT的子词级Mask：采用BPE或WordPiece分词，一个单词可能拆分成多个子词，如 “running” 拆成 [“run”, “##ning”]。

BERT的Mask预测优化：提出Span Mask增加任务难度，中文训练有命名实体Mask，提升实体识别能力。

GPT的Mask机制基于CLM：采用自回归语言模型，通过前面的词预测下一个词，不使用[MASK]。

GPT的自回归Mask：Transformer计算注意力时用 上三角Mask，屏蔽当前token之后的所有token，单向（从左到右）遮蔽。

GPT Mask方式的优缺点：适用于文本生成任务，但缺乏双向信息，对上下文理解不如BERT深入。

## GPT 和 BERT 的 Mask 机制对比：核心区别与优化策略

在 NLP 领域，GPT 和 BERT 是最具代表性的预训练语言模型之一。它们都在训练过程中使用了 Mask机制来引导模型学习语言表示，但具体实现方式和目标却有所不同。本文将深入探讨 GPT 和 BERT 的Mask 方法的核心区别，并分析其优化策略。

## 1. BERT 的 Mask 机制：基于 MLM（Masked Language Model）

BERT（Bidirectional Encoder Representations from Transformers）采用 （Masked LanguageModel） 进行训练，即在输入文本中随机 Mask 掉部分 token，让模型在上下文中预测这些被 Mask 掉的 token。

BERT 的 Mask 方式具有以下特点：

Token 级别的 Mask：BERT 会随机选择一定比例的 token 并用 [MASK] 进行替换，例如：

“I love [MASK] dogs.”这里的 my 可能会被 Mask 掉，模型需要结合上下文来预测原始词。

基于子词（Subword）级别的 Mask：BERT 采用 BPE（Byte-Pair Encoding） 或 WordPiece 进行分词，一个单词可能被拆分成多个子词（subword）。例如：

“running” → [“run”, “##ning”]如果只 Mask 掉 “##ning”，模型很容易猜测出原单词是“running”。

Mask 预测的优化：

## GPT 的 Mask 机制：基于 CLM（Causal Language Model）

与 BERT 不同，GPT（Generative Pre-trained Transformer）采用的是 自回归语言模型（CausalLanguage Model, CLM） ，其训练目标是通过前面的词预测下一个词，而不是填补 Mask 掉的部分。

因此，GPT 并不使用 [MASK] ，而是通过 自回归（Autoregressive） 方式进行训练，即：

“I love my” → 预测 “dog”

GPT 训练时，模型只能看到当前词及其之前的词，而不能看到未来的词，因此它的 Mask 机制表现为单向（从左到右）遮蔽 ：

自回归 Mask（Causal Mask）：在 Transformer 计算注意力时，GPT 采用一个 上三角 Mask，即屏蔽掉当前 token 之后的所有 token，只能依赖前面的词信息进行预测。

无显式 Mask Token：GPT 没有 [MASK]，而是通过连续生成下一个词的方式进行学习，并应用于文本生成任务，如 OpenAI 的 ChatGPT。

这种 Mask 方式的主要优点是适用于 文本生成任务 ，如对话生成、自动写作等，但缺点是缺乏双向信息，可能导致对上下文的理解不如 BERT 深入。

## GPT vs. BERT 的 Mask 机制对比
