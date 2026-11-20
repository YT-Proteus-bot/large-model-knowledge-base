# 在 LangChain 自定义 Tool 中，如果不用 Pydantic 定义参数 Schema 会怎样？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/U591weYdKiOonVkVWVwcXM4jnDu

在 LangChain 里自己写 Tool，如果不用 Pydantic 定义参数 Schema，就等于把“接口定义”的锅甩给了大模型——它会用随机的 JSON 来猜你的工具想要什么。

---

#### 📛 两种“不用 Pydantic”的常见替代（以及它们的问题）

##### 1. 纯字符串描述，让 LLM 自由生成参数

```
代码块
def search(keyword):
    return f"结果：{keyword}"
tool = Tool(
    name="search",
    func=search,
    description="搜索工具，输入一个关键词"
)
```

- 🎲 LLM 可能会生成 {"key": "天气"}、{"query": "天气"} 甚至 "天气" 纯字符串。
- 你必须在 func 里写一堆 if isinstance、字典取值、缺省值填充，业务代码里混进了大量解析脏活。

##### 2. 手动拼一个 JSON schema dict

```
代码块
tool = StructuredTool.from_function(
    func=search,
    name="search",
    description="搜索",
    args_schema={
        "type": "object",
        "properties": {
            "keyword": {"type": "string", "description": "关键词"}
        },
        "required": ["keyword"]
    }
)
```

- 虽然能约束字段名，但没有运行时校验：LLM 仍可能传 {"keyword": 123}，你照样要手动检查类型。
- 缺少 Pydantic 的自动转换（比如 "10" → 10）和嵌套模型的便利性。

---

#### 🧱 实际灾难：一个对比

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🚨 最致命的后果：错误信息无法反馈给 LLM

LangChain 的 Agent 循环中，如果工具抛出异常，错误信息会送回给 LLM 让它纠正。

- 用 Pydantic：抛出的 ValidationError 非常详细，比如 "keyword: field required"，LLM 直接能修正。
- 不用 Pydantic：你可能只会 raise ValueError("参数错误")，LLM 根本不知道哪个参数错了，陷入无效重试的死循环。

---

#### 💡 那么，是不是绝对不能用其他方式？

当然不是。如果你在做一个极简的 demo，或者 Tool 参数特别简单（比如只接受一个字符串），直接写一个 func(text: str) 然后让 Agent 传单参数也可以。但一旦参数超过两个，或参数有类型、范围约束，不用 Pydantic 就是在给自己挖坑。

---

在 LangChain 的 Tool 生态里，Pydantic 不仅是“校验器”，更是 LLM 与工具之间的可靠契约。放弃它，等于把接口的稳定性押注在模型的随机行为上——而生产环境最不缺的就是意外。
