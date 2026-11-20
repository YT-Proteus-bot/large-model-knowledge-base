---
title: 在 Agent 流式输出场景中，如果用户中途关闭了浏览器，服务端应该如何感知并停止 LLM 调用，避免产生不必要的 Token 费用？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/PzixwOOpoiXlgykJfcicGC2jnLg
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在 Agent 流式输出场景中，如果用户中途关闭了浏览器，服务端应该如何感知并停止 LLM 调用.md
questions: 1
chars: 1931
build_date: '2026-11-20'
images_removed: 0
---

# 在 Agent 流式输出场景中，如果用户中途关闭了浏览器，服务端应该如何感知并停止 LLM 调用，避免产生不必要的 Token 费用？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在 Agent 流式输出场景中，如果用户中途关闭了浏览器，服务端应该如何感知并停止 LLM 调用.md

## 题1：在 Agent 流式输出场景中，如果用户中途关闭了浏览器，服务端应该如何感知并停止 LLM 调用，避免产生不必要的 Token 费用？

这个问题在我们做智能客服 Agent 的时候特别关键——用户聊着聊着突然关掉网页，后端要是不管不顾继续生成 token，不仅浪费钱，还可能把对话上下文搞脏。下面我按 感知断开 → 取消 LLM 调用 → 实际落地 这条线来说。

🔸 第一步：怎么知道用户已经离开了？

不管你是用 SSE（Server-Sent Events）还是 WebSocket，底层都是 HTTP/1.1 的长连接或 HTTP/2 的流。客户端关闭页面时，浏览器会主动关闭 TCP 连接，服务端可以从几个层面感知到：

1️⃣ 检测写入失败（最直接）

在迭代生成器往响应体写数据的时候，如果连接断开，下一次 write 或 send 会触发 ConnectionResetError 或 BrokenPipeError。

```
try:
    for chunk in stream:
        yield chunk
except GeneratorExit:
    # FastAPI 的 StreamingResponse 迭代器被取消时会触发
    cleanup()
```

2️⃣ 监听底层请求对象的状态（Starlette/FastAPI）

```
async def event_generator(request: Request):
    for chunk in stream:
        if await request.is_disconnected():
            # 立即停止 LLM 调用
            break
        yield f"data: {chunk}\n\n"
```

FastAPI 的 Request.is_disconnected() 会检查底层 TCP 连接是否还活着，它其实是轮询底层 ASGI 的 transport.is_closing()。

3️⃣ 利用 ASGI 的取消机制

在 ASGI 应用里，当客户端断开时，Uvicorn 会往协程里抛出一个 asyncio.CancelledError。你可以在生成器里捕获它：

```
async def safe_stream(request: Request):
    try:
        # 这里调用 LLM
        async for token in llm.generate():
            yield f"data: {token}\n\n"
    except asyncio.CancelledError:
        # 客户端已断开
        await llm.cancel()  # 中断 LLM 请求
        raise
```

🔹 第二步：如何停止正在进行的 LLM 调用？

LLM 的 API 一般都是基于 HTTP 的请求，部分 SDK 支持流式读取。想要省 token，就得在连接断开时主动取消请求：

① 使用 OpenAI SDK 的取消机制

```
from openai import AsyncOpenAI
client = AsyncOpenAI()
async for token in await client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    stream=True
):
    yield token.choices[0].delta.content
```

如果迭代过程中发生 CancelledError，你需要关闭底层的 HTTP 连接，否则 SDK 可能还在缓冲数据。我们可以这样：

```
try:
    stream = await client.chat.completions.create(..., stream=True)
    async for token in stream:
        yield token.choices[0].delta.content
finally:
    await stream.response.aclose()  # 确保释放连接
```

aclose() 会优雅地关闭 HTTP 流，阻止继续接收数据，也就不会消耗更多 token（API 端也会停止生成）。
