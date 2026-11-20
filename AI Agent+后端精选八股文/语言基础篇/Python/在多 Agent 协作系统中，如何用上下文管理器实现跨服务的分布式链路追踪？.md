# 在多 Agent 协作系统中，如何用上下文管理器实现跨服务的分布式链路追踪？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/T1YywHm0ei4KoGkwu2NchjZOnmY

🧵 用 with 接管 Span 的整个生命周期，再把 Trace ID 像快递单号一样自动向后传递。

别看就这一点，它能让链路追踪从‘手动噩梦’变成‘无感自动记录’。”

---

#### 🔍 我为什么选上下文管理器？

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧪 轻量实现（不用依赖重型库）

```
代码块
from contextlib import contextmanager
import uuid, time
from contextvars import ContextVar

# 当前协程/线程的追踪上下文
_trace_ctx: ContextVar[dict] = ContextVar("trace", default=None)
@contextmanager
def traced(name: str, parent_id: str = None):
    ctx = _trace_ctx.get()
    trace_id = ctx["trace_id"] if ctx else str(uuid.uuid4())[:8]
    span_id = str(uuid.uuid4())[:6]
    start = time.time()
    # 进入 Span：设置当前上下文，供下游获取
    token = _trace_ctx.set({"trace_id": trace_id, "span_id": span_id})
    print(f"🔹 [trace:{trace_id}] ▶️  {name} ({span_id})")
    try:
        yield trace_id, span_id
    except Exception as e:
        print(f"🔸 [trace:{trace_id}] ❌ {name} failed: {e}")
        raise
    finally:
        duration = time.time() - start
        print(f"🔹 [trace:{trace_id}] ⏹️  {name} ({duration:.2f}s)")
        _trace_ctx.reset(token)
```

#### 🚀 在多 Agent 场景下怎么跨服务传递？

Agent 之间一般通过 HTTP / gRPC / 消息队列通信。传播方法极简单：

- 上游调用前，从当前上下文拿到 trace_id 和 span_id
- 塞进请求头，比如 X-Trace-Parent
- 下游服务在自己的 traced 里接收 parent_id 参数，恢复上下文

```
代码块

# 上游 Agent
def call_agent_b(payload):
    ctx = _trace_ctx.get()
    headers = {"X-Trace-ID": ctx["trace_id"], "X-Span-ID": ctx["span_id"]} if ctx else {}
    requests.post("http://agent-b/api", json=payload, headers=headers)

# 下游 Agent 接收请求时
@app.post("/api")
def handle(payload, request: Request):
    parent_trace = request.headers.get("X-Trace-ID")
    with traced("agent_b.process", parent_id=parent_trace):
        ...  # 自动纳入同一条链路
```
