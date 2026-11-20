---
title: AutoGen
topic: AI Agent+后端精选八股文
sub_topic: Agent 架构篇/Agent Framework
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/MHAaw5sSsiptF5kxdKYcdIWfnze
source_path: AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/AutoGen.md
questions: 2
chars: 1050
build_date: '2026-11-20'
images_removed: 2
---

# AutoGen

> 主题：AI Agent+后端精选八股文 › Agent 架构篇/Agent Framework　｜　原始文件：AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/AutoGen.md

### AutoGen 的 Group Chat 与 Two-Agent 对话

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

## 题1：AutoGen 的 Two‑Agent 对话和 Group Chat 各适用于什么场景？

AutoGen 的多智能体对话可以被抽象成两种拓扑：点对点的 Two‑Agent 对话 和 多对多的 Group Chat。前者像两个人之间的专业协作，后者像一个团队讨论会。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

Two‑Agent 对话的适用场景

- 代码生成与执行：一个 AssistantAgent 写代码，一个 UserProxyAgent 代表用户执行代码并反馈执行结果。这是 AutoGen 最经典的范式。
- 角色扮演协作：例如“面试官”和“候选人”的模拟面试，或者“老师”和“学生”的问答辅导。
- 条件明确的回合制任务：你清楚地知道只有两个角色参与，且它们之间的交互模式固定（A 提问 → B 回答 → A 追问）。

Group Chat 的适用场景

- 多角色共同决策：产品设计评审，需要工程师、设计师、产品经理、测试同时发言并交换意见。
- 复杂任务拆分与聚合：一个任务需要“研究员”检索资料、“分析师”解读数据、“写手”生成报告，它们之间需要多次来回讨论。
- 专家会诊：多个不同领域的专家 Agent 共同诊断一个问题，例如“安全专家”+“性能专家”+“法规专家”一起审查一份架构方案。

选型速查：

- 如果只需要一个助手+一个用户，且交互方式是“助手干活、用户反馈”，直接选 Two‑Agent。
- 如果需要三个及以上不同角色，或者发言人顺序不能事先固定（需要根据讨论内容动态决定），就用 Group Chat + GroupChatManager。

## 题2：AutoGen Group Chat 中，如何控制发言顺序和避免无限循环对话？

Group Chat 的发言由 GroupChatManager 控制，它内部通过一个 speaker_selection_method 来决定每一轮该谁说话。控制发言顺序和防止死循环，需要从发言规则、终止条件、消息约束三个层面设计。

① 控制发言顺序：指定发言选择策略

默认情况下，Manager 使用 auto 模式，让内部的 LLM 根据对话历史决定下一个发言人。你可以把它改为 round_robin（轮询），或提供一个自定义的选择函数。
