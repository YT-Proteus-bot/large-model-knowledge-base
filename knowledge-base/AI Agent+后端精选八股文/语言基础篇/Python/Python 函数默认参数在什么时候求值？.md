---
title: Python 函数默认参数在什么时候求值？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/CXHbw1cD0iQJ8lkswJYc6oxJnuh
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 函数默认参数在什么时候求值？.md
questions: 0
chars: 1450
build_date: '2026-11-20'
images_removed: 0
---

# Python 函数默认参数在什么时候求值？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 函数默认参数在什么时候求值？.md

这个问题我会用一个时间线把“定义时求值”和“调用时求值”说清楚，然后聊一下那个有名的可变对象坑，再给出我在 Agent 代码里的习惯写法。

## 默认参数是在“定义时”一次性求值的

当你写：

```
def foo(bar=some_expression()):
    ...
```

some_expression() 只会在 def 语句被执行的时候运行一次，结果就会被固化在函数对象的 defaults 属性里。之后每次调用函数，如果没传 bar，用的永远是这个预先算好的值，绝不再算一次。

用图表示就是这个节奏：

```
定义时：
   def func(x=compute())  ──▶  compute() 被调用，返回结果 42
                                 │
                                 ▼
                          func.__defaults__[0] = 42
调用时：
   func()  ──▶ 直接从 __defaults__[0] 拿 42
   func(5) ──▶ 用传入的 5
```

所以 默认参数完全是一个“初始化时”的概念，不是“每次调用时”的概念。

## 可变默认参数的经典坑

因为只算一次，如果你用了一个可变对象做默认参数，每次调用都会共享并修改同一个对象：

```
def add_history(item, log=[]):
    log.append(item)
    return log
print(add_history("turn1"))   # ['turn1']
print(add_history("turn2"))   # ['turn1', 'turn2']  ← 不是新列表
```

log 这个列表在函数定义时就创建好了，之后每次 .append 都在同一个列表上动手。Agent 里写对话记忆、收集指标时，这种写法会让不同的调用会话互相污染。

## 标准解法：用 None 然后在函数体内初始化

把默认值设为 None，再在函数体内做实际初始化，这是最通用的做法：

```
def add_history(item, loteg=None):
    if log is None:
        log = []
    log.append(item)
    return log
```

图示上就是两条路：

```
调用 add_history("turn1")
  └─ log 是 None → 新创建 [] → 结果 ['turn1']
调用 add_history("turn2")
  └─ log 是 None → 又新创建 [] → 结果 ['turn2']  (互相独立)
```

如果需要每次调用都动态生成一个默认值（比如取当前时间），也是同样的模式：

```
def process(start_time=None):
    if start_time is None:
        start_time = time.time()
    ...
```

## 原理延伸：默认参数其实就是函数属性

所有默认值都安安静静地存在 func.defaults 这个元组里。这个细节有两个用处：

- 调试时可以直接检查到底固化了什么值。
