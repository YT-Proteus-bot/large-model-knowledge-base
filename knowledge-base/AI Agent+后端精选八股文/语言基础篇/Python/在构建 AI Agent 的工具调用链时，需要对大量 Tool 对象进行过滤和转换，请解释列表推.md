---
title: 在构建 AI Agent 的工具调用链时，需要对大量 Tool 对象进行过滤和转换，请解释列表推
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/QEHfwPPHTiYigfkA2MrcqIgrncb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在构建 AI Agent 的工具调用链时，需要对大量 Tool 对象进行过滤和转换，请解释列表推.md
questions: 0
chars: 1146
build_date: '2026-11-20'
images_removed: 1
---

# 在构建 AI Agent 的工具调用链时，需要对大量 Tool 对象进行过滤和转换，请解释列表推

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在构建 AI Agent 的工具调用链时，需要对大量 Tool 对象进行过滤和转换，请解释列表推.md

在实际写 Agent 工具链的时候很关键，因为工具数量一多，懒加载、流式处理和内存控制就不得不考虑。我按“性能差异 → 图示对比 → Agent 场景选择”来说。

## 先理清三者的本质差异

- 列表推导式 [fn(t) for t in tools if cond]：直接生成一个完整的列表，立刻求值，有额外的方法栈帧但非常快。
- 生成器表达式 (fn(t) for t in tools if cond)：惰性，不产生列表，只返回一个生成器，每次迭代才计算一个值。
- map/filter：也是惰性的（Python 3），返回迭代器。但内部走 C 实现，调用函数时有回调开销。

用一个图展示它们在“执行”和“返回”上的区别：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

## ⏱️ 性能差异的核心原因

1. 列表推导式比 map+lambda 快

   因为列表推导式直接在 Python 字节码里用 LIST_APPEND 等指令，没有函数调用的开销。而 map(fn, iterable) 每一次迭代都要回调 fn（如果 fn 是 Python 函数），函数调用栈的进出比推导式的内部循环要慢一个量级。

1. map/filter 比推导式快，但仅限于函数是 C 实现的

   比如 map(str, tools)，str 是内置 C 函数，没有回调开销，此时 map 极快，且内存惰性。如果传的是 lambda，那就慢下来了。

1. 生成器表达式 vs 列表推导式

   计算速度上，生成器表达式稍快一点，因为它不分配列表内存；但你不能索引、不能算长度。两者在字节码层面非常接近，区别只在最后是否一次性产出列表对象。

1. 内存占用

   对 10 万级别的 Tool 对象过滤，列表推导式会立刻占满大块内存；而生成器或 map/filter 只占几百字节。

## Agent 工具链场景下的选择指南

我通常按照数据量级、消费次数、可读性来做决定：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

在 Agent 里经常遇到的一个典型流程：从注册表中取出所有工具 → 过滤启用的 → 转换名称 → 传给 LLM 的 prompt 模板。我的习惯是：

```
enabled_tools = (t for t in ToolRegistry.all() if t.enabled)
tool_descriptions = [f"- {t.name}: {t.desc}" for t in enabled_tools]
```

第一行用生成器表达式避免中间列表（万一注册表很大），第二行用列表推导式因为要拼接字符串并且可能会检查长度。
