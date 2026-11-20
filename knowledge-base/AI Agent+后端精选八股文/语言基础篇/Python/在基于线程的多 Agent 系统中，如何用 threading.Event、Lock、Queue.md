---
title: 在基于线程的多 Agent 系统中，如何用 threading.Event、Lock、Queue
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/XMIPwoGGiitjRWkJM6Hcb2Kenaf
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在基于线程的多 Agent 系统中，如何用 threading.Event、Lock、Queue.md
questions: 1
chars: 2492
build_date: '2026-11-20'
images_removed: 0
---

# 在基于线程的多 Agent 系统中，如何用 threading.Event、Lock、Queue

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在基于线程的多 Agent 系统中，如何用 threading.Event、Lock、Queue.md

## 题1：在基于线程的多 Agent 系统中，如何用 threading.Event、Lock、Queue 实现 Supervisor-Worker 架构，保证任务分发的线程安全和优雅停止？

### 问题重述

在一个多线程 Agent 系统中，如何用 threading.Event、Lock、Queue 搭建 Supervisor-Worker 架构，既保证任务分发线程安全，又能实现优雅停止？

核心目标：

- 任务提交与消费互不阻塞
- 多 Worker 竞争任务不出错
- 停止信号下发后，Worker 能快速、安全地退出，不丢任务、不死锁

```
🧱 架构角色（用图标一目了然）
👑 Supervisor 线程
   │
   ├── 创建任务 → 📮 Queue（线程安全任务缓冲）
   │
   ├── 维护 🔒 Lock 保护的共享状态（如活跃任务数、停止标识）
   │
   └── 收到停止信号 → 🚦 Event.set() 通知所有 Worker
👷 Worker 线程×N
   │
   ├── 循环从 📮 Queue 获取任务（阻塞或超时）
   │
   ├── 处理任务前/后更新共享状态（用 🔒 Lock）
   │
   └── 检测到 🚦 Event 置位 → 处理完当前任务后安全退出
```

### 三个同步原语的分工

> 〔原文此处为飞书内嵌组件，内容未随导出〕

### 落地实现（面试可展示的关键代码）

```
import threading
import queue
import time
import random
class SupervisorWorkerSystem:
    def __init__(self, num_workers=3):
        self.task_queue = queue.Queue()          # 📮 任务队列
        self.lock = threading.Lock()             # 🔒 保护共享计数
        self.stop_event = threading.Event()      # 🚦 停止信号
        self.active_count = 0
        self.workers = []
        self.num_workers = num_workers
    def start(self):
        # 启动 Worker 线程
        for i in range(self.num_workers):
            t = threading.Thread(target=self._worker, args=(i,), daemon=False)
            self.workers.append(t)
            t.start()
    def submit_task(self, task_data):
        """Supervisor 提交任务（线程安全，Queue本身保证）"""
        with self.lock:
            self.active_count += 1
        self.task_queue.put(task_data)
    def _worker(self, worker_id):
        """Worker 主循环：取任务→处理→检查停止信号"""
        while not self.stop_event.is_set():
            try:
                # 阻塞等待任务，但设置超时以便周期性检查停止信号
                task = self.task_queue.get(timeout=1)
            except queue.Empty:
                continue   # 超时后回到循环首部检查 stop_event
            # 处理任务
            self._process_task(worker_id, task)
            # 任务完成，扣减计数
            with self.lock:
                self.active_count -= 1
    def _process_task(self, worker_id, task):
        """模拟任务处理"""
        print(f"👷 Worker-{worker_id} 处理任务: {task}")
        time.sleep(random.uniform(0.1, 0.3))   # 模拟耗时
    def graceful_shutdown(self):
        """Supervisor 触发优雅停止"""
        print("\n🛑 Supervisor 发送停止信号...")
        self.stop_event.set()                  # 通知所有 Worker
        # 等待 Worker 线程全部结束
        for t in self.workers:
            t.join()
        # 最后检查是否还有未完成任务（实际场景可记录日志）
        with self.lock:
            remaining = self.active_count
        print(f"✅ 所有 Worker 已退出，剩余未处理任务: {remaining}")
```
