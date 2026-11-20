# Agent 消息对象用了 slots 之后，发现子类继承它时内存优化失效，怎么排查和修复？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/NGg2wUt5bidntOkzsSoctR7Mnad

🔍 出现这个现象，十有八九是子类默默地生成了 dict，把父类辛苦省下的内存又吃回去了。

---

#### 🧬 根本原因：继承链上丢了 slots

看个典型错误：

```
代码块
class BaseMessage:
    __slots__ = ('role', 'content')
class ChatMessage(BaseMessage):
    def __init__(self, role, content, timestamp):
        self.role = role
        self.content = content
        self.timestamp = timestamp  # ❌ 这里会报错，因为没有槽位
```

发现报错，于是有人这样“修”：

```
代码块
class ChatMessage(BaseMessage):
    pass   # 没定义 __slots__
msg = ChatMessage()
msg.role = "user"
msg.content = "hello"
msg.timestamp = 123   # 竟然能跑！但 __dict__ 已经悄悄回来了
```

当子类没有定义 slots，Python 会自动为它创建 dict，即使父类有 slots 也白搭。父类的槽位仍然有效，但子类多出来的属性全跑进 dict，优化瞬间失效。

---

#### 🩺 如何排查

```
代码块

# 1. 直接看实例有没有 __dict__
msg = ChatMessage()
print(hasattr(msg, '__dict__'))   # True → 内存泄漏实锤

# 2. 对比内存占用
import sys
base = BaseMessage()
base.role = "u"; base.content = "c"
print(sys.getsizeof(base))   # 比如 56
child = ChatMessage()
child.role = "u"; child.content = "c"; child.timestamp = 123
print(sys.getsizeof(child))  # 可能会大很多，且包含字典开销

# 3. 检查类的 __slots__ 是否被继承
print(ChatMessage.__slots__)  # 如果只打印父类的，没包含新属性，甚至没 __slots__ 属性
```

如果看到 dict 存在，且 slots 里没有声明新增属性，问题就定位到了。

---

#### 🛠️ 修复方案

##### ✅ 正确做法：子类显式定义 slots，只写新增属性

```
代码块
class ChatMessage(BaseMessage):
    __slots__ = ('timestamp',)   # 只写新增的，不要重复 'role', 'content'
    def __init__(self, role, content, timestamp):
        self.role = role
        self.content = content
        self.timestamp = timestamp
```
