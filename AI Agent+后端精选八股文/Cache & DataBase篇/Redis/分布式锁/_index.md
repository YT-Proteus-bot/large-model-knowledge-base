# 分布式锁

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/PcJywO5LEiO9ldkK918cGnlrnSg

> （此处为飞书知识库目录块）

## 子页面

- [Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？](Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？.md)
- [长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？](长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？.md)
- [ 多轮对话的短期记忆用 Redis 怎么存？选什么数据结构，Key 怎么设计，TTL 设多少？](多轮对话的短期记忆用 Redis 怎么存？选什么数据结构，Key 怎么设计，TTL 设多少？.md)
- [LangChain 里的 ConversationBufferMemory 如果用 Redis 做后端，底层 Key 是怎么组织的？你会关注哪些配置？](LangChain 里的 ConversationBufferMemory 如果用 Redis.md)
- [ Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？](Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？.md)
- [项目中怎么用 Redis Stream 实现多个 Agent 间的异步通信？消费者组怎么配置才能保证一个任务只被一个 Agent 消费？](项目中怎么用 Redis Stream 实现多个 Agent 间的异步通信？消费者组怎么配置才能.md)
- [Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具结果，怎么防止缓存击穿？](Agent 调用外部工具的结果用 Redis 缓存，怎么设置过期策略？如果同时大量请求同一个工具.md)
- [多个 Agent 共享一份 Redis 里的黑名单或知识库，写操作时如何保证一致性？用分布式锁吗？锁超时时间怎么设？](多个 Agent 共享一份 Redis 里的黑名单或知识库，写操作时如何保证一致性？用分布式锁吗.md)
- [ Redis 做 Agent 心跳检测是怎么设计的？每个 Agent 定时更新一个 Key 的过期时间，如果 Key 没了就判定下线，这种方案有什么缺陷？](Redis 做 Agent 心跳检测是怎么设计的？每个 Agent 定时更新一个 Key 的过期.md)
- [怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？](怎么用 Redis 的 ZSET 实现 Agent 的延迟队列？.md)
- [Agent 需要实现“检查用户配额 → 扣减配额”原子操作，为什么要用 Lua 脚本？](Agent 需要实现“检查用户配额 → 扣减配额”原子操作，为什么要用 Lua 脚本？.md)
- [聊一聊 Redis 的发布订阅模式在 Agent 配置热更新里怎么用？有哪些局限？](聊一聊 Redis 的发布订阅模式在 Agent 配置热更新里怎么用？有哪些局限？.md)
- [在 RAG 系统中，Redis 作为向量数据库怎么处理嵌入向量的更新？](在 RAG 系统中，Redis 作为向量数据库怎么处理嵌入向量的更新？.md)
- [你的 Agent 项目里，Redis 内存淘汰策略选的是什么？为什么在记忆场景下不能用默认的 noeviction？](你的 Agent 项目里，Redis 内存淘汰策略选的是什么？为什么在记忆场景下不能用默认的 n.md)
- [如何用 Redis 的 Bitmap 统计每日活跃 Agent 工具调用次数？](如何用 Redis 的 Bitmap 统计每日活跃 Agent 工具调用次数？.md)
- [如何用 Redis 实现 Agent 的滑动窗口限流？具体用了哪些 Redis 命令？如果被限流，你是返回错误还是放入等待队列？](如何用 Redis 实现 Agent 的滑动窗口限流？具体用了哪些 Redis 命令？如果被限流.md)
- [Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去重各有什么优缺点？](Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去.md)
- [ Redis 的大 Key 问题，在 Agent 对话历史存储中常见吗？](Redis 的大 Key 问题，在 Agent 对话历史存储中常见吗？.md)
- [在 AutoGPT 或 BabyAGI 这类 Agent 里，任务队列通常用 Redis 的什么数据结构？](在 AutoGPT 或 BabyAGI 这类 Agent 里，任务队列通常用 Redis 的什么.md)
- [多轮对话中，如何用 Redis 管理 Agent 的会话状态？不同用户、不同会话之间怎么隔离？TTL 和续期策略怎么设计？](多轮对话中，如何用 Redis 管理 Agent 的会话状态？不同用户、不同会话之间怎么隔离？T.md)
- [ 项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 Redis？](项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 R.md)
- [如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列？](如何用 Redis Stream 的 Pending 列表实现 Agent 任务的重试和死信队列.md)
- [ Redis 的 Persistence 你选 RDB 还是 AOF？在 Agent 记忆场景下，如果允许丢失少量最新记忆但对恢复速度要求很高，怎么选？](Redis 的 Persistence 你选 RDB 还是 AOF？在 Agent 记忆场景下，.md)
- [多个 Agent 实例并行处理任务，需要一个分布式计数器记录已完成任务数，用 Redis 的 String 还是 Hash？原子自增要注意什么？](多个 Agent 实例并行处理任务，需要一个分布式计数器记录已完成任务数，用 Redis 的 S.md)
- [在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的？如何控制消息列表的最大长度？](在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的.md)
- [为什么有些 Agent 项目用 Redis 做消息中间件而不是 RabbitMQ 或 Kafka？从运维、延迟、持久化角度说说你的理解。](为什么有些 Agent 项目用 Redis 做消息中间件而不是 RabbitMQ 或 Kafka.md)
- [Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？](Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？.md)
- [如果 Agent 的 Redis 实例出现慢查询，你怎么排查？可能和 Agent 的哪些访问模式有关？](如果 Agent 的 Redis 实例出现慢查询，你怎么排查？可能和 Agent 的哪些访问模式.md)
- [Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？](Agent 需要定时执行任务，如何用 Redis 实现一个简单的定时调度器？ (2).md)
- [聊一聊 Agent 记忆模块中 Redis 的监控，你最关注哪些指标？命中率、连接数、内存碎片率突然飙升可能是什么导致的？](聊一聊 Agent 记忆模块中 Redis 的监控，你最关注哪些指标？命中率、连接数、内存碎片率.md)
- [如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent 等待所有子 Agent 结果返回后再继续？](如何用 Redis 实现多个 Agent 的执行屏障（Barrier），比如一个父 Agent.md)
