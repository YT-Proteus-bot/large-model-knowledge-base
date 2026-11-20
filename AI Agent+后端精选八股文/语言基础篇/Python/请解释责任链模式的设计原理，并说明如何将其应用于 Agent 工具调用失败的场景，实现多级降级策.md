# 请解释责任链模式的设计原理，并说明如何将其应用于 Agent 工具调用失败的场景，实现多级降级策略？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Zz60wKy6zifCSikfGU4cEqN3nlg

“责任链模式，核心就是一句：

‘把请求沿着一条链往后传，直到有人处理它，否则掉出链尾。’

在 Agent 调用工具失败时，用这个模式做多级降级特别自然——就像给每个操作准备了多道保险。”

---

#### 🔗 模式原理，三句话讲清楚

1. 一条链：多个处理器对象串成链表。
1. 每个处理器要么处理，要么往后传：处理完就终止；处理不了就调用下一个。
1. 调用者不关心谁最终处理：它只跟链头打交道。

> （此处为内嵌电子表格，需登录飞书查看）

典型优点：单一职责、开闭原则（随时增减处理器），缺点：链太长会影响性能，且可能没人处理。

---

#### 🧰 应用到 Agent 工具调用失败：多级降级链

假设 Agent 要执行一个“查询天气”的工具，我们设计三层降级：

![image](../../../../assets/HebydVWAro0h9zxDE9Wcg7yHnag/BRHkb0j6toNqqbxjfvlcIGB6nzd.png)

每个降级都是一个处理器，它们链在一起：

```
代码块
from abc import ABC, abstractmethod
class ToolHandler(ABC):
    def __init__(self):
        self._next = None  # 后继节点
    def set_next(self, handler: "ToolHandler") -> "ToolHandler":
        self._next = handler
        return handler  # 方便链式调用
    @abstractmethod
    def handle(self, query: str) -> str:
        # 默认实现：自己处理不了就转交下一个
        if self._next:
            return self._next.handle(query)
        return "⚠️ 所有策略均失败"
```
