---
title: 怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/R4CMw9qWhi4hEhkdtCicPerunha
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？.md
questions: 0
chars: 1806
build_date: '2026-11-20'
images_removed: 0
---

# 怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？.md

延迟队列是 Agent 异步协作的刚需。Redis 的 ZSET 天生就是延迟队列的完美底座：score 存执行时间戳，member 存任务信息，消费者按时间范围拉取到期任务，原子性地出队执行。这条链路非常成熟，但要在 Agent 场景下落好，还得解决好四个核心问题：原子消费、重试机制、多实例协作和任务可靠性。

我按设计步骤来说。

## 一、数据模型与入队

ZSET 的 Key 按任务类型隔离，比如 delay:skill_retry 存放 Skill 重试任务。每个任务序列化为 JSON，作为 member 存入 ZSET，score 是预期执行时间的 Unix 毫秒戳。

入队很简单：

```
ZADD delay:skill_retry 1719123456789 '{"taskId":"t-001","skillName":"weather","params":{...}}'
```

因为 ZSET 的 member 必须唯一，如果同一个任务可能重复入队，需要在 JSON 里带上唯一 ID 作为主键。如果是纯延迟任务（不要求去重），member 可以用 任务ID:时间戳 保证唯一性。

## 二、原子出队：Lua 脚本保证不重复消费

多 Agent 并发消费时，最大的坑是一个任务被两个消费者同时拿走。单纯用 ZRANGEBYSCORE 取出到期任务再 ZREM 不是原子的，中间会出竞态。我用 Lua 脚本把“查询到期任务 + 移除”打包成原子操作：

```
local items = redis.call('ZRANGEBYSCORE', KEYS[1], 0, ARGV[1], 'LIMIT', 0, ARGV[2])
if #items > 0 then
    redis.call('ZREM', KEYS[1], unpack(items))
end
return items
```

消费者以当前时间戳为上限，每次拉取最多 N 条到期任务。因为是原子操作，不会有多个 Agent 拿到同一个任务。拿到任务的 Agent 再解析 JSON 并执行。

## 三、多 Agent 协同消费：轻量级锁 + 消费者组

如果任务量很大，多个 Agent 都去轮询同一个 ZSET，轮询间隔太短 Redis 压力大，太长延迟高。优化方式：

- 使用 BLMOVE 的变通方案：ZSET 不支持阻塞操作，但可以用 BZPOPMIN 阻塞式弹出最小 score 的任务，不过它是单个成员弹出，吞吐量偏低。我更常用的是轮询 + 自适应间隔：消费到任务时密集轮询（间隔 50ms），没任务时指数退避到 500ms。
- 对于必须严格独占的任务（如重试扣款），可以用分布式锁包裹任务执行，但一般情况下 Lua 原子出队已经足够。

## 四、任务失败的重试机制：score 滚动

Agent 执行任务如果失败，需要重新入队等待重试。实现方式就是把任务的 score 改为 当前时间 + 退避延迟：

```
ZADD delay:skill_retry 新执行时间戳 任务JSON
```

退避策略可以是固定间隔（如 1 分钟），也可以是指数退避（1min → 2min → 4min）。为了防止无限重试堵塞队列，我在任务 JSON 里记录 retryCount，超过上限（如 3 次）就不再入队，而是转入 dead:skill_retry 死信队列，由人工或定时脚本处理。

## 五、可靠性加固：ACK 与超时回收

和消息队列不同，ZSET 出队即删除，没有显式的 ACK 机制。如果 Agent 拿到任务后自己崩溃了，任务就丢了。为了提升可靠性，可以引入备胎机制：

1. 出队时，不直接从 ZSET 删除，而是用一个 Lua 脚本将任务从 delay ZSET 原子地移动到 processing ZSET（score 为超时时间）。
1. Agent 执行完成后，从 processing 里 ZREM 掉任务，表示确认。
1. 一个独立的监控线程定期扫描 processing 中 score 小于当前时间的任务（说明执行超时），把它们重新移回 delay 队列或标记为失败。

这样即使 Agent 宕机，未完成的任务也会被回收重试，可靠性大幅提升。
