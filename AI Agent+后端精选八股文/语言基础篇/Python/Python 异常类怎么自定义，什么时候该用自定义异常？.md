# Python 异常类怎么自定义，什么时候该用自定义异常？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/PhzYwS3aviFTtZkU0r9cbF8Dncb

Python 中如何自定义异常类？什么时候该用自定义异常而不用内置异常？

这题考察两点：

- 写一个自定义异常类有多简单（语法）
- 能说出什么时候必须“造轮子”（设计意识）

---

#### 🧱 自定义异常：语法上极其简单

Python 里自定义异常，只需要继承 Exception（或它的子类）。
最简形式一行代码搞定：

```
代码块
class OrderNotFoundError(Exception):
    """订单不存在时抛出的异常"""
    pass
```

然后就可以像内置异常一样使用：

```
代码块
def get_order(order_id):
    order = db.find(order_id)
    if not order:
        raise OrderNotFoundError(f"订单 {order_id} 不存在")
```

✅ 这就完成了一个自定义异常，它和 ValueError、RuntimeError 是同一个家族的。

---

#### 🧩 如果只是这样，为什么要自定义？

因为名字就是语义。看两段代码对比：

❌ 只用内置异常

```
代码块
def cancel_order(order_id):
    order = get_order(order_id)
    if order.status == "shipped":
        raise RuntimeError("已发货的订单不能取消")
    ...
```

调用方只能捕获宽泛的 RuntimeError，很容易误伤其它运行时错误，也没法区分到底是“订单不存在”还是“订单状态不允许”。

✅ 使用自定义异常

```
代码块
class OrderCancelNotAllowedError(Exception):
    pass
def cancel_order(order_id):
    order = get_order(order_id)
    if order.status == "shipped":
        raise OrderCancelNotAllowedError(
            f"订单 {order_id} 已发货，无法取消"
        )
```

调用方可以精确捕获：

```
代码块
try:
    cancel_order("X123")
except OrderNotFoundError:
    print("订单号错了")
except OrderCancelNotAllowedError:
    print("这个订单不能取消，提示用户")
```

异常类名就是一份可读的文档，也把错误类型变成了一等公民。

---

#### 🎨 自定义异常还能做什么？——携带更多上下文

自定义异常可以带任意属性，方便上层处理或记录日志：

```
代码块
class PaymentFailedError(Exception):
    def __init__(self, order_id: str, amount: float, reason: str):
        super().__init__(
            f"订单 {order_id} 支付 {amount} 元失败: {reason}"
        )
        self.order_id = order_id
        self.amount = amount
        self.reason = reason

# 使用
raise PaymentFailedError("X123", 99.9, "余额不足")
```
