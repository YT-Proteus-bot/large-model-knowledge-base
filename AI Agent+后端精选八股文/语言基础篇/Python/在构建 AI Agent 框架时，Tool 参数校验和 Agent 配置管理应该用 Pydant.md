# 在构建 AI Agent 框架时，Tool 参数校验和 Agent 配置管理应该用 Pydantic v2 还是 dataclass？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ReecwcqrniwM1ykAVJWc1FrPnOf

这两个场景的数据血统完全不同，一个来自不可信的模型输出，一个来自开发者自己写的配置。所以选型上，我不会一刀切。

---

#### 🧰 Tool 参数校验：必须上 Pydantic v2

Agent 调用 Tool 时，参数是 LLM 生成的字符串或 JSON，充满不确定性——类型错误、缺字段、幻觉键名，都是家常便饭。

```
代码块
from pydantic import BaseModel, ValidationError
class SearchParams(BaseModel):
    query: str
    max_results: int = 10

# LLM 吐出来的可能是 {"query": "天气", "max_results": "五"}
try:
    params = SearchParams.model_validate(llm_output)
except ValidationError as e:
    # 精准拿到哪个字段错了、为什么错，直接返回给 LLM 修正
    send_error_to_llm(e.errors())
```

🛃 Pydantic 在这里的优势

- 强制校验+智能转换："10" 可以变 10，"abc" 立刻拦截
- 错误信息结构化：e.errors() 直接可以喂回给 LLM 做自我纠错
- 运行时类型保障：进入执行函数的数据一定是干净的，不需要散布 isinstance

🔸 dataclass 完全不做这些，它只会让 "五" 静静躺在 max_results 里，直到工具执行时炸出难看的 TypeError。

---

#### ⚙️ Agent 配置管理：dataclass 更胜一筹

Agent 的配置（模型名、温度、最大步数等）由开发者或部署系统注入，数据是可信的，不需要运行时校验。这里追求的是：

- 快速启动、零额外依赖
- 内存占用极低（一个进程可能创建大量配置副本）
- 可以冻结、可哈希、可配合 slots 极致瘦身

```
代码块
from dataclasses import dataclass
@dataclass(slots=True, frozen=True)
class AgentConfig:
    model: str = "gpt-4o"
    temperature: float = 0.7
    max_steps: int = 15
```

✅ dataclass 在这里的优势

- 原生 Python，初始化和访问速度极快
- frozen=True 让配置不可变，安全传递
- slots=True 把内存压到最低，适合一个进程跑多个 Agent 实例
- 没有 Pydantic 的解析开销，也不需要携带 Validator 元信息

---

#### 📊 决策速查表

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧩 一个框架里的共存示例

```
代码块

# config.py —— 用 dataclass
@dataclass(slots=True, frozen=True)
class AgentConfig:
    model: str
    temperature: float

# tools.py —— 用 Pydantic
class SendEmailParams(BaseModel):
    to: str
    subject: str
    body: str
```
