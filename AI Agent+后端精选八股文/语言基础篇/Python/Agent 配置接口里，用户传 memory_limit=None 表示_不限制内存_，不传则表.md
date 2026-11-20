# Agent 配置接口里，用户传 memory_limit=None 表示"不限制内存"，不传则表示"用平台默认值"，这两种语义怎么区分？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/ONykwoaOEiZgivk9g86cVB16nxe

很实际的 API 设计问题，很多 Agent 平台的配置都会踩这个坑。我按“问题本质 → 常规方案失效 → 哨兵对象解法 → 落地写法”来讲，中间画个决策图。

---

#### 🎯 先明确两种语义的差别

在 Agent 配置接口里，我们想表达三层意思：

- 用户完全不传这个参数：Agent 应该去取平台设定的默认内存限制（比如 2GB）。
- 用户传了具体数值：比如 memory_limit=512，那就按 512MB 限制。
- 用户显式传 None：意思是“不做任何内存限制”，Agent 可以自由增长。

如果用同一个 None 去表示“没传”和“无限制”，就完全没法区分了。比如：

```
代码块
def create_agent(memory_limit=None):
    if memory_limit is None:
        memory_limit = 2048  # 到底是“用默认”，还是“无限制”？
```

函数体里只能看到一个 None，但背后是两种完全不同的意图，必须分开。

---

#### 🧭 为什么不能用普通默认值

因为 None 在 Python 里是单例，你把它写在默认参数位置，解释器就把它当成“没传”的唯一信号。如果业务上又需要 None 本身作为有效值，那就撞车了。

用图表示就是两条路径汇到同一个点上，逻辑分叉不了：

```
代码块
用户调用 create_agent(memory_limit=None)
     │
     ▼
参数 memory_limit = None  ←──── 汇合点
     │
用户调用 create_agent()         │
     │                           │
     ▼                           │
默认值也取 None ─────────────────┘
     │
     ▼
函数体无法判断来源
```

#### 🛡️ 解法：用一个看不见的“哨兵对象”作默认值

我们会造一个全局唯一的对象，它的唯一使命就是当“参数没传”的信号。这个名字没有实际语义，只用来做身份判断（is），对用户完全透明。

```
代码块

# 定义哨兵
_UNSET = object()
def create_agent(memory_limit=_UNSET, model=None):
    if memory_limit is _UNSET:
        # 用户没传，取平台默认值
        memory_limit = load_platform_default_memory()
    elif memory_limit is None:
        # 用户显式传了 None，表示不限制
        memory_limit = float('inf')  # 或者内部特殊标记
    else:
        # 用户传了具体数值
        pass
    # 后续使用 memory_limi
```

再看决策路线就非常清楚了：

```
代码块
用户调用 create_agent()
  → memory_limit = _UNSET → 取平台默认值
用户调用 create_agent(memory_limit=None)
  → memory_limit = None → 识别为“不限制”
用户调用 create_agent(memory_limit=512)
  → memory_limit = 512 → 直接用
```
