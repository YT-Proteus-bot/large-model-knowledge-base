# MySQL高可用与主从复制

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Lkf0w2Rj4iL8r4kgqkkcZt2WnIb

## 一、MySQL 主从复制架构

### 1.1 主从复制基本原理

#### MySQL 主从复制的核心流程

##### 1、基础题：MySQL 主从复制的基本原理是什么？

**⭐**（考察主从复制流程、线程角色、binlog 作用）

MySQL 主从复制通过三个线程实现数据同步：

1. **主库线程**：执行 SQL 语句，将数据变更记录到 binlog
1. **从库 IO 线程**：连接主库，请求 binlog，写入 relay log
1. **从库 SQL 线程**：读取 relay log，重放 SQL 语句

核心流程：主库写入 binlog → 从库 IO 线程拉取 binlog → 从库 SQL 线程重放执行

##### 2、进阶题：详细描述主从复制的完整流程，包括各个线程的作用和 relay log 的机制

**⭐⭐**（考察线程协作机制、relay log 作用、复制延迟原因）

1️⃣ Common Answer 主从复制就是主库写数据的时候，会把 SQL 记录到 binlog 里，然后从库有个 IO 线程去拉取 binlog，拉回来之后放到 relay log 里，然后 SQL 线程再从 relay log 里读出来执行，这样从库就和主库一样了。反正就是通过三个线程配合，主库负责写 binlog，从库负责拉取和执行，最终实现数据同步。relay log 就是中间的一个日志，用来存储从库拉过来的 binlog 内容。

2️⃣ Impressive Answer MySQL 主从复制的核心是通过三个线程的协作实现数据同步，我来详细说明整个流程：

**首先，主库侧的工作机制：** 当主库执行写操作（INSERT、UPDATE、DELETE）时，事务提交后会将数据变更记录到 binlog（二进制日志）中。binlog 是逻辑日志，记录的是 SQL 语句或行数据的变化。

**其次，从库侧的拉取机制：** 从库启动后会创建一个 IO 线程，该线程会连接到主库，并请求从指定位置开始读取 binlog。IO 线程会将读取到的 binlog 事件写入从库本地的 relay log（中继日志）中，同时记录读取位置到 [master.info](http://master.info)master.info 文件中，确保断线重连后能从正确位置继续。

**最后，从库侧的重放机制：** 从库的 SQL 线程会持续读取 relay log 中的事件，并在从库上重新执行这些 SQL 语句，从而实现数据同步。SQL 线程会将执行位置记录到 [relay-log.info](http://relay-log.info)relay-log.info 文件中，确保异常恢复后能从正确位置继续。

**关键点说明：**

- IO 线程和 SQL 线程是异步工作的，互不阻塞
- relay log 是从库本地的日志文件，作用是缓冲和持久化
- 复制延迟主要发生在 SQL 线程执行阶段，尤其是大事务或复杂查询时

```
代码块
sequenceDiagram
    participant Client
    participant Master
    participant Slave_IO
    participant Slave_SQL
    Client->>Master: 执行写操作
    Master->>Master: 写入 binlog
    Slave_IO->>Master: 请求 binlog
    Master-->>Slave_IO: 返回 binlog 事件
    Slave_IO->>Slave_IO: 写入 relay log
    Slave_SQL->>Slave_SQL: 读取 relay log
    Slave_SQL->>Slave_SQL: 重放执行 SQL
```

3️⃣ Key Differences

> （此处为内嵌电子表格，需登录飞书查看）

##### 3、场景题：生产环境主从复制延迟严重，你如何排查和解决？

**⭐⭐⭐**（考察复制延迟原因分析、优化方案、监控手段）

1️⃣ Common Answer 复制延迟的话，可能是从库机器性能不好，或者主库写入太快了。可以检查一下从库的硬件配置，看看是不是 CPU 或者磁盘 IO 不行。还有就是看看主库是不是有大事务，大事务会导致从库执行很慢。优化的话，可以升级从库硬件，或者让主库不要写那么多数据，拆分一下大事务。监控的话就看 Seconds*Behind*Master 这个指标。

2️⃣ Impressive Answer 主从复制延迟是生产环境常见问题，我遇到过多次，系统性地从以下几个方面排查和解决：

**第一，延迟原因分析：** 复制延迟主要发生在三个环节：

1. **网络延迟**：主从之间网络带宽不足或延迟高，IO 线程拉取 binlog 慢
1. **从库性能瓶颈**：从库硬件配置低，SQL 线程执行慢
1. **大事务问题**：主库执行大事务（如大批量 UPDATE、长时间事务），从库需要同步执行相同的事务，导致延迟累积
1. **锁竞争**：从库上存在长查询或锁等待，阻塞 SQL 线程执行

**第二，排查手段：**
