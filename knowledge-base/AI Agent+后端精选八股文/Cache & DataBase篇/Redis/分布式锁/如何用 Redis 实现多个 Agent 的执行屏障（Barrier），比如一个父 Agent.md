---
title: 如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent 等待所有子 Agent 结果返回后再继续？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FgDOw2pqCiNznKkLhTKcsuU8nRh
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent.md
questions: 0
chars: 1851
build_date: '2026-11-20'
images_removed: 0
---

# 如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent 等待所有子 Agent 结果返回后再继续？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent.md

这个场景本质就是分布式的 CountDownLatch。父 Agent 是等待者，子 Agent 是倒计数者，Redis 在这里充当一个原子计数器和信号广播器。我不会用复杂的锁，而是用计数器 + 发布订阅（或阻塞队列）组合，既轻量又可靠。

## 一、核心数据模型

每个并行任务批次有一个唯一 ID task_batch_id，在 Redis 里存两个 Key：

- barrier:{batchId}:total — String，存子任务总数，创建时写入，不更改。
- barrier:{batchId}:done — String，存已完成子任务计数，初始 0。

## 二、计数器 + 发布订阅的实现流程

1. 父 Agent 分配任务前：
- 计算子任务数量 N。
- SET barrier:{batchId}:total N
- SET barrier:{batchId}:done 0
- 给这两个 Key 设置一个过期时间（比如 5 分钟），防止任务异常导致 Key 永久残留。
- 父 Agent 订阅频道 barrier:{batchId}:complete，然后 SUBSCRIBE 阻塞等待。
1. 每个子 Agent 完成任务后：
- 执行 Lua 脚本，原子地递增完成计数并判断是否全部完成：

```
EVAL "
  local done = redis.call('INCR', KEYS[1])
  local total = tonumber(redis.call('GET', KEYS[2]))
  if done == total then
    redis.call('PUBLISH', KEYS[3], 'all_done')
  end
  return done
" 3 barrier:{batchId}:done barrier:{batchId}:total barrier:{batchId}:complete
```

- 这个脚本保证“递增 -> 检查 -> 发布”一气呵成，不会有竞态。
1. 父 Agent 被唤醒：
- 从 SUBSCRIBE 拿到 all_done 消息后，再 GET barrier:{batchId}:done 确认是否等于 N（双重保险）。
- 确认后清除屏障 Key，汇总结果继续执行。

三、为什么不用轮询？

用 SUBSCRIBE 是推送模式，父 Agent 不占 CPU，实时性也好。如果用 BLPOP 从队列取完成信号，需要父 Agent 提前知道由哪个子 Agent 来发最后一条，但子 Agent 是无序的，无法确定“最后一名”。计数器天然解决了无序完成的问题。

## 四、容错设计：防永远等待

并行任务最大的风险是某个子 Agent 挂了，父 Agent 永远等不到“全部完成”。我加了三层防护：

1. Key 过期兜底：barrier 相关的 Key 设 5 分钟 TTL。5 分钟后 Redis 自动清空，父 Agent 在 SUBSCRIBE 之外加一个 5 分钟超时的计时器，超时后主动检查 barrier:{batchId}:done，如果小于 N，就认为有子任务失败，启动重试或降级。
1. 心跳与重分配：子 Agent 执行期间，每隔 10 秒更新自己的“子任务心跳 Key”（如 task:{taskId}:heartbeat）。父 Agent 或监控 Agent 检测到心跳丢失，就将该子任务重新分配给另一个子 Agent，并更新 total 和 done 的对应逻辑（稍微复杂，通常用任务状态表维护）。
1. 幂等完成：如果子 Agent 完成消息被重复发送（比如网络闪断），INCR 已经保证计数只会递增一次，不会重复计数。done 永远不会超过 total。

## 五、更简单的替代方案：Redis Stream

如果子 Agent 的结果也需要收集，可以直接用 Redis Stream 的消费者组 + 计数读取。父 Agent 开一个消费者组，子 Agent 把结果 XADD 到 Stream，父 Agent 用 XREADGROUP 一次取 N 条消息，阻塞等待直到拿到 N 条。这省去了显式的屏障 Key，但需要提前知道子 Agent 数量，且适合结果收集场景。我更推荐计数器方案，因为它更轻，且不要求子 Agent 的结果统一进流。
