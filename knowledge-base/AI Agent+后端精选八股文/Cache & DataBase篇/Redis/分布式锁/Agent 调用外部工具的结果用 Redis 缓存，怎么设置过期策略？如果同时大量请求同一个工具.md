---
title: Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具结果，怎么防止缓存击穿？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/ZX1RwDO8cijY6ikIujccn17lnJc
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具.md
questions: 0
chars: 1576
build_date: '2026-11-20'
images_removed: 0
---

# Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具结果，怎么防止缓存击穿？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具.md

“缓存策略的核心是平衡时效性和资源保护。过期设得太长，数据陈旧影响决策；设得太短，高并发下工具被打爆。防击穿则是加一把分布式锁，让失效瞬间只有一个请求能去拉数据，其他等待或降级。我分两块来讲。

## 一、过期策略：分级 TTL + 随机偏移 + 主动预热

我不会一刀切设一个固定值，而是按工具类型和业务容忍度分成三级：

- 时效敏感型（如实时股价、当前温度）：TTL 设 30 秒 ~ 2 分钟，且带上随机偏移 ±10%。比如股价设 60 秒 ±6 秒，避免大量缓存在同一秒失效引发雪崩。
- 相对稳定型（如航班时刻表、酒店列表）：TTL 设 5 ~ 15 分钟，但会在 TTL 过半时，异步触发一个后台任务主动刷新缓存。这样用户请求拿到的是最新值，且极少遇到真正过期回源。
- 基础信息型（如城市列表、固定规则）：TTL 设 1 小时甚至更长，可以用 @Cacheable 配合 Spring Cache，极少变动。

同时，所有缓存 Key 都携带数据版本号或时间戳前缀，方便灰度升级时一键失效。比如 tool:weather:v2:beijing，发布新版本工具时换 v3 即可。

## 二、防缓存击穿：互斥锁 + 逻辑过期 + 降级兜底

缓存击穿是某个热点 Key 过期瞬间，大量 Agent 同时去调同一个外部工具。我们分三层防线：

1. 分布式互斥锁（主防线）

   Agent 发现缓存没有命中，先用 Redis 的 SET lock:key unique_id NX PX 5000 争抢一把锁。抢到锁的线程去调工具并把结果写回缓存；没抢到的线程自旋等待一小段时间（如 50ms）后再查缓存，而不是直接穿透到工具。如果锁一直不释放（抢到锁的线程挂了），锁本身有 5 秒的过期时间，不会死锁。伪代码：

```
def get_tool_result(key):
    val = redis.get(key)
    if val: return val
    lock_key = f"lock:{key}"
    if redis.set(lock_key, "1", "NX", "PX", 5000):
        try:
            val = call_external_api()
            redis.set(key, val, "PX", ttl_ms())
            return val
        finally:
            redis.delete(lock_key)
    else:
        # 没抢到锁，短暂休眠后重试读缓存（最多重试 3 次）
        for _ in range(3):
            sleep(0.05)
            val = redis.get(key)
            if val: return val
        # 重试耗尽，返回降级数据或空
        return fallback_value
```

1. 逻辑过期（热点 Key 特殊处理）

对于必须极致低延迟且容忍几秒旧数据的热点（比如首页推荐），我会给缓存值包装一层，包含逻辑过期时间戳和真实数据。读取时发现逻辑过期，先返回旧数据给用户，同时异步起一个线程去刷新缓存，用户无感知。这避免了加锁带来的串行等待。

1. 降级兜底与熔断

如果工具调用持续失败（比如上游宕机），互斥锁也不能解决问题，反而会让所有请求堆积。此时缓存可以记录最后一次成功的结果，在工具不可用时，即使逻辑过期了也继续返回这个“陈旧但可用”的数据，同时触发告警。这就是缓存的“救命”模式。
