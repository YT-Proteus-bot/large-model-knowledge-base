# Python 的 MRO 是什么，怎么查看一个类的 MRO？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/OL74wN3W2iBIFGkFjugcQHALnih

🗺️ 一句话定义

MRO = Method Resolution Order（方法解析顺序），决定了在多继承下，Python 查找属性和方法时的线性顺序。

🧩 为什么要有它？

菱形继承（钻石继承）时，必须约定一个无歧义的查找路径。Python 3 用的是 C3 线性化——结果保证：子类在前，父类在后，书写顺序优先。

---

🔎 怎么查看 MRO？ 两条路，效果一样：

```
代码块
类名.__mro__   # 返回元组，类本身的直接属性
类名.mro()     # 返回列表，更方便打印查看
```

直接上经典菱形继承：

```
代码块
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
print(D.__mro__)

# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

顺序解析：

- D 找自己
- 然后找 B（D 左边先写的父类）
- 再找 C（右边父类）
- 然后才是共同祖先 A，最后 object

---

🧠 能加分的理解

- super() 的协同工作全依赖 MRO，不是简单调父类，而是按 MRO 链向后找下一个。
- Mixin 模式能玩转，前提就是你吃透了 MRO 的线性顺序。
- 如果继承图违反了 C3 约束（比如顺序冲突），直接 TypeError，这体现了一致性优于兼容性。
