# contextvars.ContextVar 解决了什么问题？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/B6Bew2s7Ti0ALeksCo1cALK1nte

我们来拆这道题。面试官问 contextvars.ContextVar 解决了什么，其实是在看你对异步上下文传递的痛点有没有体感。我试着把逻辑链串起来，尽量不堆概念，从真实踩坑出发。

---

#### 🧩 先看一个非常常见的需求

在 Web 服务里，一个请求进来，我们想给这次请求分配一个唯一 request_id，然后在日志、数据库查询、下游调用中都能带上它，便于链路追踪。

最笨但最干净的办法：把 request_id 作为参数一层层传下去。

```
代码块
def handle(request_id):
    log(request_id, "开始处理")
    query_db(request_id, "SELECT ...")
```

这在小项目里没问题。但一旦中间件、ORM、缓存、消息队列都参与进来，每个函数签名都要改，很快失控。

于是我们想：能不能把这种贯穿整个调用链、又不改变业务函数签名的数据，放在一个“全局又只属于当前请求”的地方？

---

#### 🧵 threading.local 的旧方案及其崩溃点

在纯同步多线程时代，threading.local() 就是标准答案：

```
代码块
import threading
local = threading.local()
def get_request_id():
    return getattr(local, 'request_id', 'unknown')
```

每个线程有自己独立的 local 命名空间，线程 A 的 request_id 不会串到线程 B，完美。

但到了异步时代，事情变了。 在一个线程内，可能同时存在多个协程（或 Task），它们交替执行。如果还用 threading.local，协程 A 在 await 处暂停，协程 B 被调度进来，把 local.request_id 覆盖掉；等协程 A 恢复执行时，读到的就是协程 B 的值 —— 灾难性串台。

```
代码块
时间 ──────────────────────────────►
Task A:  set(request_id='A') → await → 读 request_id  → 期望 'A'，实际 'B' ❌
Task B:                  set(request_id='B') → await ...
```

这就是所谓的协程不安全的上下文状态。

---

#### ✨ ContextVar：为异步重新设计的“当前上下文”变量

contextvars.ContextVar（Python 3.7+）就是专门解决这个问题的。它把“上下文”的概念从线程细化到了异步 Task。每个 Task 有自己独立的上下文栈，ContextVar 的读写作用在当前 Task 的当前上下文里。

- 不同 Task 自动隔离：Task A 的变量修改不会污染 Task B。
- Task 内调用链自动继承：当你在一个协程里 await 另一个协程，当前上下文会被复制给子任务，子任务可以读取、甚至修改（修改不影响父上下文，通过 token 机制实现回滚，但通常我们利用浅拷贝即可）。
- 不受线程调度影响：无论多少协程在同一个线程里交替执行，各自上下文都被妥善保存和恢复。

用图表达就是：

```
代码块
        Task A 上下文                Task B 上下文
   ┌─────────────────┐       ┌─────────────────┐
   │ request_id='A'  │       │ request_id='B'  │
   │ user='Alice'    │       │ user='Bob'      │
   └─────────────────┘       └─────────────────┘
               隔离，互不干扰
```

#### 🔧 极简对比代码

threading.local（异步下有问题）

```
代码块
import asyncio, threading
local = threading.local()
async def set_and_read(name):
    local.v = name
    await asyncio.sleep(0.1)
    print(f"{name}: {local.v}")   # 可能读到别的协程的值
async def main():
    await asyncio.gather(set_and_read("A"), set_and_read("B"))
```
