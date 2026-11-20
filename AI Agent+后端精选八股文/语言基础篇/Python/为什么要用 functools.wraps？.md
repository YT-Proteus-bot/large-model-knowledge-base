# 为什么要用 functools.wraps？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ZZQSwAoViiy0FdkdfLfc6uPinSh

“functools.wraps 解决的其实是一个身份丢失的问题。装饰器本质上是用一个新函数替换原函数，如果不加 wraps，原函数的名字、文档字符串、参数签名这些元信息就全丢了。这对调试、日志、API 文档生成都是灾难性的。

我举个具体的例子说清楚。

一、不加 wraps 时发生了什么？

假设我写了一个最普通的装饰器，统计函数执行时间：

```
代码块
import time
def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"耗时: {time.time() - start:.2f}秒")
        return result
    return wrapper
@timer
def get_user(user_id: int) -> str:
    """根据 ID 获取用户名"""
    return f"用户_{user_id}"
```

这个装饰器看起来完全没问题。但如果我运行下面的代码：

```
代码块
print(get_user.__name__)   # 输出: wrapper
print(get_user.__doc__)    # 输出: None
help(get_user)             # 看不到任何有用的签名信息
```

原函数 get_user 的名字变成了 wrapper，文档字符串没了，参数签名也丢了。在 Flask 或 FastAPI 这种依赖函数签名自动生成路由规则的框架里，这会直接导致路由注册失败或参数校验失效。

二、wraps 怎么解决？

加 functools.wraps(func) 就是一句话的事：

```
代码块
import functools
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"耗时: {time.time() - start:.2f}秒")
        return result
    return wrapper
```

现在再查：

```
代码块
print(get_user.__name__)   # 输出: get_user
print(get_user.__doc__)    # 输出: 根据 ID 获取用户名
```

wraps 做的事情就是：把 func 的 name、doc、module、dict 等元信息，全部复制到 wrapper 上。同时，wraps 还会在 wrapper 上挂一个 wrapped 属性，指向原始函数。这样你随时可以通过 get_user.wrapped 拿到最底层的函数，这在单元测试或者装饰器的多层嵌套排查时非常有用。

三、核心价值：对工具链友好

很多 Python 工具依赖函数的元信息工作：

- IDE 智能提示：PyCharm 看到 doc 才能弹出文档说明。
- Sphinx / pydoc 文档生成：读 name 和 doc 来生成 API 文档。
- Flask / FastAPI 路由：用函数名作为路由端点，丢了就冲突或者 500。
- mock.patch：patch 一个被装饰的函数时，需要拿到正确的函数路径，wrapped 让它能穿透装饰器直接 mock 原函数。

四、wraps 的等价实现

其实 functools.wraps 本身没有魔法，它内部就是调用了 functools.update_wrapper，而我也可以手动实现一个简化版：

```
代码块
def my_wraps(original_func):
    def decorator(wrapper_func):
        wrapper_func.__name__ = original_func.__name__
        wrapper_func.__doc__ = original_func.__doc__
        wrapper_func.__wrapped__ = original_func
        return wrapper_func
    return decorator
```
