---
title: 在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的？如何控制消息列表的最大长度？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Kh5gwJS8SizueRkW4JDcqywqnzc
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的.md
questions: 4
chars: 2976
build_date: '2026-11-20'
images_removed: 0
---

# 在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的？如何控制消息列表的最大长度？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/在 LangChain 里，RedisChatMessageHistory 底层是怎么存储消息的.md

“这个问题很细，但特别实战。我正好在生产环境里深度用过，我分两块拆开讲：一是底层到底用了Redis的什么数据结构，二是怎么在保留上下文和节省Token之间控制长度。”

### 一、底层存储：用的不是String，是List

## 题1：Redis 结构：一个 Session 一个 List

- 核心数据结构：RedisChatMessageHistory 内部用的是 Redis 的 List（列表），而不是 String 或 Hash。
- Key 的命名规则：默认的 Key 格式是 message_store:{session_id}，那个 session_id 就是你初始化时传进去的。例如会话ID是 user_123，实际 Redis 里的 Key 就是 message_store:user_123。你可以通过 key_prefix 参数定制这个前缀。
- 为什么用 List？ 因为聊天消息天然是一个有序的序列，而且只做追加操作。用 List，写入时往右边 RPUSH，读取时用 LRANGE 从头拉到尾，完美保持时间顺序，还支持高效的基于索引的范围查询。

## 题2：消息的序列化：JSON字符串

- 每一条消息（不管是 HumanMessage 还是 AIMessage）在存入 Redis 之前，都会被序列化成一个 JSON 字符串。
- 具体来说，LangChain 会调用消息对象的 .dict() 方法，生成一个包含 type、content、additional_kwargs 等字段的字典，然后 json.dumps。
- 所以你在 Redis 里执行 LRANGE message_store:user_123 0 -1，会看到类似这样的字符串队列：

```
{"type": "human", "content": "帮我查下订单", "additional_kwargs": {}}
{"type": "ai",   "content": "好的，正在查询...", "additional_kwargs": {}}
{"type": "human", "content": "再查一下物流", "additional_kwargs": {}}
```

- 读取还原：取出这些字符串后，LangChain 用 messages_from_dict 这个工具函数，根据 type 字段反向构造出对应的 Message 对象。

## 题3：一次完整的写入调用链（加深理解）

当你调用 add_message(message) 时，背后其实就干了一件事：

```
redis_client.rpush(session_key, message.json())
```

没有复杂的二级索引，就是单纯的追加。这个简单的设计意味着：查全量消息时，性能跟列表长度成正比，这一点马上会联系到长度控制的必要性。

## 题4：二、如何控制最大长度？

“原生的 RedisChatMessageHistory 本身不自带自动截断，如果你只调用 add_message，它会无限增长，最后撑爆 Redis 内存，或者每次拉全部消息时把网络堵死。所以控制长度是必然要做的事，有三种实战方法。”

### 方法1：直接修剪 Redis List（最直接、最底层）

- 利用 Redis 的 LTRIM 命令。每次 add_message 后，顺手执行：

```
redis_client.ltrim(session_key, -max_count, -1)
```

- 这行命令意思是：只保留列表右边（最新的）max_count 条消息。比如 LTRIM key -100 -1 就是始终保留最近100条。
- 原子性问题：如果你直接分开写 RPUSH 和 LTRIM，高并发下可能有两个 Agent 同时操作，一个刚 RPUSH 完，另一个 LTRIM 就把新消息切掉了。解决办法：用 Redis Lua 脚本把 RPUSH 和 LTRIM 打包成一个原子操作，或者用官方 API——LangChain 的 RedisChatMessageHistory 虽然没有内置 trim，但可以继承重写 add_message 来封装这个 Lua 脚本。

### 方法2：使用 LangChain 链上修剪（更偏框架层）

- LangChain 提供了 trim_messages 这个工具，它不直接操作 Redis，而是在构建 Prompt 之前，基于策略（token 数量、消息数量）对消息列表进行裁剪。
- 用法示例：在 chain 的 prompt 阶段前套一个 trim_messages(strategy="last", max_tokens=3000)，它会自动帮你仅保留最靠近当前对话的部分消息，然后再喂给 LLM。
- 注意区分：这个修剪不会写回 Redis。也就是 Redis 里还是全量存着的，只是在当前推理请求里裁剪上下文。如果你想“持久化修剪”，必须在 add_message 时结合方法1，否则存储压力依然在。

### 方法3：从 Session 工厂层面做一个“修剪版”的 MessageHistory

- 自己写一个 TrimmedRedisChatMessageHistory 子类，重写 add_message 方法：

```
def add_message(self, message):
    super().add_message(message)
    # 原子修剪：保留最近 max_len 条
    lua_trim_and_push = """
    redis.call('RPUSH', KEYS[1], ARGV[1])
    redis.call('LTRIM', KEYS[1], -ARGV[2], -1)
    return redis.call('LLEN', KEYS[1])
    """
    self.redis_client.eval(lua_trim_and_push, 1, self.key, message.json(), self.max_len)
```

- 然后把自定义的 History 类注入到 get_session_history 里，这样无论哪个 Agent 写消息，都会自动遵守长度限制，不用散落在业务各处。

### 三、一个绕不开的工程坑：消息窗口≠存储窗口

“面试官，我多提一句真实经验。很多人以为控制了 Redis List 的长度就万事大吉，其实存储长度和发给大模型的上下文窗口是两回事。哪怕 Redis 存了 500 条消息，你的 Prompt 可能只能塞 20 条，否则 Token 爆了。

所以我一般会做两层限制：

- Redis 层保留较大窗口（比如最近 200 条），方便冷数据回溯调试；
- LLM 调用层通过 trim_messages 或人工拼接，只取最近 10~20 轮对话进 Prompt。

  这样既省内存、不丢调试数据，又能精确控制推理成本。”
