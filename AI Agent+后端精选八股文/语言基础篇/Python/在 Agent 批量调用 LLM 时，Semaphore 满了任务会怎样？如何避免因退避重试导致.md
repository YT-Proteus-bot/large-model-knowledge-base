# 在 Agent 批量调用 LLM 时，Semaphore 满了任务会怎样？如何避免因退避重试导致槽位被长期占用？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/IVKHwUmNFitAH6ktzCtcTiWjnze

面试官追问到这一层，通常是在验证你有没有真的在生产环境里处理过背压和资源泄漏。我一般会先画个简单的状态图，把“等槽位”和“占着槽位不拉”分清楚。

---

#### 🔒 Semaphore 满了，任务到底在干嘛？

假设我们设了 asyncio.Semaphore(5)，同时有 8 个协程试图进入：

```
代码块
sem = asyncio.Semaphore(5)
async def task():
    async with sem:          # ← 第6个及之后会挂在这里
        await call_llm()
```

- 前 5 个协程立刻拿到槽位，开始执行 LLM 调用。
- 第 6~8 个协程会被挂起，它们的状态变成“等待信号量”，事件循环知道它们正在等 sem.acquire()。
- 它们不占用 CPU，不会阻塞其他协程，只是排在一个 FIFO 的等待队列里。

用图表示就是：

![image](../../../../assets/ZrTadXZvLoRCxXxFm6Ec0maGnBb/EpDIbnF9woINgyxDL2zctRpRnnN.png)

任一槽位释放，等待队列最前面的协程立即被唤醒，拿到槽位继续执行。这套机制本身是健康的。

---

#### 🐍 问题出在哪：重试退避把槽位“腌”住了

结合 tenacity 做重试时，最容易踩的坑是退避等待发生在信号量内部

```
代码块

# ❌ 有问题的写法
@retry(wait=wait_exponential(min=1, max=30))
async def call_with_retry(payload):
    async with sem:                     # 拿到槽位
        response = await client.post(url, json=payload)
        if response.status_code == 429:
            raise RateLimitError        # 触发 tenacity 等待
        return response
```

tenacity 的 wait 会在协程内做 await asyncio.sleep(...)。此时信号量并没有释放——协程只是睡在那里，槽位却被空占着。如果几个协程同时被限流，它们会各占一个槽位集体睡觉，外面的任务全部饥饿。

实际效果：并发数从 5 跌成 2 甚至 0，明明还有大量请求排队，但槽位都在“静坐”。

---

#### 🔧 解法一：重试期间释放槽位（推荐）

思路很简单：退避等待时不持有信号量，重试时再重新获取。

```
代码块
async def call_with_smart_retry(payload):
    for attempt in range(MAX_RETRIES):
        async with sem:                      # 每次尝试都排队取号
            response = await client.post(url, json=payload)
            if response.status_code == 429:
                retry_after = response.headers.get("Retry-After", 1)
            else:
                return response
        # 离开 with 块，信号量已释放，在外面等待
        await asyncio.sleep(float(retry_after) * (2 ** attempt))
    raise RuntimeError("Max retries exceeded")
```
