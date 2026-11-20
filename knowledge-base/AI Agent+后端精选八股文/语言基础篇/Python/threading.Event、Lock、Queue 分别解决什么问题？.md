---
title: threading.Event、Lock、Queue 分别解决什么问题？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/DxH1wd3lRi427OkznwlcZFnnnrd
source_path: AI Agent+后端精选八股文/语言基础篇/Python/threading.Event、Lock、Queue 分别解决什么问题？.md
questions: 0
chars: 1321
build_date: '2026-11-20'
images_removed: 0
---

# threading.Event、Lock、Queue 分别解决什么问题？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/threading.Event、Lock、Queue 分别解决什么问题？.md

```

# 多线程协调三件套：
# Lock — 防踩踏
# Event — 等信号
# Queue — 传数据
```

这三者解决的并发问题截然不同，但在脑里分清楚之后，写多线程就不会乱塞锁。

## Lock：解决“此时此地只能我一个人”

典型场景：抢票、计数器、写日志文件。

```
import threading
lock = threading.Lock()
seats = 10
def buy(n):
    global seats
    with lock:                 # 拿到钥匙才能往下走
        if seats >= n:
            seats -= n
            print(f"出票{n}张，剩余{seats}")
```

- 📛 不加锁：线程A、B同时看到剩余10张，都认为自己能买，结果超卖。
- ✅ 加了锁：把“查余量→减库存”变成原子块，同一时刻只有一个线程在执行。

🧠 记法：Lock 是互斥，保护的是一块代码不被同时进入，核心是 acquire/release。

## Event：解决“我好了，你们可以动了”

典型场景：服务就绪通知、主线程等所有子线程初始化完毕、定时器唤醒。

```
ready = threading.Event()
def worker():
    print("worker 正在加载模型...")
    time.sleep(3)
    ready.set()         # 🟢 举旗：我准备好了
def boss():
    print("boss 等工人就绪...")
    ready.wait()        # 🔴 死等，直到旗子举起
    print("开始干活！")
```

- event.wait() 阻塞当前线程，直到另一线程调用 event.set()。
- event.clear() 可以把旗子降下来，用于循环事件。
- 与 Lock 的本质区别：Event 是跨线程的布尔信号，不保护任何共享对象，只管“行不行”的状态传递。

🚦 记法：Event 像红绿灯，红灯（未set）就停，绿灯（set）就走。

## Queue：解决“我生产，你消费，别直接握手”

典型场景：爬虫（一个线程抓链接，多个线程解析）、AI Agent 里消息队列、流水线。

```
from queue import Queue
import threading
q = Queue(maxsize=10)
def producer():
    for i in range(20):
        q.put(i)          # 队列满则自动阻塞等待
        print(f"生产 {i}")
def consumer():
    while True:
        item = q.get()    # 队列空则自动阻塞等待
        print(f"消费 {item}")
        q.task_done()
```
