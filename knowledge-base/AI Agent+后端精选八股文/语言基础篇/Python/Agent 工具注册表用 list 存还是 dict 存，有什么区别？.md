---
title: Agent 工具注册表用 list 存还是 dict 存，有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/KlUrwiEeBia3NukX0aYc6qHmnXc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Agent 工具注册表用 list 存还是 dict 存，有什么区别？.md
questions: 0
chars: 980
build_date: '2026-11-20'
images_removed: 0
---

# Agent 工具注册表用 list 存还是 dict 存，有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Agent 工具注册表用 list 存还是 dict 存，有什么区别？.md

⚙️ 工具注册表的核心操作是按名字找工具。选 list 还是 dict，其实就是在选“遍历”还是“索引”。

## List：顺序存储，查找靠遍历

```
tools = [
    Tool("search", ...),
    Tool("calculator", ...),
    Tool("weather", ...)
]
def find_tool(name):
    for t in tools:
        if t.name == name:
            return t
```

- 添加：O(1)，直接 append
- 查找：O(n)，要一个一个比对名字
- 内存：只存元素本身，省空间

🔹 如果注册表很小（比如就 5~10 个工具），遍历开销几乎可以忽略，用 list 简单直接。

## Dict：键直接映射到工具对象

```
tools = {
    "search": Tool("search", ...),
    "calculator": Tool("calculator", ...),
    "weather": Tool("weather", ...)
}
def find_tool(name):
    return tools[name]   # 或 .get()
```

- 添加：O(1) 均摊
- 查找：O(1)，通过哈希直接定位
- 内存：多了哈希表开销（indices + entries）

🔸 当工具数量上到几十甚至上百，或者 Agent 每处理一条消息就要解析好几次工具名时，O(1) 的查找优势就非常明显。

## 直接对比

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 选哪个？

- 工具少，且常用模式是“全部遍历”（比如每个工具都试一遍），list 很合适。
- 工具多，或者调用频繁，核心操作是按名派发，dict 是标准答案。绝大多数 Agent 框架（LangChain、AutoGPT）内部都是 dict 或类似映射结构，因为运行时绝大多数情况是“已知工具名，找到函数然后调它”。

实际项目里我甚至会用 dict[name] → callable，连 Tool 对象都省了，直接映射到可执行函数。毕竟越靠近热路径，越值得用哈希。
