---
title: new 和 init 分别负责什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/UqaLw33nFixMD3kmakWcaJfxnPg
source_path: AI Agent+后端精选八股文/语言基础篇/Python/new 和 init 分别负责什么？.md
questions: 1
chars: 768
build_date: '2026-11-20'
images_removed: 1
---

# new 和 init 分别负责什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/new 和 init 分别负责什么？.md

new 和 init 的分工一句话就能讲清：

- new：负责造对象（分配空间、返回实例）。
- init：负责装修对象（给实例绑属性、设初值）。

### 创建流程

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

用图表示就是：

```
MyClass(1,2)
   │
   ▼
__new__(cls, 1, 2) → 创建并返回 self (一片刚申请的内存)
   │
   ▼
__init__(self, 1, 2) → 给 self 挂上各种属性，准备就绪
   │
   ▼
 返回 self 给你
```

### 关键区别

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 题1：什么时候必须动 new

1. 单例模式 —— 在 new 里判断类是否已有实例，有就直接返回，不另创建。
1. 继承不可变类型（int, str, tuple）—— 因为它们的值在创建时就得确定，init 根本改不了。只能在 new 里传值。
1. 元类里定制类的创建 —— 元类的 new 是在控制类对象如何出生，和这里的逻辑同理，只是层级更高。

### 一个单例的极简例子

```
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    def __init__(self):
        print("init called")  # 每次 Singleton() 都会执行，但对象还是同一个
```
