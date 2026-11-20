# 什么是单例模式，Python 里最简单的单例实现是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Qa3YwmyYTiUXpkkfDnPcN50in0b

#### 🎯 问题聚焦

什么是单例模式？Python 里最简单的单例实现是什么？

这个问题考察两点：

1. 能否准确说出单例模式的本质和适用场景；
1. 能否结合 Python 语言特性给出最简洁、最 Pythonic 的实现。

---

#### 📚 单例模式的定义

单例模式保证一个类在全局范围内仅有一个实例，并提供一个全局访问点。

用一句话概括：无论在哪里创建该类的对象，拿到的永远是同一个实例。

核心特征：

- ✅ 唯一性：内存中只存在一个对象
- ✅ 全局可访问：通过统一入口获取
- ✅ 延迟初始化（通常）：用到时才创建

---

#### 🧩 为什么需要单例？实际场景

在实际项目中，常会遇到需要统一控制资源或状态的情况，例如：

- 🌐 配置管理器 – 整个应用只需一份配置，避免多处读取不一致
- 📊 日志记录器 – 所有模块向同一个 Logger 写日志，防止文件句柄冲突
- 🔌 数据库连接池 – 连接池必须是全局唯一的，否则资源会相互争抢
- 🧠 Agent 的大脑/记忆 – 多工具共享同一份上下文状态

简单说：凡是“只应有一份”的对象，都适合用单例。

---

#### 🐍 Python 里最简单的实现：模块单例

在 Python 中，模块本身就是天然的单例。

因为 Python 的 import 机制会保证：一个模块无论被 import 多少次，其代码只会执行一次，全局变量只初始化一次。

这就意味着，你完全不需要类继承、装饰器或元类，只要把一个普通对象定义在模块顶层，它就是一个线程安全的、开箱即用的单例。

---

#### 💻 直观代码演示

📁 文件 config_manager.py

```
代码块

# 模块顶层创建一个实例——它就是全局唯一的配置对象
class _Config:
    def __init__(self):
        self.settings = {
            "model": "gpt-4",
            "temperature": 0.7,
            "max_tokens": 2048
        }
    def update(self, key, value):
        self.settings[key] = value

# 在模块级别直接实例化，外界导入的就是这个对象
config = _Config()
```

📁 其它文件使用

```
代码块

# agent.py
from config_manager import config
config.update("model", "claude-4")
print(config.settings)   # 修改会影响到所有引用

# tool.py
from config_manager import config

# 这里拿到的 config 和 agent.py 里的是同一个实例
print(config.settings["model"])   # 输出: claude-4
```
