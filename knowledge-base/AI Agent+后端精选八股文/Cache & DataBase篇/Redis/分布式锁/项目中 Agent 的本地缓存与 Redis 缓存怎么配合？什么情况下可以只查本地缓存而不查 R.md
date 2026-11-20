---
title: 项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 Redis？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/T0DMwO42Ui552MkljoMc5VNBnLb
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 R.md
questions: 0
chars: 1884
build_date: '2026-11-20'
images_removed: 0
---

# 项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 Redis？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/项目中 Agent 的本地缓存与 Redis 缓存怎么配合？什么情况下可以只查本地缓存而不查 R.md

“本地缓存和 Redis 在 Agent 里是速度和共享的分工。本地缓存（我们一般用 Caffeine）是微秒级的纯内存访问，但只能当前实例自己用；Redis 是中心化的，毫秒级，所有 Agent 实例都能共享。配合起来的核心是多级缓存：本地一级，Redis 一级，数据源一级。

## 一、多级缓存的查询流程

1. Agent 需要某份数据（比如 Skill 的元信息）时，先查本地缓存。命中直接返回，微秒级。
1. 本地未命中，再查 Redis。Redis 命中后，把值写回本地缓存，再返回，同时给本地缓存设一个短 TTL（比如 30 秒）。
1. Redis 也没命中，才去数据库或远程服务查，拿到结果后先写 Redis，再写本地缓存，最后返回。

这套三级流程用 Caffeine + Redis 非常成熟，但关键在于什么数据可以直接跳过 Redis，永远只查本地缓存。

二、什么情况下可以只查本地缓存？

核心判断标准就一条：数据的不一致性是否可以被业务容忍，且本地能覆盖绝大多数读取场景。我分四类：

第一类：低频变更的静态配置

Skill 的元信息（名称、描述、参数 schema、超时时间）通常一天甚至一周才变更一次。Agent 启动时把这些全量加载到 Caffeine，设置半小时或一小时刷新一次。运行时直接读本地，Redis 只作为变更通知通道：配置后台修改后，发一条 Pub/Sub 消息，所有 Agent 监听到就清空本地缓存，下次请求时重新加载。这种模式下平常读从不访问 Redis，只在极偶尔的清空重建时才会访问。

第二类：会话级别的临时状态

当前对话的上下文、Slot 槽位、已解析的用户意图——这些数据本身就是该 Agent 实例当前会话私有的，其他实例根本不需要知道。这种数据完全不需要进 Redis，用本地缓存（或直接存线程变量）即可。只有当会话需要跨实例转移（如负载均衡开粘滞失败）时，才把断点状态序列化到 Redis，由另一个实例接管。

第三类：高频读、允许短时不一致的实时数据

比如“用户偏好语言”这个字段，读写比极高，一天改不了两次。直接本地缓存，设置 5 分钟过期，5 分钟内用户即使改了语言，Agent 仍用旧语言也不影响理解。或者用版本号：Redis 里存用户偏好的版本号，本地缓存存实际值，每次查询先检查版本号是否变了，没变直接返回本地值。版本号的读取本身就非常轻量，比读整个对象快得多。

第四类：经过“本地优先”策略确认的热点数据

用 Caffeine 的 refreshAfterWrite 异步刷新能力，本地缓存快过期时自动开一个后台线程去 Redis 刷新，同步请求永远不阻塞在缓存加载上。这样对业务看起来“永远只查本地”，Redis 只是默默的异步补充。

## 三、保证本地缓存不“脏”的兜底机制

只查本地不是放任自流，我们加了三个保护：

- 最大容忍过期时间：本地缓存必须设 expireAfterWrite，不能永不过期，至少每 30 分钟强制刷新一次，防止长期脏读。
- 变更通知 + 主动失效：Redis Pub/Sub 或 Redis Keyspace Notification 通知配置变更，收到后立即 invalidate 本地缓存。
- 兜底对账：每 5 分钟用定时任务扫描本地热点 Key，去 Redis 对比版本号，不一致就异步更新。

## 四、一个生产案例

我们的 Agent 在调用外部 Skill 之前，必须读 Skill 的参数 schema 来做意图解析。这个 schema 变更频率极低，但调用频率极高（每次对话都触发）。我们把所有 Skill 的 schema 在 Agent 启动时全量加载到 Caffeine，设 30 分钟过期。Redis 只存一份全量 JSON，在配置变更时通过 Pub/Sub 发一个 skill:schema:refresh。平时 99.99% 的调用都只查本地，Redis 几乎零负载。就这一项优化，Agent 平均响应时间降了 8ms。

## 本地缓存 vs Redis vs 数据源 对比图（文本）

```
查询请求
   │
   ▼
本地缓存 (Caffeine) ── 命中 → 返回 (微秒级)
   │ 未命中
   ▼
Redis 缓存 ── 命中 → 回填本地 → 返回 (毫秒级)
   │ 未命中
   ▼
数据库/远程服务 → 写 Redis → 写本地 → 返回 (数十毫秒)
```
