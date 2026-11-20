---
title: async def 和普通函数有什么本质区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FGHYwB5tZiYMSSkUuM6cprdonMc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/async def 和普通函数有什么本质区别？.md
questions: 1
chars: 711
build_date: '2026-11-20'
images_removed: 0
---

# async def 和普通函数有什么本质区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/async def 和普通函数有什么本质区别？.md

async def 和普通函数的本质区别，一句话：

普通函数是“做完了才回来”，async def 是“先给你一个承诺，之后在事件循环里一步步完成”。

下面我快速拆一下👇

### 核心差异对比

> 〔原文此处为飞书内嵌组件，内容未随导出〕

### 一个最简例子

```

# 普通函数
def normal():
    return "done"

# 异步函数
async def async_func():
    await some_io()   # 这里可以暂停，不阻塞线程
    return "done"
```

- normal()：立刻执行，当前线程必须等它结束。
- async_func()：得到的是一个协程对象，不会立刻执行，必须交给事件循环（asyncio.run 或 await）才真正运行。

## 题1：为什么这是“本质”区别？

- 普通函数是“给你结果”，调用栈是连续的，返回意味着结束。
- async def 是“给你一个任务包”，任务可以分步进行，中间可以穿插其他任务，实现了单线程内的协作式多任务。

这就是为什么 async def 能做高并发 IO，普通函数不能——普通函数的阻塞会卡死整个线程，而 async 函数的阻塞会被 await 转化为让出执行权。

### 选用口诀

- 全是 CPU 计算 → 普通函数
- 有大量 IO（网络、文件）且想单线程高并发 → async def + 异步库
- 简单脚本、不需要并发 → 普通函数就够了

这样讲，是不是把最本质的“调用生成协程对象、可暂停恢复”讲清楚了？如果需要，我们可以再聊聊协程调度那层。
