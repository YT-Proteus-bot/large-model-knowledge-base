---
title: Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Eqtywb5itimDlsk0fVdcxnUan6d
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？.md
questions: 1
chars: 1368
build_date: '2026-11-20'
images_removed: 0
---

# Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？.md

## 题1：Redis 内存淘汰策略有哪些？在 Agent 场景如何选择？

面试官：“Redis 内存淘汰策略有哪些？如果在 Agent 系统里用 Redis 做缓存和上下文存储，你会怎么选？”

候选人：

“Redis 的内存淘汰策略有八种，但本质上是在回答一个问题：内存满了，该踢谁出去？ 它们分成三大类：

- 不淘汰：noeviction，内存满了直接报错，不删任何数据。
- 在有过期时间的键里淘汰：volatile-lru、volatile-lfu、volatile-random、volatile-ttl。只删设置了 TTL 的 key。
- 在所有键里淘汰：allkeys-lru、allkeys-lfu、allkeys-random。不管有没有设过期时间，全范围挑着删。

具体原理大家比较熟，我就不逐条展开了。我更想聚焦在 Agent 场景下该怎么选，这是真正考验工程判断力的地方。

Agent 场景的缓存特征

Agent 用 Redis 通常是三层数据：

1. 上下文数据：当前任务或会话的状态快照，用来断点续执行或跨 Skill 共享。这是强生命线，丢了任务就丢了。
1. Skill 执行结果缓存：比如天气查询结果、用户画像摘要。有 TTL，丢了可以重算，但会拖慢响应。
1. 临时限流/熔断计数器：高频读写，增量操作，丢了只是重新计数。

三层数据对淘汰的容忍度完全不同。

我的选择策略：分层 + 分类淘汰

我不会给整个 Agent 系统设一个全局策略，而是按数据层用不同的淘汰策略和 Redis 实例：

1. 对会话上下文：用 noeviction 或独立实例不设淘汰

   任务上下文是 Agent 的“记忆”，丢了等于用户得从头再来。所以存储上下文的 Redis 实例我会设 maxmemory-policy noeviction，同时给每个 key 设合理的 TTL（比如任务超时 30 分钟自动过期）。如果真的快满了，我宁愿让它报错触发告警，人工介入扩内存，也不能静默丢数据。

1. 对 Skill 结果缓存：用 allkeys-lru 或 volatile-lfu

   这些缓存丢了不可怕，可以重算。我倾向用 allkeys-lru 或 allkeys-lfu 淘汰最近最少用或最不常用的缓存。选择取决于数据模式：

- 如果 Skill 调用分布有明显的冷热，比如热门城市天气查询远多于冷门，选 LFU，保护高频热点。
- 如果访问模式更随机，用 LRU 更稳妥，简单有效。

同时，这类数据一定要设置 TTL，因为 Agent 环境里很多知识有时效性——10 分钟前的天气缓存已经没意义了。TTL + LFU 双保险，能有效控制内存。

1. 对限流/熔断计数器：用 volatile-ttl 或 noeviction 配合小内存

   计数器通常自带过期时间（比如滑动窗口计数器 1 分钟过期）。如果因为内存满了丢计数器，可能导致限流判断失效。所以我会优先给计数器用 volatile-ttl 策略，优先淘汰即将过期的计数器，避免删掉刚更新的窗口。如果计数器写在一个独立的小 Redis 里，甚至可以设置 noeviction，因为它本身体量很小，配足够内存即可。
