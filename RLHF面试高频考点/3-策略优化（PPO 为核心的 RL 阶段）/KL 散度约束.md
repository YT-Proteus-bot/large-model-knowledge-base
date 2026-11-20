# KL 散度约束

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/BAN9wgGBJiVSTWkpMHXcKrxFn7f

#### 1. 详细推导 RLHF 中带 KL 惩罚的修改奖励函数：R_total = R_RM - β * KL(π_θ || π_ref)

1.1 从原始优化目标谈起

RLHF 的终极目标是训练一个语言模型策略 πθ*πθ*，使其能生成符合人类偏好的回答。最直接的想法是最大化奖励模型的期望评分：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/MTHPbu5l6oYYF7xIB14ctdXGnEo.png)

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/SKxHbmOm9osYY3xQ1AZcEjp3nfh.png)

**1.3 转化为无约束的拉格朗日对偶问题**

使用拉格朗日乘子法，我们将约束优化转化为无约束的极大化问题。引入拉格朗日乘子 β>0，得到：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/STc7bKSLQowb4Wx93fKcRuU2noc.png)

由于 *δ* 是常数，在优化时不影响梯度，因此等价于最大化：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/WmwrbGUD8oD2GqxCWZTcUSnynKh.png)

这里的 *β* 就扮演了 KL 惩罚系数的角色。它控制着约束的强度：β*β* 越大，惩罚越强，策略越不敢偏离参考模型；*β* 越小，惩罚越弱，策略越倾向于最大化奖励而可能远离参考模型。

**1.4 展开到 token 级别的强化学习**

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/YJ6lb4Le8od0shxiE4ochg2BnPh.png)

但在实际 PPO 训练中，由于我们已经在采样阶段获得了具体的 token 序列，我们可以使用该序列下每个 token 的对数概率来近似 KL 散度。于是，对于每个 token *at*，定义即时 KL 惩罚：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/J4SfbBSw7oVLlNxrONaciUCvnJb.png)

注意这里是对数比率，不是真正的 KL 散度（KL 需要加期望），但在给定样本下，它提供了无偏的梯度估计。因此，token 级别的总即时奖励变为：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/N9rCbXsWrooeFWxvXxCcAp3Pnbg.png)

**1.5 推导总结**

因此，最终在 PPO 中使用的修改奖励函数为：

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/BkVxbcmV4oeim4xCVDAccjWpn1b.png)

其中 KL 项在实际计算中由所有生成 token 的对数概率比之和近似，除以序列长度或保持为总和，这取决于实现。这个目标完美地平衡了“追求高奖励”与“保持语言流畅性”之间的矛盾。

---

#### 2. 为什么 KL 惩罚能防止策略崩塌？从信息论和策略梯度的角度分析。

2.1 策略崩塌的本质

策略崩塌（Policy Collapse）是指语言模型在 RLHF 中为了片面追求奖励模型的高分，逐渐丧失其原有的语言能力，输出变得重复、无意义或充满模板化用语。从概率分布的角度看，策略崩塌表现为策略熵的急剧下降，概率质量高度集中在少数几个能获得高奖励的 token 或 token 序列上。从行为上看，它像是一种严重的“过拟合”，过度适应了奖励模型的表面特征。

2.2 信息论视角：KL 散度作为信息约束

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/CscJbWlqXowsCFx6GZhcJu0Bn6c.png)

- **最小描述长度原则：**KL 惩罚确保了策略在寻找高奖励输出时，仍然倾向于选择那些在参考模型下“自然”的路径。这符合奥卡姆剃刀原则：如果两个解释（回复）都能获得同样高的奖励，那么选择那个对参考模型来说更“常见”的（即 KL 散度更小的）解释。这避免了策略走向怪异、不自然的解空间。

2.3 策略梯度视角：KL 惩罚稳定了更新方向

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/WjdtbtjeCojciLxBLMRcLsKtnEr.png)

![image](../../../assets/TFi4dIwwVolRPkxNhBBcxrdfnL0/MNNabORUsoSRbqx5XAIcIU5tn7e.png)

- **降低方差：**KL 惩罚作为每个 token 的即时成本，为 Critic 提供了密集的、相对稳定的训练信号。这使得价值函数的学习变得更容易，优势估计的方差更低。没有它，Critic 就必须完全依赖那个稀疏且可能有噪声的最终奖励来学习价值函数，其方差会极高，导致训练极其不稳定。

因此，KL 惩罚既是一个防止遗忘语言先验的信息瓶颈，也是一个持续将策略锚定在安全流形的引力场，还是降低 RL 训练方差的稳定器。三者共同作用，有力地防止了策略崩塌。

---
