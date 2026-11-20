# 请解释 asyncio 事件循环的工作原理，以及如何在 Agent 中正确使用 asyncio.gather 并发调用 LLM、用 Semaphore 防止 Rate Limit？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/N020wxCbWiGWtfkE18Uchwhwnyg

🔷 1. 事件循环（Event Loop）到底在干什么？

你可以把事件循环理解成一个永不停歇的“调度员”，只在一个线程里工作。

- 🔄 它手里握着两个东西：一个任务队列、一个 I/O 多路复用器（比如 epoll）。
- 📋 任务队列里排着各种协程任务。事件循环每次从队列里取出一个，跑一小段，直到碰到 await。
- ⚡ 一碰到 await 一个 I/O 操作，协程就会暂停，并且告诉事件循环：“我要等这个 socket 可读/可写，注册进去”。然后事件循环立刻切换去跑别的任务。
- 📡 内核 I/O 复用器在后台监听所有注册的事件。哪个就绪了，事件循环就把它对应的协程重新扔回任务队列，等下次调度继续跑。
- 🧩 因为所有这些都在一个线程里，没有锁的开销，核心就是“遇到等待就切走，不浪费 CPU 空转”。

说白了：单线程 + 协作式多任务 + 回调驱动 I/O。这就是为什么 asyncio 适合 IO 密集型，而不适合 CPU 密集型。

---

🧩 2. 在 Agent 中用 asyncio.gather 并发调 LLM

Agent 经常要做一件事：同一个问题拆成 3 份，同时调 3 个 LLM 服务，或者并行检索 + 推理。串行太慢，asyncio.gather 就很关键。

- 🎯 把每个 LLM 调用封装成一个 async 函数，比如 async def call_llm(prompt, semaphore): ...。
- 🔧 然后 results = await asyncio.gather(call_llm(p1, sem), call_llm(p2, sem), call_llm(p3, sem))。
- 🧠 这样做的好处：三个请求几乎同时发起，http 请求一发出就 await 挂起，事件循环会并行等待三个响应，总耗时近似最慢那个请求的时长，而不是三倍。

✅ 实战中的小技巧：gather 默认一个任务抛异常，其他任务会被取消。一般会加上 return_exceptions=True，这样单个 LLM 超时或报错不影响其余结果，最后手动过滤。

---

🚦 3. 用 Semaphore 防止 Rate Limit

LLM API 都是有 Rate Limit 的，比如 60 RPM，一并发就是几十个请求，很容易 429 被打。

- 🛂 asyncio.Semaphore 就是一个内部计数器。设定 Semaphore(5)，意思是同时最多允许 5 个协程进入保护区。
- ⛓️ 在 LLM 调用函数里用 async with semaphore: 包裹真正的 HTTP 请求。信号量内部计数减 1，如果计数为 0，后来协程就在 with 这里挂起，等前面释放。
- ⏳ 效果：无论你 gather 了 100 个任务，同时真正发出的 HTTP 请求永远只有 5 个，其余的都排队。这就用背压控制把并发限制在了 API 允许的窗口内。

结合使用示例结构（伪代码风格）：

```
代码块
sem = asyncio.Semaphore(5)
async def call_llm(prompt):
    async with sem:
        # 实际 HTTP 请求，比如 httpx.AsyncClient.post
        resp = await client.post(url, json={"prompt": prompt})
        return resp.json()

# Agent 并发入口
async def run_agent(prompts):
    tasks = [call_llm(p) for p in prompts]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # 处理结果、重试429等
    return [r for r in results if not isinstance(r, Exception)]
```

⚠️ 注意 Semaphore 要在 gather 外边创建，然后传给同一个 sem，否则每个协程独立计数就没意义了。

---

🔚 最后，这套组合拳的本质就是 “用有限的并发窗口，去消化任意大的任务列表”。面试时如果能再提一句“其实还可以配合 asyncio.Queue 做生产者-消费者式的动态调度”，基本就是加分项了。
