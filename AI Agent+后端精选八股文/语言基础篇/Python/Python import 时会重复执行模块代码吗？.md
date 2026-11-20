# Python import 时会重复执行模块代码吗？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/GOdJw5pZqiwR6pkV7MEcL3z9nzo

#### 为什么不会重复执行？

Python 导入模块时会经历三个步骤：

1. 查找：在 sys.path 里找 .py 或 .so 文件
1. 加载：编译成字节码
1. 执行：运行模块的顶层代码块（定义函数、类、变量等）

但关键在于——执行完的结果会被缓存到一个全局字典里：

```
代码块
sys.modules['module_name'] = module_obj
```

之后任何地方再 import 这个模块，Python 都会先查 sys.modules，发现已经有了，就直接返回缓存对象，完全跳过执行步骤。

---

#### 📦 一个直观验证

```
代码块

# foo.py
print("foo 被执行了！")
x = 42
```

```
代码块

# main.py
import foo          # 输出：foo 被执行了！
import foo          # 没有任何输出
print(foo.x)        # 42
```

即使在不同文件里 import，或者在同一个文件 import 多次，效果都一样——只有第一次会看到打印。

---

#### 🔄 例外：什么情况下会重新执行？

1. 显式使用 importlib.reload

```
代码块
import importlib
importlib.reload(foo)   # 强制重新执行模块顶层代码
```

这常用于开发热更新，但日常业务代码几乎不用。

1. 不同解释器进程，或不同 Python 实例
   每个进程有自己的 sys.modules，所以每次重启程序当然会重新执行。

1. 用奇怪的方式绕过缓存
   比如直接操作 sys.modules 删掉缓存项再 import，但这是作死操作，面试里不用提。

---

#### 🧠 理解“顶层代码只执行一次”的意义

- 定义语句是执行的一部分：函数 def 和类 class 也是运行时才创建的，但只会创建一次。
- 单例模式天然适配：模块里的全局变量就是模块唯一的实例。
- 避免副作用：不要在模块顶层做高开销、有副作用的事（如打开数据库连接），因为它们真的只会做一次，且会在第一次 import 时发生。

---

#### 📌 总结图

```
代码块
首次 import a → 查 sys.modules 没有 → 执行 a.py → 存入缓存
再次 import a → 查 sys.modules 有    → 直接返回缓存，不执行代码
```

👉 所以，Python 的 import 就是典型的“懒执行 + 强缓存”模式，保证模块代码不重复执行，同时让程序启动更快、内存更省。

这个机制也是 Python 项目组织代码时很可靠的一个基础特性。
