---
title: Agent 系统中每次 LLM 调用都创建一个记录对象，如何优化这类高频小对象的内存占用？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FOmlwthAQie1zAknByGcSNCAns4
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Agent 系统中每次 LLM 调用都创建一个记录对象，如何优化这类高频小对象的内存占用？.md
questions: 1
chars: 1463
build_date: '2026-11-20'
images_removed: 0
---

# Agent 系统中每次 LLM 调用都创建一个记录对象，如何优化这类高频小对象的内存占用？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Agent 系统中每次 LLM 调用都创建一个记录对象，如何优化这类高频小对象的内存占用？.md

在 Agent 高频调用 LLM 的场景里，每次请求生成一个记录对象，看似无害，但当 QPS 上去后，GC 压力和内存毛刺会让你头疼。我优化这类问题有一个固定思路，不是上来就改代码，而是先分清楚：这对象是“胖”还是“多”，是“短命”还是“长驻”。

## 题1：为什么小对象会压垮内存？

每个 Python 对象的固定开销大约 56 字节（PyObject 头部），如果你用普通的 class 且没做 slots，每个属性又有一个 dict 开销（又是 64+ 字节）。一个原本十几字节的数据，包装成对象可能膨胀到 150 字节以上。

1000 个请求/秒 → 150KB/s → 每秒产生大量垃圾，GC 频繁触发，CPU 也会被拖累。

### 第一步：把对象“压扁”——slots 或 dataclass(slots=True)

这是我几乎会第一时间做的事情。

```

# ❌ 没优化
class LLMRecord:
    def __init__(self, ts, model, tokens, latency):
        self.ts = ts
        self.model = model
        self.tokens = tokens
        self.latency = latency

# ✅ 优化后：无 __dict__，内存直降 50%-70%
from dataclasses import dataclass
@dataclass(slots=True)
class LLMRecord:
    ts: float
    model: str
    tokens: int
    latency: float
```

> 〔原文此处为飞书内嵌组件，内容未随导出〕

如果字段更少、值域更小，甚至可以考虑用 namedtuple（不可变，内存更小）或 array 模块分层存储。

### 第二步：对象复用——池化高频短命对象

不要每次调用都 LLMRecord(...) 然后丢弃，用一个预分配的对象池来循环使用，尤其是那些生命周期只在“生成本地日志或缓冲”的记录。

```
import queue
class RecordPool:
    def __init__(self, size=2000):
        self._pool = queue.Queue(size)
        for _ in range(size):
            self._pool.put(LLMRecord(0, "", 0, 0.0))  # 预置空对象
    def acquire(self, ts, model, tokens, latency):
        r = self._pool.get()          # 取出一个，可能阻塞或直接拿到
        r.ts = ts
        r.model = model
        r.tokens = tokens
        r.latency = latency
        return r
    def release(self, record):
        # 重置敏感字段，避免泄露
        record.model = ""
        self._pool.put(record)
```
