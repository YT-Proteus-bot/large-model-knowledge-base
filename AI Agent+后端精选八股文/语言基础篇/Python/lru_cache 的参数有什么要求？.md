# lru_cache 的参数有什么要求？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ABBPwprifisRYtkUepGcyRjcnZb

lru_cache 参数不多，但踩坑的人前赴后继。直接拆解要求。

---

### 📦 核心参数一览

#### 1️⃣ maxsize — 缓存容量上限

要求：必须是 None 或 非负整数 (>=0)。

- None：无界缓存（LRU禁用），小心内存泄漏
- 0：不缓存结果，每次重新计算（但统计功能可用）
- 负数：直接ValueError，比如 @lru_cache(maxsize=-1) 💥

#### 2️⃣ typed — 类型敏感开关

要求：必须是布尔值 (True / False)。

- False（默认）：f(3) 和 f(3.0) 视为同一参数，共享缓存
- True：按类型区分，3与3.0不同，各自缓存

---

### ⚠️ 隐形铁律：所有参数必须可哈希

lru_cache 内部用字典以参数为键，若传入不可哈希对象（list, dict, set），调用时直接 TypeError: unhashable type。
✅ 解决：用 tuple 代替 list，或用 frozenset。

---

### 🧠 加分回答

- maxsize=0 的妙用：只看调用统计不存结果，配合 cache_info() 做热点分析。
- Python 3.9 后 @functools.cache 就是 @lru_cache(maxsize=None) 的简化版，但无界且不支持 typed。
- 想缓存带 dict 的函数？别手动转元组，用 frozenset(d.items()) 作为键，或封装一个“可哈希包装器”类。

最后考你一下：如果一个方法用了 lru_cache，而方法所属对象变了，缓存会不会自动失效？👀
