---
title: Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis/分布式锁
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Oy4CwgIBCi2hiakL2P5cBw1vnZd
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？.md
questions: 0
chars: 2082
build_date: '2026-11-20'
images_removed: 0
---

# Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis/分布式锁　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/分布式锁/Agent 需要根据历史对话检索相似记忆，Redis 怎么做向量相似度搜索？.md

“这正是 Redis 在 AI 场景下的杀手锏能力。我们不是只做精确匹配，而是要用语义向量去捞记忆。实现上依赖 Redis Stack 的 RediSearch 模块，它原生支持向量索引和 KNN 近似搜索，速度极快，数据不离 Redis，没有额外组件的延迟。

我分四个步骤讲清楚：向量怎么存、索引怎么建、怎么搜、以及生产上怎么优化。

## 一、记忆的存储结构：JSON + 嵌入向量

每条历史对话记忆，我会封装成一个 Redis JSON 文档（用 JSON.SET），里面包含：

- content：记忆的原始文本，比如“用户喜欢靠窗的座位”。
- metadata：领域、时间戳、重要性等结构化的过滤条件。
- embedding：用嵌入模型（如 text-embedding-3-small）把内容转成的向量数组，维度通常是 1536 或 768。

存储命令类似：

```
JSON.SET memory:user:10086:1 $ '{"content":"用户喜欢靠窗的座位","metadata":{"category":"flight","ts":1719123456,"importance":0.8},"embedding":[0.023,-0.11,...,0.045]}'
```

这样，每条记忆既是结构化文档，又携带了语义向量，后续检索和过滤都能用上。

## 二、向量索引的创建：HNSW 算法加速

要让向量能被搜索，需要建索引。我用 FT.CREATE 创建一个 RediSearch 索引，指定向量字段的算法：

```
FT.CREATE idx:user_memory
  ON JSON
  PREFIX 1 memory:user:
  SCHEMA
    $.content AS content TEXT
    $.metadata.category AS category TAG
    $.metadata.importance AS importance NUMERIC
    $.embedding AS embedding VECTOR HNSW 6
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
```

关键配置说明：

- HNSW 算法：分层可导航小世界图，平衡了检索速度和召回率，适合记忆库这种百万级规模。
- DISTANCE_METRIC COSINE：用余弦距离衡量语义相似度，和嵌入模型的训练目标一致。
- 索引前缀：指定只索引 memory:user: 开头的 Key，隔离不同数据。

## 三、执行相似度搜索：KNN + 混合排序

当用户说“我想选上次那种安静的位置”，Agent 会把这句话用同一个嵌入模型实时转成查询向量，然后发起搜索：

```
FT.SEARCH idx:user_memory
  "@category:{flight}
   @embedding:[KNN 5 @embedding $query_vec AS vec_score]"
  SORTBY importance DESC
  RETURN 3 content vec_score
  DIALECT 2
```

这一步做了几件事：

- 结构化过滤：只搜 category 为 flight 的记忆，缩小候选集。
- KNN 检索：在满足过滤条件的文档里，找向量最相似的 Top-5，并返回相似度得分 vec_score。
- 多因子排序：最终排序可以结合 vec_score 和 importance（重要性）加权，让高频重要记忆排在前面。如果用 SORTBY 不能满足复杂权重，我会在应用层拉出候选集后用代码做最终加权，保留向量相似度 60%、重要性 25%、时间衰减 15% 的公式。

检索结果直接返回最相关的记忆内容，Agent 把它注入 LLM 上下文，生成更个性化的回复。

## 四、生产环境的优化配置

向量搜索虽然强大，但不加控制会很耗资源。我主要关注三点：

- 索引参数调优：HNSW 的 M（每层最大连接数）设 16 或 32，EF_CONSTRUCTION 设 200，平衡构建速度和查询精度。查询时用 EF_RUNTIME 动态控制搜索范围。
- 向量降维与量化：如果记忆量极大，可考虑用产品化的向量数据库代理，但 Redis 本身支持 FP32，存储和计算压力不小。实践中对于千万级记忆，我会做分层存储：近期记忆用 Redis 向量索引保证快速访问，历史归档记忆存在 Milvus 或 pgvector，通过统一路由层查询。
- 监控索引内存：向量索引占用额外内存（尤其是 HNSW 图），需要监控 [FT.INFO](http://FT.INFO)FT.INFO 中的 num_docs 和 memory，设置合理的内存上限和淘汰策略，避免 OOM。
