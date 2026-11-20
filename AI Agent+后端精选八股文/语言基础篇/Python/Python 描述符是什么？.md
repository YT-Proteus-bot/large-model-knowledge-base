# Python 描述符是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Gnndwo4I1iEop0k4hU9cb1kYnZd

🧠 描述符，一句话：一个能“挟持”另一个类属性访问的对象。它让你能把 obj.attr 这个最简单的操作，变成一段自定义逻辑。

---

#### 🎭 背后协议：get / set / delete

任何定义了 get 方法的类，它的实例就可以作为另一个类的类属性。当访问这个属性时，Python 会调用 get；赋值时调用 set；删除时调用 delete。

![image](../../../../assets/Iii1d8lgwovS1Px5gDbcJ8uYnPc/QUsDbrzMoo19jxxLqhuciijPnrh.png)

- 非数据描述符：只定义了 get。实例字典里的同名属性会遮盖它。
- 数据描述符：同时定义了 get 和 set（或 delete）。优先级最高，实例字典里的同名属性都打不过它。

---

#### 🛠️ 一个你每天在用却可能没意识到的例子

```
代码块
class Person:
    def __init__(self, name):
        self._name = name
    @property
    def name(self):
        return self._name
    @name.setter
    def name(self, value):
        if not isinstance(value, str):
            raise TypeError("名字必须是字符串")
        self._name = value
```

这里的 @property 实际上创建了一个描述符对象 name。p.name 会调用描述符的 get，p.name = 'Tom' 会调用 set。整个 property 机制就是建立在描述符协议上的。

---

#### 🧩 手写一个描述符：类型检查器

```
代码块
class TypedField:
    def __init__(self, field_type):
        self.field_type = field_type
        self.data = {}          # 用一个字典存不同实例的值
    def __get__(self, obj, objtype=None):
        if obj is None:         # 类访问返回描述符本身
            return self
        return self.data.get(id(obj))
    def __set__(self, obj, value):
        if not isinstance(value, self.field_type):
            raise TypeError(f"需要 {self.field_type}, 收到 {type(value)}")
        self.data[id(obj)] = value
    def __delete__(self, obj):
        del self.data[id(obj)]
class Person:
    name = TypedField(str)
    age = TypedField(int)
```
