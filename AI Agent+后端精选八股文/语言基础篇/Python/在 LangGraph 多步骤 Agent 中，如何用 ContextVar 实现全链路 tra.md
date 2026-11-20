# 在 LangGraph 多步骤 Agent 中，如何用 ContextVar 实现全链路 trace_id 透传，而不污染 AgentState？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ItywwJhspiFyprkJDxDcPfwanUe

在 LangGraph 这类多步骤 Agent 框架里，trace_id 透传是一个典型的横切关注点。如果处理不好，要么把业务状态（AgentState）污染成“大杂烩”，要么日志断断续续，排查问题像猜谜。下面我按一条清晰的逻辑线，聊聊怎么用 ContextVar 干净地解决它。

---

#### 🎯 问题厘清：为什么不能往 AgentState 里塞 trace_id？

AgentState 是 LangGraph 的“官方数据流”，节点通过它来传递业务信息，框架也依赖它做检查点、中断、重放。如果把 trace_id 放进 AgentState，会出现几个问题：

- 🧹 污染业务模型：trace_id 不描述业务，只是运维需要。混入后，所有节点都要知道“这个字段别碰”，徒增心智负担。
- 💾 序列化冗余：每次检查点保存都会带上 trace_id，纯属浪费，而且重放时 trace_id 可能是旧值，反而混淆。
- 🔗 图逻辑干扰：条件边如果不小心依赖了 trace_id，可能引入隐性 bug。

我们需要一种不通过 AgentState，却能贯穿所有节点、LLM 调用、工具执行的透传方式，这就是 ContextVar 的用武之地。

---

#### 🔧 方案拆解：用 ContextVar 构建“隐形光纤”

整个思路用一张图概括：

![image](../../../../assets/QXX7d0f9LoZV3IxZnqHc3lSbnCb/Mghnb5p9PoaKOexS2Gkcxv4QnRg.png)

下面分四步落地。

---

#### 1️⃣ 定义全局“上下文变量”，而非全局变量

在一个中心模块（比如 app/context.py）中定义，所有模块导入的是同一个 ContextVar 对象：

```
代码块
import contextvars
trace_id_var: contextvars.ContextVar[str] = contextvars.ContextVar(
    'trace_id', default='unknown'
)
```

这样全服务只有一个“变量框”，但每个异步 Task 里存的值互相隔离。

---

#### 2️⃣ 请求入口“种入” trace_id，图执行前完成

LangGraph 通常配合 FastAPI 等服务框架，在中间件或依赖注入里完成设置：

```
代码块
from fastapi import Request
@app.middleware("http")
async def trace_middleware(request: Request, call_next):
    # 优先取上游传的，没有则自生成
    trace_id = request.headers.get("X-Trace-ID") or generate_trace_id()
    token = trace_id_var.set(trace_id)
    try:
        response = await call_next(request)
        response.headers["X-Trace-ID"] = trace_id
        return response
    finally:
        trace_id_var.reset(token)   # 请求结束清理，防止泄露
```

这样，当 LangGraph 的图被调用时（例如 app.ainvoke(state)），当前 Task 的上下文里已经有了 trace_id。

---

#### 3️⃣ 在 LangGraph 节点内部“无痛读取”

节点函数不要从 AgentState 里取 trace_id，直接读 ContextVar：

```
代码块
from app.context import trace_id_var
async def my_agent_node(state: AgentState) -> dict:
    tid = trace_id_var.get()
    logger.info(f"[{tid}] 进入节点 my_agent_node")
    # 业务逻辑...
```
