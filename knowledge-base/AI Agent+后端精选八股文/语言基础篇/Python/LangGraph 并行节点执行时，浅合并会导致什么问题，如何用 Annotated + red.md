---
title: LangGraph 并行节点执行时，浅合并会导致什么问题，如何用 Annotated + reducer 正确解决？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/UfhswEX1piUgcWkSIfNcSEXqnrh
source_path: AI Agent+后端精选八股文/语言基础篇/Python/LangGraph 并行节点执行时，浅合并会导致什么问题，如何用 Annotated + red.md
questions: 0
chars: 1462
build_date: '2026-11-20'
images_removed: 0
---

# LangGraph 并行节点执行时，浅合并会导致什么问题，如何用 Annotated + reducer 正确解决？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/LangGraph 并行节点执行时，浅合并会导致什么问题，如何用 Annotated + red.md

## 先看浅合并的「坑」

假设状态里有一个列表 items，两个并行节点都要往里追加内容：

```

# 错误示例：普通 dict 作为状态，LangGraph 默认浅合并
class State(TypedDict):
    items: list
def node_a(state):
    return {"items": ["A"]}
def node_b(state):
    return {"items": ["B"]}
```

执行流程：

```
State 初始 {items: []}
        /        \
   node_a       node_b
返回 {items:[A]}  返回 {items:[B]}
        \        /
      浅合并 (后覆盖前)
          ↓
   最终 {items: ["B"]}  ❌ 丢失了 A
```

🔴 问题本质：浅合并只做字段级别的 整体替换，同一个 key 的返回值会互相覆盖，无法合并「追加」「累加」这类增量操作。并行节点之间没有顺序保障，结果不可预测且数据会丢失。

## 正确解法：Annotated + reducer

在状态声明里，为字段绑定一个 合并策略（reducer），告诉 LangGraph 「如何把多个返回值合到一起」：

```
from typing import Annotated, TypedDict
from langgraph.graph import add_messages  # 也可以自己写 reducer
import operator

# ✅ 自定义合并逻辑：追加到列表
def merge_items(left: list, right: list) -> list:
    return left + right
class State(TypedDict):
    items: Annotated[list, merge_items]   # 关键！绑定 reducer
    total: Annotated[int, operator.add]   # 数字累加也用一样的方法
```

现在即使两个节点并行返回 {"items": ["A"]} 和 {"items": ["B"]}，LangGraph 会自动调用 merge_items：

```
node_a 返回 ["A"]    node_b 返回 ["B"]
        \                /
    merge_items(left, right)   ← 逐个两两合并
              ↓
      最终 ["A", "B"]  ✅ 完整保留
```

📌 执行机制：并行节点更新状态时，每个字段都使用你绑定的 reducer 归并，而不是简单 dict.update。顺序问题、数据丢失问题一次性解决。

## 面试加分回答模板

“LangGraph 的并行节点更新状态，默认浅合并只能做字段级替换，会导致并发写同 key 的数据互相覆盖。

我会用 Annotated 为该字段声明一个 reducer，比如 operator.add 做累加，或者自定义一个合并函数来处理列表、字典等复合结构。这样框架会在合并状态时自动调用 reducer，保证数据的正确性和完整性，即使是多个并行节点同时产出结果也没问题。”
