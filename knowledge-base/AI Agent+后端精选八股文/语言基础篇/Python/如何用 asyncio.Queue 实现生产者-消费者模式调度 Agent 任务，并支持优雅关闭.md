---
title: 如何用 asyncio.Queue 实现生产者-消费者模式调度 Agent 任务，并支持优雅关闭和优先级调度？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/SQYww6fB2i2D6DkZzijcFlX5nUh
source_path: AI Agent+后端精选八股文/语言基础篇/Python/如何用 asyncio.Queue 实现生产者-消费者模式调度 Agent 任务，并支持优雅关闭.md
questions: 0
chars: 1588
build_date: '2026-11-20'
images_removed: 1
---

# 如何用 asyncio.Queue 实现生产者-消费者模式调度 Agent 任务，并支持优雅关闭和优先级调度？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/如何用 asyncio.Queue 实现生产者-消费者模式调度 Agent 任务，并支持优雅关闭.md

面对 Agent 这种既要稳定吞吐又要弹性响应的系统，我几乎每次都把 asyncio.Queue 作为任务流的中枢神经。它天生就是为异步生产者-消费者准备的，不过要把它用到“优雅关闭”和“优先级调度”的程度，需要一些约定好的手势。

## 基础模式：一个队列 + 一组合格的工人

最简单的模型：主 Agent 是生产者，把任务（比如 LLM 请求、工具调用）往队列里 put；一组 worker 协程是消费者，不断 get 并执行。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

```
import asyncio
async def producer(queue: asyncio.Queue):
    for task in incoming_tasks:
        await queue.put(task)
async def worker(queue: asyncio.Queue, name: str):
    while True:
        task = await queue.get()
        try:
            await process(task)
        except Exception as e:
            log_error(e)
        finally:
            queue.task_done()   # 通知队列本任务完成
```

启动时：

```
queue = asyncio.Queue(maxsize=100)  # 有背压上限
workers = [asyncio.create_task(worker(queue, f"w{i}")) for i in range(5)]
producer_task = asyncio.create_task(producer(queue))
```

## 优雅关闭：让队列说“打烊了”，别直接拔电源

直接 cancel() worker 会导致正在执行的任务中断，且队列里的任务丢失。正确的做法是先关门，再清场：

1. 生产者先停止放新任务。
1. 等待队列里已有的任务全部被取走处理完 (queue.join())。
1. 向每个 worker 发送 sentinel 信号（例如 None），让它们自行退出。

```
async def graceful_shutdown(queue, workers, producer_task):
    # 1. 等待生产者完成
    await producer_task
    # 2. 等待队列清空
    await queue.join()
    # 3. 发送退出信号（每个 worker 一个）
    for _ in workers:
        await queue.put(None)   # None 作为哨兵
    # 4. 等待所有 worker 退出
    await asyncio.gather(*workers)

# worker 中处理：
async def worker(queue, name):
    while True:
        task = await queue.get()
        if task is None:          # 收到打烊通知
            queue.task_done()
            break
        try:
            await process(task)
        finally:
            queue.task_done()
```
