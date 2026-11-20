---
title: '@dataclass 自动生成了哪些方法？'
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/StVswc0nBipkkTke6czcsb2xndg
source_path: AI Agent+后端精选八股文/语言基础篇/Python/@dataclass 自动生成了哪些方法？.md
questions: 0
chars: 705
build_date: '2026-11-20'
images_removed: 0
---

# @dataclass 自动生成了哪些方法？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/@dataclass 自动生成了哪些方法？.md

“这个好记——

🎁 @dataclass 的核心是帮你把‘样板方法’自动补全，让你少写 90% 的重复代码。

默认情况下，主要生成这 四个半 方法。我用一个类直接跑一遍，效果最直观。”

## 默认生成的方法

```
from dataclasses import dataclass
@dataclass
class Agent:
    name: str
    model: str = "gpt-4"
```

你不用写一行方法，它已经等价于你手写了下面这些：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 快速验证

```
a1 = Agent("Bob")
a2 = Agent("Bob")
print(a1)                # Agent(name='Bob', model='gpt-4')  <- __repr__
print(a1 == a2)          # True  <- __eq__
print({a1: "value"})     # 若没 frozen 会报错，因为 __hash__ 默认是 None
```

如果你加了 frozen=True，hash 就自动生成了，实例可哈希，能放进 set/dict 键。

## 我面试时一般这样总结

> 〔原文此处为飞书内嵌组件，内容未随导出〕

最后一句话：

📌 “@dataclass 不是魔法，它就是帮你把 Python 里最啰嗦的 init 和 repr 给省了；记住默认有 init/repr/eq，frozen 触发 hash，order 触发排序，就够应付面试了。”
