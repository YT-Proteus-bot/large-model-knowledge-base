---
title: Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去重各有什么优缺点？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/TLWdwInIeiMEEEkR0GVcXSfInpd
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去.md
questions: 0
chars: 908
build_date: '2026-11-20'
images_removed: 0
---

# Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去重各有什么优缺点？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 接收的消息需要幂等处理，用 Redis Set 去重和 Bloom Filter 去.md

“Agent 场景下消息重复是常态——上游重试、任务超时重分配、网络抖动都可能导致同一条指令被消费多次。幂等去重就是保证‘同一条消息只生效一次’。Redis 里常用的两种方案是 Set 精确去重 和 布隆过滤器概率去重，它们本质上是精度与内存的交换。

## 一、Redis Set 方案：精确、稳妥、但有成本

最简单的做法是：每条消息处理前，用 SADD msg:idempotent:set <message_id>，如果返回 1 就说明是新消息，照常处理；返回 0 说明已经处理过，直接跳过。消息 ID 用业务唯一键（如 order_id + event_type）。

- 优点：
- 绝对精确，不会漏判，也不存在误判。
- 可以配合 EXPIRE 做自动清理，比如保留 24 小时内的去重记录。
- 实现简单，不需要额外插件。
- 缺点：
- 内存占用随消息量线性增长。如果每天千万条消息，保存 24 小时就要存千万个 ID，每个 ID 按 30 字节算，再加上 Redis 数据结构开销，轻松吃掉几个 GB 内存。
- 大量 Key 过期时可能触发 Redis 的被动淘汰或阻塞，需要调优过期策略。

## 二、Bloom Filter 方案：极致节省内存，但有小概率误判

布隆过滤器用一组哈希函数和一个 Bitmap 来判断一个元素“可能存在”还是“绝对不存在”。我们可以用 Redis 的 BF.ADD 和 BF.EXISTS 命令（需 Redis Stack 或布隆过滤器模块），或者用 SETBIT 配合多个哈希函数自己实现。

- 优点：
- 内存占用极小且恒定。比如为 1 亿条消息准备过滤器，误判率 1%，只需约 120 MB。
- 写入和查询都是 O(k)（k 为哈希函数个数），性能极高。
- 缺点：
- 存在误判（假阳性）：过滤器可能说“这条消息见过”，但实际上没见过。这会漏杀，认为新消息是重复的，导致消息被错误丢弃，这在交易、支付等场景下可能造成事故。
- 不支持删除，也不支持精确的过期淘汰。通常只能定期重建整个过滤器。
- 需要预先估计数据规模，配置错误会导致误判率飙升。
