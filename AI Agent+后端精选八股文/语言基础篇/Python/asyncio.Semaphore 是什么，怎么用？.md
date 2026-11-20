# asyncio.Semaphore 是什么，怎么用？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/YXquwprLri1WSLk7A7pcW9BqnYf

asyncio.Semaphore 是什么”时，我通常不会直接背定义，而是先扔一个场景——因为它就是为解决并发过载而生的，尤其在我们刚才聊过的 API 限流、工具调用那些场景里。

---

#### 🚦 一句话理解

asyncio.Semaphore 就是一个协程世界的红绿灯，它允许多个协程“同时运行”，但最多只放行固定数量。
你可以把它当成一个只有 N 个名额的洗手间，出来一个人，才能放进去下一个人。

---

#### 🧰 基本用法 & 核心方法

```
代码块
import asyncio

# 创建信号量，最多允许 3 个协程同时进入
sem = asyncio.Semaphore(3)
async def limited_task(name):
    async with sem:              # 获取名额，没有空闲就挂起等待
        print(f"{name} 进入")
        await asyncio.sleep(1)   # 模拟 IO 操作
        print(f"{name} 离开")
    # 出了 with 块自动释放名额
```

- semaphore = asyncio.Semaphore(value) ：value 是最大并发数
- await semaphore.acquire() ：尝试减一，如果内部计数器为 0 则挂起当前协程
- semaphore.release() ：加一，唤醒一个等待的协程
- async with semaphore ：等价于 acquire + try...finally release

重点：release() 可以在不同于 acquire() 的协程里调用，这点和 Lock 不同，使它非常灵活。

---

#### 🏊 典型场景一：限制并发 API 调用

这也是我们之前计算 RPM 时用到的那个信号量。比如调用一个允许同时 10 个请求的 API：

```
代码块
db_sem = asyncio.Semaphore(20)
async def query(sql):
    async with db_sem:
        async with pool.acquire() as conn:
            return await conn.execute(sql)
```

即使有 1000 个 URL，同时在飞的请求也永远不超过 10 个，不会打爆对方服务。

---

#### 🔗 典型场景二：限制数据库连接池

很多异步数据库驱动有连接数上限。与其让任务报错再重试，不如用信号量在协程层排队：

```
代码块
db_sem = asyncio.Semaphore(20)
async def query(sql):
    async with db_sem:
        async with pool.acquire() as conn:
            return await conn.execute(sql)
```

这样 100 个查询不会同时去抢 20 个连接，避免 Too many connections。

---

#### ⚠️ 和 Lock 的区别（常被问）

> （此处为内嵌电子表格，需登录飞书查看）

因为 Semaphore 不绑定所有权，你可以用它实现跨协程的流量控制，甚至把 release 当成“补充名额”的操作，比如动态调整并发数。
