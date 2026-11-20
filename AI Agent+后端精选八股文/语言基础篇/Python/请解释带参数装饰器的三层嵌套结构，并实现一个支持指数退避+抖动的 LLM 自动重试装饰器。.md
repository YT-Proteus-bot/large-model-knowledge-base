# 请解释带参数装饰器的三层嵌套结构，并实现一个支持指数退避+抖动的 LLM 自动重试装饰器。

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Q1RowTZtgi1gUok6Wrrc5dM8ncg

你问的三层嵌套，其实拆开来看，就是“配置→装饰→包装”这三次传递。我用一个实战场景来讲：给调用大模型接口的函数做自动重试，并且要指数退避+随机抖动。

---

#### 🧱 为什么必须是三层

- 第一层（最外层）：接收你传的配置，比如 max_retries=3, base_delay=1, max_delay=60，这叫“参数工厂”。它返回真正的装饰器。
- 第二层（中间层）：标准的 decorator(func)，接收被装饰的函数。
- 第三层（最内层）：wrapper(*args, **kwargs)，实际执行函数、加重试逻辑的地方。

如果只有两层，你只能拿到 func，没办法在定义时传入可配置的参数。三层本质上是 工厂函数(配置) → 装饰器 → 新函数 这条链。

---

#### 🧪 指数退避 + 抖动的重试装饰器实现

下面是可运行的代码，我嵌入了图标来标出每层职责。

```
代码块
import time
import random
import functools
from collections.abc import Callable
from typing import Type, Tuple

# 🏭 第一层：参数工厂，接收重试策略配置
def llm_retry(
    max_retries: int = 3,
    base_delay: float = 1.0,      # 秒
    max_delay: float = 60.0,      # 秒
    jitter: bool = True,
    retry_on: Tuple[Type[BaseException], ...] = (Exception,)
):
    """
    返回一个装饰器，用于自动重试可能失败的 LLM 调用。
    """
    # 🎯 第二层：真正的装饰器
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        # ⚙️ 第三层：包装函数，加入重试逻辑
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except retry_on as e:
                    last_exception = e
                    if attempt == max_retries:
                        raise  # 重试耗尽，抛出最后一次异常
                    # 📈 指数退避：base_delay * 2^attempt
                    sleep_time = base_delay * (2 ** attempt)
                    # 🎲 可选抖动：在 0 到 sleep_time 之间取随机值
                    if jitter:
                        sleep_time = random.uniform(0, sleep_time)
                    # 🛑 封顶，防止无限等待
                    sleep_time = min(sleep_time, max_delay)
                    print(f"⏳ 第{attempt+1}次重试，等待 {sleep_time:.2f}s "
                          f"（异常: {type(e).__name__}）")
                    time.sleep(sleep_time)
            # 理论上不会执行到这里
            raise last_exception
        return wrapper
    return decorator
```
