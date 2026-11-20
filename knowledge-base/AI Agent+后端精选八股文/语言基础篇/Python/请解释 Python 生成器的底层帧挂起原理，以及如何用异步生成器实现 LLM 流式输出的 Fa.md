---
title: 请解释 Python 生成器的底层帧挂起原理，以及如何用异步生成器实现 LLM 流式输出的 FastAPI SSE 接口？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/GNWPwqF2liEjpVklyATcJpQenhc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/请解释 Python 生成器的底层帧挂起原理，以及如何用异步生成器实现 LLM 流式输出的 Fa.md
questions: 1
chars: 2245
build_date: '2026-11-20'
images_removed: 0
---

# 请解释 Python 生成器的底层帧挂起原理，以及如何用异步生成器实现 LLM 流式输出的 FastAPI SSE 接口？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/请解释 Python 生成器的底层帧挂起原理，以及如何用异步生成器实现 LLM 流式输出的 Fa.md

## 题1：请解释 Python 生成器的底层帧挂起原理，以及如何用异步生成器实现 LLM 流式输出的 FastAPI SSE 接口？

问这个，一般是看你有没有把 Python 的协程机制和 Web 流式输出打通。我之前在智能客服项目里，就是用异步生成器把 LLM 的 token 流转成 SSE 推给前端。下面先扒一下生成器挂起的底层，再串到 FastAPI 实现。

🔹 生成器的底层：帧挂起，不是魔法

Python 的生成器能把一个函数执行到一半“冻”住，靠的是帧对象（frame）的现场保存。

当你调用一个生成器函数时，CPython 做了这几件事：

1. 创建生成器对象（PyGenObject）

   这个 C 结构体里有两个关键字段：

- gi_frame：指向当前执行帧（PyFrameObject），包含代码对象、局部变量、指令指针（f_lasti）、栈等。
- gi_code：编译后的字节码对象。
1. 执行到 yield 时

   解释器正常执行字节码，遇到 YIELD_VALUE 指令：

- 把栈顶的值作为 yield 的返回值传给调用方。
- 把当前帧的指令指针、栈深度等信息完整保留在 gi_frame 里，然后把 gi_frame 字段保存，生成器状态变为 Gi_SUSPENDED。
- 注意：此时帧并没有被销毁，只是从解释器的调用栈上摘下来，就地冻结。
1. 下一次 next() 或 send() 时

   解释器找到生成器的 gi_frame，重新压入调用栈，从 f_lasti 记录的字节码偏移后一条指令继续执行，局部变量、栈内容原封不动恢复——就像电影暂停后继续播放。

这样，生成器函数就实现了无状态回调的“有状态暂停”。从 Python 层面看，就是一个对象，但它的背后挂着一整个活着的执行上下文。

📦 用个图说话（逻辑示意）

```
生成器对象 (gen)
   ├─ gi_frame ──► 帧对象 (Frame)
   │                  ├─ f_lasti: 12 (下一条指令偏移)
   │                  ├─ f_locals: {'i': 3, 'data': [...]}
   │                  ├─ f_stack: [值1, 值2]
   │                  └─ f_code: Code Object
   └─ gi_code ────► 同样的 Code Object
```

gi_frame 在生成器耗尽（StopIteration）后会变成 None，意味着帧被释放。

⚙️ 异步生成器：挂起还能等 I/O

异步生成器（async def + yield）底层更复杂，用的是 asyncgenobject，帧挂起原理一样，但 yield 点必须与事件循环协作。

当执行到 yield 时，它把当前协程的帧保存，然后将控制权通过 await 链返回给事件循环，这样循环可以去调度其他任务。

等到下一轮 async for 调用 anext 时，再次唤醒这个帧继续往后跑。

这就让它天然适合一边生成数据一边等待异步操作的场景，比如 LLM 的 token 流。

🔸 用异步生成器实现 LLM 流式输出的 FastAPI SSE

SSE（Server-Sent Events）是单向的流式协议，数据格式是 data: <内容>\n\n。

FastAPI 支持用 StreamingResponse 包裹一个异步生成器，并设置 media_type="text/event-stream"。

我们线上最简化的实现是：

```
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
app = FastAPI()
async def llm_token_generator(prompt: str):
    """模拟 LLM 异步生成 token"""
    # 这里实际会调用 openai.AsyncOpenAI 等
    for token in ["今天", "的", "天气", "很", "好", "。"]:
        await asyncio.sleep(0.1)   # 模拟生成延迟
        # SSE 格式
        yield f"data: {token}\n\n"
    yield "data: [DONE]\n\n"
@app.get("/stream")
async def stream(prompt: str):
    return StreamingResponse(
        llm_token_generator(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # 关掉 Nginx 缓冲
        }
    )
```
