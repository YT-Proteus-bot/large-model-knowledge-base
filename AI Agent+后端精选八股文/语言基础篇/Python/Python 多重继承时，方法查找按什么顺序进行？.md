# Python 多重继承时，方法查找按什么顺序进行？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/DYIPwf2uTi9UaZkct8GcD3ALnXg

多重继承的方法查找顺序，核心就是 MRO（Method Resolution Order，方法解析顺序），Python 用的算法叫 C3 线性化。我按“为什么需要它 → 它怎么算 → 怎么看它 → 实际怎么用”来讲，中间画个钻石继承的图。

---

#### 🧱 为什么需要 MRO？

单继承时，找方法很简单：从当前类开始，一路沿着父类链往上找。但多重继承可能出现“同一个方法在多个父类里都有”，而且这些父类之间还有可能共享一个祖先（钻石继承）。

比如经典的：

![image](../../../../assets/NqGNdtXymouOWmxkgomcRqQGnGe/WxuxbjVXJoucRHxxjISctDeUnub.png)

D 同时继承 B 和 C，B、C 又都继承 A。如果 B 和 C 都重写了 A 的方法，那 D 的实例调这个方法时，到底用哪个？这就是 MRO 要解决的问题。

---

#### 🧭 Python 的解法：C3 线性化

Python 3 所有类都是新式类，MRO 用 C3 线性化算法。它的目标是把所有父类按照“保留定义顺序、子类优先于父类、每个类只出现一次”的原则，排成一个从左到右的顺序链表。

用上面的钻石继承，代码这样写：

```
代码块
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
print(D.__mro__)

# (D, B, C, A, object)
```

顺序是 D → B → C → A → object。方法查找时，就是从 D 开始，顺着这个元组一路往后找，找到第一个匹配的就停下来。

---

#### 📐 用图画出查找路径

```
代码块
钻石继承结构：
        A
       / \
      B   C
       \ /
        D
对应的 MRO 线性序列：
   D ──▶ B ──▶ C ──▶ A ──▶ object
查找过程（假设调 d.method()）：
   1. 看 D 里有没有 method，没有就往下
   2. 看 B 里有没有，有就执行，停止
   3. 如果 B 没有，看 C
   4. 如果还没有，看 A，最后 object
```

这样做的好处是：从不跳过子类，且每个祖先只访问一次，避免了旧式类那种同一祖先被多次调用或深度优先造成的混乱。

---

#### 🧪 几个关键点

1. 顺序由基类书写顺序决定

class D(B, C) 和 class D(C, B) 的 MRO 完全不同。前者 D, B, C, A, object；后者 D, C, B, A, object。所以多个父类的顺序就是查找优先级。

1. 不可行的 MRO 会直接报错

如果写了一个 C3 无法线性化的结构（比如循环或冲突的优先顺序），Python 会在类创建时就抛出 TypeError：

```
代码块
class A: pass
class B(A): pass
class C(A, B): pass  # 报错！A 和 B 的顺序冲突
```
