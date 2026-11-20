---
title: Doing
topic: AI Agent+后端精选八股文
sub_topic: 大模型基础篇
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/AxkEwVG8kibSjokIPVEc6aggnjo
source_path: AI Agent+后端精选八股文/大模型基础篇/Doing.md
questions: 1
chars: 1634
build_date: '2026-11-20'
images_removed: 1
---

# Doing

> 主题：AI Agent+后端精选八股文 › 大模型基础篇　｜　原始文件：AI Agent+后端精选八股文/大模型基础篇/Doing.md

## 题1：简易自注意力机制

要求：手写 Scaled Dot-Product Attention，含 Q/K/V 线性变换及 mask 处理。

核心代码（PyTorch风格，但底层可用矩阵乘法实现）：

```
import torch
import torch.nn.functional as F
import math
def scaled_dot_product_attention(Q, K, V, mask=None):
    # Q,K,V 形状: (batch, heads, seq_len, d_k)
    d_k = Q.size(-1)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
    if mask is not None:
        # mask中True位置置为极小值
        scores = scores.masked_fill(mask == 0, float('-inf'))
    attn = F.softmax(scores, dim=-1)
    output = torch.matmul(attn, V)
    return output, attn
class SimpleSelfAttention(torch.nn.Module):
    def __init__(self, d_model, n_heads):
        super().__init__()
        assert d_model % n_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads
        self.W_q = torch.nn.Linear(d_model, d_model)
        self.W_k = torch.nn.Linear(d_model, d_model)
        self.W_v = torch.nn.Linear(d_model, d_model)
        self.out_proj = torch.nn.Linear(d_model, d_model)
    def split_heads(self, x):
        batch, seq, _ = x.size()
        return x.view(batch, seq, self.n_heads, self.d_k).transpose(1, 2)
    def forward(self, x, mask=None):
        Q = self.split_heads(self.W_q(x))
        K = self.split_heads(self.W_k(x))
        V = self.split_heads(self.W_v(x))
        attn_out, _ = scaled_dot_product_attention(Q, K, V, mask)
        # 合并多头
        attn_out = attn_out.transpose(1, 2).contiguous().view(x.size(0), -1, self.d_model)
        return self.out_proj(attn_out)
```

关键点：mask 在解码器自回归时使用下三角，值为 -inf 让 softmax 归零；多头拆分与合并时注意维度变换。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
