# Python 多继承的 MRO C3 算法原理是什么，super() 在多继承中如何工作？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/JKlcwE1zHiseW0k69WlcyOKqnRh

多继承不是乱炖，Python 靠 C3 线性化 定规矩。下面用最精要的方式拆解，保证你听完就能讲给同事听。

---

### 🧬 MRO 是什么？

Method Resolution Order —— 方法解析顺序。
当你在 class D(B, C) 里调 self.method()，Python 必须知道先去 B 还是 C 找。
MRO 就是那个排好序的类列表，让每次查找都是确定的、无歧义的。

---

### 🔢 C3 算法原理（一句话版）

MRO = 自己 + 合并所有父类的 MRO + 父类本身

合并规则（merge）是精髓：

1. 取第一个列表的头（第一个元素）
1. 如果这个头没有出现在任何其他列表的尾部，就把它拿出来，并从所有列表中删除这个头
1. 否则，看下一个列表的头
1. 循环直到所有列表为空
1. 如果找不到合法头 → 💥 抛出 TypeError，拒绝创建不一致的继承

公式感：
L[C] = C + merge(L[B1], L[B2], ..., B1, B2, ...)

经典钻石：
D(B, C) 且 B(A), C(A)

- L[A] = A, object
- L[B] = B, A, object
- L[C] = C, A, object
- L[D] = D + merge(B,A,object , C,A,object , B,C)
  → 头 B 不在其他尾部 → 取出 B
  → 下一头 C 不在其他尾部 → 取出 C
  → 然后 A、object
  最终 MRO：D → B → C → A → object ✅

---

### 🚀 super() 的协同舞蹈

super() 不是直接调父类，而是沿着 MRO 链，找当前类后面的下一个类。
这造就了多重继承下“协作”模式：

```
代码块
class A:
    def action(self):
        print("A")
class B(A):
    def action(self):
        super().action()
        print("B")
class C(A):
    def action(self):
        super().action()
        print("C")
class D(B, C):
    def action(self):
        super().action()
        print("D")
```

D().action() 输出顺序（MRO: D→B→C→A）：

```
代码块
A
C
B
D
```

🔁 调用链是一根线穿到底：D 的 super 调 B，B 的 super 调 C，C 的 super 调 A，A 结束回溯。

这就是“协同”的意义：每个类只负责自己的事，把接力棒交给 MRO 链的下一个，完全不依赖具体父类是谁。

---

### ⚠️ 三个致命坑
