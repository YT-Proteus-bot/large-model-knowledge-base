# Agent 在处理 LLM 流式响应时，应该用什么方式逐 token 处理输出？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/B3k0wp9cLipmdpk3af2cZPYrnRc

面试官问这个，一般是想看你有没有真正在项目里用 LangChain 搭过工具链，而不是仅仅跑通 Demo。我去年在给内部客服系统接自定义工具时，正好把这三个点都踩了一遍。下面按照 装饰器做了什么 → 依赖关系 → 注册为什么失败 → JSON Schema 怎么生成的 这个线索来聊。

---

![image](../../../../assets/WTItdL4YRoyCW1xyet7cr5K1nsc/ESNsboPJoo05B7xPf6YcC3t8ncd.png)

🔸 @tool 装饰器到底干了什么？

它本质上是一个函数 → Tool 对象的转换器。

- 把你的 Python 函数包装成一个 BaseTool 子类实例（通常是 Tool 或 StructuredTool）。
- 自动提取函数名、描述（docstring）、参数类型和默认值，生成一个符合 OpenAI Function Calling 格式的 JSON Schema。
- 把这个 Tool 注册到 LangChain 的全局或局部工具注册表中，供 Agent 调用。

可以理解为，它帮你把“普通函数”翻译成了 LLM 能理解的“插件说明书”。

---

📦 依赖关系：哪些库在背后撑着

```
代码块
@tool 装饰器
  │
  ├─ Pydantic           ← 核心，用于参数模型定义和校验
  ├─ typing_extensions  ← 为了兼容低版本 Python 的 TypedDict 等
  ├─ langchain_core     ← 提供 BaseTool、Tool 等基类
  └─ (可选) docstring_parser  ← 从 Google/NumPy 风格注释里抠描述
```

- Pydantic v1 vs v2 是最大的坑：LangChain 某些版本强依赖 Pydantic v1，如果你的环境里装了 v2，@tool 可能会在参数校验时直接崩掉，报错信息还很隐蔽（比如 AttributeError: 'FieldInfo' object has no attribute 'extra'）。
- 如果函数的参数没写类型注解，@tool 就没办法生成 Schema，这时候它虽然不会直接报错（会用 Any 类型），但 Agent 传参可能很乱，属于隐性失败。

---

⚠️ 工具注册失败的几个典型原因

> （此处为内嵌电子表格，需登录飞书查看）

🔹 最隐蔽的一次：我们有一个工具函数的参数名是 from，Pydantic 生成 Schema 时 from 成了非法 JSON key，直接报错。因为 from 是 Python 关键字，虽然变量名能用，但 JSON Schema 里会炸。改成 from_date 解决。

---

🧩 JSON Schema 的生成机制

这个过程可以拆成三步：

1️⃣ 签名分析
@tool 通过 inspect.signature 拿到函数的参数列表，包括参数名、类型注解、默认值。

2️⃣ Pydantic 模型构建
内部会动态创建一个 Pydantic BaseModel，字段来自函数参数。每个参数变成模型的一个 Field，类型注解就是 Field 的 type，默认值就是 Field 的 default。

3️⃣ 导出 JSON Schema
调用这个动态模型的 .schema() 方法（Pydantic v1）或 .model_json_schema()（Pydantic v2），生成符合 JSON Schema Draft 7 的定义。

举个例子，这样一个函数：

```
代码块
@tool
def get_weather(city: str, unit: str = "celsius") -> str:
    """获取指定城市的天气"""
    return f"{city} 温度 25 度"
```
