# Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/G3OYw0oIWi5bH6kWkHxc4hPanff

“一听这个需求，我脑子里的第一反应不是引入 Celery 或者 Airflow 那种大家伙，而是用 Redis 的有序集合（Sorted Set）手搓一个轻量级延迟队列。Agent 的定时任务通常就几个，比如清理过期会话、定时拉取数据源，为这个去搭一套分布式调度框架，运维成本比业务逻辑还重。所以 Redis ZSET 的方案，几分钟就能跑起来。”

---

#### 🧠 一、核心设计：把“定时”转成“排序”

“核心思路很简单：把任务的执行时间戳作为 score，任务信息作为 member，塞进一个 ZSET 里。然后有个 Worker 或者 Agent 自己的主循环，不断去取 score 小于等于当前时间戳的任务，拿出来执行。”

##### 🗺️ 逻辑示意图

```
代码块
[ 任务生产者 ]
      │
      ▼
   ZADD tasks:scheduled 1719600000 "task:cleanup"
   ZADD tasks:scheduled 1719603600 "task:report"
      │
      ▼
┌─────────────────────────┐
│  Redis Sorted Set       │
│  Key: tasks:scheduled   │
│  ┌──────────────┬─────┐ │
│  │ Score (执行时间)│ Member (任务ID/Payload) │
│  ├──────────────┼─────┤ │
│  │ 1719600000   │ "task:cleanup" │
│  │ 1719603600   │ "task:report"  │
│  └──────────────┴─────┘ │
└─────────────────────────┘
      │
      ▼   ZRANGEBYSCORE 取到期任务
┌─────────────────┐
│   轮询 Worker   │  取出后 ZREM 原子删除，防止重复
└─────────────────┘
      │
      ▼  执行具体业务逻辑
[ Agent 执行任务 ]
```

这一步的巧妙之处：ZSET 天然就是按 score 排序的，我们不需要自己维护一个优先队列，Redis 帮你排好了。取到期任务就是一条命令：ZRANGEBYSCORE key -inf current_timestamp。

---

#### 🔧 二、一步步搭出来，连命令都给你

1. 添加定时任务（生产者）

```
代码块
ZADD tasks:scheduled 1719600000 "cleanup::sessions"
```

如果你需要传参数，就把 JSON 字符串当 member：

```
代码块
ZADD tasks:scheduled 1719603600 '{"type":"cleanup","target":"sessions","ttl":3600}'
```

1. Worker 取出到期任务（消费者）
   不能直接 ZRANGEBYSCORE 然后执行，因为可能会有多个 Worker 实例抢同一个任务。所以要用原子操作把“取出”和“删除”合成一步：

- 推荐方案：Redis 的 Lua 脚本，原子性地取出并删除所有到期任务：

```
代码块
local tasks = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1])
if #tasks > 0 then
    redis.call('ZREM', KEYS[1], unpack(tasks))
end
return tasks
```

- 这个脚本传入当前时间戳，返回这批到期任务列表。Worker 拿到的就是“已领取”的任务，其他 Worker 不会再抢到。
1. 主循环逻辑（伪代码）

```
代码块
while agent_running:
    current_ts = int(time.time())
    # 原子取出并删除到期任务
    tasks = redis.eval(LUA_SCRIPT, 1, "tasks:scheduled", current_ts)
    for task in tasks:
        execute_task(task)   # 这里可以开协程处理，避免阻塞
    time.sleep(0.5)   # 轮询间隔，Agent 场景 0.5-1 秒足够
```
