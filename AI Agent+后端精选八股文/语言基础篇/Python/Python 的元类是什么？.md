# Python 的元类是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/OPMww7LlrililNkl2u2cJZEonmh

我从“类也是对象”这个底层事实讲起，画个关系图，然后落到具体用法上。

---

#### 🧩 类也是对象，那类的类就是元类

Python 里常说“一切皆对象”，这句话没说完：类本身也是对象。

```
代码块
class Dog:
    pass
print(type(Dog))  # <class 'type'>
```

Dog 这个类，它的类型是 type。既然 Dog 是对象，那它肯定也是由某个类实例化出来的，这个“创建类的类”，就是元类。

用图把这几层关系画清楚：

```
代码块
   实例化                 实例化
对象  <──────  类  <──────  元类
(实例)          (类对象)       (type / 自定义元类)
  │               │               │
  └─ 由类创建      └─ 由元类创建   └─ type 是所有类的默认元类
```

拿 Dog 举例：

- d = Dog() → d 是 Dog 的实例。
- Dog 类是 type 的实例。
- 所以 type 就是 Dog 的元类。

---

#### ⚙️ class 语句的背后：元类在偷偷干活

当我们写：

```
代码块
class Dog(Animal):
    leg = 4
```

解释器实际干的事大致等于：

```
代码块
Dog = type('Dog', (Animal,), {'leg': 4})
```

type 作为元类，它的 new 和 init 负责：

1. 收集类名、基类、类属性字典。
1. 创建真正的类对象。
1. 返回这个类对象，并绑定到变量 Dog 上。

所以元类就是拦截类创建过程的钩子。你可以在这里偷天换日：改类名、加属性、注册到某个中心、强制检查接口，啥都行。

---

#### 🔧 自定义元类：一个加前缀的例子

比如我想让某个包下面所有类的属性，自动加上 my_ 前缀，防御命名冲突。自定义元类这样做：

```
代码块
class PrefixMeta(type):
    def __new__(mcs, name, bases, namespace):
        # 把所有非内置、非私有属性加上前缀
        prefixed = {}
        for k, v in namespace.items():
            if not k.startswith('__'):
                prefixed['my_' + k] = v
            else:
                prefixed[k] = v
        return super().__new__(mcs, name, bases, prefixed)
class User(metaclass=PrefixMeta):
    name = 'alice'
    age = 30
print(User.my_name)  # 'alice'
print(User.my_age)   # 30
print(hasattr(User, 'name'))  # False
```
