# LangChain

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SuxqwVBLli4q94kJFBAcOLkFnZc

#### 🧱 1. LangChain 1.0 的四大核心组件

LangChain 1.0 的使命是：用搭积木的方式，把 LLM 调用、工具、记忆这些零散的能力组装成可运行的应用。 它的设计围绕四个抽象展开。

![image](../../../../assets/KdtUdbFF1oxR9yxijAscqSjhnBo/FoNvbXc3iosQAuxxjwAch1Z2nCc.png)

##### ① Chain（链）

本质：把多个步骤串起来，定义一个确定的执行流程。
你可以像写配置文件一样，声明“先用这个 prompt 调 LLM，再把输出传给下一个 prompt，或者传给某个 Python 函数”。最常见的 LLMChain 就是 PromptTemplate + LLM + 可选的输出解析器 的组合。

```
代码块
from langchain.prompts import PromptTemplate
from langchain.llms import OpenAI
from langchain.chains import LLMChain
prompt = PromptTemplate.from_template("给一家卖{product}的公司起三个名字")
chain = LLMChain(llm=OpenAI(), prompt=prompt)
print(chain.run("智能手表"))
```

Chain 让“多次 LLM 调用 + 数据处理”变成一个可复用的管道。

##### ② Agent（智能体）

本质：让 LLM 自己决定“下一步做什么”。

它不按固定顺序执行，而是用 LLM 作为推理引擎，在一个循环里重复“思考 → 选择工具 → 执行 → 观察结果”，直到认为可以回答用户。Agent 的核心是决策，而不是死板的流程。

```
代码块
from langchain.agents import load_tools, initialize_agent, AgentType
tools = load_tools(["serpapi", "llm-math"], llm=llm)
agent = initialize_agent(tools, llm, agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION, verbose=True)
agent.run("马斯克几岁了？他的年龄的平方根是多少？")
```

这段代码背后，Agent 会先搜索“马斯克年龄”，然后用计算器开平方，最后整合回答。

##### ③ Tool（工具）

本质：LLM 的外部手和脚。

任何被包装成“名字 + 描述 + 参数 schema + 可调用函数”的东西都可以是 Tool。Agent 通过 Tool 的 description 来决定什么时候用它、传什么参数。搜索、计算器、数据库查询、API 调用，都是 Tool。

```
代码块
from langchain.tools import Tool
def get_weather(city: str) -> str:
    return f"{city}：晴，25°C"
weather_tool = Tool(
    name="Weather",
    func=get_weather,
    description="输入城市名，返回天气。参数 city 为字符串。"
)
```

##### ④ Memory（记忆）

本质：跨轮次保持的对话状态。
