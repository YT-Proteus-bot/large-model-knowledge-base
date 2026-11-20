---
title: Redis集群与高可用
topic: AI Agent+后端精选八股文
sub_topic: Cache & DataBase篇/Redis
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/WYybwaFDnib3uYkGNzpcRvJsnzd
source_path: AI Agent+后端精选八股文/Cache & DataBase篇/Redis/Redis集群与高可用.md
questions: 3
chars: 1418
build_date: '2026-11-20'
images_removed: 2
---

# Redis集群与高可用

> 主题：AI Agent+后端精选八股文 › Cache & DataBase篇/Redis　｜　原始文件：AI Agent+后端精选八股文/Cache & DataBase篇/Redis/Redis集群与高可用.md

## 一、Redis 主从复制

## 题1：1 主从复制原理

### 主从复制机制

## 题2：Redis 主从复制的基本原理是什么？

Redis 的主从复制，简单说就是 “从库认主、全量快照、命令追赶” 三个步骤。它让一个或多个从库（Replica）保持与主库（Master）的数据一致，是实现读写分离、数据冗余和故障转移的基础。

整个复制过程可以分为四个阶段：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

阶段一：握手与协商

从库执行 REPLICAOF <master-ip> <master-port> 后，主动向主库发起连接。双方通过 REPLCONF 交换能力，从库发送 PSYNC ? -1 表示要进行一次全量同步（因为不知道主库的 replication ID）。

阶段二：主库生成 RDB 快照

主库收到全量同步请求后，执行 fork() 创建一个子进程。子进程将当前内存数据写入一个 RDB 文件。在这个 fork 瞬间，主库还会创建一个 replication buffer（复制缓冲区），此后所有新的写命令都会被同时写入这个 buffer，防止快照期间的数据丢失。

阶段三：RDB 传输与从库加载

主库将生成的 RDB 文件通过网络发送给从库。从库收到后，先清空自己的旧数据，然后加载 RDB 文件到内存。此时从库的数据与主库 fork 瞬间一致。

阶段四：增量命令追赶与命令传播

主库将 replication buffer 中积压的增量命令发送给从库，从库逐条执行。当从库追赶上主库的那一刻，复制状态进入 实时命令传播：此后主库每执行一条写命令，都会异步地广播给所有从库。从库执行完这条命令后，会向主库发送 REPLCONF ACK <offset>，报告自己的复制偏移量。

关键的偏移量（Replication Offset）

主库和从库各自维护一个偏移量。主库的 master_repl_offset 表示自己已经处理了多少字节的写命令；从库的 slave_repl_offset 表示自己追到了哪里。通过对比这两个值，我们可以监控复制延迟。

```

# 在主库上查看复制状态
redis-cli INFO replication

# master_repl_offset:1234567890
# 在从库上查看
redis-cli INFO replication

# slave_repl_offset:1234567890  # 与主库一致则同步正常
```

## 题3：Redis 2.8 之后引入的 PSYNC2 相比 SYNC 有什么改进？

在 2.8 之前，Redis 使用 SYNC 命令进行复制。SYNC 有一个致命缺陷：断线重连后，从库总是触发全量同步。哪怕只断开了 1 秒钟，从库也得重新接收整个 RDB 文件，主库也要 fork 子进程重新生成 RDB。在高并发下，这种频繁的 fork 和全量传输会严重消耗 CPU、内存和网络带宽，甚至可能导致主库 OOM。

PSYNC2 的核心创新：部分重同步

2.8 之后引入的 PSYNC 命令（以及后续 4.0 的 PSYNC2）让从库在断线重连后，可以只补发丢失的那一小段增量命令，而不必全量同步。这依赖于两个核心组件：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
