---
title: Elasticsearch 搜索
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/MjSawrHIci2PTTktAdVc2UF3nQb
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Elasticsearch 搜索.md
questions: 2
chars: 1818
build_date: '2026-11-20'
images_removed: 1
---

# Elasticsearch 搜索

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Elasticsearch 搜索.md

## Elasticsearch 搜索与 Agent 知识库

## 题1：Elasticsearch 的倒排索引原理是什么？为什么搜索快？

要理解 Elasticsearch 为什么快，核心在于倒排索引。它和传统的“正向索引”（比如书的目录）完全不同。

正向索引 vs 倒排索引

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

倒排索引的核心结构

它由两个核心部分组成：词典 (Term Dictionary) 和 倒排列表 (Posting List)。

- 词典：存储所有经过分词处理后的不重复单词，并排序（比如按字典序）。为了快速定位，ES 内部用 Finite State Transducers (FST) 或跳表 (Skip List) 来加速 Term 查找。
- 倒排列表：每个单词对应一个列表，记录所有包含该单词的文档 ID、词频、位置等信息。列表本身也做了压缩和跳表优化，方便快速求交集、并集。

```
词典 (Term Dictionary)          倒排列表 (Posting List)
────────────────────────        ────────────────────────
"elastic"  ──────────────→      [Doc1: tf=3, pos=<2,15,37>]
"search"   ──────────────→      [Doc1: tf=2, pos=<10,30>, Doc3: tf=1, pos=<5>]
```

为什么搜索快？

1. 直接寻址：不需要全表扫描，通过 Term 索引直接跳转到对应的倒排列表，时间复杂度是 O(1) 或 O(log N)，而不是 O(N)。
1. 索引压缩与缓存：词典常驻内存（通过 FST 压缩），倒排列表在磁盘上也是高度压缩的，读取时批量加载，结合操作系统的 Page Cache，极大减少磁盘 IO。
1. 高效的布尔运算：多个词查询时，ES 会在倒排列表上做交集、并集，并利用跳表在排序列表上快速跳过大量无关文档，避免了真正的全量遍历。
1. 打分与排序：ES 不仅找出文档，还基于 TF-IDF 或 BM25 算法实时计算相关性分数，这个过程也是基于倒排索引的统计信息，无需额外扫描。

示例代码：用 Python 模拟一个极简的倒排索引

```
from collections import defaultdict
def build_inverted_index(docs):
    index = defaultdict(list)
    for doc_id, text in docs.items():
        for word in set(text.lower().split()):
            index[word].append(doc_id)
    return index
docs = {
    1: "Elasticsearch is a search engine",
    2: "Search is fast with inverted index",
    3: "Elasticsearch uses inverted index for fast search"
}
inverted = build_inverted_index(docs)

# 查询 "search"
print(inverted.get("search", []))  # [1, 2, 3]

# 查询 "search" AND "fast"
result = set(inverted.get("search", [])) & set(inverted.get("fast", []))
print(result)  # {2, 3}
```

在实际的 ES 中，字典和倒排列表都经过精心压缩和索引，使得亿级数据也能在毫秒内返回。

收束：倒排索引的本质是把“找文档”变成了“找词”，再通过词的文档列表直接拿到结果。这种结构天然为全文搜索而生，配合现代硬件的高速 IO 和 ES 的分布式架构，才成就了它极高的查询性能。

## 题2：在 Agent 知识库场景，ES 和向量数据库如何配合做混合检索？
