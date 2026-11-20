# Python 字符串驻留机制的触发条件是什么？对 Agent 开发有什么实际影响？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/DONQwtVCcir2KZk2z3Xcrut8nCg

### 一句话回答

Python 字符串驻留的触发条件为：编译期确定的、长度不超过 20 个字符、且仅由字母、数字、下划线组成的字符串会自动驻留；运行时动态创建的字符串默认不驻留，需通过 sys.intern() 手动强制。在 Agent 开发中，驻留机制既能加速工具名/角色名的等值比较，也可能因无限制 intern 导致内存常驻，需结合场景合理使用。

---

### 一、触发条件详解

#### 1.1 自动驻留的四种情况

> （此处为内嵌电子表格，需登录飞书查看）

代码验证：

```
代码块
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

#### 1.2 不会自动驻留的情况

> （此处为内嵌电子表格，需登录飞书查看）

代码验证：

```
代码块

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
