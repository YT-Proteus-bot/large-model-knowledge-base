# Agent 需要实现“检查用户配额 → 扣减配额”原子操作，为什么要用 Lua 脚本？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/GC1vwslxmim2nykCEIncjNYsnre

“这个场景太经典了，几乎所有 Agent 系统都会遇到。比如用户每天只能调用天气 Skill 10 次，调用前必须先检查还能不能调，然后扣一次。这两个动作如果分开执行，在高并发下就是经典的检查-修改竞态，一定会出问题。用 Lua 脚本就是把这两个动作焊在一起，让 Redis 单线程一次性执行完，彻底消除并发窗口。

我把为什么必须用 Lua、不用会怎样、以及 Agent 场景的特殊要求分三层说清楚。

一、不用 Lua 的竞态灾难

最简单的做法是分两步：

1. 检查：GET user:123:weather_quota → 返回 3（还有 3 次）
1. 扣减：DECR user:123:weather_quota → 变成 2

在单线程下没问题，但 Agent 是高并发的。假设两个 Agent 同时为用户服务，都读到了 3，都判断“还有配额”，然后都执行 DECR，结果配额从 3 变成了 2，实际只应该扣两次却扣了两次，用户多用了服务。这就是丢失更新。

二、Lua 脚本如何原子化

把整个逻辑写进 Lua 脚本，交给 Redis 执行：

```
代码块
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")
if current < limit then
    redis.call('INCR', key)
    return 1  -- 成功扣减
else
    return 0  -- 配额已满
end
```

执行命令：

```
代码块
EVAL "上述脚本" 1 user:123:weather_quota 10
```

Redis 执行 Lua 脚本时，整个脚本会以原子方式执行，中间不会插入任何其他命令。因为 Redis 是单线程执行命令的，脚本一旦开始，就会一口气跑完，其他客户端的所有请求都排队等着。这样就保证了检查配额和扣减之间的窗口被彻底关闭。

三、Agent 场景的特殊要求

Agent 场景下这个原子操作还需要考虑几个额外点：

1. 带 TTL 的配额重置：用户配额通常是周期性的，比如每天 10 次。如果 key 不存在，说明新周期开始，需要先设置配额上限并设置 TTL 到当天结束。这些也可以封装在同一个 Lua 脚本里：

```
代码块
local exists = redis.call('EXISTS', key)
if exists == 0 then
    redis.call('SET', key, 1, 'EX', ttl_seconds)
    return 1
end
-- 否则继续检查逻辑
```

1. 返回值需要丰富：Agent 不仅需要知道成功或失败，还需要知道剩余配额，以便生成更友好的用户提示。脚本可以返回 {status, remaining}，一次调用拿回所有需要的信息。
1. 降低网络开销：在 Agent 编排引擎中，一次任务可能触发多个 Skill，每个 Skill 都可能有配额检查。如果用传统方式，每次检查需要至少两次网络往返（GET + DECR），高并发下网络延迟会被放大。Lua 脚本合并为一次往返，在大规模 Agent 集群中对 Redis 的负载和延迟都有显著优化。
1. 与限流器的整合：配额检查常常和限流器放在一起。用同一个 Lua 脚本既扣配额又更新滑动窗口计数器，既保证原子性，又避免了多个 Redis 命令的编排复杂度和网络消耗。

四、什么时候不适合用 Lua？

Lua 脚本也有一些限制需要注意：

- 脚本执行时间过长会阻塞其他请求。配额扣减这种 O(1) 操作完全没问题，但如果在脚本里做复杂计算或者循环遍历大量 key，就要拆开或用 EVAL_RO。
- 集群模式下，脚本操作的 key 必须在同一个 slot。配额 key 通常按用户 ID 哈希，天然满足这个条件。
- 调试相对困难，需要把脚本的幂等性和边界条件提前测试好。
