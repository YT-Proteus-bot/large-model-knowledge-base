# LangServe

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/K973wYln7iD4oCkYW2acmsT4nke

#### 1. 🚀 LangServe 是什么？它解决了什么问题？

LangServe 是 LangChain 官方推出的部署工具，它能将你的 LangChain 链（LCEL 链、Agent 等）自动转化为生产级的 REST API。它不是一个独立的框架，而是基于 FastAPI 构建的一个高级封装，能够感知 LangChain 的 Runnable 接口，并暴露相应的端点。

在 LangServe 出现之前，开发者要将一条链部署为 API，通常需要手动编写大量的 Web 框架（如 FastAPI）代码：定义输入输出模型、处理请求参数、处理流式响应、编写 Swagger 文档、实现错误处理等。这是一套重复且易出错的“体力活”。LangServe 通过简单的 add_routes 函数，将这些常见的部署需求自动化了。

LangServe 具体解决了以下问题：

- 零样板代码部署：只需几行代码就能将任何 LCEL 链（Runnable 对象）暴露为 REST API，无需手动编写路由函数。
- 自动生成 OpenAPI 文档：由于 LangChain 的 Runnable 有明确的输入输出 schema（通过 with_types 或 Pydantic 模型），LangServe 能够自动生成符合 OpenAPI 规范的文档，并附带 Swagger UI。
- 原生流式支持：自动为链创建 /stream 和 /stream_log 等端点，支持 Server-Sent Events (SSE) 和 WebSocket，让流式输出即开即用。
- 内置 Playground：提供一个可选的 Web 界面，开发者可以在浏览器中直接测试链，调整参数，查看输出，极大地方便了调试和演示。
- 客户端自动生成：LangServe 服务端可以提供一个 /openapi.json，LangChain 的 RemoteRunnable 客户端可以直接通过这个 URL 远程调用链，实现了“服务端代码即客户端定义”。
- 配置与扩展性：支持通过 config 动态传递回调、标签等运行时配置；可以集成 FastAPI 的依赖注入、中间件等，便于添加认证、日志等功能。
- 生产级特性：基于 FastAPI，天然支持高性能异步、自动验证、错误处理等生产必备能力。

简单来说，LangServe 把 LangChain 开发者的主要精力从“如何部署”转移回了“如何构建更好的链”，它把部署过程标准化、自动化了，是 LangChain 从原型到产品的桥梁。

---

#### 2. 💻 用 LangServe 将一个 LCEL 链部署为 API，最少需要几行代码？写出示例。

最少只需要 6 行有效代码（除去导入和模型初始化）。以下是一个最简示例，将一个简单的问答链部署为 API：

```
代码块

# 1. 导入必要模块
from fastapi import FastAPI
from langserve import add_routes
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# 2. 创建FastAPI应用
app = FastAPI(title="我的LLM服务")

# 3. 构建LCEL链
prompt = ChatPromptTemplate.from_template("请解释：{topic}")
llm = ChatOpenAI(model="gpt-3.5-turbo")
chain = prompt | llm

# 4. 将链添加为路由（核心！）
add_routes(app, chain, path="/explain")
```

这段代码做了以下事情：

- 创建了一个 FastAPI 实例。
- 定义了一个简单的 LCEL 链：接受 topic，生成解释。
- 使用 add_routes 将这个链挂载到 /explain 路径上。

启动服务（uvicorn app:app --reload）后，你将自动获得以下端点：

- POST /explain/invoke：单次调用，返回完整结果。
- POST /explain/stream：流式调用，返回 SSE 流。
- POST /explain/batch：批量调用。
- GET /explain/playground/：一个在线测试页面（需显式开启 playground=True）。

如果想进一步控制输入输出，可以为链添加类型注解，LangServe 会自动生成对应的 JSON Schema：

```
代码块
from pydantic import BaseModel
class TopicInput(BaseModel):
    topic: str
chain = prompt | llm

# 指定输入类型
add_routes(app, chain.with_types(input_type=TopicInput), path="/explain")
```

这样 OpenAPI 文档就会清晰地展示 topic 是一个必填的字符串字段。可见 LangServe 将部署复杂度降到了最低，让开发者把精力集中在链的逻辑上。

---
