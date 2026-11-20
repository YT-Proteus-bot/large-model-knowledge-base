# 分布式 RLHF 系统架构

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/TWMuwtjygiXKQYkjGELcArHan1b

#### 1. 🏗️ 设计一个大规模的 RLHF 训练系统，包含采样、奖励计算和训练三大模块，画出架构图并说明数据流。

架构概览

一个生产级 RLHF 系统通常由三大解耦的模块组成：采样集群（Actor Rollout）、奖励计算服务（Reward & Ref Server）、训练集群（PPO Trainer）。它们之间通过高性能消息队列或分布式数据总线连接，形成异步流水线。

架构图（文字版）

![image](../../../assets/IRmVdPFdlo1Ug2xTflgcimkSncb/Cdd0bbj03olYwWxre5KcYEiknDg.png)

🔁 数据流详述

1. 采样阶段：调度器从 Prompt 池中拉取一批无标签 prompt，推送给采样集群。Actor 模型使用当前最新权重进行自回归生成，为每个 prompt 生成候选回答，并记录每一步的对数概率 log⁡πθ(at∣st)log*πθ*(*at*∣*st*)。回答和概率数据被打包成 RolloutBatch，发送到经验缓冲区。
1. 奖励计算阶段：奖励计算服务从缓冲区拉取 RolloutBatch，将其中的 prompt 和 response 分别送入冻结的 Reward Model 和 Reference Model。Reward Model 输出序列级标量奖励 rRM*r*RM；Reference Model 输出每 token 对数概率 log⁡πreflog*π*ref。服务将 KL 惩罚 −β(log⁡πθ−log⁡πref)−*β*(log*πθ*−log*π*ref) 加到奖励上，形成最终奖励序列，再写回缓冲区。
1. 训练阶段：PPO Trainer 从缓冲区拉取完整的经验数据（包含状态、动作、旧对数概率、奖励、价值估计等）。使用 GAE 计算优势函数，然后执行多轮 PPO 更新，更新 Actor 和 Critic 的参数。更新完成后，Trainer 将最新的 Actor 权重推送到模型仓库，供采样集群拉取。

---

#### 2. 🆚 为什么需要将生成（Inference）和训练（Training）解耦到不同的 GPU 资源上？如果不解耦会有什么问题？

🔍 核心矛盾：推理与训练的资源需求截然不同

> （此处为内嵌电子表格，需登录飞书查看）

💥 不解耦的灾难性后果

如果强行让推理和训练共用同一批 GPU：

- 显存崩溃：PPO 训练同时需要 Actor、Critic、Ref、RM 四个模型，已经濒临显存上限。若再叠加推理时的 KV Cache，显存几乎必然溢出。
- 计算资源竞争：推理的串行解码会阻塞训练的大规模矩阵乘法，GPU 利用率在两者间剧烈震荡，墙钟效率极低。
- 同步开销爆炸：推理需要频繁拉取最新 Actor 权重，训练需要等待推理完成才能开始，两者串行交替会导致大量 GPU 空闲时间。

✅ 解耦的优势：

- 独立扩展：推理集群可单独水平扩展，提升采样吞吐；训练集群可聚焦梯度计算，使用高带宽网络。
- 资源利用率：推理集群可使用更适合推理的 GPU（如 T4、A10），训练集群使用顶级训练 GPU，成本最优。
- 异步流水线：采样和训练可以流水线化，隐藏数据传输和模型同步的延迟。

---

#### 3. 🚀 如何利用 vLLM 或 TensorRT-LLM 等推理引擎加速在线采样？它们如何与 PyTorch 训练框架交互？

🔧 推理引擎的作用

vLLM 和 TensorRT‑LLM 是专为大语言模型推理优化的引擎，核心优化包括：

- PagedAttention：将 KV Cache 分页管理，减少显存碎片，支持更大 batch 和更长序列。
- 连续批处理（Continuous Batching）：动态合并来自不同请求的 token 生成，最大化 GPU 利用率。
- 算子融合与量化：将注意力、层归一化等操作融合，支持 INT8/FP8 量化。
