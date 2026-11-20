# Agent 系统里有多个模块都需要访问 LLM 客户端（如 openai.AsyncOpenAI），如何设计才能避免重复创建连接池，同时保证可测试性？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/GrqbwHfhnieU05k7iPnclH4Knad

这个问题本质上是两个需求的平衡：

既要全局共享一个连接池避免重复创建，又要方便替换以便测试。

我会用依赖注入 + 模块级单例来解决。”

---

#### ❌ 先看坏味道

多个模块各自实例化客户端：

```
代码块

# agent_a.py
client = AsyncOpenAI()  # 创建一个连接池

# agent_b.py
client = AsyncOpenAI()  # 又创建一个连接池
```

🔴 问题：

- 连接池被复制多份，资源浪费；
- 测试时你没法统一 mock，得去改每个模块的源码或者做猴子补丁，非常难看。

---

#### ✅ 正确设计：单例创建，依赖注入

把客户端的创建和使用分开：

```
代码块

# llm_client.py —— 全局只建一次
import openai
from functools import lru_cache
@lru_cache(maxsize=1)
def get_llm_client() -> openai.AsyncOpenAI:
    return openai.AsyncOpenAI()
```

然后每个模块不自己创建，而是通过参数接收：

```
代码块

# agent_a.py
class AgentA:
    def __init__(self, llm: AsyncOpenAI):
        self.llm = llm
```

组装时注入同一个实例：

```
代码块

# main.py
client = get_llm_client()
agent_a = AgentA(client)
agent_b = AgentB(client)
```

📊 直观示意图：

![image](../../../../assets/GXgyd3P3AoruOgxjBCRccbFgn1z/Zl1qbBuCPoHOuxxLB7ucawLXnoh.png)

#### 🧪 可测试性怎么来的？

测试时，你根本不用走真实网络：

```
代码块
from unittest.mock import AsyncMock
mock_client = AsyncMock()
agent = AgentA(mock_client)

# 直接断言行为，完全隔离
```
