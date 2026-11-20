# Transformer 与大型语言模型

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/CJfrwNOhFip8O6kTe57cTXq7n8e

#### 1. Transformer 的注意力机制公式，为什么要缩放？自注意力的计算复杂度。

注意力机制公式
Transformer的核心是缩放点积注意力（Scaled Dot-Product Attention）。给定查询（Query）矩阵 Q、键（Key）矩阵 *K* 和值（Value）矩阵 *V*，注意力输出为：

![image](../../../assets/Ec1qdDcUGoWAIexY0v4cofY8nwb/TiAgbnXlfoHxIexSCVkcXpnZn9d.png)

**为什么要缩放？**

![image](../../../assets/Ec1qdDcUGoWAIexY0v4cofY8nwb/UKsvbrTv2okWdixxsUDcgwhtn6k.png)

![image](../../../assets/Ec1qdDcUGoWAIexY0v4cofY8nwb/AScXbxjgUoOaLSxzWdycvgMWnYc.png)

---

#### 2. BERT 和 GPT 的预训练目标分别是什么？Masked LM vs Autoregressive LM。

BERT：Masked Language Model (MLM)

- 预训练目标：随机遮盖输入序列中的一部分词（如15%），让模型预测被遮盖的词。训练目标是复原被遮盖的词，利用上下文双向信息。
- 具体操作：在输入句子中，对选中的token：
  - 80%概率替换为特殊标记 [MASK]。
  - 10%概率替换为随机词。
  - 10%概率保持不变（但仍需预测）。
    这种策略缓解了预训练和微调时 [MASK] 标记不匹配的问题。

- 双向建模：BERT的Transformer编码器可以同时看到被遮盖词左右两边的上下文，因此能融合双向信息，对理解类任务（如文本分类、问答、命名实体识别）非常有效。
- 辅助任务：除MLM外，BERT还使用下一句预测（NSP），判断句子B是否是句子A的下一句，以学习句子间关系。后续变体（如RoBERTa）移除了NSP，仅靠MLM即可。

GPT：Autoregressive Language Model (AR LM)

![image](../../../assets/Ec1qdDcUGoWAIexY0v4cofY8nwb/J24ybd1Gto0UPdxzSN3crgA3nic.png)

- 单向建模：GPT使用Transformer解码器，带有因果掩码（Causal Mask），确保每个位置只能看见自身和之前的token，不能看未来信息。这使得GPT天生适合文本生成任务（如对话、续写、翻译）。
- 自回归生成：推理时，模型逐词生成，每一步将已生成的词作为上下文预测下一个词，直到输出结束符。

对比：

> （此处为内嵌电子表格，需登录飞书查看）
