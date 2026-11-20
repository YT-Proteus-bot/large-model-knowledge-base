---
title: asyncio.gather 和 asyncio.as_completed 分别是什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/IEICwIfAHiY8aZktu4NcJq4unxd
source_path: AI Agent+后端精选八股文/语言基础篇/Python/asyncio.gather 和 asyncio.as_completed 分别是什么？.md
questions: 0
chars: 1370
build_date: '2026-11-20'
images_removed: 1
---

# asyncio.gather 和 asyncio.as_completed 分别是什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/asyncio.gather 和 asyncio.as_completed 分别是什么？.md

面试时被问到 asyncio.gather 和 asyncio.as_completed，我通常会先说一个形象的比喻，帮对方快速抓住本质，再落到代码上。

## 一句话抓住本质

- gather：像老师收卷子——必须等所有人交齐，我才拿到全部结果，顺序和你交卷的顺序（或安排好的顺序）一致。
- as_completed：像客服中心处理工单——谁先处理完，我就立刻响应谁，不需要等最慢的那个人。

## 先看签名的差别（很重要）

```

# gather：一批协程，返回一个列表，保持传入顺序
results = await asyncio.gather(*tasks)

# as_completed：返回一个异步迭代器，按完成顺序逐个产出
for coro in asyncio.as_completed(tasks):
    result = await coro
```

- gather 一次性拿到所有结果，如果某个任务抛异常，默认会立刻抛出，但也支持 return_exceptions=True 收集异常。
- as_completed 是按完成顺序逐个处理，异常不会中断其他任务，你可以在循环里单独捕获。

## 用图示理解执行过程

假设有三个任务，耗时分别为 快(1s)、中(2s)、慢(3s)：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

```
as_completed 的时间线：
[快]--1s--> ✓ 先拿到结果，立即处理
[中]------2s------> ✓ 第二个拿到
[慢]---------3s---------> ✓ 最后拿到
```

👉 如果“快”的结果足以驱动后续逻辑（比如提前返回），用 as_completed 可以大幅降低整体响应时间。

## 代码对比 + 常见场景

## 场景 1：批量调用多个 API，需要所有结果，且顺序重要

```
async def fetch(url):
    # 模拟请求
    ...
urls = ["/api/a", "/api/b", "/api/c"]
tasks = [fetch(u) for u in urls]
results = await asyncio.gather(*tasks)

# results[0] 一定对应 /api/a，即使 /api/b 先回来
```

## 场景 2：调用多个搜索源，只要有第一个成功就返回

```
async def search_provider(provider):
    ...
providers = [search_provider(p) for p in ["A", "B", "C"]]
for coro in asyncio.as_completed(providers):
    try:
        result = await coro
        # 拿到第一个成功结果，取消其余任务
        for t in providers:
            t.cancel()
        return result
    except Exception:
        continue
```
