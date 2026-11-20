---
title: LangGraph
topic: AI Agent+后端精选八股文
sub_topic: Agent 架构篇/Agent Framework
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/BGoPwvMLhic7wKkrioacP0oynIh
source_path: AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/LangGraph.md
questions: 1
chars: 1289
build_date: '2026-11-20'
images_removed: 2
---

# LangGraph

> 主题：AI Agent+后端精选八股文 › Agent 架构篇/Agent Framework　｜　原始文件：AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/LangGraph.md

### LangGraph 的状态图模型与工作流控制

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 题1：LangGraph 的核心概念是什么？State、Node、Edge 三者什么关系？

LangGraph 是一个用有向图来编排 Agent 行为的框架。 它把 Agent 的每一次决策、每一个工具调用、每一段思考都抽象为图的 节点，把控制流和数据流抽象为图的 边。整张图共享一份 状态。

一图胜千言：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

三者的职责与关系：

- State（状态）：一份贯穿全图的共享数据字典（TypedDict 或 Pydantic 模型）。每个节点都可以读它、写它。它承载了 Agent 的全部上下文：对话历史、中间结果、控制标志等。
- Node（节点）：一个 Python 函数，接收当前 State，返回更新后的 State（或部分更新）。它可以是一个 LLM 调用、一个工具执行、一段业务逻辑。
- Edge（边）：定义节点之间的流转规则。普通边就是“A 执行完一定去 B”；条件边是“A 执行完后，根据 State 的某个字段值决定去 B 还是去 C”。

关系一句话：State 是大脑的记忆，Node 是会做的事情，Edge 是做完一件事之后接着做什么的决定。

最简单的代码骨架：

```
from typing import TypedDict
from langgraph.graph import StateGraph, END
class AgentState(TypedDict):
    messages: list
    next_step: str
def node_a(state):
    # 处理逻辑...
    return {"messages": state["messages"] + ["A done"]}
def node_b(state):
    return {"messages": state["messages"] + ["B done"]}
def router(state):
    if "done" in state["messages"][-1]:
        return END
    return "node_b"
graph = StateGraph(AgentState)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.set_entry_point("node_a")
graph.add_conditional_edges("node_a", router, {"node_b": "node_b", END: END})
graph.add_edge("node_b", "node_a")  # 循环
app = graph.compile()
```
