---
title: Python 字符串为什么不能原地修改？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/WfjywMqsJiHWC9k95uecoZw6nVc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 字符串为什么不能原地修改？.md
questions: 0
chars: 979
build_date: '2026-11-20'
images_removed: 0
---

# Python 字符串为什么不能原地修改？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 字符串为什么不能原地修改？.md

## 一句话回答

因为 Python 的 str 是不可变对象（immutable）：一旦创建，内容就不能被修改。任何看似“修改”的操作，实际上都是创建了一个新字符串对象，变量只是重新指向了新对象。

## 核心原因

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 代码示例

```
s = "hello"
print(id(s))          # 例如 140234...

# 下面这行会报错：TypeError: 'str' object does not support item assignment
# s[0] = 'H'
# 看似修改，实际创建新对象
s += " world"
print(s)              # hello world
print(id(s))          # id 变了，说明是新对象

# 验证不可变
original = "abc"
new = original.upper()
print(original)       # abc
print(new)            # ABC
print(id(original), id(new))  # 不同对象

# 高效拼接：用 join，而不是循环 +=
parts = ["a", "b", "c", "d"]
result = "".join(parts)
print(result)         # abcd

# 可变替代：bytearray
ba = bytearray(b"hello")
ba[0] = ord("H")
print(ba)             # bytearray(b'Hello')
```

## 底层视角

CPython 中 str 对应 PyUnicodeObject，大致包含：

```
typedef struct {
    PyObject_HEAD
    Py_ssize_t length;   // 长度
    Py_hash_t hash;      // 缓存的 hash
    // ... 字符数据
} PyUnicodeObject;
```

- 长度和 hash 在创建后固定。
- 字符数据区不可变。
- 因此无法原地修改某个字符。
