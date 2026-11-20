# 请解释 Python 描述符的 get、set、delete 协议，并说明如何用描述符实现 Agent 配置的类型校验和环境变量注入？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/LIn1w7ChdicIDJkWzy4ch354n30

“描述符协议，其实就是 Python 给类属性安的 ‘访问拦截器’。
你只要在一个类里定义了 get / set / delete，这个类的实例再被挂到别的类上，就成了描述符，能完全接管那个属性的读写删。”

---

#### 🔧 协议怎么工作

> （此处为内嵌电子表格，需登录飞书查看）

只要对象实现了这三个中的任意一个，它就是一个描述符。

优先级比实例字典高，所以能做到“看起来是普通属性，背后全是逻辑”。

---

#### 🧪 应用1：类型校验

我写过最爽的一个描述符，就是用来给 Agent 的配置做运行时类型守护：

```
代码块
class Typed:
    """给 Agent 配置字段加类型锁"""
    def __init__(self, expected_type):
        self.expected_type = expected_type
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        if instance is None: return self
        return instance.__dict__.get(self.name)
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"💥 {self.name} 应为 {self.expected_type.__name__}，"
                f"实际收到 {type(value).__name__}"
            )
        instance.__dict__[self.name] = value
```

这样定义配置类：

```
代码块
class AgentConfig:
    temperature = Typed(float)
    max_tokens  = Typed(int)
config = AgentConfig()
config.temperature = 0.7        # ✅
config.temperature = "hot"      # ❌ TypeError
```

所有校验逻辑完全集中在描述符内部，配置类自己不需要写任何 setter。

---

#### 🌍 应用2：环境变量注入

同一个描述符机制，还能让 Agent 配置自动感知环境变量——这在 12-factor 应用里特别实用：

```
代码块
import os
class EnvVar:
    """自动从环境变量取值，支持默认值"""
    def __init__(self, env_key, default=None, cast=str):
        self.env_key = env_key
        self.default = default
        self.cast = cast
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        if instance is None: return self
        # 优先从环境变量读取，找不到就用默认值
        raw = os.environ.get(self.env_key)
        if raw is not None:
            return self.cast(raw)
        return self.default
    def __set__(self, instance, value):
        # 允许手动覆盖
        instance.__dict__[self.name] = value
```
