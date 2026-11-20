---
title: 在 RAG 管道中，需要同时处理文档解析、Embedding 计算和向量存储写入三个步骤，如何设计并发架构最大化吞吐量？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Lo7AwMwnSijrhckvvM5czuXnnwf
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在 RAG 管道中，需要同时处理文档解析、Embedding 计算和向量存储写入三个步骤，如何设.md
questions: 0
chars: 3001
build_date: '2026-11-20'
images_removed: 1
---

# 在 RAG 管道中，需要同时处理文档解析、Embedding 计算和向量存储写入三个步骤，如何设计并发架构最大化吞吐量？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在 RAG 管道中，需要同时处理文档解析、Embedding 计算和向量存储写入三个步骤，如何设.md

其实是在问如何把离线批处理的吞吐量打到最高。三个步骤——文档解析、Embedding、向量写入——性能瓶颈完全不一样，所以不能一锅烩，得拆成流水线。

## 先看每步到底卡在哪

- 文档解析：读文件（IO）+ 解析成文本（CPU 密集，尤其 PDF 布局分析、OCR）。卡 CPU 或磁盘。
- Embedding 计算：如果调远程 API（如 OpenAI），卡网络 IO；如果是本地模型（sentence-transformers），卡 GPU/CPU 计算，但大多会释放 GIL，可以用线程并行。
- 向量存储写入：调向量数据库的写入 API，通常是 网络 IO + 数据库内部索引开销，支持批量接口的话一次写多条效率远高于逐条写。

所以最大化吞吐的核心思路就是：流水线解耦 + 各段分别调并发度 + 攒批写入。

## 流水线架构（Producer-Consumer）

我把架构画成一个三段的异步管道：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

每一段之间用 asyncio.Queue 解耦，容量设个上限比如 500，起到背压作用，避免前端把后端撑爆。

## 各段并发方案怎么选

## 阶段 1：文档解析 → 多进程 + 异步文件读

- 用 aiofiles 异步读取文件（PDF、Word 等），把 IO 和解析分离。
- 解析本身是 CPU 重活，直接通过 loop.run_in_executor(process_pool_executor, parse_func, file_bytes) 扔进 ProcessPoolExecutor，进程数一般等于 CPU 物理核数。
- 每解析完一个文档，把切分好的文本块 put 进 Queue_A。

这里一个小优化：如果文档特别多，还可以先并行读一批文件到内存，再用进程池 map 解析，减少进程间传递大数据。

## 阶段 2：Embedding → 根据模型来源分情况

- 远程 API：全部走 asyncio。用一个 Semaphore 限制并发请求窗口，每拿到一个文本批次，async with sem 请求 API，返回向量后 put 进 Queue_B。
- 本地模型：加载一个模型实例（避免每个进程都加载一份），在线程池中跑推理。通过 loop.run_in_executor(thread_pool, model.encode, batch_texts) 调用。线程池大小按 GPU 并发上限或者 CPU 线程数调，配合 asyncio.wait 做并发。
- 关键：一定要攒小批（mini-batch）再送进模型，一个批次 32/64 条文本，比逐条调用高效一个数量级。所以这一阶段我会从 Queue_A 取到若干个文本块，凑满一小批再喂给 Embedding。

## 阶段 3：向量写入 → 异步批量写入 + 动态攒批

- 大部分向量数据库都有异步客户端（如 asyncpg for pgvector，qdrant-client 的 async 接口）。把多个向量攒到一个本地列表，达到 batch_size（比如 100）或者等待超时（比如 0.5 秒）就调用一次批量 upsert。
- 写入前同样用一个 Semaphore 限制与数据库的并发连接数，避免把数据库连接池占满。
- 如果数据库写入很慢，Queue_B 会自然堆积，形成背压，最终让前面的 Embedding 和解析也慢下来，整体不崩。

## 动态调节和防止抖动

单纯固定并发度不够，实际负载会有波动（文档大小不一、API 偶发延迟），我会加两个自适应手段：

1. Embedding 阶段根据 Queue_A 的堆积量动态调整批次大小：堆积多了就把 mini-batch 从 32 加到 64，加快消费；如果队列快空了，就减小 batch，降低延迟。
1. 向量写入的 batch_size 和最大延时动态匹配：吞吐高时尽量凑满 batch 一次写；快结束时要强制 flush 掉残留数据，别让最后一批卡着不写。

## 完整的代码结构感觉（伪代码）

```
async def doc_parser(files, queue_a, process_pool):
    for f in files:
        raw = await aiofiles.read(f)
        chunks = await loop.run_in_executor(process_pool, parse, raw)
        for c in chunks:
            await queue_a.put(c)
    await queue_a.put(None)  # 结束信号
async def embedder(queue_a, queue_b, sem, model, thread_pool):
    batch = []
    while True:
        chunk = await queue_a.get()
        if chunk is None:
            break
        batch.append(chunk)
        if len(batch) >= 32:
            vecs = await loop.run_in_executor(thread_pool, model.encode, batch)
            for v in vecs:
                await queue_b.put(v)
            batch = []
    if batch:
        # 剩余批次
        vecs = await loop.run_in_executor(thread_pool, model.encode, batch)
        for v in vecs:
            await queue_b.put(v)
    await queue_b.put(None)
async def vector_writer(queue_b, client, sem, batch_size=100):
    buffer = []
    while True:
        vec = await queue_b.get()
        if vec is None:
            break
        buffer.append(vec)
        if len(buffer) >= batch_size:
            async with sem:
                await client.upsert(buffer)
            buffer = []
    if buffer:
        async with sem:
            await client.upsert(buffer)
```
