---
title: getattr 什么时候会被调用？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/ORlNwnZLCi8DAzkpBaCcYIrOnne
source_path: AI Agent+后端精选八股文/语言基础篇/Python/getattr 什么时候会被调用？.md
questions: 0
chars: 996
build_date: '2026-11-20'
images_removed: 0
---

# getattr 什么时候会被调用？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/getattr 什么时候会被调用？.md

关于 getattr 什么时候被调用，这里要区分两个东西：

一个是 Python 的内置函数 getattr()，另一个是对象特殊方法 getattr。

面试通常问的是后者，它其实是属性查找的“兜底机制”。

## 核心结论

getattr(self, name) 只在正常属性查找失败时才被调用。

正常查找走 getattribute，它包揽所有属性访问。只有当它抛出 AttributeError 时，Python 才会去调用 getattr（如果定义了）。

## 触发时机（都有哪些情况）

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 直观例子

```
class User:
    def __init__(self):
        self.name = "张三"
    def __getattr__(self, item):
        print(f"属性 {item} 不存在，进入 __getattr__ 后备")
        return "默认值"
u = User()
print(u.name)      # 正常找到 → 输出"张三"，不触发 __getattr__
print(u.age)       # 不存在 → 触发 __getattr__，输出"默认值"
print(getattr(u, "age"))        # 不带默认值 → 同上，触发
print(getattr(u, "age", 18))   # 带默认值 → 输出18，不触发
```

## 关键注意点

- getattr 只处理缺失属性，已有的属性直接跳过它。
- 如果在 getattr 里写了 self.xxx，而 xxx 也不存在，会递归调用自己，导致无限循环。应该直接操作实例字典或调用 super()。
- 想要拦截所有属性（包括存在的），得重写 getattribute，但这是进阶操作，容易踩坑。

## 一句话总结

getattr 就像属性世界的“备用钥匙”——只有主钥匙（正常查找）打不开门时，它才会被拿出来试一下。

内置 getattr 函数也只是这扇门的一种打开方式，底层同样尊重这个逻辑。

这样解释，把调用时机和特殊方法的角色串起来了吗？如果需要，我们还可以继续聊聊和 getattribute 的区别。
