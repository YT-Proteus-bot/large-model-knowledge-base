# Agent 需要同时处理用户实时请求（高优先级）和后台批量任务（低优先级），如何设计任务调度？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Lw1wwY4VNi9agikABZnc1qSnnyf

面试时遇到这个场景题，说明对方在考察你在资源有限的情况下，如何做任务的“阶级划分”和“优雅降级”。我通常会先用一个贴地的比喻切入，再拆解成具体可落地的调度架构。

---

#### 🚦 先理解本质：这不是功能需求，是资源分配的“政治问题”

你的 Agent 就像一个只有 3 个窗口的办事大厅：

- VIP 窗口（用户实时请求）：立刻响应，等久了会骂人。
- 普通窗口（后台批量任务）：可以排队，但不能永远不被叫号。

如果来一个 VIP 就插队，普通窗口的人会“饿死”；如果完全公平，VIP 的体验就崩了。所以我们要设计一套带优先级的、可抢占的、防饿死的调度规则。

---

#### 🧩 核心方案：双队列 + 优先级调度 + 动态并发隔离

别把所有任务丢进一个池子里搅和。分成两个逻辑队列，配合 asyncio.PriorityQueue 或双队列协调：

#### ⚡ 为什么不用单一 PriorityQueue？

单一优先级队列在纯协程模型里有坑：不能真正抢占。一个已经在运行的低优任务，不会因为队列后来插入高优任务而自动让出资源，必须等它主动 await。
所以我们用双队列 + 独立的 Worker 组，既能保证高优快速进入，又能通过动态调整 Worker 数量实现“软抢占”。

---

#### 🔧 代码骨架：用 asyncio 实现分层调度

```
代码块
import asyncio
from asyncio import Queue, Task
class PriorityScheduler:
    def __init__(self, max_concurrency: int = 10):
        self.max_concurrency = max_concurrency
        self.high_queue = Queue()       # 高优任务
        self.low_queue = Queue()        # 低优任务
        self._running = False
    async def _worker(self, worker_id: int):
        """每个 worker 循环：优先取高优，无高优再取低优"""
        while self._running:
            # 优先尝试高优队列，用 get_nowait 非阻塞检查
            try:
                task = self.high_queue.get_nowait()
            except asyncio.QueueEmpty:
                # 高优空，取低优（可阻塞等待，给低优任务保底）
                try:
                    task = await asyncio.wait_for(
                        self.low_queue.get(), timeout=0.5
                    )
                except asyncio.TimeoutError:
                    continue   # 没任务就继续循环
            try:
                await task()
            except Exception as e:
                # 错误处理 + 日志
                pass
            finally:
                # 标记任务完成
                if task in self.high_queue._queue:
                    self.high_queue.task_done()
                else:
                    self.low_queue.task_done()
    async def run(self):
        self._running = True
        # 启动固定数量的 workers
        workers = [asyncio.create_task(self._worker(i))
                   for i in range(self.max_concurrency)]
        # 动态调整高优槽位：用单独的协程监控
        asyncio.create_task(self._adaptive_reserve())
        await asyncio.gather(*workers)
    async def _adaptive_reserve(self):
        """定期调整，保证低优任务不被饿死"""
        while self._running:
            # 这里可以根据低优队列长度和等待时间动态提升低优 worker 数
            await asyncio.sleep(1)
```
