---
title: Python 字符串驻留机制的触发条件是什么？对 Agent 开发有什么实际影响？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/DONQwtVCcir2KZk2z3Xcrut8nCg
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 字符串驻留机制的触发条件是什么？对 Agent 开发有什么实际影响？.md
questions: 2
chars: 852
build_date: '2026-11-20'
images_removed: 0
---

# Python 字符串驻留机制的触发条件是什么？对 Agent 开发有什么实际影响？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 字符串驻留机制的触发条件是什么？对 Agent 开发有什么实际影响？.md

### 一句话回答

Python 字符串驻留的触发条件为：编译期确定的、长度不超过 20 个字符、且仅由字母、数字、下划线组成的字符串会自动驻留；运行时动态创建的字符串默认不驻留，需通过 sys.intern() 手动强制。在 Agent 开发中，驻留机制既能加速工具名/角色名的等值比较，也可能因无限制 intern 导致内存常驻，需结合场景合理使用。

### 一、触发条件详解

## 题1：1 自动驻留的四种情况

> 〔原文此处为飞书内嵌组件，内容未随导出〕

代码验证：

```
import sys

# 条件 1：长度 0 或 1
a = ""
b = ""
print(a is b)  # True

# 条件 2：符合标识符规则
a = "hello_world"
b = "hello_world"
print(a is b)  # True

# 条件 3：编译期折叠
a = "hell" + "o"   # 编译期变成 "hello"
b = "hello"
print(a is b)      # True

# 条件 4：长度 ≤ 20
a = "a" * 20
b = "a" * 20
print(a is b)      # True
a = "a" * 21
b = "a" * 21
print(a is b)      # False
```

## 题2：2 不会自动驻留的情况

> 〔原文此处为飞书内嵌组件，内容未随导出〕

代码验证：

```

# 含空格——不驻留
a = "hell o"
b = "hell o"
print(a is b)  # False

# 运行时拼接——不驻留
s1 = "hell"
s2 = "hello"
print(s1 + "o" is s2)  # False

# 长度 > 20——不驻留
a = "a" * 21
b = "a" * 21
print(a is b)  # False
```
