# Redis

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Q9QMw2nf7ixgxGkQ9NUcjdpZno0

## Redis 核心基础与持久化

#### ⚡ 1. Redis 为什么快？单线程模型为什么能支撑高并发？

Redis 快，是因为它把所有数据都放在内存里，并且用单线程扫清了并发控制的一切障碍。

一个常见的误解是“单线程 = 慢”。在 Redis 的场景里恰恰相反：单线程让它避免了多线程的锁竞争、上下文切换、以及 CPU 缓存失效的开销。当一个命令执行时间在微秒级时，把这些时间花在调度线程上反而是最大的浪费。

![image](../../../../assets/FFOFdzqhio3Iawx6CYcc3FcJn6w/W57cbBygsoymUXxuv1KcufPVnjc.png)

单线程能支撑高并发的三个支柱：

1. 纯内存操作：一次 SET 或 GET 只需要几微秒。瓶颈不在 CPU 计算，而在网络 IO。只要单线程能在微秒级处理完一个命令，它就能在 1 秒内处理几十万次请求。
1. IO 多路复用（epoll）：Redis 使用 epoll（Linux 下）同时监听成千上万个客户端 socket。哪个 socket 有数据到达，就处理哪个；没有数据时，主线程就阻塞在 epoll_wait 上，不浪费任何 CPU。这使得单线程可以轻松管理数万并发连接。
1. 避免锁和上下文切换：多线程程序在访问共享数据结构时必须加锁，锁竞争和线程调度会吃掉大量 CPU。Redis 单线程直接操作内存中的数据结构，零锁开销，CPU 缓存永远热乎。这就是为什么 Redis 的“单线程”反而比很多“多线程”存储系统更快。

关于 Redis 6.0 引入的多线程 IO：它只用于网络读写，把数据从 socket 搬进搬出的工作分给了 IO 线程，但命令执行仍然是单线程的。这就像在餐厅里，服务员帮你端菜、收碗筷（IO 线程），但厨房里仍然只有一个大厨（主线程）在炒菜。大厨的速度没变，但顾客的等待时间却因为并行上菜而缩短了。

一个简单的代码示例：感受一下 Redis 的单线程命令有多快。

```
代码块
import redis, time
r = redis.Redis(host='localhost', port=6379)

# 测试单线程吞吐量
start = time.time()
for i in range(100000):
    r.set(f"key{i}", f"value{i}")
elapsed = time.time() - start
print(f"10 万次 SET 耗时: {elapsed:.2f} 秒, QPS: {100000/elapsed:.0f}")

# 典型输出: 10 万次 SET 耗时: 0.78 秒, QPS: 128000
```

在 Agent 场景里，我们经常用 Redis 来存储对话状态、缓存 Embedding 结果，每一次读写都是微秒级完成的，这正是 Agent 能做到毫秒级响应的基础。

---

#### 💾 2. RDB 和 AOF 持久化有什么区别？混合持久化怎么工作？

Redis 数据在内存里，断电就丢。持久化就是给内存数据拍“快照”或记“日记”，让重启时能恢复。RDB 和 AOF 是两种完全不同的哲学。

```
代码块
RDB (快照)                        AOF (追加日志)
───────────                       ──────────────
  Redis 数据 → 二进制 dump.rdb      每一条写命令 → 追加到 appendonly.aof
  周期性保存 (如每 5 分钟)           实时记录 (每 1 秒或每条命令 fsync)
  恢复时直接加载二进制，极快         恢复时逐条重放命令，较慢
  可能丢失最近几分钟的数据           数据安全性极高，最多丢 1 秒
  文件小，适合备份                  文件大，但可重写压缩
```

## 子页面

- [分布式锁](分布式锁/_index.md)
- [Redis集群与高可用](Redis集群与高可用.md)
