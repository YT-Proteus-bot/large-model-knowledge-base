# Agent 调用外部工具时，如何区分"应该重试"和"不应该重试"的错误？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/QhdxwSzW6isySdkzoqHc0i5tnWb

当 Agent 调用外部工具（搜索、计算、发送请求等）出错时，有些错误重试一下就能成功，有些重试反而雪上加霜。如何自动区分“该重试”和“不该重试”的错误？

这个问题本质上是：如何把异常从一堆 Exception 变成有语义的决策信号。

---

#### 🧠 核心思路：让异常“说话”

重试策略不该靠“猜”，而应让每个异常明确回答两个问题：

- 🔄 瞬态还是持久？ 网络闪断是瞬态的，权限拒绝是持久的。
- ♻️ 重复执行安全吗？ 即接口是否幂等（例如查询天然可重试，创建订单重复提交就可能扣款两次）。

因此，我们需要在错误抛出时就打上标签，重试框架根据标签自动决策。

---

#### 🏷️ 错误三分法：可重试 / 不可重试 / 需谨慎

> （此处为内嵌电子表格，需登录飞书查看）

有了这个分类，我们把错误包装成不同异常类，重试器只看类型即可。

---

#### ⚙️ 落地实现：异常分层 + 重试装饰器

##### 1️⃣ 设计异常树

```
代码块
class ToolError(Exception):
    """所有工具异常的基类"""
class RetryableError(ToolError):
    """标记：此错误可重试"""
    pass
class NonRetryableError(ToolError):
    """标记：此错误不可重试"""
    pass

# 具体例子
class ExternalServiceTimeout(RetryableError):
    pass
class InvalidParameterError(NonRetryableError):
    pass
```

##### 2️⃣ 工具调用时抛出对应异常

```
代码块
def call_search_api(query):
    resp = requests.get("...", timeout=5)
    if resp.status_code == 503:
        raise ExternalServiceTimeout("服务暂时不可用")
    if resp.status_code == 400:
        raise InvalidParameterError(f"参数错误: {query}")
    return resp.json()
```

##### 3️⃣ 智能重试装饰器（识别异常类型）

```
代码块
import time
from functools import wraps
def tool_retry(max_retries=3, backoff=1.5):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, max_retries+1):
                try:
                    return func(*args, **kwargs)
                except RetryableError as e:
                    last_exc = e
                    print(f"🔄 可重试错误，第{attempt}次重试: {e}")
                    time.sleep(backoff * (attempt-1))  # 线性退避
                except NonRetryableError as e:
                    raise  # 不可重试，直接抛出
            raise last_exc  # 重试耗尽
        return wrapper
    return decorator
```
