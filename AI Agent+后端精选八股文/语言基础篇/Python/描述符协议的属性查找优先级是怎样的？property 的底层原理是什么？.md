# 描述符协议的属性查找优先级是怎样的？property 的底层原理是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/YnnTw7TwqihqhpkPPWzcP6cvnke

🔍 描述符的属性查找，其实就一条铁序：
数据描述符（有 set） > 实例 dict > 非数据描述符（只有 get）

---

#### 🗺️ 查找路线图

![image](../../../../assets/RXIsdB2lfowxiyxEBKackqOjnzh/H4acbPWptoUOljxOzivcco95nIg.png)

重点：只要在类里定义了 set（或 delete），这个描述符就成了“闸门”。哪怕你在实例字典里硬塞同名属性，访问时也完全不理你，直接走描述符。

---

#### 🧬 @property 的底层：一个 C 实现的“闸门”

```
代码块
class Dog:
    @property
    def name(self):
        return self._name
    @name.setter
    def name(self, val):
        self._name = val
```

这里的 name 本质上是一个 property 对象，它内置了 get 和 set，所以是数据描述符。

用纯 Python 可以描出它的骨架：

```
代码块
class MyProperty:
    def __init__(self, fget=None, fset=None):
        self.fget = fget
        self.fset = fset
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return self.fget(obj)
    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("read-only")
        self.fset(obj, value)
    def setter(self, fset):
        return type(self)(self.fget, fset)
```
