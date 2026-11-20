# Agent 概念

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/L6pCw9CICiNRXDkqTmbcNMZ5nzh

#### 1. 🧠 什么是 LangChain Agent？它和 Chain 最本质的区别是什么？

LangChain Agent 不是一条固定的流水线，而是一个能根据任务动态选择行动的智能体。你可以把它想象成一个项目经理：你告诉它最终目标（用户的问题），它自己决定需要什么工具、以什么顺序调用工具、如何分析结果，最终完成任务。

Chain（链）则是事先编排好的固定操作序列。比如一个 RetrievalQA 链，它的逻辑是写死的：收到问题 -> 向量检索 -> 拼接到 Prompt -> 调用 LLM -> 返回答案。链不具备自主决策能力。

最本质的区别：

- 决策权：Agent 自主决定下一步做什么；Chain 硬编码了执行流程。
- 灵活性：Agent 能处理未知的、多步推理的问题，可以根据中间结果调整策略；Chain 只能处理预设流程内的问题。
- 循环执行：Agent 内建了“思考-行动-观察”的循环，直到达成目标；Chain 调用一次就结束。

举个例子：用户问“明天北京的天气怎么样？如果下雨，给我推荐室内活动”。

- 如果用 Chain，你必须事先写死：先调用天气 API，再根据返回值判断，再调用活动推荐 API。任何一个环节变化都需要修改链。
- 如果用 Agent，你只需告诉它有两个工具：WeatherTool 和 ActivityTool。Agent 会自己推理：“我需要先查天气 -> 调用天气工具 -> 发现下雨 -> 再调用活动工具搜索室内活动 -> 整理回答”。所有决策都由 LLM 做出。

LangChain 中的实现：Agent 通常与 AgentExecutor 配合使用。AgentExecutor 负责运行 Agent 的循环、管理工具调用、处理错误，是 Agent 的“执行引擎”。

✅ 经验之谈：我刚用 LangChain 时，总想把所有流程都写成 Chain，结果越写越复杂。后来发现，一旦业务逻辑稍微复杂一点，用 Agent 会省很多事，因为它把“怎么完成”的决策交给了 LLM，你只需要专心把工具做好。但 Agent 也带来了不确定性，需要更多调试。

---

#### 2. 🔄 画出 Agent 的执行循环：思考 → 行动 → 观察 → 思考……并解释每一步。

Agent 的执行循环就是著名的 ReAct 模式（Reasoning + Acting），其流程如下：

![image](../../../assets/LgDvdOz6komMfox5ui6c5X4yndh/XttIbAHa9oREqoxAsV9cZYeqnSg.png)

每一步的含义：

- 思考 (Thought)：Agent 分析当前状态和已有信息，决定下一步应该做什么。它用自然语言表达推理过程，比如“我需要先获取天气数据，所以应该使用 Weather 工具”。
- 行动 (Action)：Agent 输出一个具体的工具调用指令，包括工具名称和输入参数。例如 Action: Weather, Action Input: "北京"。
- 观察 (Observation)：工具执行后返回的结果。Agent 会把这个结果作为新的信息，反馈到下一轮思考中。例如 Observation: 北京明天晴，25度。
- 循环：Agent 根据观察再次思考，判断是否还需要进一步行动，或者已经可以给出最终答案。如果需要更多信息，就继续循环；否则输出最终答案。

AgentExecutor 的内部循环：

1. 将用户输入、可用工具列表、历史交互（scratchpad）组装成 Prompt，发给 LLM。
1. LLM 返回 Thought/Action/Action Input。
1. 解析输出，如果包含 Action，则调用相应工具。
1. 将工具输出作为 Observation 追加到 scratchpad。
1. 重复步骤 1，直到 LLM 输出 Final Answer 或达到最大步数。

✅ 经验之谈：这个循环让 Agent 有了“试错”能力。但也要警惕，如果工具描述不清晰，Agent 可能会在循环里打转，反复调用同一个工具。这时需要设置合理的 max_iterations，并且让工具返回足够明确的信息。

---

#### 3. 📚 Agent 有哪几种主要类型？（ReAct, OpenAI Functions, StructuredChat 等）各自的特点。

LangChain 支持多种 Agent 类型，它们的核心区别在于如何构造 Prompt 和如何解析 LLM 的输出。

1. ReAct Agent
- 特点：经典模式，强调“Thought, Action, Observation”。Prompt 通常包含详细的示例（Few-shot），指导 LLM 按固定格式输出。
- 适用场景：开源模型，或者对格式控制要求高的场景。
- 优势：逻辑清晰，可解释性强，不依赖模型特定的 API。
- 局限：输出格式有时不稳定，需要较长的 Prompt，会消耗更多 token。
1. OpenAI Functions Agent
- 特点：专门为 OpenAI 的 Function Calling 能力设计。LLM 不再输出文本格式的 Action，而是直接生成一个函数调用 JSON。
