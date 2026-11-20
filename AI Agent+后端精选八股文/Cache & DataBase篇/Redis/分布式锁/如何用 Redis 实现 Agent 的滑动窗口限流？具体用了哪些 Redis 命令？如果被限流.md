# 如何用 Redis 实现 Agent 的滑动窗口限流？具体用了哪些 Redis 命令？如果被限流，你是返回错误还是放入等待队列？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/VYNywbPh9iHtNNkI7Tgc13lWnpb

“滑动窗口限流是保护 Agent 和下游工具的第一道防线，比固定窗口更平滑，不会出现‘边界突刺’——比如上一秒和下一秒各放行大量请求。核心思路是：用一个 ZSET 维护每次请求的时间戳，每次请求进来先清理掉窗口外的旧记录，再统计当前窗口内的请求数，超过阈值就限流。

一、具体实现和 Redis 命令

我以一个 Skill 每分钟最多 10 次调用为例，具体步骤是：

1. 每个用户对该 Skill 的限流用一个 ZSET，Key 为 rate:skill:weather:user:123。
1. 当用户发起调用时，拿到当前毫秒时间戳 now。
1. 执行以下 Lua 脚本，保证整个逻辑原子化：

```
代码块
-- KEYS[1]: ZSET Key
-- ARGV[1]: 当前时间戳 (毫秒)
-- ARGV[2]: 窗口长度 (毫秒), 比如 60000 表示 1 分钟
-- ARGV[3]: 限流阈值
-- ARGV[4]: 唯一请求 ID (用于去重和幂等)
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local request_id = ARGV[4]
-- 1. 删除窗口外的旧请求（滑动窗口的核心）
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
-- 2. 统计当前窗口内的请求数
local count = redis.call('ZCARD', key)
if count < limit then
    -- 3. 未超限，加入当前请求时间戳
    redis.call('ZADD', key, now, request_id)
    -- 4. 给 Key 设置过期时间，略长于窗口，避免冷数据占用内存
    redis.call('PEXPIRE', key, window + 1000)
    return 1  -- 放行
else
    return 0  -- 限流
end
```

这里用到的 Redis 命令就几个：

- ZREMRANGEBYSCORE：删除窗口外的旧条目，滑动窗口的关键。
- ZCARD：获取当前窗口内的请求数量。
- ZADD：把当前请求时间戳写入。
- PEXPIRE：给 Key 设一个略长于窗口的过期时间，防止无用 Key 堆积。

整个脚本在 Redis 里单线程执行，天然原子，不会出现竞态。

二、为什么不用固定窗口的 INCR + EXPIRE？

固定窗口（比如用 INCR rate:key + EXPIRE）的问题是边界突刺。比如窗口是 1 分钟，用户在 11:00:59 秒发 10 个请求，又在 11:01:01 秒发 10 个请求，两个窗口分别放行，实际 2 秒内通过了 20 个请求，限流形同虚设。滑动窗口用 ZSET 严格按时间戳计数，不认分钟边界，只要过去 60 秒内超过 10 个就限，丝滑得多。

三、被限流后怎么处理？

Agent 场景下我优先返回错误，绝不内部排队等待。原因有三：

- Agent 是同步等待用户的，如果限流后排队，用户会干等很久，体验极差，而且排到一半可能任务上下文都过期了。
- 排队会引发雪崩，大量请求堆积在 Agent 内存队列里，瞬间把堆内存打满。
- Agent 编排引擎本身已经有重试机制，收到限流错误后，可以根据任务配置决定是否自动重试、降级换另一个工具、或者生成一条友好提示给用户：“当前使用太频繁，请稍后再试”。

所以我的限流脚本返回 0 后，Agent 拿到这个信号，不走等待队列，而是直接抛出一个 RateLimitException，编排引擎捕获后根据 DAG 边的降级策略处理。降级方式可能是用缓存结果、提示用户，或者隔几秒后自动重试（重试间隔由编排层控制，不在 Redis 里排队）。
