# 请解释 Python 元类的工作原理，并实现一个基于元类的 Tool 自动注册系统，同时与 init_subclass 方案对比。

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/BX9xwAT3ei9befksFqDcL4rgnuc

我从元类的工作机理讲起，画个创建流程图，然后直接落地到一个 Tool 自动注册的实现，最后把元类和 init_subclass 两种方案摆在一起比一下。

---

#### ⚙️ 元类在类创建时到底做了什么

Python 里 class 语句的执行顺序是这样的：

1. 解释器收集类体里所有赋值语句、函数定义，塞进一个临时的命名空间字典 namespace。
1. 根据继承链，找到最优先的元类（如果有 metaclass=... 就用它，否则用基类的元类，默认是 type）。
1. 调用这个元类的 new 和 init，把类名、基类元组、命名空间传进去。
1. 返回新创建的类对象，绑定到类名上。

用图追一下 Tool 的创建过程：

![image](../../../../assets/XzjXdyln3ocOELxFEMVcqXqanwh/IMeEbFbGAo0zgxxzYGTc24Y7nUg.png)

核心思想：元类控制了从“代码文本”到“类对象”这一步转化。任何你想在类定义瞬间自动干的事（检查、修改、注册），都可以挂在这个钩子上。

---

#### 🔧 实战：用元类做 Tool 自动注册系统

假设咱们在写一个 AI Agent 框架，开发者只要写一个类继承 BaseTool，就自动注册到工具库，不用手动调用任何 register 函数。

先看元类的实现：

```
代码块

# tool_registry.py
class ToolRegistryMeta(type):
    registry = {}  # 全局工具库
    def __new__(mcs, name, bases, namespace):
        # 跳过基类 BaseTool 本身，不注册
        cls = super().__new__(mcs, name, bases, namespace)
        if name != 'BaseTool':
            # 约定：必须有 name 属性和 execute 方法
            if 'execute' not in namespace or not callable(namespace['execute']):
                raise TypeError(f"Tool {name} must implement 'execute' method")
            tool_name = namespace.get('name', name.lower())
            mcs.registry[tool_name] = cls
        return cls

# 基类用这个元类
class BaseTool(metaclass=ToolRegistryMeta):
    def execute(self, **kwargs):
        raise NotImplementedError

# 使用：开发者只管写子类，自动注册
class SearchTool(BaseTool):
    name = "search"
    description = "Search the web"
    def execute(self, query, **kwargs):
        return f"Searching for {query}..."
class CalculatorTool(BaseTool):
    name = "calc"
    def execute(self, expression, **kwargs):
        return eval(expression)

# 查看自动注册结果
print(ToolRegistryMeta.registry)

# {'search': <class 'SearchTool'>, 'calc': <class 'CalculatorTool'>}
```
