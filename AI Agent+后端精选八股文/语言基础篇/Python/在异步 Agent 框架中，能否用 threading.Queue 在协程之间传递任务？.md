# 在异步 Agent 框架中，能否用 threading.Queue 在协程之间传递任务？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/AlB8wbqtViP7VYkS4VPcyXFPnof

#### ⚡ 一句话结论

原则上不要直接用，但如果你清楚自己在做什么，可以通过线程桥接的方式“蹭”一下。协程间通信，首选用 asyncio.Queue。

---

#### 🔍 1. 先看 threading.Queue 是什么

它本质是一个线程安全的 FIFO 队列，常用于多线程生产者-消费者模型。它的 put 和 get 是阻塞型调用：队列满了就卡住生产者，队列空了就卡住消费者。

```
代码块
import threading, queue, time
q = queue.Queue()
def producer():
    for i in range(3):
        q.put(i)          # 队列满时阻塞当前线程
        print(f"生产 {i}")
        time.sleep(0.1)
def consumer():
    while True:
        item = q.get()    # 队列空时阻塞当前线程
        print(f"消费 {item}")
        q.task_done()
```

🧵 这里阻塞的是操作系统线程，多线程场景下没问题，因为每个线程有自己的栈，一个线程卡住不影响其他线程。

---

#### ⚡ 2. asyncio 协程的运行方式完全不同

asyncio 是单线程内协作式调度。所有协程共享同一个线程，通过 await 主动让出控制权。事件循环就像一位不停轮询的管家，任何一个任务里如果出现同步阻塞，管家就被钉死在那里，其他任务全部饿死。

```
代码块
import asyncio, queue, time
async def main():
    q = queue.Queue()
    async def worker():
        while True:
            item = q.get()    # ❌ 阻塞了当前线程！
            print(item)
    asyncio.create_task(worker())
    await asyncio.sleep(0.1)  # 永远没机会执行
```

⚠️ 你会发现，q.get() 一调用，整个事件循环都停摆——它阻塞的不是协程，而是运行协程的那个唯一线程。这就是为什么 threading.Queue 不能直接用于协程间通信。

---

#### 🔄 3. 那硬要用怎么办？——桥接方案

某些复杂系统（比如既有同步 Agent 又有异步 Agent）确实需要混用队列。这时可以用 loop.run_in_executor 把阻塞操作丢给线程池，相当于给异步世界开一条通往同步世界的“小路”。

```
代码块
import asyncio, queue, concurrent.futures
q = queue.Queue()
async def async_producer():
    for i in range(5):
        await asyncio.get_event_loop().run_in_executor(
            None, q.put, i
        )
        await asyncio.sleep(0.1)
async def async_consumer():
    loop = asyncio.get_running_loop()
    while True:
        item = await loop.run_in_executor(None, q.get)
        print(f"异步拿到: {item}")
```
