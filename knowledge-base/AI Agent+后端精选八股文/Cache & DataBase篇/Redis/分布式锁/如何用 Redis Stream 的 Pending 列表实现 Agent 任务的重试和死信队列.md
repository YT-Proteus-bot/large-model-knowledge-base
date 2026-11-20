---
title: 如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/DVV6wXmfBiwIeCkTaTEcS5bYnkb
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列.md
questions: 0
chars: 1541
build_date: '2026-11-20'
images_removed: 0
---

# 如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列.md

“Redis Stream 的 Pending 列表是我做 Agent 任务可靠性保障的核心工具。它天然记录了‘已分发但未确认’的消息，这个特性直接映射到我们的重试和死信机制。我分四层说清楚：消息的生命周期、如何利用 Pending 做重试、如何从重试升级到死信、以及完整流程。

## 一、消息的四个生命周期阶段

在 Redis Stream 消费者组里，一条消息经历四个阶段：

1. 就绪（Stream 中）：消息通过 XADD 写入，等待消费者拉取。
1. 待确认（Pending）：消费者通过 XREADGROUP 拉取后，消息进入该消费者的 Pending 列表，状态变为已分发但未确认。
1. 完成（ACK）：消费者处理成功后调用 XACK，消息从 Pending 移除。
1. 异常（Pending 超时）：消费者崩溃或处理失败，消息一直留在 Pending 中，需要其他消费者接管。

## 二、用 Pending 实现重试：XAUTOCLAIM 认领超时消息

重试的核心是检测哪些消息卡在 Pending 太久了，把它们认领回来重新处理。

我们部署一个独立的监控 Agent（或每个消费者自带一个后台任务），定期扫描所有 Stream 的 Pending 列表：

1. 用 XPENDING stream group 获取所有待确认消息的概览，包括最小/最大空闲时间、每个消费者的 Pending 数量。
1. 用 XAUTOCLAIM stream group consumer min-idle-time count 认领那些空闲时间超过阈值（比如 30 秒）的消息。这个命令会把这些消息的所有权转移给当前消费者，并返回消息列表。
1. 认领成功后，当前消费者重新处理这些消息，处理完正常 XACK。

这样，如果一个 Agent 在处理消息时崩溃，或者业务逻辑失败没有 ACK，消息不会丢。30 秒后会被另一个健康的 Agent 捡起来重试。这比手动 ACK 超时检测要可靠得多。

重试策略放在哪里？

我们不在 Redis 里做复杂的退避调度，而是在 Agent 的消费逻辑里实现：

- 每次重入的消息体里，我们会嵌入一个 retry_count 字段和 max_retries。消费端先递增这个计数，再执行任务。
- 退避策略（固定间隔、指数退避）用 Thread.sleep 或延迟队列，如果不想阻塞消费者线程，就把重试任务重新投递到一个延迟 ZSet 队列，等延迟过后再写回 Stream。

## 三、从重试到死信：重试上限后的转移

不是所有失败都值得无限重试。当 retry_count 达到上限（比如 3 次），我们就认为这个任务在当前条件下无法成功，把它转入死信队列：

- 死信队列也是一个 Redis Stream，比如 dead:task:stream。
- 转移过程用 Lua 脚本保证原子性：将消息从原 Stream 的 Pending 中 XACK 掉，同时 XADD 到死信 Stream，并附带原始错误堆栈、重试次数、失败时间等诊断信息。
- 死信队列有独立的消费者组，可以是人工处理的后台，也可以定时批量重放。

## 四、处理中的幂等和重复

因为 XAUTOCLAIM 认领后消息可能被处理多次（比如消费者 A 正在处理，但网络慢，监控 Agent 误认为超时并把消息认领给了 B），所以消费端必须做好幂等：

- 每条消息有一个业务唯一键（如 task_id），用 Redis Set 存已成功处理的任务 ID（设 TTL）。
- 消费前检查 Set，若已存在则直接 XACK 跳过，不再执行业务逻辑。
