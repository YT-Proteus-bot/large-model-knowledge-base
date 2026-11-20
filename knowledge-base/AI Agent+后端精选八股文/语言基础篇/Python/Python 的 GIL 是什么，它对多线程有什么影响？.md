---
title: Python 的 GIL 是什么，它对多线程有什么影响？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/LeDwwf37aiyuXnkxXp0cRk7jnJf
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 的 GIL 是什么，它对多线程有什么影响？.md
questions: 0
chars: 1118
build_date: '2026-11-20'
images_removed: 0
---

# Python 的 GIL 是什么，它对多线程有什么影响？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 的 GIL 是什么，它对多线程有什么影响？.md

面试官抛出 GIL，其实是想看你有没有在真实场景里被它“坑”过，以及是否知道如何绕开它。我先从一次线上故障讲起，再拆原理和影响，最后落到怎么选型。

🔸 一句话讲清 GIL 是什么

GIL（Global Interpreter Lock）是 CPython 解释器里的一把全局互斥锁，它规定：同一时刻，只有一个线程可以执行 Python 字节码。哪怕你有 16 核 CPU，多线程也只能在单核上轮转，根本做不到真正的并行计算。

⚙️ 为什么 CPython 非要搞这把锁？

根本原因是为了内存管理的线程安全。

CPython 内部用引用计数管理对象生命周期，比如 a = [] 这个对象的引用计数就 +1。

如果多个线程同时修改同一个对象的引用计数，没有锁就会发生竞争，导致内存泄漏或对象提前释放。

最简单的解法就是加一把全局大锁，让任何线程操作 Python 对象之前都必须先拿到 GIL。代价就是多线程变成“并发”而不是“并行”。

📌 对多线程的具体影响，分三种场景说

## 1️⃣ CPU 密集型任务：多线程反而更慢

我们有一次在 API 里用多线程做图片哈希计算（纯 Python 循环），4 核机器上开 4 个线程，理论应该快 4 倍，结果反而比单线程慢了 15%。

原因：

- 线程频繁争抢 GIL，上下文切换开销巨大。
- 操作系统以为线程在工作，实际 CPU 时间大量消耗在锁竞争和唤醒上。

```

# ❌ 多线程做 CPU 密集型，等于负优化
def compute_hash(data):
    # 纯 Python 字节码，全程霸占 GIL
    ...
```

结论：CPU 密集必须用 multiprocessing 多进程，或 C 扩展（Cython/Numba）释放 GIL。

## 2️⃣ I/O 密集型任务：多线程很合适

当线程在等待网络响应、磁盘读写时，它会主动释放 GIL（比如在 socket.recv 等 C 函数内部）。

这时其他线程可以立即获得 GIL 开始工作，所以 I/O 密集场景下，多线程能显著提升吞吐。

我们自研的爬虫调度器，用 ThreadPoolExecutor 做并发下载，10 个线程跑满带宽，GIL 几乎不是瓶颈。

## 3️⃣ 混合型任务：需要实测，不能想当然

我们做过一个报表生成服务，前半部分是数据库查询（I/O 密集），后半部分是 JSON 序列化和压缩（CPU 密集）。

最终方案是：线程池做并发查询，返回数据后交给进程池做序列化，中间用 Queue 传递数据，这样 GIL 对 CPU 部分的影响被多进程绕开了。
