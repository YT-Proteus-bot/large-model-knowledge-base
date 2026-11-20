---
title: 异步支持 (Async)
topic: LangChain面（高频）
sub_topic: 八、回调、流式处理与异步
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/XRv0wx736iBPPjk1VBdcvmwSnmb
source_path: LangChain面（高频）/八、回调、流式处理与异步/异步支持 (Async).md
questions: 3
chars: 2990
build_date: '2026-11-20'
images_removed: 0
---

# 异步支持 (Async)

> 主题：LangChain面（高频） › 八、回调、流式处理与异步　｜　原始文件：LangChain面（高频）/八、回调、流式处理与异步/异步支持 (Async).md

## 题1：🌊 LangChain 的异步方法（ainvoke, astream, abatch）是如何实现的？底层用了什么异步库？

LangChain 的异步方法本质上是将同步执行流程封装成协程，利用 Python 的 asyncio 库实现非阻塞并发。底层主要依赖 asyncio 的协程、Task、以及 run_in_executor 等机制，并结合 concurrent.futures 处理同步组件。

核心实现路径：

- 对于 LLM 调用，LangChain 内部使用 AsyncOpenAI 等异步客户端，其底层是 httpx.AsyncClient 或 aiohttp，实现了真正的异步 I/O。
- 对于自定义 Chain/Runnable，当你调用 ainvoke 时，会触发 _acall 或 atransform，这些方法内部会 await 下游组件的异步方法。
- 如果某个组件只实现了同步版本，LangChain 会通过 asyncio.to_thread 或 run_in_executor 将其调度到线程池，避免阻塞事件循环。

线程池的隐性存在：

- LangChain 内部有一个全局的 ThreadPoolExecutor，默认用于执行同步工具调用、同步回调等。在异步链中，如果工具是同步的，AgentExecutor 会在线程池中执行它。
- 这意味着即便使用了 ainvoke，如果某些组件是同步的，仍会引入线程切换开销，但不会阻塞主事件循环。

如何确认底层是否真正的异步：

- 检查 LLM 的 _astream 是否被重写。例如 ChatOpenAI 的 _astream 直接调用了 async for chunk in client.chat.completions.create(...)，这是原生异步。
- 回调方面，如果你使用了 AsyncCallbackHandler，则所有 on_* 方法都是协程，会被 await；如果使用同步回调，则在线程池执行。

实践经验：在一个高并发问答服务中，我全部替换为 AsyncChatOpenAI 并确保所有工具都是异步实现（用 aiohttp 访问外部API），配合 asyncio.Semaphore 限流，QPS 从同步版的 30 提升到 300+。如果混用同步工具，性能会急剧下降，因为线程池默认只有 min(32, os.cpu_count()+4) 个线程，成为瓶颈。

## 题2：⚡ 异步调用 LLM 时，你如何控制并发数量？使用 asyncio.Semaphore 可行吗？

asyncio.Semaphore 是控制异步并发的标准工具，完全可行，且非常高效。

使用方法：

- 创建一个全局或请求级的 Semaphore，设置最大并发数。
- 在每次 LLM 调用前 async with semaphore:，超过限制的协程会挂起等待。

示例：

```
import asyncio
from langchain_openai import ChatOpenAI
sem = asyncio.Semaphore(10)  # 最多10个并发LLM调用
async def ask_llm(prompt):
    async with sem:
        llm = ChatOpenAI(model="gpt-3.5-turbo")
        return await llm.ainvoke(prompt)
async def main():
    tasks = [ask_llm(f"Question {i}") for i in range(100)]
    results = await asyncio.gather(*tasks)
```

为什么 Semaphore 可行？

- 它工作在协程层级，当达到上限时，新的协程会阻塞在 acquire 上，但不会阻塞事件循环，其他协程可继续运行。
- 相比线程池的 max_workers，Semaphore 更轻量，且不会引入线程切换开销。

额外考量：

- API 限流：如果你的 API 有 RPM (每分钟请求数) 限制，Semaphore 只能控制并发，不能精确控制频率，需要结合令牌桶或滑动窗口（见问题9）。
- 连接池限制：异步 HTTP 客户端（如 httpx）本身有连接池限制，需要配合调整 limits 参数，否则 Semaphore 形同虚设。
- 多实例：在分布式环境，Semaphore 只能限制单进程并发，全局限制需要借助 Redis 等集中式计数器。

实践技巧：我通常将 Semaphore 封装成一个上下文管理器，并集成到 langchain 的回调中，在 on_llm_start 申请，on_llm_end 释放，实现透明化限流。

## 题3：🧪 写一个 asyncio.gather 并发执行多个链的示例，并处理异常。

假设我们要并行处理 3 个不同主题的问答，使用同一个链（或不同链），并优雅地处理个别任务的失败。

```
import asyncio
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 创建三条链，参数不同
async def run_chain(topic: str):
    llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)
    prompt = ChatPromptTemplate.from_template("用一句话解释：{topic}")
    chain = prompt | llm | StrOutputParser()
    return await chain.ainvoke({"topic": topic})
async def main():
    topics = ["量子计算", "区块链", "深度学习"]
    tasks = [run_chain(t) for t in topics]
    # gather 并发执行，return_exceptions=True 让异常作为结果返回
    results = await asyncio.gather(*tasks, return_exceptions=True)
    for topic, result in zip(topics, results):
        if isinstance(result, Exception):
            print(f"[错误] {topic}: {result}")
        else:
            print(f"[成功] {topic}: {result}")
```
