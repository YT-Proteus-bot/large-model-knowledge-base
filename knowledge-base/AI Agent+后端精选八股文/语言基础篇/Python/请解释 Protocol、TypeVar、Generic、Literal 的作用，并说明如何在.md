---
title: 请解释 Protocol、TypeVar、Generic、Literal 的作用，并说明如何在
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/YU6VwVY4diBtKvkd6FVc2HhVndc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/请解释 Protocol、TypeVar、Generic、Literal 的作用，并说明如何在.md
questions: 1
chars: 1696
build_date: '2026-11-20'
images_removed: 0
---

# 请解释 Protocol、TypeVar、Generic、Literal 的作用，并说明如何在

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/请解释 Protocol、TypeVar、Generic、Literal 的作用，并说明如何在.md

### 用 Protocol、TypeVar、Generic、Literal 构建类型安全的 Agent 框架

### 一句话理解它们的分工

- Protocol：定义“你能做什么”，不靠继承，靠结构。
- TypeVar：类型变量，保住泛型函数和泛型类里的“关系”。
- Generic：让类可以参数化，比如 Agent[MyTool]。
- Literal：把参数的取值精确到字面量，像枚举但更轻。

在 Agent 框架里，这四个组合起来，能让你在不运行代码时就靠静态检查发现“工具签名不对”“Agent 能力不匹配”等问题。

### 逐一拆解

### Protocol —— 结构化的“能力契约”

Protocol 就是给鸭子类型上户口。任何对象，只要它具备约定的方法签名，不用继承，就被类型系统认可为这个接口。

```
from typing import Protocol
class Tool(Protocol):
    name: str
    def __call__(self, input: str) -> str: ...
```

上面这个 Tool，任何有 name 属性和 call 方法的对象都自动符合，静态检查器会认。

### TypeVar —— 保持类型关联的“变量”

写泛型函数时，我们希望输入是什么类型，输出就是什么类型，TypeVar 就是那个占位符。

```
from typing import TypeVar
T = TypeVar("T")
def identity(x: T) -> T:
    return x
```

用它，类型关系就“绑”住了。

### Generic —— 参数化类的模具

Generic 配合 TypeVar，让类可以带上类型参数，比如一个 Agent 能携带它处理的“状态类型”。

```
from typing import Generic, TypeVar
S = TypeVar("S")
class Agent(Generic[S]):
    state: S
    def get_state(self) -> S:
        return self.state
```

### Literal —— 把值变成类型的一部分

当你希望参数只能是指定的几个字符串时，Literal 就是最精准的选择。

```
from typing import Literal
def set_mode(mode: Literal["fast", "precise"]) -> None: ...
```

set_mode("fast") 合法，set_mode("slow") 直接静态报错。

### 实战：设计一个类型安全的 AI Agent 框架

假设我们要做一个可扩展的 Agent 框架，核心有两个部分：工具注册 和 Agent 接口约束。我们直接用上面四个组件来搭。

## 题1：工具注册：强类型工具协议

先用 Protocol 定义一个工具的样子，并用 TypeVar 让工具能处理不同的输入输出，保留精确签名。

```
from typing import Protocol, TypeVar, Generic, Literal, Any

# 泛型工具：支持任意输入/输出，但保持一对一的类型关系
T_in = TypeVar("T_in", contravariant=True)
T_out = TypeVar("T_out", covariant=True)
class Tool(Protocol[T_in, T_out]):
    """任何符合此结构的可调用对象都是工具"""
    name: str
    category: Literal["search", "code", "api", "custom"]
    def __call__(self, input: T_in) -> T_out: ...
```
