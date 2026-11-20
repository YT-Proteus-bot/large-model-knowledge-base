# Agent 并发调用多个 LLM 时，如何正确处理超时和取消？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/UOoewle0hi0IjpkWKFwcSrSynZg

Agent 并发调多个 LLM 时，超时和取消的核心难点是：既要保证整体响应速度，又不能留下“孤儿任务”白白消耗 token 和连接。

我直接说我们常用的处理方式👇

---

#### 🎯 核心思路：结构化并发 + 超时 + 取消传播

原则只有一个：所有子任务的生命周期必须嵌套在父任务之内，超时或某个子任务失败时，取消必须级联传播，确保资源全部释放。

---

#### 🛠️ 方案一：asyncio.wait + FIRST_COMPLETED + timeout（Python 原生灵活方案）

适合：部分结果即够用，超时后直接放弃未完成者。

```
代码块
import asyncio
async def call_llm(prompt, model):
    # 模拟调用大模型
    ...
async def agent_orchestrate():
    tasks = {
        asyncio.create_task(call_llm("...", "gpt-4")),
        asyncio.create_task(call_llm("...", "claude-3")),
    }
    done, pending = await asyncio.wait(
        tasks, timeout=10.0, return_when=asyncio.FIRST_COMPLETED
    )
    # 关键：取消所有还在跑的任务
    for task in pending:
        task.cancel()
    # 收集已完成的结果（过滤取消异常）
    results = []
    for task in done:
        try:
            results.append(task.result())
        except asyncio.CancelledError:
            pass
    return results
```

✅ 传播链：wait 的超时会直接让循环退出，我们显式调用 task.cancel()，内部 call_llm 中的 await 会抛出 CancelledError，如果底层 http 库支持取消（如 httpx 的 aclose），连接也会被中断。

---

#### 🛠️ 方案二：asyncio.gather + asyncio.wait_for（需要全部结果时）

```
代码块
async def agent_all_needed():
    try:
        results = await asyncio.wait_for(
            asyncio.gather(
                call_llm("...", "gpt-4"),
                call_llm("...", "claude-3"),
            ),
            timeout=15.0
        )
    except asyncio.TimeoutError:
        # gather 会自动取消内部所有任务，无需手动 cancel
        raise
```
