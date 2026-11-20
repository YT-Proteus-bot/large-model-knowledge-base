---
title: Python 普通实例为什么会有额外的内存开销？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/ZPE8wXojbi71oUkE9l0cXqrNnIg
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 普通实例为什么会有额外的内存开销？.md
questions: 1
chars: 1126
build_date: '2026-11-20'
images_removed: 0
---

# Python 普通实例为什么会有额外的内存开销？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 普通实例为什么会有额外的内存开销？.md

🧠 Python 的普通实例，背后背着一个你看不见的“万能属性背包”——dict。就是它把内存撑大了。

### 每个普通实例都自带一个字典

当你写这样一个类：

```
class Dog:
    def __init__(self, name):
        self.name = name
```

每个 Dog 实例内部，都会创建这样一个映射：

```
实例对象
  └─ __dict__ = {'name': 'Buddy'}   # 独立的 dict 对象
```

这个 dict 是一个完整的 Python 字典，即使里面只有一个键值对，它也需要维护：

- 哈希表结构（至少 8 个槽位，取决于实现）
- 键对象的指针、值对象的指针
- 版本标记、负载计数等元信息

用 sys.getsizeof 测一下就一目了然：

```
import sys
d = Dog('Buddy')
print(sys.getsizeof(d))           # 56 字节（实例本身）
print(sys.getsizeof(d.__dict__))  # 104 字节（字典，只有一个键！）
```

🗂️ 区区一个属性，字典开销居然是实例本身的近两倍。

### 对比：用 slots 关掉“背包”

如果预先知道属性有哪些，可以告诉 Python：“别给我背包了，给我几个固定格子就好”。

```
class DogSlim:
    __slots__ = ('name',)
    def __init__(self, name):
        self.name = name
```

这时实例内部没有 dict，属性直接存储在实例内存的固定槽位里：

```
实例对象
  └─ [name] 直接存值，无字典
```

内存表现：

```
d2 = DogSlim('Buddy')
print(sys.getsizeof(d2))  # 48 字节，且不会随属性增多而膨胀字典
```

> 〔原文此处为飞书内嵌组件，内容未随导出〕

当你有几十万个对象时（比如游戏粒子、数据流记录），不带 slots 的内存浪费会急剧放大。

## 题1：为什么字典那么“重”？

Python 字典是为动态查找、任意键、高适应性设计的通用容器，不是为你每个实例存两三个属性优化的。就算只有一个键，它也要：

1. 分配一个哈希表数组（最小一般是 8 个槽）
1. 每个槽存哈希值、键指针、值指针
1. 维护负载因子、版本计数

这就像 用集装箱搬家只运了一本书——车（实例）不大，集装箱（dict）却死沉。
