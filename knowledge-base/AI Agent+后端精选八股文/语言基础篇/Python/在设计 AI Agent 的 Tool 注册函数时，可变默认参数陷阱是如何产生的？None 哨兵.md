---
title: 在设计 AI Agent 的 Tool 注册函数时，可变默认参数陷阱是如何产生的？None 哨兵模式怎么解决？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Weo1wrZcciNbwfklYh3cBWlVnbc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在设计 AI Agent 的 Tool 注册函数时，可变默认参数陷阱是如何产生的？None 哨兵.md
questions: 0
chars: 1335
build_date: '2026-11-20'
images_removed: 1
---

# 在设计 AI Agent 的 Tool 注册函数时，可变默认参数陷阱是如何产生的？None 哨兵模式怎么解决？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在设计 AI Agent 的 Tool 注册函数时，可变默认参数陷阱是如何产生的？None 哨兵.md

在设计 Agent 的 Tool 注册系统时，如果对 Python 的默认参数机制不够警惕，很容易埋下“跨调用污染”的坑。我先把陷阱怎么形成的拆开，再给出 None 哨兵模式的解法，最后落到 Tool 注册函数里该怎么做。

## 陷阱根源：默认参数在函数定义时只算一次

Python 在执行 def 语句时，会把所有默认参数表达式立即求值，然后把结果存进函数的 defaults 元组。之后每次调用，如果没传该参数，就直接复用这个预存的对象，不再重新计算。

用图来表示这个过程：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

所以如果你拿可变对象（列表、字典、集合）当默认参数，所有调用共享同一个对象，会互相污染。

## 在 Tool 注册场景的具体表现

假设我们写了一个简化的注册函数，让每个工具可以带一组标签，默认是空列表：

```
def register_tool(name, tags=[]):
    tags.append(name)
    tool_registry[name] = tags
    return tags
register_tool("search")
register_tool("calculator")
print(tool_registry)

# {'search': ['search', 'calculator'], 'calculator': ['search', 'calculator']}
```

本来期望每个工具的标签列表是独立的，结果第一个工具的标签里居然包含第二个工具的名字。这就导致 Agent 按标签过滤工具时，工具被错误归类，甚至出现循环依赖式的脏数据。

究其本质，就是这个 tags=[] 只在函数定义时创建了一次，后面每次 append 都在同一个列表上操作。

## 解决方案：None 哨兵模式

把默认值设为 None，然后在函数体内部判断，如果是 None 就新建一个真正的空列表。这样每次调用不传参数时，都会得到全新的列表对象。

代码就是：

```
def register_tool(name, tags=None):
    if tags is None:
        tags = []
    tags.append(name)
    tool_registry[name] = tags
    return tags
```

再用图对照一下：

```
调用 register_tool("search")
  → tags 是 None，所以 tags = []   (新建列表 A)
  → A.append("search")            → 结果 {"search": ["search"]}
调用 register_tool("calculator")
  → tags 又是 None，所以 tags = [] (新建列表 B)
  → B.append("calculator")        → 结果 {"calculator": ["calculator"]}
```

两个列表完全独立，不会再串。
