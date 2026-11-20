# 百度AI Agent开发面试题集

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/OERYwng0viix75k3TdUcUj4envf

### 一、 大模型底层原理与训练优化（5题）

#### 1. 请详细推导Transformer中Self-Attention的复杂度。如果序列长度从2k扩展到1M，在Attention计算上会面临什么挑战？有哪些优化方案？

答案要点：

- 复杂度推导：Self-Attention的计算包含Q·K^T（复杂度O(n²·d)）和Softmax（复杂度O(n²)），总复杂度O(n²·d)，其中n为序列长度，d为隐层维度。
- 挑战：n=1M时，n²达到10¹²量级，显存和计算时间都无法接受。
- 优化方案：
  - 稀疏Attention：如Longformer、BigBird，将全连接改为局部窗口+全局token
  - 线性Attention：如Performer，通过核方法将复杂度降至O(n·d²)
  - FlashAttention：通过分块计算和重计算，在不改变结果的前提下降低显存访问次数
  - KV-cache优化：在生成阶段缓存KV，避免重复计算

#### 2. RLHF中的PPO算法存在哪些训练不稳定的问题？DPO是如何从数学上规避这些问题的？

答案要点：

- PPO的不稳定性：
  - 奖励模型（RM）与策略模型分布不匹配，导致奖励信号偏移
  - 需要同时维护多个模型（策略、参考策略、价值函数、奖励模型），资源消耗大
  - 优势函数估计的方差高，需要大量超参数调优
- DPO的核心数学原理：
  - DPO利用Bradley-Terry模型，将偏好概率表示为：
  - p(y1≻y2)=σ(βlog⁡πθ(y1∣x)πref(y1∣x)−βlog⁡πθ(y2∣x)πref(y2∣x))*p*(*y*1≻*y*2)=*σ*(*β*log*π*ref(*y*1∣*x*)*πθ*(*y*1∣*x*)−*β*log*π*ref(*y*2∣*x*)*πθ*(*y*2∣*x*))
  - 直接通过最大化偏好数据集的似然来优化策略，无需显式奖励模型
  - 损失函数为：
  - LDPO(θ)=−E(x,yw,yl)∼D[log⁡σ(βlog⁡πθ(yw∣x)πref(yw∣x)−βlog⁡πθ(yl∣x)πref(yl∣x))]LDPO(*θ*)=−E(*x*,*yw*,*yl*)∼*D*[log*σ*(*β*log*π*ref(*yw*∣*x*)*πθ*(*yw*∣*x*)−*β*log*π*ref(*yl*∣*x*)*πθ*(*yl*∣*x*))]
  - 优势：训练稳定、只需2个模型、收敛速度更快

#### 3. 什么是Mamba？它与Transformer相比在序列建模上有什么本质区别？在Agent的长期记忆场景中谁更有优势？

答案要点：

- Mamba的核心：基于结构化状态空间模型（SSM），将序列建模视为线性状态方程：
- ht=Aˉht−1+Bˉxt,yt=Cht*ht*=*A*ˉ*ht*−1+*B*ˉ*xt*,*yt*=*Cht*
- 其中Aˉ*A*ˉ、Bˉ*B*ˉ是输入相关的参数，实现选择性状态空间
- 与Transformer的区别：
  - Transformer：Attention机制，O(n²)复杂度，显式建模所有token交互
  - Mamba：线性复杂度O(n)，通过状态传递隐式建模长程依赖
- 在Agent长期记忆场景：
  - Mamba优势：处理百万级token时显存和速度优势明显
  - Transformer优势：上下文窗口内可灵活回溯任意位置，对于需要精确检索的场景更优
  - 实际中常采用混合架构：短期交互用Attention，长期记忆用SSM

#### 4. MoE（混合专家模型）在训练和推理阶段分别有哪些工程挑战？如何解决负载不均衡问题？

答案要点：

- 训练挑战：
  - 负载不均衡：少数专家被频繁激活，多数专家闲置
  - 通信开销：All-to-All通信在大规模集群上成为瓶颈
  - 路由崩溃：路由器可能收敛到将所有token分配给同一个专家
- 解决方案：
  - 负载均衡损失：添加辅助损失鼓励专家使用率均衡
  - Lbalance=α⋅∑i=1Efi⋅piLbalance=*α*⋅*i*=1∑*Efi*⋅*pi*
  - 其中fi*fi*是专家i被选中的频率，pi*pi*是路由概率
  - 专家容量限制：限制每个专家处理的token数量，超出部分残差连接
  - 随机路由：训练时加入噪声，避免确定性过拟合
  - 推理优化：使用专家并行+张量并行混合策略，预取专家参数到本地显存

#### 5. 请推导LoRA的数学原理。为什么在微调Agent时，LoRA往往比全参数微调效果更好？

答案要点：
