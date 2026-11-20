# 在异步 AI Agent 服务中，如何用 ContextVar 实现请求级别的上下文隔离，避免并发请求间的数据污染？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/VJo8weeOriXgdQkkkebcyfXonVb

这个问题特别务实，是在真实项目中几乎一定会碰到的。面试官问这个，其实是在考察你有没有在并发环境下写过有状态的 AI Agent 服务，以及是否理解上下文隔离的工程细节。下面我按从问题到解决方案的逻辑链来组织，尽量不背概念，像在聊架构经验。

---

#### 🧨 先还原事故现场：数据污染到底怎么发生的？

想象一个基于 FastAPI + asyncio 的 AI Agent 服务，它对外提供 /chat 接口。这个 Agent 在调用 LLM、执行工具时，都需要知道“当前请求的用户是谁、会话 ID 是什么、对话历史到哪了”。

新手可能写出这样的代码：

```
代码块

# 全局状态或类属性（反例）
class Agent:
    def __init__(self):
        self.current_user_id = None
        self.conversation_history = []
agent = Agent()
@app.post("/chat")
async def chat(request: ChatRequest):
    agent.current_user_id = request.user_id   # 直接赋值
    agent.conversation_history = load_history(request.session_id)
    reply = await agent.run(request.message)
    return reply
```

在异步并发下，请求 A 刚设置完 current_user_id = "alice"，await 让出控制权，请求 B 进来把它覆盖成 "bob"。等请求 A 后续调用 LLM 时，拿到的就是 "bob"，导致 Alice 收到了 Bob 的回复，或者更糟——把 Bob 的历史发给了 Alice。

这就是典型的请求级状态污染，根源是：用“线程不安全 + 协程不安全”的共享变量来承载请求级数据。

---

#### 🎯 解决思路：给每个请求分配一个隔离的“上下文空间”

threading.local 解决不了这个问题，因为在一个线程里可以跑几十个协程，它们共享同一个 local 空间。我们需要的是 Task 级隔离——同一个协程 Task 内，所有子调用链自动共享同一份上下文，但不同 Task 之间绝对隔离。

这正是 contextvars.ContextVar 的设计初衷。把它用到 Agent 服务里，整个架构一下子就干净了。

---

#### 🏗️ 实现四步走（带图标）

🔹 Step 1：定义请求级上下文变量

不要零散定义，最好集中在一个模块里，像这样：

```
代码块
import contextvars
from typing import Optional, List, Dict

# 请求级上下文：用户ID、会话ID、对话历史等
user_id_var: contextvars.ContextVar[Optional[str]] = contextvars.ContextVar('user_id', default=None)
session_id_var: contextvars.ContextVar[Optional[str]] = contextvars.ContextVar('session_id', default=None)
history_var: contextvars.ContextVar[List[Dict]] = contextvars.ContextVar('history', default=[])
```

这样全局定义是安全的，因为具体的“值”是绑定在 Task 上的，不是绑定在变量本身。

---

🔹 Step 2：在请求入口“种”上下文（中间件）

这是最关键的一步，必须保证在任何业务逻辑执行之前，当前 Task 的上下文已经初始化完成。在 FastAPI 里通常用中间件：

```
代码块
from fastapi import Request
@app.middleware("http")
async def context_middleware(request: Request, call_next):
    # 从请求头、JWT token 等获取当前用户信息
    user_id = request.headers.get("X-User-ID", "anonymous")
    session_id = request.cookies.get("session_id", "unknown")
    # 设置上下文，注意保存 token，用于请求结束后恢复或清理
    token_user = user_id_var.set(user_id)
    token_session = session_id_var.set(session_id)
    try:
        response = await call_next(request)
        return response
    finally:
        # 清理当前 Task 的上下文，避免残留（尤其重要）
        user_id_var.reset(token_user)
        session_id_var.reset(token_session)
```
