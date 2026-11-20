# 循环神经网络 (RNN)

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/H9klwWRiMic1XAkzqs5csdQ1nvg

#### 1. 普通 RNN 的前向传播公式是什么？隐藏状态是如何更新的？

![image](../../../assets/PPRxdIN52oP2ILxlwXvcQ7slnY5/NMfPbdyDboy0IRxaecdcSLaZnBT.png)

基本 RNN 的前向传播

![image](../../../assets/PPRxdIN52oP2ILxlwXvcQ7slnY5/WR54bUyILog0Rax5y4ZcOkKGnzh.png)

隐藏状态更新

![image](../../../assets/PPRxdIN52oP2ILxlwXvcQ7slnY5/RSsrbOMrloSoDLx06MbchdUPnQb.png)

深层 RNN：可以堆叠多个 RNN 层，第 l*l* 层的隐藏状态公式为：

![image](../../../assets/PPRxdIN52oP2ILxlwXvcQ7slnY5/K6rCbvFQGoidLvxjwducj6WOnIQ.png)

---

#### 2. RNN 为什么能处理变长序列？与 CNN 处理序列的方式有何不同？

RNN 处理变长序列的原理

![image](../../../assets/PPRxdIN52oP2ILxlwXvcQ7slnY5/EO9Eb5r5koz9xHxRCdVcj9RlnUh.png)

入。不论序列多长，模型均按时间顺序逐步处理，因此天然适配变长序列。在实现时，只需在输入时按实际长度展开计算图，序列结束后停止更新隐藏状态。对于变长 batch，通过 padding 和 masking 忽略无效时间步即可。

与 CNN 处理序列的不同

- 处理机制：
  - RNN：串行循环处理，每个时间步的隐藏状态依赖于前一步。本质是时间维度上的“动态系统”，适合序列依赖建模。
  - CNN：在序列上应用一维卷积，利用固定大小的滑动窗口并行计算每个位置的局部上下文特征。它不显式维护贯穿序列的隐藏状态，感受野由卷积核大小和层数决定。为了捕捉长距离依赖，需要堆叠多层或使用空洞卷积来扩大感受野。
- 并行性：
  - RNN 的递归特性导致时间步之间必须串行计算，难以并行化（但 batch 内不同样本可并行）。
  - CNN 在整个序列上可以完全并行计算，训练和推理速度远快于 RNN。
- 长距离依赖：
  - RNN 理论上可以记忆无限长的历史，但实际受梯度消失影响，只能捕获有限的上下文。
  - CNN 长距离依赖需要很深的网络或特殊设计（如膨胀卷积），否则感受野受限。
