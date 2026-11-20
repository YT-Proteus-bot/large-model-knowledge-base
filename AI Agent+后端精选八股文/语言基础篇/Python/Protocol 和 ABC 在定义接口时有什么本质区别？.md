# Protocol 和 ABC 在定义接口时有什么本质区别？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/YdZiwfVk1iFzowkFURacpkvpnu1

### 🔷 Protocol 和 ABC 在定义接口时的本质区别？

#### 🎯 先给结论

ABC 看的是“你继承了我吗？”（名义子类型），Protocol 看的是“你能做这些事吗？”（结构子类型）。

换句话说，ABC 讲血缘，Protocol 讲能力。

---

#### 🔍 1. ABC——抽象基类：必须“签合同”

用 abc.ABC 和 @abstractmethod 定义接口，子类必须显式继承，否则运行时根本不认。

```
代码块
from abc import ABC, abstractmethod
class Flyable(ABC):
    @abstractmethod
    def fly(self) -> None:
        ...
class Bird(Flyable):       # 显式继承
    def fly(self) -> None:
        print("扑翼飞行")

# 运行时检查立刻生效
print(isinstance(Bird(), Flyable))  # True
```

📌 特点：

- 强侵入性，想用接口就得改类定义或显式 register
- 运行时 isinstance / issubclass 会严格检查继承链
- 典型的“框架基类”思维：你在我的体系里，就得按我的规则来

---

#### 🔷 2. Protocol——协议：看你会不会“做事”

typing.Protocol 定义了结构接口，任何类只要拥有了规定的方法签名，无需继承，就自动被视为该类型。这本质是静态鸭子类型。

```
代码块
from typing import Protocol
class Flyable(Protocol):
    def fly(self) -> None:
        ...
class Bird:                # 不继承任何东西
    def fly(self) -> None:
        print("扑翼飞行")
class Airplane:
    def fly(self) -> None:
        print("喷气飞行")
def take_off(f: Flyable) -> None:
    f.fly()
take_off(Bird())      # ✅ 静态检查通过
take_off(Airplane())  # ✅ 静态检查通过
```

⚠️ 注意：上面这种“类型兼容”默认只在静态检查器（mypy、pyright）里生效，运行时 isinstance(Bird(), Flyable) 会直接报错，除非用 @runtime_checkable 装饰器显式开启运行时支持。

#### 深入一点：为什么会有这种区别？

ABC 的本质是运行时强制契约。当我在一个框架里写出 isinstance(obj, MyBase) 时，我希望得到绝对确定的答案，所以必须依靠继承关系。

Protocol 的本质是静态类型系统里的鸭子类型。Python 一直崇尚“如果它走起来像鸭子，叫起来像鸭子，那它就是鸭子”。Protocol 就是把这种哲学搬进了类型标注里，让静态检查器也能懂“你只要有 fly() 方法，就是 Flyable”。
