# dataclass 和 Pydantic BaseModel 的核心区别是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ZWn1wjf3yiLuq4kXmqhcxvHonLd

🔍 它俩虽然都能“省掉写 init”，但核心区别在于运行时对数据的控制力，一个完全信任你，一个会严格验票。

```
代码块
from dataclasses import dataclass
from pydantic import BaseModel

# 1️⃣ dataclass：纯粹的 Python 数据容器
@dataclass
class UserDC:
    name: str
    age: int

# 2️⃣ Pydantic：带“安检”的强类型模型
class UserPD(BaseModel):
    name: str
    age: int
```

#### ⚡ 核心分歧

> （此处为内嵌电子表格，需登录飞书查看）

---

#### 🧪 同样不靠谱的输入，结局完全不同

```
代码块

# dataclass：照单全收，类型只是个装饰
u1 = UserDC(name="Alice", age="not-a-number")
print(u1.age)          # "not-a-number"，完全不管

# Pydantic：当场翻脸
try:
    u2 = UserPD(name="Bob", age="not-a-number")
except Exception as e:
    print(e)
    # 1 validation error: age: value is not a valid integer
```

#### 🧩 一个简单的实战对照

```
代码块

# Pydantic 对“脏数据”的转换能力
class Product(BaseModel):
    price: float
p = Product(price="19.99")
print(p.price)          # 19.99 (float)

# dataclass 得自己动手
@dataclass
class ProductDC:
    price: float
p2 = ProductDC(price="19.99")
print(p2.price)         # "19.99" (str) – 埋雷
```

#### 💡 所以怎么选？

```
代码块
需要严格校验 / 来自外部输入 / 涉及序列化反序列化 → 上 Pydantic
        │
        ▼
需要速度 / 内部大量临时对象 / 完全信任数据源 → 用 dataclass（甚至可以 sloted）
```
