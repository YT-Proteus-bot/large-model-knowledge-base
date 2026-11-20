---
title: 在 LangChain 里不加 @functools.wraps 会怎样？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/PwiSwQwaEivpIUk00CZc9Imtn0b
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在 LangChain 里不加 @functools.wraps 会怎样？.md
questions: 0
chars: 1741
build_date: '2026-11-20'
images_removed: 0
---

# 在 LangChain 里不加 @functools.wraps 会怎样？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在 LangChain 里不加 @functools.wraps 会怎样？.md

这个问题太实在了，我正好在封装自定义 Agent 工具时踩过这个坑。LangChain 内部大量依赖函数名和 docstring 来生成工具描述、注册路由和写日志，一旦装饰器不加 wraps，原函数的这些‘身份信息’就全被替换成装饰器内部函数的名字，轻则日志看不懂，重则 Agent 直接调不到工具。

我分三个层面来说这个连锁反应。

## 一、工具注册失败或描述错误

LangChain 最常见的写法是用 @tool 装饰器把一个普通函数包装成 Agent 可用的工具。比如：

```
from langchain_core.tools import tool
@tool
def get_weather(city: str) -> str:
    """查询指定城市的实时天气"""
    return f"{city}: 晴，26°C"
```

@tool 内部会把函数名和 docstring 自动生成工具的 name 和 description。Agent 根据这两个字段决定什么时候调用什么工具。

但如果我写了一个自定义装饰器，忘了加 wraps：

```
def log_calls(func):
    def wrapper(*args, **kwargs):
        print("调用工具")
        return func(*args, **kwargs)
    return wrapper
@log_calls
@tool
def get_weather(city: str) -> str:
    """查询指定城市的实时天气"""
    return f"{city}: 晴，26°C"
```

这里 log_calls 套在最外层，没有用 wraps。结果是：

- get_weather.name 变成了 'wrapper'
- get_weather.doc 变成了 None

LangChain 的工具注册逻辑读到的是 wrapper 这个无名函数，生成的工具名可能是 "wrapper" 或者干脆报错。Agent 在规划时看到的工具描述变成空的，或者一堆工具都叫 wrapper，根本无法区分。

## 二、日志、追踪和可观测性全乱

LangChain 的 verbose 模式或回调系统（如 LangSmith）会记录每个工具调用的名字。如果所有工具都叫 wrapper，你的 trace 里就会看到：

```
调用工具: wrapper
调用工具: wrapper
调用工具: wrapper
```

排查问题时完全不知道是哪个工具被调用了，也看不到原来的 docstring。这对线上排错是灾难性的。

## 三、工具冲突与依赖注入失效

更隐蔽的是，有些 LangChain 组件会根据函数名做依赖注入或缓存。如果多个工具被同一个不带 wraps 的装饰器包裹，它们的 name 全变成 wrapper。Agent 框架内部可能用函数名字符串作为字典 key，导致后面的工具覆盖前面的工具，莫名其妙少了一个工具。

## 四、解决办法：装饰器顺序和 wraps 缺一不可

最常见的修复就是：

```
import functools
def log_calls(func):
    @functools.wraps(func)  # 把原函数的身份复制过来
    def wrapper(*args, **kwargs):
        print("调用工具")
        return func(*args, **kwargs)
    return wrapper
```

同时，装饰器顺序也很重要。LangChain 的 @tool 需要包裹在最接近函数定义的地方，业务装饰器要在 @tool 之上：

```
@log_calls           # 业务装饰器（有 wraps）
@tool                 # LangChain 装饰器
def get_weather(city: str) -> str:
    ...
```
