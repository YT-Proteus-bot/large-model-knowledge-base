# @dataclass 自动生成了哪些方法？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/StVswc0nBipkkTke6czcsb2xndg

“这个好记——

🎁 @dataclass 的核心是帮你把‘样板方法’自动补全，让你少写 90% 的重复代码。

默认情况下，主要生成这 四个半 方法。我用一个类直接跑一遍，效果最直观。”

---

#### 📦 默认生成的方法

```
代码块
from dataclasses import dataclass
@dataclass
class Agent:
    name: str
    model: str = "gpt-4"
```

你不用写一行方法，它已经等价于你手写了下面这些：

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 👀 快速验证

```
代码块
a1 = Agent("Bob")
a2 = Agent("Bob")
print(a1)                # Agent(name='Bob', model='gpt-4')  <- __repr__
print(a1 == a2)          # True  <- __eq__
print({a1: "value"})     # 若没 frozen 会报错，因为 __hash__ 默认是 None
```

如果你加了 frozen=True，hash 就自动生成了，实例可哈希，能放进 set/dict 键。

---

#### 🧠 我面试时一般这样总结

> （此处为内嵌电子表格，需登录飞书查看）

最后一句话：

📌 “@dataclass 不是魔法，它就是帮你把 Python 里最啰嗦的 init 和 repr 给省了；记住默认有 init/repr/eq，frozen 触发 hash，order 触发排序，就够应付面试了。”
