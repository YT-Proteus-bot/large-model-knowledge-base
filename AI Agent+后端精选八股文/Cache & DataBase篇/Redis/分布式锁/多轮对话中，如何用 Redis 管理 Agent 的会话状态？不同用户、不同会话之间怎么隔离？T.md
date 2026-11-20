# 多轮对话中，如何用 Redis 管理 Agent 的会话状态？不同用户、不同会话之间怎么隔离？TTL 和续期策略怎么设计？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/KURhwPKP5irrxjkQpTQcE3xxnTY

好，这个问题其实挺经典的，我结合之前做智能客服机器人的经历来说一下思路，尽量讲得落地一点。

---

#### 1. 整体模型：把 Redis 当作“短期大脑”

多轮对话里，Agent 需要记住上下文、槽位、本轮意图甚至历史消息，但这些信息都是临时的，不需要持久化到 DB。Redis 天然适合做这个“会话级缓存”，读写都在内存，延迟低，又有 TTL，省得自己写清理逻辑。

我习惯把一次会话的状态抽象成一个 Session Object，用 Redis 的 Hash 或 String(JSON) 存，看团队偏好。

---

#### 2. 隔离设计：Key 的命名就是天然的“路由”

隔离分两层：用户隔离 和 会话隔离。

我会用这种 Key 结构：

```
代码块
agent:session:{user_id}:{session_id}
```

- user_id：来自认证层（JWT / API Key），100% 可信后才拿来拼 Key，避免越权。
- session_id：会话创建时由服务端生成（UUID），返回给客户端，客户端每次请求都要带上。

为什么把 user_id 放进 Key？
因为在 Redis 里这是最硬的隔离——Key 空间本身就分开了。就算某个请求的 session_id 被篡改，也很难跨用户撞 Key。反过来，如果只用 session_id 作为 Key，必须在 Value 里存 user_id，每次读取后做校验，多一步，也容易疏漏。

如果要支持“查看我的历史会话列表”，我会再加一个辅助数据结构：

```
代码块
user:sessions:{user_id}  →  Sorted Set (score = last_active_timestamp, member = session_id)
```

这样隔离和查询兼顾，互不冲突。

---

#### 3. 会话状态里存什么？——一个直观的 Hash 设计

我偏向用 Hash 存储，因为对话状态字段会慢慢变多，Hash 可以按需读写，不用整个序列化。

典型字段：

> （此处为内嵌电子表格，需登录飞书查看）

结构示例：

```
代码块
HMSET agent:session:u1001:sess_abc123
  status "waiting_confirmation"
  intent "book_flight"
  slots '{"dep":"北京","arr":"上海","date":"2026-07-01"}'
  context '{"last_question":"出发城市"}'
  history '[{"role":"user","content":"...","time":...}]'
  created_at 1719158400
  last_active 1719158500
```

为什么不是全存 JSON 字符串？
因为更新槽位的时候，我不需要读整个对象，只需要 HSET slots，降低并发冲突概率。但如果团队更习惯 JSON 且历史消息较少，直接 SET key JSON + GET 也是完全可行的，简单就是美。

---

#### 4. TTL 策略：别卡得太死，也别无限续命

初始 TTL 设置：

会话创建时，给一个合理的“空闲超时”，比如 30 分钟：

```
代码块
EXPIRE agent:session:u1001:sess_abc123 1800
```

选择 30 分钟的依据：

- 用户咨询客服平均间隔很少超过 5 分钟。
- 30 分钟足够覆盖短暂离开（接电话、查信息），又不至于 Redis 内存爆掉。
- 结合产品类型，电商售后可能设 15 分钟，教育答疑可能设 1 小时。

辅助 Key 的 TTL 最好保持一致，比如 user:sessions 那个 Sorted Set，可以用同一套过期策略，或者定期 job 清理过期成员。

---

#### 5. 续期策略：少写 Redis，但要精准

最简单的做法：每次请求都刷新 TTL。
流程是：收到消息 → 读会话 → 处理 → 更新会话 → EXPIRE key 1800。

但有个小问题：如果高并发下，频繁 EXPIRE 虽然开销小，但积少成多，而且很多时候没必要，比如用户才过了 2 秒又发一条。

所以我惯用的优化是 “门槛续期”：
