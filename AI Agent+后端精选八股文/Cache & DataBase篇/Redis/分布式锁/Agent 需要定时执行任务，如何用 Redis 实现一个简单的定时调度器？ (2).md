# Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/XnzwwIAcqiWyIgk56ABcw9aFntc

先搭骨架：

“这个需求我一般会拆成三个动作：存任务 → 取任务 → 执行任务。Redis 里很多数据结构都能做，但最省心、最准确的还是有序集合（Sorted Set）。它天然按时间排序，还能原子操作，适合不需要引入 Celery 那种重型调度框架的场景。我画个简单的流程，然后一步步说透。”

---

#### 🧭 逻辑流图

```
代码块
                  ┌──────────────┐
                  │   任务生产者   │
                  │(Agent / 管理端)│
                  └──────┬───────┘
                         │  ZADD tasks:scheduled <未来时间戳> <任务负载>
                         ▼
              ┌──────────────────────┐
              │   Redis Sorted Set   │
              │  Key: tasks:scheduled│
              │  ┌─────────┬──────┐  │
              │  │ Score   │ Member│  │
              │  ├─────────┼──────┤  │
              │  │ 1719... │ 任务A │  │
              │  │ 1719... │ 任务B │  │
              │  └─────────┴──────┘  │
              └──────────┬───────────┘
                         │ 1. ZRANGEBYSCORE -inf now LIMIT 0 50
                         │ 2. ZREM 删除已取任务
                         ▼
              ┌──────────────────────┐
              │   Worker (轮询器)     │
              │   - 单线程循环        │
              │   - 协程/线程池执行   │
              └──────┬───────────────┘
                     │ 实际执行
                     ▼
              ┌──────────────┐
              │  Agent 业务   │
              │ (清理/拉取等) │
              └──────────────┘
```

这个流的核心就是一句话：把时间当权重，取到即删除，保证不重跑。

---

#### 🔧 具体实现细节（含命令级还原）

1. 添加任务
   直接用 ZADD，score 是执行时刻的 Unix 秒级时间戳，member 可以是任务类型或 JSON 字符串：

```
代码块
ZADD tasks:scheduled 1719660000 '{"action":"cleanup","session":"abc"}'
```

如果任务不需要复杂参数，直接用 action::session_id 这种简单字串也可以。

1. 轮询领取到期任务

这是整个调度器的灵魂。为了保证多个 Worker 实例并行时不抢同一个任务，必须用 Lua 脚本把“范围查询 + 批量删除”原子化：

```
代码块
-- KEYS[1] = tasks:scheduled, ARGV[1] = 当前秒级时间戳, ARGV[2] = 单次拉取上限
local tasks = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, tonumber(ARGV[2]))
if #tasks > 0 then
    redis.call('ZREM', KEYS[1], unpack(tasks))
end
return tasks
```

Worker 拿到返回结果后，就可以逐个（或并发）执行了。

为什么不用事务（MULTI/EXEC）？
事务不保证原子性中的“判断”，它只保证命令序列顺序执行，但你在事务里无法用 ZRANGEBYSCORE 的结果来决定后续的 ZREM 逻辑。Lua 脚本在 Redis 中执行时是阻塞且原子的，完美绕开竞态。
