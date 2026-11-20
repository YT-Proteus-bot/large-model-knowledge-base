# Agent 并发调用多个工具，有的工具响应快有的慢，如何让用户尽早看到可用结果？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ZLCIwu195iR4DNk4DbQc7gYDnhe

这个问题，其实是在考察你如何设计 Agent 系统的“响应感知层” ，而不仅仅是工具调用本身。我通常会先抛出底层模型差异，再给出分层方案。

---

#### 🧠 先认清一个事实：Agent 的结果交付已经从“批处理”走向“流式交互”

过去的 Agent 循环是：LLM 决策 → 串行调用工具 A → 等 A 返回 → 调用工具 B → 等 B 返回 → 汇总回答。

现在的 LLM 可以一次决策出多个并行工具调用（如 OpenAI 的 parallel tool calls）。这时如果还等所有工具全返回再显示，用户看到的白屏时间就会等于最慢那个工具的耗时，体验很糟糕。

---

#### ⚡ 核心思路：把“全量收集再展示”改为“谁先回来，谁先上屏”

我们让 Agent 后端充当流式事件生产者，前端充当异步事件消费者，用下面分层方案落地：

![image](../../../../assets/FJv9dXgcxobq7Bx0QwPctCpunyg/PqyCbi6XgorhqUxmA9Hcn4Qdnso.png)

#### 🔧 核心实现：用 [asyncio.as](https://asyncio.as/)asyncio.as_completed 驱动“即完即推”

在 Agent 拿到 LLM 返回的多个工具调用后，我们不会用 gather 死等，而是用 as_completed 逐个处理：

```
代码块
import asyncio
from typing import AsyncIterator
async def execute_tools_streaming(tool_calls) -> AsyncIterator[dict]:
    tasks = [execute_single_tool(call) for call in tool_calls]
    for coro in asyncio.as_completed(tasks):
        try:
            result = await coro
            yield {"type": "tool_result", "data": result}
        except Exception as e:
            yield {"type": "tool_error", "error": str(e)}
```

这个异步生成器每产出一个结果，上层就立刻封装成 SSE 消息发给前端，前端立即渲染对应工具卡片——其他慢的工具卡片继续保持骨架屏或加载动画。

---

#### 🧩 前端配合：用“增量填充”降低焦虑

前端在收到 LLM 决定调用哪些工具时，立即生成一组占位卡片（Skeleton），每个卡片对应一个工具调用，并标记为 pending。
当后端推送某个工具的结果时，前端根据 tool_call_id 找到对应卡片，把状态切换为 completed 并填入内容。

```
代码块
用户体验时间线：
0.0s ─ 展示思考链 + 工具名称列表 (骨架)
0.8s ─ 工具A（搜索）完成 → 卡片A填充，用户已经开始阅读
1.2s ─ 工具B（计算）完成 → 卡片B填充
2.5s ─ 工具C（复杂SQL）完成 → 最后卡片填充
       用户不用等 2.5 秒才看到任何东西
```

#### ⚠️ 容易踩的坑 & 进阶设计
