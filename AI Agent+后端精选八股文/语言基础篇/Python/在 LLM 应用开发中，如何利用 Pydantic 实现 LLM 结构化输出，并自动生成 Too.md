# 在 LLM 应用开发中，如何利用 Pydantic 实现 LLM 结构化输出，并自动生成 Tool Schema？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/XakpwSDVgidbNykqpnbcmFcmnIc

这个问题其实触及了 LLM 应用从“能聊天”到“能干活”的关键一步。下面我从定义模型 → 生成 Schema 约束输出 → 自动转成工具描述这条链路，把我的实践方式说清楚。

---

#### 🎯 先明确要解决什么

LLM 本质是文本进、文本出。想让它的输出被代码可靠消费，要么写又臭又长的正则，要么手写 JSON Schema 还不一定跟数据模型同步。我的思路是：让 Pydantic 成为唯一的真相来源，Schema 和解析全由它自动派生。

---

#### 📦 第一步：定义 Pydantic 模型，顺便把提示词“喂饱”

一切从业务模型开始，用 Field 把字段含义写清楚——这很重要，因为后面这些描述会直接交给 LLM 做约束。

```
代码块
from pydantic import BaseModel, Field
class UserIntent(BaseModel):
    """用户意图识别结果"""
    intent: str = Field(..., description="意图类别，如 '下单','售后','咨询'")
    confidence: float = Field(..., ge=0, le=1, description="置信度")
    entities: list[str] = Field(default_factory=list, description="涉及的产品或人名")
```

💡 一个踩坑心得：Field(description=...) 就是你的 prompt engineering 素材，写得越像“对 AI 的自然语言指令”，后续输出越稳。

---

#### 🔄 第二步：用 model_json_schema() 把模型变成 LLM 的“输出协议”

Pydantic 能直接导出标准的 JSON Schema。把它嵌入 system prompt 或作为 response_format（部分官方 API 可直接用），让 LLM 必须按这个模子输出。

```
代码块
schema = UserIntent.model_json_schema()

# 得到完整的 JSON Schema，包含类型、必填、字段描述等
prompt = f"""你是一个意图分析器。请严格按照以下 JSON Schema 输出（只输出 JSON，不要解释）：
{schema}
用户输入：{user_input}
"""
```

响应回来后，一行代码完成解析+校验：

```
代码块
try:
    intent_data = UserIntent.model_validate_json(llm_response)
except ValidationError as e:
    # 可以把 e.json() 反馈给 LLM 让它自纠正，形成一个重试闭环
    ...
```

🔁 我习惯在验证失败时把 Pydantic 的报错信息（它也是 JSON）原样丢回给 LLM 做二次修复，效果出奇的好。

---

#### 🧰 第三步：同一个模型，自动生成 Tool Schema（Function Calling 的灵魂）

做工具调用时，我们需要给 OpenAI/其他平台提供 tools 参数，里面包含函数名、描述和参数 JSON Schema。完全没必要手写，直接基于 Pydantic 模型自动生成。

1. 给模型写一个 docstring（会被转成工具描述），类名转成函数名。
1. 用 model_json_schema() 拿参数 schema。
1. 拼成标准的工具定义字典。

```
代码块
def pydantic_to_tool_schema(model: type[BaseModel], name: str = None):
    return {
        "type": "function",
        "function": {
            "name": name or model.__name__,
            "description": model.__doc__ or "",
            "parameters": model.model_json_schema()
        }
    }

# 定义一个可调用的工具模型
class WeatherQuery(BaseModel):
    """查询指定城市的实时天气"""
    city: str = Field(..., description="城市名称，英文如 'Beijing'")
    unit: str = Field(default="celsius", description="温度单位，celsius 或 fahrenheit")
tool_def = pydantic_to_tool_schema(WeatherQuery)

# 这个 tool_def 可以直接扔进 api 的 tools 列表里
```
