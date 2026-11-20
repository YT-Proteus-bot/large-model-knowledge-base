# 在 LangGraph 中，如果 Agent State 里有一个记录"已访问 URL"的集合，并行节点各自爬取不同的页面，怎么设计这个字段才不会出问题？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/KYwKwJt4ziaL29kg88jcvn6unqf

“先说坑在哪。”

并行节点都往 visited_urls 这个集合里加新 URL，如果状态只是普通的 TypedDict，LangGraph 的默认合并是浅合并——把整个字段直接替换掉。

比如节点 A 返回 {"visited_urls": {"/page1"}}，节点 B 返回 {"visited_urls": {"/page2"}}，浅合并一执行，最后可能只剩一个节点加进去的 URL，另一个就被盖掉了。
并行场景下到底谁盖谁，完全看调度时机，不可控。

🔴 核心问题：集合被整体替换，不是增量合并。

---

✅ 正确设计：用 Annotated 绑定一个 reducer

在定义 State 时，告诉 LangGraph 这个字段「怎么合并」，而不是直接替换：

```
代码块
from typing import Annotated, Set, TypedDict

# 自定义 reducer：对集合取并集
def merge_sets(left: Set[str], right: Set[str]) -> Set[str]:
    return left | right
class AgentState(TypedDict):
    visited_urls: Annotated[Set[str], merge_sets]
    # 其他字段...
```

现在无论有多少个并行节点，各自返回自己的 visited_urls 片段，LangGraph 都会自动调用 merge_sets 做并集，所有节点发现的 URL 会完好无损地汇总到一起。

📊 流程可以这样记：

```
代码块
节点A 返回 {visited_urls: {🔗A, 🔗B}}
节点B 返回 {visited_urls: {🔗C, 🔗D}}
          \                /
        merge_sets(left, right)
               ↓
   最终 state.visited_urls = {🔗A, 🔗B, 🔗C, 🔗D}   ✅
```

🧩 为什么不用 add_messages？
那是给对话消息列表设计的，会自动去重和保留消息 ID，对普通集合反而不合适。自己写一个 merge_sets 简单直接，也体现你对 reducer 机制的理解。

📌 一句话总结：
并行写的集合字段，一定要用 Annotated + 自定义并集 reducer，把「替换」变成「归并」。
