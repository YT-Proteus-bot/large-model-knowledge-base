---
title: 流式处理 (Streaming)
topic: LangChain面（高频）
sub_topic: 八、回调、流式处理与异步
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/CZWewxKPziW9NLkL7iBcV35cncy
source_path: LangChain面（高频）/八、回调、流式处理与异步/流式处理 (Streaming).md
questions: 3
chars: 2538
build_date: '2026-11-20'
images_removed: 0
---

# 流式处理 (Streaming)

> 主题：LangChain面（高频） › 八、回调、流式处理与异步　｜　原始文件：LangChain面（高频）/八、回调、流式处理与异步/流式处理 (Streaming).md

## 题1：🌊 LangChain 中实现流式输出的核心方法是什么？stream 和 astream 的区别。

在 LangChain 中，流式输出是指模型每生成一个 token 就立即返回给客户端，而不是等整个回答生成完。实现流式输出的核心方法是 stream（同步）和 astream（异步），它们是 Runnable 接口的一部分，所有 LCEL 链、模型、组件都支持。

- stream(input, config)：返回一个 Python 迭代器，遍历它能逐个获取事件。对于 LLM，每个事件通常是一个 AIMessageChunk，携带增量内容（token）。它阻塞当前线程，适合同步环境。
- astream(input, config)：返回一个异步迭代器，适用于 asyncio 环境。它不阻塞事件循环，可以与其他异步任务并发执行，是构建高性能 Web 服务的基石。

两者本质上都是生产-消费模型的封装，底层依赖 LLM 自身的流式 API（例如 OpenAI 的 stream=True），LangChain 内部通过回调系统（CallbackManager）把这些 token 转化为事件流。

区别：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

在实际项目中，如果你使用 FastAPI 等异步框架，必须使用 astream，否则会阻塞整个服务；而在 Jupyter Notebook 或命令行脚本中，stream 更为便捷。

## 题2：🛠️ 在自定义链中，如何实现流式输出？需要继承哪些方法？

如果你要编写一个自定义链并支持流式输出，需要遵循 LangChain 的 Runnable 协议。相比于传统的 Chain 基类，LCEL 的 Runnable 是更推荐的实现方式。

关键步骤：

1. 继承 Runnable 而不是 Chain。Runnable 提供了统一的接口，包括 invoke、stream、astream 等。
1. 实现核心方法：
- invoke(self, input, config)：返回完整结果，用于非流式调用。
- stream(self, input, config)：返回同步迭代器。通常你需要重写此方法，在内部调用下游 LLM 的 stream，并处理中间逻辑，然后 yield 每个 chunk。
- transform(self, input, config)：这是一个生成器函数，框架默认的 stream 就是调用它。你可以重写 transform 来逐个产出事件。
- 对于异步，还有 ainvoke、astream、atransform。
1. 使用 @chain 装饰器（LangChain 提供）可以快速将一个函数转换为支持流式的 Runnable，但自定义复杂链时手动继承更灵活。

示例：一个简单的自定义流式链

```
from langchain_core.runnables import Runnable
from typing import Iterator
class MyStreamingChain(Runnable):
    def __init__(self, llm):
        self.llm = llm
    def invoke(self, input, config=None):
        return self.llm.invoke(input)
    def stream(self, input, config=None) -> Iterator:
        # 直接委托给 LLM 的 stream，返回每个 token chunk
        for chunk in self.llm.stream(input):
            yield chunk
```

如果你想在中间做一些处理，比如在生成 token 后添加一个前缀，可以在 stream 中实现：

```
def stream(self, input, config=None):
    yield "🤖 "  # 自定义输出
    for chunk in self.llm.stream(input):
        yield chunk
```

重要提示：要确保内部组件（如 LLM）已经开启流式模式，否则 stream 返回的可能是一次性结果而不是迭代器。

## 题3：📝 写出一个简单的 LCEL 链，并使用 stream 方法逐 token 打印结果。

下面是一个经典的 LCEL 链：Prompt → LLM → StrOutputParser，然后流式输出。

```
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

# 1. 创建组件
prompt = ChatPromptTemplate.from_template("请用50字以内解释：{topic}")
llm = ChatOpenAI(model="gpt-3.5-turbo", streaming=True)  # 必须开启 streaming
output_parser = StrOutputParser()

# 2. LCEL 链
chain = prompt | llm | output_parser

# 3. 流式调用
topic = "量子计算"
print("🤖 ", end="", flush=True)
for chunk in chain.stream({"topic": topic}):
    print(chunk, end="", flush=True)  # chunk 就是一个个字符串
print()  # 换行
```
