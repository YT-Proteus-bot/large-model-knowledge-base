# 在 RAG 系统中，对 Embedding 函数加 lru_cache 时踩过哪些坑？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/AgRFweuIriROUqkaPiYcw6QAnaf

“给 Embedding 函数加 lru_cache，看起来只是加个装饰器的事，但在 RAG 系统里，我踩过三个比较典型的坑，都是跟数据特性和调用模式有关的。”

---

#### 🪤 坑1：拿 ndarray / Tensor 当缓存键 —— 直接报错

很多人一开始会把 Embedding 函数写成这样：

```
代码块
@lru_cache(maxsize=1024)
def embed(text: str) -> np.ndarray:
    ...
```

这本身没问题，但如果为了性能传入预处理后的 token ids：

```
代码块
@lru_cache(maxsize=1024)
def embed_from_tokens(tokens: list[int]) -> np.ndarray:
    ...
```

⚠️ 这里就埋雷了——如果某天你把 list 换成了 np.ndarray，lru_cache 会直接抛 TypeError，因为 ndarray 不可哈希。

🔧 解决：缓存键一律用可哈希类型，比如 tuple(tokens)，或者直接用原始文本做键。

---

#### 🪤 坑2：内存爆炸 —— Embedding 向量太大了

这个坑最痛。
假设一条文本向量是 float32[1536]，约 6KB。缓存 100 万条就是 6GB 纯向量数据，加上缓存字典开销，内存直接飙红。

而且 lru_cache 只管条目数量，不管每条多大，它不会替你估算内存占用。

🔧 解决：

- 设置合理的 maxsize，根据可用内存倒推，比如只缓存 10 万条。
- 配合 cachetools 或 diskcache，换成 LRU + 大小限制 的缓存，或落地磁盘。

```
代码块
from cachetools import LRUCache
cache = LRUCache(maxsize=10_000)  # 或按字节限制
```

#### 🪤 坑3：缓存失效 —— 模型更新后老向量还在用

线上模型升级（比如 text-embedding-3-small → text-embedding-3-large），如果只更新了函数实现，lru_cache 里还存着旧模型的向量，返回结果就全乱了，而且很难发现。

🔧 解决：

- 缓存键里带上 模型版本号，f"{model_version}:{text}"。
- 或者给函数包一层，模型升级时 cache_clear() 清空整个缓存。

---

#### 🪤 坑4（进阶）：批量接口 & 缓存粒度的冲突

生产环境你一般会用批量 Embedding 接口一次传多条文本，但缓存是单条的。

如果缓存了一层批量接口：

```
代码块
@lru_cache
def embed_batch(texts: tuple[str, ...]) -> np.ndarray:
    ...
```

缓存命中率会很低，因为查询组合稍微不同就是新的键。

但反过来只缓存单条，又可能造成大量重复的 API 调用。

🔧 解决：

- 在批量函数内部，先查单条缓存，只对未命中部分调 API，再合并结果。
- 这样既利用了缓存，又不会为不同组合重复计算。

```
代码块
def embed_batch(texts):
    cached, miss_idx, miss_texts = [], [], []
    for i, t in enumerate(texts):
        vec = single_cache.get(t)
        if vec is not None: cached.append((i, vec))
        else: miss_idx.append(i); miss_texts.append(t)
    if miss_texts:
        new_vecs = api.embed(miss_texts)
        for i, v in zip(miss_idx, new_vecs): single_cache[miss_texts] = v
    ...
```
