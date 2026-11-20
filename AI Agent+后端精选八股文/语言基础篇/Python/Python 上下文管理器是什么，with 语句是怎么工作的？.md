# Python 上下文管理器是什么，with 语句是怎么工作的？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/L100wR238iOBqzkOiCLcdcZEnLf

🗝️ 一句话本质
上下文管理器就是一个实现了 enter 和 exit 的协议对象，负责资源的获取与释放，让你告别 try-finally 的繁琐。

⚙️ with 是怎么工作的？
步骤拆解，一次看透：

```
代码块
with 管理器对象 as 变量:
    # 业务代码
```

1. 进入 with → 调用 管理器.enter()，返回值赋给 as 后面的变量
1. 执行代码块 → 运行业务逻辑
1. 无论是否异常，离开 with → 调用 管理器.exit(异常类型,异常值,回溯)
  - __exit__ 返回 True → 吞掉异常，继续执行
  - 返回 False 或 None → 异常正常抛出

---

#### 🧪 经典示例：文件操作

```
代码块
with open("file.txt") as f:
    data = f.read()

# 自动关闭文件，哪怕中间报错也不怕
```

背后等价于手写：

```
代码块
f = open("file.txt")
try:
    data = f.read()
finally:
    f.close()
```

with 把 获取和释放 写成一体，清晰又安全。

---

#### 🧠 面试加分项

- 自定义上下文管理器 → 实现 enter / exit，比如数据库连接、锁的自动获取释放。
- contextlib.contextmanager 装饰器 → 把生成器函数变上下文管理器，yield 前是 enter，yield 后是 exit，优雅简洁。
- 异步上下文管理器 → aenter / aexit，配合 async with。
- 处理异常的能力 → 你可以在 exit 里分析异常并决定是否压制，这是 with 隐藏的高级玩法。

面试官心里话：能画清楚 enter → 业务 → exit 的流程，再随口抛出一个自定义实现或 contextlib，这题稳过。 ✅
