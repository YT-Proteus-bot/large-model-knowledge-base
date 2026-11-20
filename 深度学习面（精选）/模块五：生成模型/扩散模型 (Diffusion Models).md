# 扩散模型 (Diffusion Models)

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/B3cWw9MTcipDGVk2lMWcLYBBnIh

## 深度学习面（精选）

> （此处为嵌入式文档部件，需登录飞书查看）

#### 1. 扩散模型的基本思想是什么？前向过程和反向过程分别指什么？

扩散模型是一类基于马尔可夫链的生成模型，其灵感来自非平衡热力学。基本思想可以这样理解：

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/Vpp3bCDy1ouyAixyadGcOyf0nph.png)

- **前向过程（扩散过程）：**给定真实数据 **x**0，我们不断地、逐渐地向其中加入微量高斯噪声，经过 *T* 步后，数据被完全破坏成一个**各向同性的标准高斯噪声 x*****T*****∼N(0,I)**。这个过程是固定的、无参数的马尔可夫链，每一步都只依赖上一步。
- **反向过程（去噪过程）：**如果我们能够学习到如何逆转前向过程的每一步——即从 **x***t* 恢复出 **x***t*−1，那么我们就可以从纯噪声 xT**x***T* 开始，逐步去噪，最终生成新的数据样本。反向过程同样被建模为一个马尔可夫链，但转移概率 pθ(xt−1∣xt) 由神经网络参数化，需要从数据中学习。

因此，扩散模型的核心就是学习一个能够逐步将噪声转化为有意义数据的逆扩散过程。前向加噪提供了一条从数据到噪声的“破坏路径”，反向去噪则学习其“重建路径”。

---

#### 2. DDPM 的前向加噪过程是如何定义的？写出从 x0x0 到 xtx*t* 的直接采样公式。

在 DDPM（Denoising Diffusion Probabilistic Models）中，前向过程是一个固定方差的高斯马尔可夫链。给定一个预定义的方差序列 *β*1,…,*βT*∈(0,1)，前向过程定义如下：

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/L4z7bhlmtoCDLOx7bcvcp7evn3f.png)

这相当于对当前数据先缩放，再加上微小的高斯噪声。

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/EGU3bH2c0oX9czxJlCFc9Mu9nub.png)

等价地，条件分布可以写为：

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/C5LNb7m91oMwaexvl3GcEDJSnZe.png)

---

#### 3. 在扩散模型中，反向去噪过程是如何参数化的？为什么可以学习噪声预测？

反向过程被建模为马尔可夫链，其转移概率也用高斯分布来近似：

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/UJZebFffto1NclxP6nKcGEGLnMf.png)

为什么可以学习噪声预测
前向过程是确定性的加噪，给定了 **x**0 和噪声 ***ϵ*** 就可以得到 **x***t*。反向过程中，我们希望从 **x***t* 推断出其中混合的噪声，从而恢复出**x***t*−1。因为真实的噪声 ***ϵ*** 是已知的（在训练时），我们可以让神经网络直接预测这个噪声。预测噪声等价于预测数据 **x**0 或预测得分函数，但预测噪声在实践中效果最好、训练最稳定。从损失函数角度，最终简化为最小化预测噪声与真实噪声之间的 MSE。

---

#### 4. 推导 DDPM 的训练损失函数，为什么最终简化为预测噪声的 MSE？

![image](../../../assets/Kaoydwto6oFARcxukQ5cJwDEnNg/RqtmbnCbZokDulxuVs2cZv3vnAh.png)
