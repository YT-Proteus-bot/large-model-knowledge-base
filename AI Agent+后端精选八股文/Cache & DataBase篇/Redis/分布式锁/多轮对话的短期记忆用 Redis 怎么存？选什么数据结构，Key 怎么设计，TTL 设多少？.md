#  多轮对话的短期记忆用 Redis 怎么存？选什么数据结构，Key 怎么设计，TTL 设多少？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SNyxwWhDRi3NTfkxIJJcT0Aynrb

“短期记忆是多轮对话的血液，它不像长期记忆那样需要永久归档，但必须在对话期间随时可用、高效读写，并且会话结束后自动清理，不能变成内存僵尸。我在设计时遵循三个原则：结构服务于读写模式、Key 携带生命周期、TTL 配合续期机制。

具体方案我拆成数据结构选型、Key 命名规范、TTL 策略三层来说。

---

一、数据结构：分层存储，各司其职

一次对话的短期记忆其实包含两类数据，读写模式完全不同，我用了两种结构。

1. 当前对话状态 —— 用 Hash

对话过程中 Agent 需要快速读取或更新当前槽位、已解析参数、Skill 中间状态等。这类数据字段多、更新频繁、需要按字段读写。Hash 天然适合：

- 每个字段独立，不用序列化整个对象。
- 支持 HGET/HSET 单字段操作，无锁竞争。
- 可以原子性地读取多个字段。

示例结构：

```
代码块
HSET session:abc123:state
  intent "book_flight"
  departure "北京"
  destination "上海"
  date "2026-06-25"
  last_skill "flight_search"
  step "confirm"
```

每次用户补充信息，编排引擎就更新对应字段，比如 HSET session:abc123:state date "2026-06-26"。

1. 对话历史消息 —— 用 ZSet 或 List

对话历史需要按时间排序、支持截断最近 N 条、以及向 LLM 注入上下文窗口。我选 ZSet 而不是 List，原因有二：

- List 只能按插入顺序，无法方便地按时间戳范围裁剪。
- ZSet 以毫秒时间戳为 score，消息体为 member，可以轻松 ZREVRANGE 取最近 N 条，也可以用 ZREMRANGEBYSCORE 删除过期的历史。

示例：

```
代码块
ZADD session:abc123:messages 1719123456123 "user: 帮我订明天去上海的机票"
ZADD session:abc123:messages 1719123457890 "agent: 好的，请问您想选什么舱位？"
```

要取最近 10 轮对话，直接 ZREVRANGE session:abc123:messages 0 9。当对话历史过长时，用 ZREMRANGEBYRANK 保留最近 50 条。

如果不需要按时间裁剪，用 List 更轻量，左推右截也行。但我在实践中发现 ZSet 的时间戳能力在调试和断点恢复时非常有用。

---

二、Key 设计：命名空间隔离，生命周期绑定

Key 命名遵循统一规范：

```
代码块
{namespace}:{会话ID}:{数据类型}
```

具体：

- session:{sessionId}:state → 当前对话状态（Hash）
- session:{sessionId}:messages → 对话历史（ZSet）
- session:{sessionId}:checkpoint → 断点续执行用的完整上下文快照（String，JSON 序列化）
- session:{sessionId}:locks → 并发控制锁（String，用 SET NX EX）

所有 Key 共享同一个会话 ID 前缀，方便批量管理。会话结束时可以一次 DEL 或 UNLINK 清理所有相关 Key。

会话 ID 设计：不是简单的随机串，而是嵌入了用户 ID 和创建时间戳，方便排查时直接定位用户。格式如：uid10086_20260625_a1b2c3。

---

三、TTL 策略：设定超时 + 操作续期

短期记忆最大的风险是会话忘了关，内存一直涨。TTL 就是强制遗忘的闹钟。

我设置两档 TTL：

1. 基础 TTL：对话闲置 30 分钟自动过期。在创建第一个 Key 时用 EXPIRE 设 1800 秒。这个值来自业务统计——绝大多数用户 30 分钟内要么完成会话，要么放弃。太长浪费内存，太短用户上个厕所回来就丢了上下文。
1. 续期机制：每次用户发送新消息，编排引擎用 EXPIRE 刷新所有相关 Key 的 TTL 为 30 分钟。这样活跃对话不会断，僵尸对话自动清理。
1. 断点快照的特殊处理：checkpoint 的 TTL 设得更短，比如 10 分钟。因为断点恢复只在极短时间内有效，如果用户 10 分钟都没响应，这个快照多半用不上了，早清理更安全。

另外，为了防突发流量打满内存，短期记忆所在的 Redis 实例开启 volatile-ttl 淘汰策略，万一内存真满了，优先踢掉即将过期的 Key，而不是直接报错。这对用户体感的影响最小。
