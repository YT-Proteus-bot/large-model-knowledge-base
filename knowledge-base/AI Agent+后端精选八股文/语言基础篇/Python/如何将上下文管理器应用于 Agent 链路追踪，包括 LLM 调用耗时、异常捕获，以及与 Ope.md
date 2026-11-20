---
title: 如何将上下文管理器应用于 Agent 链路追踪，包括 LLM 调用耗时、异常捕获，以及与 OpenTelemetry 的结合？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/AzMawKRIOinkUmksMn9cM2wunOc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/如何将上下文管理器应用于 Agent 链路追踪，包括 LLM 调用耗时、异常捕获，以及与 Ope.md
questions: 0
chars: 1602
build_date: '2026-11-20'
images_removed: 0
---

# 如何将上下文管理器应用于 Agent 链路追踪，包括 LLM 调用耗时、异常捕获，以及与 OpenTelemetry 的结合？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/如何将上下文管理器应用于 Agent 链路追踪，包括 LLM 调用耗时、异常捕获，以及与 Ope.md

Agent 调用链往往涉及工具调用、LLM 推理、检索增强等多个环节，排查慢调用和异常全靠日志是地狱模式。

上下文管理器天生为“进入-退出”模式设计，做链路追踪是一等公民。

## 核心思路：用 with 块包裹“追踪单元”

每个关键操作（LLM 调用、工具执行、RAG 检索）都作为一个 Span，用上下文管理器自动记录：

- ⏱️ 耗时
- ❌ 异常
- 📝 元数据（模型名、prompt token 数、工具名等）

跟 OpenTelemetry 集成后，这些 Span 自然串成 Trace，直接推到 Jaeger / Zipkin。

## 实战：定制 @traceable 上下文管理器

```
from contextlib import contextmanager
from time import perf_counter
import traceback
from opentelemetry import trace
tracer = trace.get_tracer(__name__)
@contextmanager
def traced_span(name: str, attributes: dict = None):
    span = tracer.start_span(name, attributes=attributes or {})
    start = perf_counter()
    try:
        yield span          # 可在 with 体内添加事件
    except Exception as e:
        span.record_exception(e)
        span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
        raise
    finally:
        duration = perf_counter() - start
        span.set_attribute("duration_ms", duration * 1000)
        span.end()
```

## Agent 场景应用：LLM 调用追踪

```
def call_llm(prompt: str, model="gpt-4"):
    with traced_span("LLM Call", {"model": model, "prompt_len": len(prompt)}) as span:
        try:
            response = openai.chat.completions.create(...)
            span.set_attribute("usage.total_tokens", response.usage.total_tokens)
            span.set_attribute("status", "ok")
            return response
        except Exception as e:
            span.set_attribute("status", "fail")
            raise
```

🔗 多层嵌套自动形成父子关系（OpenTelemetry 的 context 传播）：

```
Agent Loop
 └─ Tool Select
     └─ LLM Call
     └─ Web Search
         └─ HTTP Request
```

## 升级：装饰器一步到位
