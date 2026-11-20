# Agent 评估体系

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SEdiwaoGBiWMFYk0pl3c9iW4n7c

## AI Agent精选八股文

> （此处为嵌入式文档部件，需登录飞书查看）

### 1. 多 Agent 系统中，怎样评估个体 Agent 的贡献与整体协同效果？如何处理非独立交互和涌现行为？

在多 Agent 系统里，**评估的难题是 “1+1>2” 那部分究竟该归功于谁**。个体的输出往往依赖前置 Agent 的结果，而整体又可能展现出设计之外的新能力。面对这种非独立交互和涌现行为，需要建立一套分层+反事实的评估框架。

**评估架构图**

![image](../../../assets/StjvdZqX0o7vQzxYzNZc4NMynbb/WOAWbsWuzopPpLxrUvDcfip1nTg.png)

**个体贡献归因：用反事实思维**

**要想知道 Agent A 是不是真的不可替代，最直接的方法就是“拿掉它试试”。这被称为 Leave-One-Out (LOO) 消融。实现时有两种变体：**

- **移除该 Agent：**从流程里完全去掉 A，让 B 直接对接 C，或让人类临时替代 A 的角色。
- **降级该 Agent：**把 A 替换成更简单的规则或更弱的模型，观察整体效果下降多少。

**为了量化这个下降，我们可以定义一个 边际贡献 指标：**

```
代码块
def marginal_contribution(agent_name, full_crew_score, crew_without_agent_score):
    return full_crew_score - crew_without_agent_score

# 示例
baseline = 0.85   # 完整团队的任务成功率
without_A = 0.72  # 拿掉 Agent A 后，系统任务成功率
contribution_A = marginal_contribution("A", baseline, without_A)  # 0.13
```

如果 Agent 数量较多，可以进一步用 沙普利值（Shapley Value） 的近似算法，遍历所有子集组合，计算每个 Agent 的平均边际贡献。但实际工程中，LOO 已经足够诊断关键瓶颈。

捕捉协同与涌现

“协同”**体现在输出的一致性、冗余度下降、**或需要多 Agent 来回协商才能解决的问题增多。我们可以通过过程指标来捕获：

- 消息交互轮次：Group Chat 中，有效决策前的平均消息数。
- 信息增益：每次 Agent 发言后，任务状态的不确定性减少了多少（可以用 LLM 对后续候选答案的置信度变化来衡量）。
- 涌现行为：定义为测试集中“团队成功完成、但任一单个 Agent 无法独自完成”的问题比例。这类问题需要设计专门的 协同压力测试集，例如必须由安全专家和合规专家共同签审才能通过的任务。

对于涌现行为，最重要的是建立日志回放与人工标注机制。每次运行后，由评估员（或强模型）回答：这个成功的决策是否来自于任何单一 Agent 的独立能力？如果答案是否定的，就标记为涌现。长期积累这类案例，可以反向优化 Agent 间的交互协议。

示例代码：简易的 LOO 评估循环

```
代码块
def evaluate_crew(crew, test_cases):
    scores = []
    for case in test_cases:
        result = crew.kickoff(inputs=case)
        scores.append(score(result))
    return sum(scores) / len(scores)
agents = [planner, analyst, reviewer]
full_score = evaluate_crew(full_crew, test_set)
for agent in agents:
    reduced_crew = [a for a in agents if a != agent]
    score = evaluate_crew(reduced_crew, test_set)
    print(f"移除 {agent.role} 后，整体得分下降: {full_score - score:.3f}")
```
