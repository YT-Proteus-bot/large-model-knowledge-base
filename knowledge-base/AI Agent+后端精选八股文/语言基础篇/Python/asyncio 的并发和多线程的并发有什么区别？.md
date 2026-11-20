---
title: asyncio 的并发和多线程的并发有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/YiBvw7JLWiVTqKkVc7ScMwsSnMe
source_path: AI Agent+后端精选八股文/语言基础篇/Python/asyncio 的并发和多线程的并发有什么区别？.md
questions: 3
chars: 1713
build_date: '2026-11-20'
images_removed: 0
---

# asyncio 的并发和多线程的并发有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/asyncio 的并发和多线程的并发有什么区别？.md

> ✅ **一句话答案**：asyncio 是**单线程内的协作式并发**，靠事件循环在 I/O 等待点主动让出控制权；多线程是**操作系统级的抢占式并发**，靠内核调度多个线程并行执行。两者解决的都是"并发"问题，但底层模型、调度方式和适用场景完全不同。

## 核心区别一览

**对比维度**

**asyncio 并发**

**多线程并发**

执行模型

单线程 + 事件循环

多线程，由 OS 调度

调度方式

协作式，协程在 await 处主动让出

抢占式，内核时间片轮转

切换成本

极低（用户态切换，微秒级）

较高（内核态切换 + 上下文保存）

CPU 密集任务

不适合（受 GIL 限制，无法利用多核）

受限（Python 线程同样受 GIL 限制）

I/O 密集任务

非常适合，单线程可支撑海量连接

适合，但线程数和切换开销是瓶颈

共享数据

天然单线程，无竞态，无需加锁

存在竞态条件，需要加锁保护

调试难度

相对容易（执行流可预测）

较难（线程交错难以复现）

典型场景

高并发网络服务、爬虫、Web 框架（FastAPI 等）

阻塞式 I/O 混合、GUI 后台任务、调用不支持异步的库

## 关键差异详解

## 题1：调度模型不同

- **asyncio**：单线程内由事件循环统一调度，协程执行到 await（如网络请求、读写文件、sleep）时主动让出控制权，等 I/O 完成后再回来继续执行。全程只有一条线程，不存在"并行"。
- **多线程**：多个线程由操作系统内核抢占式调度，每个线程可能同时运行（多核时），也可能随时被切走，开发者无法预知线程间的执行顺序。

## 题2：并发与并行的本质

- **asyncio 是"并发"而非"并行"**：任意时刻只有一个任务在执行，靠快速切换制造"同时进行"的效果。
- **多线程（多核 CPU）可以"并行"**：多个线程真正同时运行在不同核心上；但 Python 受 GIL 限制，纯计算线程也无法真正并行，I/O 时才会释放 GIL。

## 题3：数据安全与竞态

- **asyncio**：协程切换只发生在 await 点，非等待代码段是原子的，天然避免竞态，不需要加锁。
- **多线程**：线程可能在任意指令间被切换，共享变量容易产生竞态条件，必须借助 Lock、RLock 等同步原语保护。

## 代码示例对比

### asyncio：遇到 await 自动切换

```
asyncio 示例
import asyncio
async def task(name, delay):
    print(f"{name} 开始")
    await asyncio.sleep(delay)   # 这里让出控制权
    print(f"{name} 完成")
async def main():
    # 两个协程"同时"推进，总耗时约 2s 而非 4s
    await asyncio.gather(
        task("任务A", 2),
        task("任务B", 2),
    )
asyncio.run(main())
```

### 多线程：操作系统抢占调度

```
threading 示例
import threading, time
def task(name, delay):
    print(f"{name} 开始")
    time.sleep(delay)            # 阻塞当前线程，由 OS 调度
    print(f"{name} 完成")
t1 = threading.Thread(target=task, args=("任务A", 2))
t2 = threading.Thread(target=task, args=("任务B", 2))
t1.start(); t2.start()           # 两个线程并行推进
t1.join(); t2.join()
```
