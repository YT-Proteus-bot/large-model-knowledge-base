---
title: Python 描述符协议是什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/SAEJwJfqzihp5ykMUSXcNCEOnmz
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 描述符协议是什么？.md
questions: 1
chars: 1369
build_date: '2026-11-20'
images_removed: 0
---

# Python 描述符协议是什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 描述符协议是什么？.md

描述符协议，面试里属于那种“你不会时觉得它很神秘，会了才发现它其实藏在 Python 底层每个角落”的知识点。我一般不从定义讲起，而是先抛一个你天天用但没意识到的例子。

### 先来感受它的存在

```
class MyClass:
    def __init__(self, x):
        self.x = x
obj = MyClass(10)
print(obj.x)   # 你以为只是读属性，背后可能有描述符在干活
```

如果 MyClass 的类属性 x 是一个描述符对象，obj.x 就不再是简单的 dict 查找，而是触发了一段代码。这就是描述符协议：它定义了对象作为类属性时，被访问、修改、删除的行为。

### 协议的核心：三个魔法方法

任何一个类，只要实现了以下任意一个方法，它就是描述符：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

优先级规则：描述符分为两种，取决于它实现了哪些方法——

- 数据描述符：同时有 get 和 set（或 delete） → 访问优先级最高，会覆盖实例字典。
- 非数据描述符：只有 get → 实例字典的优先级高于它。

这个优先级机制是理解 property、classmethod、staticmethod 这些内置设施的关键。

### 最简单的例子：自己写一个类型检查描述符

```
class TypedField:
    def __init__(self, name, expected_type):
        self.name = name          # 属性名
        self.expected_type = expected_type
    def __get__(self, instance, owner):
        if instance is None:      # 通过类访问，返回描述符自身
            return self
        return instance.__dict__.get(self.name)
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"{self.name} 期望 {self.expected_type}")
        instance.__dict__[self.name] = value
class Person:
    age = TypedField("age", int)
    name = TypedField("name", str)
p = Person()
p.age = 30       # 正常
p.name = "Alice" # 正常
p.age = "30"     # ❌ TypeError
```

这里的 age 和 name 是类属性，但每次对实例赋值，都通过 set 被拦截并校验。描述符让属性访问变成了方法调用，这就是它强大的地方。

## 题1：为什么说它无处不在？
