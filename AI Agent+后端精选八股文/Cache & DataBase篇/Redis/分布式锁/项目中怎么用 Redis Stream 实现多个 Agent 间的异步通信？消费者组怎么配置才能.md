# 项目中怎么用 Redis Stream 实现多个 Agent 间的异步通信？消费者组怎么配置才能保证一个任务只被一个 Agent 消费？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/M0y0w1kuQiX4TpkA09lcqrVwnNY

“Redis Stream 是我们 Agent 系统的消息中枢，专门解决多 Agent 间异步任务分发和协作。它比 Pub/Sub 多了持久化和消费者组，比专业 MQ 更轻量，非常适合 Agent 集群内部通信。

整个方案的核心是消费者组 + 消息确认，我来拆成三个层面说清楚。

一、为什么用 Stream 而不是 Pub/Sub

Agent 发出一条任务消息（比如‘Skill A 执行完毕，下一步需要 Skill B’），如果用的是 Pub/Sub，订阅者不在线消息就丢了，而且没法控制多个 Agent 实例谁处理这条消息。Stream 解决了这两个致命问题：

- 消息持久化：写入 Stream 的消息会保留，不因消费者离线而丢失。
- 消费者组：同一个组里的多个 Agent 实例，每条消息只会被一个实例消费，天然支持负载分担和故障恢复。

二、消费者组怎么保证独占消费

我设计消费者组时，严格遵循 Redis 的 XREADGROUP 语义：

1. 创建消费者组：在 Stream 创建后，用 XGROUP CREATE 命令初始化一个组，名字例如 agent_skill_executor，从最早的消息开始消费 0-0。
1. 消费消息时指定组和消费者名：每个 Agent 实例在消费时，必须提供组名和唯一的消费者标识（比如 agent_host_pid）。Redis 保证一条消息在组内只会分发给一个消费者，不会重复投递。
1. 消费确认（XACK）：实例收到消息后，处理完成后必须调用 XACK 确认。若实例在处理期间崩溃，消息会留在该消费者的 PEL（待处理条目列表）中。组内其他消费者可以通过 XCLAIM 或 XAUTOCLAIM 把超时未确认的消息认领过来处理，防止消息卡死。
1. 独占保证体现在：同一消费者组内，一条消息只出现在一个消费者的 PENDING 列表中，不会被其他消费者同时拿到。这就实现了任务独占。

三、架构设计：两个 Stream 实现双向通信

实际场景中，两个 Agent 或者编排引擎与 Skill 执行器之间通常需要双向异步通信。我设计成两个 Stream：

- 任务流（task:stream）：编排引擎发布任务，多个 Skill 执行器 Agent 作为消费者组去抢任务。
- 结果流（result:stream）：Skill 执行器处理完后，把结果写回结果流，编排引擎消费结果并继续编排。

消费者组配置示例（用 Redis 命令模拟）：

```
代码块

# 创建消费者组
XGROUP CREATE task:stream agent_executors $ MKSTREAM

# Agent 实例消费（伪代码）
XREADGROUP GROUP agent_executors agent-1 COUNT 1 BLOCK 5000 STREAMS task:stream >
```

四、关键配置防止消息丢失和重复

Redis Stream 的 at-least-once 语义下，我们还需要考虑幂等性和故障恢复：

- 消息 ID 幂等：每条消息在应用层带唯一任务 ID（如 task_uuid），消费端根据任务 ID 做幂等。这样即便消息被重复投递（因 XACK 丢失），也不会重复执行。
- XAUTOCLAIM 的合理使用：设一个 worker 每 10 秒扫描 PEL，把超过 30 秒未确认的消息通过 XAUTOCLAIM 认领给其他消费者，并重试。同时限制重试次数，超过上限就转入死信 Stream 人工处理。
- MAXLEN 限制：用 XADD ... MAXLEN ~ 10000 限制 Stream 长度，防止内存无限增长。
- 合理的 BLOCK 时间：消费时设 BLOCK 5000 毫秒，长轮询既省 CPU 也保证实时性。

五、实践中踩过的坑

- 消费者组不自动创建：必须手动在启动时检查 XGROUP CREATE，否则直接消费会报错。
- 消息序列化：消息体统一用 JSON，包含 task_id、skill_name、params、timestamp 等字段，并带 version，方便协议演化。
- 监控：用 XINFO GROUPS 和 XINFO CONSUMERS 监控每个组的滞后和消费者状态，及时告警。
