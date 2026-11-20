---
title: 主流 AI 框架对比与选型
topic: AI Agent+后端精选八股文
sub_topic: Agent 架构篇/Agent Framework
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/AuVxwYFDWiT663kKLskczm8mnIh
source_path: AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/主流 AI 框架对比与选型.md
questions: 1
chars: 2067
build_date: '2026-11-20'
images_removed: 1
---

# 主流 AI 框架对比与选型

> 主题：AI Agent+后端精选八股文 › Agent 架构篇/Agent Framework　｜　原始文件：AI Agent+后端精选八股文/Agent 架构篇/Agent Framework/主流 AI 框架对比与选型.md

## [LangChain / LangGraph / SpringAl / LangChain4J /

## Llamalndex / AutoGen / CrewAI ÷FXJłk]

### 框架总览速查表

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

## 题1：🔗 LangChain vs 🕸️ LangGraph：核心区别是什么？

### ▎一句话说清

LangChain 是“搭积木”的链式框架，LangGraph 是“画流程图”的有状态图框架。

前者解决“怎么把多个组件串起来”，后者解决“怎么让组件根据运行时状态循环、分支、暂停甚至回退”。

### ▎深度解析

1. 执行模型不同
- LangChain 的 LCEL（LangChain Expression Language）构建的是有向无环图（DAG）。链一旦 invoke，数据就从头流到尾，中间无法动态循环或根据中间结果回到上游。
- LangGraph 基于状态图（State Graph），节点之间可以形成环，边上可以挂条件判断，甚至支持暂停等待人工确认后再恢复执行。这让 Agent 的“观察-思考-行动”循环天然可被建模。
1. 状态管理能力
- LangChain 的 Chain 本质是无状态的，每次调用独立，需要记忆时得额外接 Memory 组件。
- LangGraph 把状态作为一等公民，每个节点都接收状态、返回状态更新，图自动管理状态的持久化和传递。这对于多步 Agent 或需要纠错的流程至关重要。
1. 复杂度天花板
- 简单 RAG、单一工具调用、顺序流水线，LangChain 足够优雅。
- 多 Agent 协作、带人工审核的工作流、根据运行结果动态决定下一步（甚至重试），LangGraph 的图结构不会变成“意大利面条”，因为它本身就是可观察、可测试的图。

### ▎示例代码对比

LangChain：搭建一个简单的 poet chain（无循环）

```
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一位诗人，用古诗风格回应。"),
    ("user", "{input}")
])
model = ChatOpenAI()
chain = prompt | model | StrOutputParser()
print(chain.invoke({"input": "春天"}))
```

这条链只能顺序执行：填 prompt → 调模型 → 解析输出，无法根据输出内容决定是否重新生成。

LangGraph：一个带自循环的简易 Agent（可重试直到满意）

```
from langgraph.graph import StateGraph, END
from typing import TypedDict
class AgentState(TypedDict):
    query: str
    answer: str
    attempts: int
def generate_answer(state):
    # 模拟调用LLM
    state["answer"] = f"回答：{state['query']} (第{state['attempts']}次)"
    state["attempts"] += 1
    return state
def should_continue(state):
    # 如果尝试次数<2且答案中不含“满意”，则循环回生成节点
    if state["attempts"] < 2 and "满意" not in state["answer"]:
        return "generate"
    return END
graph = StateGraph(AgentState)
graph.add_node("generate", generate_answer)
graph.set_entry_point("generate")
graph.add_conditional_edges("generate", should_continue, {
    "generate": "generate",
    END: END
})
app = graph.compile()
print(app.invoke({"query": "今天天气", "attempts": 0}))
```
