# 在定义 Agent 组件接口时，Python 的 Protocol 和 abc.ABC 各有什么特点，在 Agent 框架设计中应该如何选择？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/XOUiwn5NwimURBkydV5cxEaknyf

“Protocol 和 ABC，其实代表两种完全不同的接口设计哲学。

在 Agent 框架里选型，核心就一句话：你是想管‘能做什么’，还是想管‘是什么’。”

---

#### 📋 先看直观对比

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧩 各有什么特点

🦆 Protocol

完全遵循鸭子类型：“走起来像鸭子，叫起来像鸭子，那它就是鸭子”。

- 不侵入用户代码，不用非得继承你的基类。
- 只要一个对象实现了 run(task) 方法，它就自动满足 AgentLike 协议。
- 非常适合作为框架对外暴露的接口，降低使用门槛。

```
代码块
from typing import Protocol
class AgentLike(Protocol):
    def run(self, task: str) -> str: ...

# 任何有这个方法的对象都是 AgentLike，无需继承
```

🧱 ABC

需要明确声明“我是这个类型”，适合有公共逻辑或需要isinstance 检查的场合。

- 可以定义抽象方法，强迫子类重写。
- 可以在基类里实现工具方法、钩子、模板方法。
- 运行时能做实例判断，框架内部路由更安全。

```
代码块
from abc import ABC, abstractmethod
class BaseAgent(ABC):
    @abstractmethod
    def run(self, task: str) -> str: ...
    def log(self, msg: str):  # 公共实现
        print(f"[LOG] {msg}")
```

#### 🎯 Agent 框架里怎么选？

我一般会这样拆解：

- 对外接口（插件、自定义工具） → 优先用 Protocol
  理由：让开发者自由，不用被迫继承你的类。耦合度低，符合“面向能力编程”。

- 框架内部基类（有共享逻辑） → 用 ABC
  理由：集中管理通用逻辑，子类继承后少写重复代码，同时做类型检查也更安全。

- 两者混用也很常见：
  对外暴露 Protocol 约束，内部实现用 ABC，然后让 ABC 类显式实现那个 Protocol。
  这样既保持了灵活性，又保留了代码复用的好处。

---

📌 一句话总结

Protocol 管能力，ABC 管身份。Agent 框架想低耦合就用 Protocol，想强约束和多复用就用 ABC，通常“外 Protocol 内 ABC”是最舒服的组合。
