---
title: NLP高频面（17）什么是KV Cache
topic: NLP专项高频面试（学习检测_面试都可用）
sub_topic: ''
doc_type: pdf
source_url: ''
source_path: NLP专项高频面试（学习检测_面试都可用）/NLP高频面（17）什么是KV Cache.pdf
questions: 2
chars: 3475
build_date: '2026-11-20'
pages: 4
---

# NLP高频面（17）什么是KV Cache

> 主题：NLP专项高频面试（学习检测_面试都可用）　｜　原始文件：NLP专项高频面试（学习检测_面试都可用）/NLP高频面（17）什么是KV Cache.pdf

本文讨论了在大语言模型领域，随着模型参数增多和输入上下文增加，推理计算量和显存消耗显著增加的背景下，KV Cache 作为重要优化技术的相关内容，包括其原理、优势、实现、优化方案及面临的挑战等。关键要点包括：

KV Cache 定义：KV Cache 即 Key-Value Cache，是 Transformer 模型推理时为减少重复计算、降低内存开销的缓存机制，存储已计算好的 Key 和 Value 供复用。

使用原因：大模型推理分预填充和解码阶段，解码阶段逐词生成新 token 时，使用 KV Cache 可避免重复计算，降低内存带宽需求和计算成本，提升推理速度。

实现原理：在 Transformer 模型的每个 self-attention 层存储过去的 Key 和 Value，生成新 token时，计算当前 token 对应的 Key 和 Value 并追加到缓存后。

MQA 优化：多查询注意力（MQA）中所有头共享一组 Key 和 Value 矩阵，缓存的 KV 体积显著降低，但易损失精度，需额外训练优化。

GQA 优化：分组查询注意力（GQA）是 MHA 和 MQA 的折中方案，将注意力头分组，组内头共享 Key和 Value，能平衡内存效率和模型精度，如 Llama2 70B 模型采用。

内存占用挑战：KV Cache 在推理阶段加速明显，但批量推理和长序列时内存占用大。

优化策略：有分页缓存、Flash Attention、量化和稀疏化等策略，可减少碎片、提高内存利用率、降低存取开销和内存占用。

在当今火热的大语言模型领域，模型的参数动辄数十亿甚至上千亿，随着输入的上下文（token长度）

增加，推理过程中的计算量和显存消耗都会显著增加。其中， KV Cache 是大模型推理过程中的一种重要优化技术。

本文将围绕 KV Cache 详细展开，帮助你深入理解这个关键技术的原理、优势以及相关的优化方案。

## 题1：一、什么是 KV Cache？

KV Cache，全称为 Key-Value Cache，是在 Transformer模型 推理过程中，为减少重复计算、降低内存开销而设计的一种缓存机制。具体来说：

Transformer 模型中，每生成一个新词（token）时，都需要计算该词与前面所有词之间的注意力（attention）。

注意力计算涉及 Query(Q)、Key(K) 和 Value(V) 三个张量，其中 Key 和 Value 对于已生成的 token 是不变的，只有 Query 会随每次生成而更新。

KV Cache 就是将这些已经计算好的 Key 和 Value 存储起来，供下一次生成 token 时直接复用，而不必重复计算。

这种 缓存机制 极大提升了推理效率，尤其在长序列的自回归生成场景中非常有效。

## 题2：二、为什么需要 KV Cache？

以大模型的推理过程为例，我们一般分为两个阶段：

预填充阶段（Prefill stage）模型处理整个输入序列，这个阶段高度并行化，利用 GPU 效率高。

解码阶段（Decode stage）模型逐个生成新 token，每生成一个 token，都需要与之前所有 token 进行注意力计算，这个阶段效率较低。

在解码阶段，如果每次生成新 token 都重新计算 Key 和 Value，将产生大量冗余计算，推理效率极低。通过使用 KV Cache，模型可以避免重复计算，大大降低了内存带宽需求和计算成本，显著提升推理速度。

### 三、KV Cache 的实现原理（附代码示例）

KV Cache 的实现非常简单，其核心思想是：

在 Transformer 模型的每个 self-attention 层存储过去的 Key 和 Value。

当生成新 token 时，只需计算当前 token 对应的 Key 和 Value，并将它们追加到过去缓存的 Key 和Value 后面。

下面是一段使用 PyTorch 实现的简化示例：

代码块import torch

# 假设 key_states 和 value_states 是当前 token 的计算结果# past_key_value 是缓存的历史 Key 和 Value

def update_kv_cache(past_key_value, key_states, value_states, use_cache=True):

if past_key_value is not None:

# 将历史的 Key 和当前的 Key 拼接key_states = torch.cat([past_key_value[0], key_states], dim=-2)

value_states = torch.cat([past_key_value[1], value_states], dim=-2)

# 更新缓存past_key_value = (key_states, value_states) if use_cache else Nonereturn past_key_value

这里的 past_key_value 通常是一个元组 (key_states, value_states) ，每个 tensor 的形状一般为 (batch_size, num_heads, seq_len, head_dim) 。

值得注意的是， Query 并不缓存 ，因为每次推理只关心最新生成的那个 token，因此每次只需用最新的 Query 向量即可。

### 四、优化 KV Cache：MQA 与 GQA

随着 KV Cache 应用的广泛，一些变种注意力机制诞生，例如：

### （1）多查询注意力 (MQA, Multi-Query Attention)

MQA 中所有的头共享一组 Key 和 Value 矩阵，相比传统的 MHA (Multi-Head Attention)，缓存的 KV体积显著降低：

MHA 缓存大小：(num_heads, seq_len, head_dim)

MQA 缓存大小：仅需 (seq_len, head_dim)

这种方式虽然有效节约了缓存，但容易损失精度，通常需要额外的训练优化。

### （2）分组查询注意力 (GQA, Grouped Query Attention)

GQA 是 MHA 和 MQA 的一种折中方案：

将注意力头分成若干组，每组内头共享一份 Key 和 Value。

GQA-N 表示头数被分成 N 组，GQA-1 即为 MQA，GQA-头数即为传统 MHA。

GQA 能够在内存效率和模型精度之间取得平衡，像知名的 Llama2 70B 模型正是采用了 GQA。

GQA实现示例：

代码块def repeat_kv(x: torch.Tensor, n_rep: int) -> torch.Tensor:

bs, slen, n_kv_heads, head_dim = x.shapeif n_rep == 1:

return xreturn (x[:, :, :, None, :].expand(bs, slen, n_kv_heads, n_rep, head_dim)

.reshape(bs, slen, n_kv_heads * n_rep, head_dim)

这段代码的核心作用就是将少数几个 Key 和 Value 头扩展到更多的头，实现分组共享。

### 五、KV Cache 的挑战和进一步优化

虽然 KV Cache 在推理阶段提供了显著的加速，但其内存占用仍然较大，尤其是在批量推理和长序列情况下。

为了解决这一挑战，目前已有多种策略：

分页缓存（Paged KV Cache）仿照操作系统分页机制，将 KV 缓存分成固定大小的块，动态分配和释放，从而减少碎片、提高内存利用率。

Flash Attention通过重新安排注意力计算顺序，将多次内存读写优化为一次性处理，极大提高了 GPU利用效率，降低 KV Cache 存取开销。

量化（Quantization）和稀疏化（Sparsity）对 KV 缓存做低精度量化或稀疏表示，进一步降低内存占用。
