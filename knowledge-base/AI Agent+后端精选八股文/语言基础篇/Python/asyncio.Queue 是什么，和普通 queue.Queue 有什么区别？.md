---
title: asyncio.Queue 是什么，和普通 queue.Queue 有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/XYSTwk2MuiUqAsk4IdwcUWUFn5d
source_path: AI Agent+后端精选八股文/语言基础篇/Python/asyncio.Queue 是什么，和普通 queue.Queue 有什么区别？.md
questions: 0
chars: 1238
build_date: '2026-11-20'
images_removed: 0
---

# asyncio.Queue 是什么，和普通 queue.Queue 有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/asyncio.Queue 是什么，和普通 queue.Queue 有什么区别？.md

聊到 asyncio.Queue 和 queue.Queue，面试官其实在考察你对 同步/异步模型 的理解深度，而不仅仅是背 API。我一般会从它们服务的“世界”讲起。

## 一句话定位

queue.Queue 是线程世界的管道，靠“阻塞”等待；asyncio.Queue 是协程世界的管道，靠“挂起”等待。

换个比喻：一个是实体店排队，人在那里干站着等；一个是拿号等叫号，你可以先去旁边喝杯咖啡。

## queue.Queue —— 线程安全的同步队列

它来自标准库 queue，设计给多线程环境。

```
import queue
import threading
q = queue.Queue(maxsize=5)
def producer():
    q.put(item)         # 队列满时，这里会阻塞线程
def consumer():
    item = q.get()      # 队列空时，这里也会阻塞线程
    q.task_done()
```

- 阻塞方式：put / get 会让整个线程停住，CPU 切走，线程睡在那里。
- 适用场景：线程池、生产者-消费者模型，线程间传递数据。
- 关键特性：内部有 threading.Lock 保证线程安全。

## ⏳ asyncio.Queue —— 协程友好的异步队列

来自 asyncio，专为 async/await 设计。

```
import asyncio
async def main():
    q = asyncio.Queue(maxsize=5)
    await q.put(item)       # 队列满时，当前协程挂起，事件循环不阻塞
    item = await q.get()    # 队列空时，同样挂起
    q.task_done()
```

- 阻塞方式：put/get 返回 awaitable，遇到等待条件时，将当前协程挂起，事件循环可以立刻跑去执行其他任务。
- 适用场景：异步爬虫、异步消息处理、需要限流的异步 API 调用等。
- 关键特性：协程安全，不阻塞线程，不会卡死整个事件循环。

## 对比表，一眼看清

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 核心原理：阻塞 vs 挂起

这是面试中拉开分差的点。

- 线程阻塞：当 q.put() 满时，操作系统把线程从 CPU 上拿下来，直到有空位。线程是操作系统资源，调度开销大，几百个线程已经很多了。
- 协程挂起：当 await q.put() 满时，事件循环记录这个协程在等待队列有空位，然后立刻转去执行其他协程。协程是用户态轻量任务，几万个轻松跑。

因此，在协程代码里错用 queue.Queue，相当于你在高速公路上突然停车，整个事件循环被卡死，所有异步任务全部暂停，这是非常典型的性能事故。

## 我的实际选用原则
