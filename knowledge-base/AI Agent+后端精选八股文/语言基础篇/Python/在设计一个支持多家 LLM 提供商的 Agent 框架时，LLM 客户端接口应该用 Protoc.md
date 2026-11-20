---
title: 在设计一个支持多家 LLM 提供商的 Agent 框架时，LLM 客户端接口应该用 Protocol 还是 ABC？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/BygBwIxJMigfN9kuf1ncHhm0nZf
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在设计一个支持多家 LLM 提供商的 Agent 框架时，LLM 客户端接口应该用 Protoc.md
questions: 2
chars: 1161
build_date: '2026-11-20'
images_removed: 0
---

# 在设计一个支持多家 LLM 提供商的 Agent 框架时，LLM 客户端接口应该用 Protocol 还是 ABC？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在设计一个支持多家 LLM 提供商的 Agent 框架时，LLM 客户端接口应该用 Protoc.md

这个问题问到点上了。我直接给结论吧——用 Protocol，而且几乎不用犹豫。👇”

## 题1：为什么是 Protocol？

```
📌 一句话：我们需要的是“长成这样的对象”，而不是“我儿子”。
```

> 〔原文此处为飞书内嵌组件，内容未随导出〕

### 我的真实设计思路

“做框架的时候，我只关心外部 LLM 客户端有没有 generate(prompt) 和 stream(prompt)，

它内部是 HTTP 调用、gRPC 还是本地推理，我完全不 care。

那自然就是 Protocol 的天下。”

极简示例（Python 3.10+）：

```
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

## 题2：什么时候我才会考虑 ABC？

- 需要在基类里放公共逻辑（比如统一的重试、token 计数），且这个逻辑确实不方便用装饰器/组合解决。
- 要求所有适配器强制注册到某个中心，且需要 isinstance 检查。
- 但即便这样，我也会优先考虑“Protocol + Mixin”或“Protocol + 工厂函数”，尽量不绑死。

### 最后一句总结

🦆 “面对多家 LLM 提供商，鸭子类型才是保持框架轻盈的秘诀。

选 Protocol，让接口回归‘约定’，而不是‘控制’。”

✨ 这样回答既有明确立场，又有代码佐证，还带点经验之谈，面试官会觉得你 务实、有架构感、不被过度设计困住。
