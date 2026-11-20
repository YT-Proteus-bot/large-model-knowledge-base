# 在设计一个支持多家 LLM 提供商的 Agent 框架时，LLM 客户端接口应该用 Protocol 还是 ABC？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/BygBwIxJMigfN9kuf1ncHhm0nZf

这个问题问到点上了。我直接给结论吧——用 Protocol，而且几乎不用犹豫。👇”

---

#### 🧠 为什么是 Protocol？

```
代码块
📌 一句话：我们需要的是“长成这样的对象”，而不是“我儿子”。
```

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧩 我的真实设计思路

“做框架的时候，我只关心外部 LLM 客户端有没有 generate(prompt) 和 stream(prompt)，
它内部是 HTTP 调用、gRPC 还是本地推理，我完全不 care。
那自然就是 Protocol 的天下。”

极简示例（Python 3.10+）：

```
代码块
from typing import Protocol, Iterator
class LLMClient(Protocol):
    def generate(self, prompt: str, **kwargs) -> str: ...
    def stream(self, prompt: str, **kwargs) -> Iterator[str]: ...

# 完全不需要继承！任何满足这个形状的对象都可以
class OpenAICompat:
    def generate(self, prompt, **kwargs):
        # ... 调用 OpenAI API
        return response
    def stream(self, prompt, **kwargs):
        # ... 流式生成
        yield chunk
def run_agent(client: LLMClient, task: str):
    return client.generate(task)
```

🎯 看，OpenAICompat 和我的框架零耦合，以后换 Anthropic、Claude、Ollama，只要包一层满足 Protocol 就行。

---

#### ⚠️ 什么时候我才会考虑 ABC？

- 需要在基类里放公共逻辑（比如统一的重试、token 计数），且这个逻辑确实不方便用装饰器/组合解决。
- 要求所有适配器强制注册到某个中心，且需要 isinstance 检查。
- 但即便这样，我也会优先考虑“Protocol + Mixin”或“Protocol + 工厂函数”，尽量不绑死。

---

#### 🎤 最后一句总结

🦆 “面对多家 LLM 提供商，鸭子类型才是保持框架轻盈的秘诀。
选 Protocol，让接口回归‘约定’，而不是‘控制’。”

---

✨ 这样回答既有明确立场，又有代码佐证，还带点经验之谈，面试官会觉得你 务实、有架构感、不被过度设计困住。
