---
title: 长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Pq9IwluleiOutzkVuW0c4sbZnCf
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？.md
questions: 0
chars: 1673
build_date: '2026-11-20'
images_removed: 0
---

# 长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/长期记忆和短期记忆在 Redis 里如何区分？长期记忆用什么数据结构存储，如何支持高效检索？.md

“区分长期和短期记忆，核心不是存的内容不同，而是生命周期、访问模式和检索方式不同。短期记忆像你正在处理的草稿纸，用完或过时就扔；长期记忆是归档的档案，跨会话保留，需要能按语义快速翻出来。

我按这三个层次来设计。

## 一、生命周期与命名空间隔离

我在 Redis 里用不同的 key 前缀和 TTL 策略来硬性区分：

- 短期记忆：session:{session_id}:*，比如 session:abc123:weather_last_city。这类 key 的 TTL 设得很短，和会话生命周期绑定，会话结束或超时就自动过期。如果会话中断需要恢复，就用 session:abc123:checkpoint 把整个会话上下文序列化存一个 hash，设 30 分钟 TTL。
- 长期记忆：user:{user_id}:memory:*，比如 user:10086:memory:flight_preference。这类 key 不设 TTL 或设非常长（比如 90 天）。它们是用户级别的，跨任何会话都能读到。

命名空间一隔离，运维上也清晰——短期记忆用独立的 Redis 实例或 DB 编号，长期记忆用另一个实例，避免短期大量写入把长期记忆踢出去。

## 二、长期记忆的数据结构选型

长期记忆在 Agent 场景下有一个核心需求：不能只按精确键查，必须支持语义检索。用户说“上次我选的那个靠窗的座位”，Agent 需要去记忆里搜“航班偏好+靠窗”，而不是精确匹配 user:10086:memory:seat_preference。

Redis 原生的 Hash、ZSet 做不了语义检索，所以我实际用的是 Redis Stack（带 RediSearch 模块），它支持向量相似度搜索和结构化过滤。

具体设计：

- 每条长期记忆存成一个 Redis JSON 文档（用 JSON.SET），包含记忆内容、类型标签、创建时间戳、重要性权重、以及记忆内容的嵌入向量。
- 示例文档：

```
{
  "type": "preference",
  "content": "用户喜欢靠窗的座位",
  "metadata": {
    "category": "flight",
    "timestamp": 1719123456,
    "importance": 0.8
  },
  "embedding": [0.023, -0.11, ..., 0.045]
}
```

- 用 FT.CREATE 建立索引，定义向量字段使用 FLAT 或 HNSW 算法。HNSW 在检索速度上更优，适合大规模记忆库。
- 检索时，把用户当前查询意图用同一个嵌入模型实时转成向量，然后执行 KNN 近似最近邻搜索：

```
FT.SEARCH idx:user_memory
  "@vector:[KNN 5 @embedding $query_vec AS score]"
  SORTBY score ASC
  RETURN 3 content metadata.importance
```

- 这样就能按语义找出最相关的 5 条记忆，同时用重要性分数做加权。

## 三、高效检索的优化策略

纯向量检索有时会召回不够精准，因为重要性和时效性没有参与排序。我引入 混合检索 + 时间衰减：

- 混合得分：最终得分 = 向量相似度 × 0.6 + 重要性权重 × 0.25 + 时间衰减因子 × 0.15。时间衰减用指数函数，越老的记忆衰减越多，除非重要性特别高。
- 元数据过滤：在搜索时加上结构化过滤条件，比如 @category:{flight}，只搜和当前任务领域相关的记忆，缩小搜索空间。
- 分层存储：不是所有记忆都走向量检索。高频、近期记忆可以用 ZSet 做时间线索引，按时间戳排序直接取最近 N 条，只对模糊查询才走向量搜索。这样热数据用精确索引，冷数据用语义搜索，兼顾性能和成本。
