# 记忆与 Agent 集成

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/PGF2wlLMeiiO4SkG7iycSQbwnNf

#### . 💡 在 Agent 中，除了对话历史，还需要记住什么？比如工具的返回结果。

Agent 的工作方式是“思考-行动-观察”循环（ReAct），在这个过程中，它会调用工具、获取结果，并基于此进行下一步推理。因此，除了用户与Agent的对话历史，Agent还必须记住以下内容：

- 当前任务的上下文与子目标：用户最初提出的完整指令可能被拆解为多个子任务。Agent 需要记住“我已经完成了哪几步，下一步要做什么”，以及“最终目标是什么”。否则在多步推理中容易迷失方向。
- 工具调用的历史与返回值：Agent 调用工具后，必须记住工具的名称、参数和返回结果。这些信息是后续推理的依据。例如，查询了天气，记住“上海今天25度”，后续才能推荐合适的衣服。
- 中间推理过程（scratchpad）：Agent 在单次任务执行中，会记录自己的思考（Thought）、行动（Action）和观察（Observation）。这些是瞬时的“草稿”，用于在当前任务闭环内进行多步推理，但通常不会存入长期记忆（Memory）。
- 从历史对话中提取的实体与偏好：在多轮对话中，用户可能在不同时间点提及个人偏好（如“我吃素”）、名字、地点等。Agent 需要把这些“知识”记住，以便个性化回应。这属于长期记忆的范畴，常需要持久化到向量库。
- 任务执行状态与错误信息：当工具调用失败时，Agent 需要记住错误原因，避免重复相同的错误调用，或者切换到备选方案。

实践建议：在 LangChain 中，对话历史通常由 Memory 组件自动管理；而工具调用历史和 scratchpad 则由 AgentExecutor 内部维护，并通过 Prompt 注入。如果需要记忆工具返回结果以便在未来的独立会话中引用，则应将这些结果显式地存入长期记忆（如向量库）。

---

#### 1. 📝 Agent 的中途输出（scratchpad）是如何被记录和传递的？这与 Memory 有什么关系？

Scratchpad 是 Agent 在一次任务执行中产生的“草稿纸”，记录了它多步推理的完整轨迹，包括 Thought, Action, Action Input, Observation。它的生命周期仅限于单次 agent_executor.run() 调用。一旦 Agent 输出 Final Answer，scratchpad 就会被丢弃。

记录与传递机制：

- 当你调用 AgentExecutor.run() 时，LangChain 内部会启动一个循环（最多 max_iterations 次）。
- 在每次循环中，Agent 根据当前 Prompt（包含用户输入、可用工具、以及所有之前的 scratchpad 内容）调用 LLM，LLM 返回下一步的 Thought/Action/Action Input。
- 如果 LLM 输出的是 Action，AgentExecutor 执行相应工具，得到 Observation。
- 这个 (Thought, Action, Observation) 三元组会被追加到 scratchpad 列表中。
- 在下一次循环开始时，更新后的 scratchpad 会再次被格式化并注入 Prompt。因此，Agent 能看到自己“刚才想了什么、做了什么、结果如何”，从而进行连贯的多步推理。

与 Memory 的关系：

- Memory（如 ConversationBufferMemory）存储的是跨轮次的对话历史。比如上一轮用户问了天气，这一轮问“那明天呢？”，Memory 使得 Agent 知道“那”指的是天气。
- Scratchpad 是轮次内的推理轨迹。即使没有 Memory（如单次任务），Agent 也需要 scratchpad 来完成多步操作。两者共同构成 Agent 的上下文窗口。
- 在 Prompt 组装时，LangChain 通常将 Memory 的历史放在前面，scratchpad 紧接其后，再是用户的最新输入。这样 Agent 既了解对话背景，也清楚当前任务的进展。

“坑”与思考：如果 Memory 过长，会挤压 scratchpad 的可用空间，导致 Agent 在长对话中失去推理能力。因此需要限制 Memory 大小，或使用摘要记忆，为 scratchpad 留足 token 预算。

---

#### 3. 🧭 如果一个 Agent 执行过程中需要多轮推理，如何让它在后续步骤中不忘记最初的目标？

Agent “忘记目标”是长链推理中的经典问题。根本原因是 LLM 的注意力随着 Prompt 增长而衰减，或者 scratchpad 中的中间步骤噪声过大，淹没了原始指令。

防御策略：

1. 在 Prompt 中固化目标：在构建 Agent Prompt 时，将用户原始问题放在一个显眼的位置，并使用 SystemMessage 强调“你的最终目标是{original_question}”。每次循环，这个目标都会出现在上下文中，不会被后续的 observation 淹没。
1. 利用 Memory 保持目标：将用户初始问题存入 Memory，并在每次 Agent 推理时，将 Memory 中的目标摘要作为额外上下文注入。例如，在 ConversationSummaryMemory 的摘要中，明确包含“用户希望完成X任务”。
1. 显式追踪子目标完成状态：要求 Agent 在每一步思考时，先回顾最终目标，再列出已完成和未完成的子目标。这可以通过 Prompt 工程设计实现：

```
代码块
Thought: 我的最终目标是安排一次从北京到上海的旅行。目前我已经查了航班，还没有预订酒店。接下来我应该搜索上海的酒店。
```

1. 限制推理步数并设置检查点：当步数超过阈值时，强制 Agent 暂停并总结已完成的工作，然后重新明确下一步计划。这相当于在 scratchpad 中插入一个人工检查点。
1. 利用长期记忆存储任务状态：对于需要跨会话的复杂任务，可将任务状态（如 JSON 格式的已完成列表）存储在向量库或 Redis 中。每次 Agent 启动时，先检索当前任务状态，然后基于此继续工作。

实践反思：单纯依赖 LLM 的注意力来记住初始目标并不靠谱。应该在 Prompt 结构上下功夫，把目标做成“固定背景板”，每次推理都强制 LLM 先看目标，再看历史。这就像给人一份任务清单，每完成一项就划掉，始终知道终点在哪。

---

#### 4. 🗄️ 你如何设计一个 Agent 的记忆，使其能根据当前问题自动检索相关的长期记忆？

设计这种记忆的核心是 “检索增强记忆”。Agent 不应无差别地加载所有历史，而应根据当前问题，从长期记忆库中动态拉取最相关的片段，注入 Prompt。

设计步骤：

1. 存储长期记忆：使用向量数据库（如 Chroma、Pinecone）存储对话片段、提取的实体、用户偏好等。每条记忆可以带元数据（时间戳、重要性评分等）。
1. 自动检索：在 Agent 执行前，用当前用户输入（或改写后的查询）去向量库中检索最相关的 top-K 条记忆。
1. 注入 Prompt：将检索到的记忆格式化为文本，放入 Prompt 的特定位置（例如 {long_term_memory}）。
1. 更新机制：在对话结束后，使用 LLM 从最新交互中提取值得长期保留的信息，存入向量库。旧记忆可通过时间衰减或重要性评分来淘汰。

LangChain 实现：

```
代码块
from langchain.memory import VectorStoreRetrieverMemory
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

# 1. 初始化向量库和检索器
vectorstore = Chroma(embedding_function=OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 2. 创建长期记忆
long_term_memory = VectorStoreRetrieverMemory(
    retriever=retriever,
    memory_key="long_term_memory",
    input_key="input",
)

# 3. 在 Agent 中使用
agent = create_openai_functions_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, memory=long_term_memory)
```
