# 如何用描述符实现 Agent 配置类的参数校验？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/NJmiwyi41in7ySk5KwWcrMtUnUb

直接上干货。描述符的本质，是把属性的存取逻辑从类里抽出来，封装成可复用的校验组件。对 Agent 配置类这种“参数多、校验规则杂”的场景，用描述符比在 init 里写一堆 if 干净得多。

---

#### 🔧 描述符的工作流

描述符就是实现了 set（或 get）的对象。当你在类属性上赋值时，Python 会先看这个类属性有没有 set，有就交给它处理，否则走实例字典。

用一个最简图表示校验过程：

```
代码块
config.temperature = 2.5
       │
       ▼
┌───────────────┐
│ 描述符 __set__ │
│ - 检查类型     │
│ - 检查范围     │
│ - 合规则写入   │
│   实例字典     │
│ - 不合规抛错   │
└───────────────┘
```

#### 🧱 实现两个常用校验描述符

为了让例子看着像真实代码，我写一个 FloatRange 和一个 StringChoice，都能带默认值。

```
代码块
class FloatRange:
    def __init__(self, low, high, default=None):
        self.low = low
        self.high = high
        self.default = default
        self.name = None  # 在 __set_name__ 里自动拿到字段名
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        if instance is None:
            return self
        # 从实例字典取，没有则返回默认值
        return instance.__dict__.get(self.name, self.default)
    def __set__(self, instance, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f"{self.name} 必须是数字")
        if not (self.low <= value <= self.high):
            raise ValueError(f"{self.name} 必须在 [{self.low}, {self.high}] 之间")
        instance.__dict__[self.name] = value
class StringChoice:
    def __init__(self, choices, default=None):
        self.choices = set(choices)
        self.default = default
        self.name = None
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name, self.default)
    def __set__(self, instance, value):
        if value not in self.choices:
            raise ValueError(f"{self.name} 必须是 {self.choices} 之一")
        instance.__dict__[self.name] = value
```
