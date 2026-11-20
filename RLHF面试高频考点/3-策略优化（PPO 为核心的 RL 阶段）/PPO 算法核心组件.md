# PPO 算法核心组件

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ShXzw5qhYiJep9kS2mYcaT57nZc

#### 1. 写出 PPO 的裁剪目标函数，并解释其每一项的含义。

PPO 的裁剪目标函数 (Clipped Surrogate Objective) 是用于更新 Actor (策略网络) 的核心损失函数。它的标准形式如下：

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/Ln2Wbumc1osqBnx5Xn3cOpMCnzg.png)

每一项的含义：

- rt(θ)：重要性采样比率 (probability ratio)。定义新旧策略在状态 *st* 下选择动作 at*at* 的概率之比：

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/R8sCb6tZ4oU3ssxyCONclrDrnjh.png)

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/NtqLbGokcoqbQMxjFpMctZ4cnKf.png)

---

#### 2. PPO 中，为什么需要对重要性采样比率进行裁剪？不裁剪会出现什么问题？

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/Xa8MboawroHeZBx8I2ncbpZRncM.png)

**不裁剪的第一个问题：策略崩溃与高方差**

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/PJdzbAWSroqx8XxgKJpcv4Tvnib.png)

**不裁剪的第二个问题：破坏信任区域**

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/XbcXbmzweovRCLxryDFcgESVn9g.png)

**裁剪所起的作用：**

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/JUITbiABbojJ8vxxIoachvA4ngF.png)

---

#### 3. 裁剪率 ε 通常设为多少？过大或过小会怎样？

在 RLHF 的 PPO 实现中，ϵ*ϵ* 通常设在 0.1 到 0.2 之间，0.2 是最经典和常见的默认值。这个值是在大量实验基础上找到的一个平衡点。

**如果 ε 过大（例如 0.5 甚至 1.0）：**

- 裁剪约束变得非常宽松，几乎退化为普通的策略梯度方法。
- 策略可能在单次更新中发生巨大跃迁，过大的步长会导致训练剧烈波动，出现“悬崖式”的性能崩塌，即 loss 或奖励突然变成 NaN。
- 语言模型特别脆弱，一个过大的 ε 可能让模型一夜之间学会奖励黑客行为（如无意义地反复重复某个高分 token），而由于 KL 惩罚没有相应加强，模型可能迅速陷入这个局部最优并无法自拔。

**如果 ε 过小（例如 0.01）：**

- 裁剪约束过于严格，策略几乎被冻结在原地。
- 每次更新的幅度极小，模型需要极多的训练步数才能有可见的优化效果，收敛速度极慢。
- 在存在奖励模型噪声的情况下，过于保守的更新可能让模型完全停滞，无法有效放大那些微弱的正奖励信号，导致训练陷入瓶颈，无法达到更好的对齐效果。

**自适应 ε 的高级实践：**

一些工程实践中会引入动态调整 ε 的策略。例如，当监控到 KL 散度持续超出阈值时，主动降低 ε；当优化进展缓慢且 KL 很低时，可以适当增大 ε 以加快进度。但大多数稳健的基线仍使用固定的 0.2，并配合强大的 KL 惩罚。

---

#### 4. 什么是优势函数 A(s,a)？在 PPO 中如何利用它来降低梯度方差？

优势函数 A(s,a)*A*(*s*,*a*) 衡量的是“在状态 s*s* 下采取动作 a*a*，比在状态 s*s* 下的平均预期回报高出了多少”。其标准定义为：

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/FzplbzrRuotvygxffbYcQw7Mnzc.png)

![image](../../../assets/D4kZdD7MNofwPexlocWcjQ6bn3s/S9yrbBJCaoKUZsxGNhGcBPXynbf.png)
