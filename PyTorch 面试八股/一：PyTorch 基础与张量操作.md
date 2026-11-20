# 一：PyTorch 基础与张量操作

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Vf94wFO62ij7vwkGFVbcOxpdnRe

#### 1. 如何创建一个形状为(3,4)的全零张量，并指定数据类型为float32？

创建全零张量最直接的方式是使用 torch.zeros 工厂函数，它允许我们通过 dtype 参数显式地指定张量的数据类型。

```
代码块
import torch

# 创建一个形状为 (3, 4) 的张量，并用0填充，指定类型为32位浮点数
x = torch.zeros(3, 4, dtype=torch.float32)
print(x.shape)  # torch.Size([3, 4])
print(x.dtype)  # torch.float32
```

深入理解与最佳实践：

- torch.zeros 申请的内存并不会被立刻物理分配和归零，而是采用了“延迟初始化”或“写时复制”的机制，仅在第一次被访问时才会真正分配。这使得创建大尺寸张量非常高效。
- 与 torch.zeros 类似的还有 torch.ones 和 torch.empty。torch.empty 不会对内存进行任何初始化，速度快但包含“脏数据”，使用时需要立即覆盖所有值，否则可能导致难以调试的数值问题。
- 在深度学习微调实践中，强烈建议始终显式指定 dtype。例如，在混合精度训练（AMP）时，明确指定 torch.float16 或 torch.bfloat16 可以避免因框架默认推断导致的类型不匹配错误。

---

#### 2. torch.tensor 与 torch.Tensor 在使用上有何区别？

这是一个非常经典的“陷阱”问题。torch.tensor 是一个函数，而 torch.Tensor 是一个类。这个区别看似微小，但在实际编码中至关重要。

- torch.tensor(data)（函数）
  - 数据拷贝：它会为传入的data创建一个全新的副本，并推断其数据类型。
  - 灵活性：你可以通过dtype参数强制指定返回的张量类型，实现int到float的转换。
  - 推荐用法：这是目前PyTorch官方推荐的构造方式，因为它的行为是确定的、可预测的。
- torch.Tensor(data)（类构造器）
  - 数据共享：如果传入的data是一个Python列表，行为与torch.tensor类似。但如果传入一个NumPy数组，它会尽可能地共享内存，而不是拷贝。这意味修改原始NumPy数组会影响到张量，反之亦然。
  - 类型锁定：当传入Python列表时，torch.Tensor会强制将其转换为全局默认的张量类型（torch.float32），无论原始列表是什么类型。它不接受dtype参数，无法在构造时改变类型。这个行为非常隐蔽，很容易导致输入整数却得到浮点数的bug。

```
代码块
import numpy as np

# 场景1：传入 Python 列表
data_list = [1, 2, 3]
a = torch.tensor(data_list)   # 推断为 torch.int64
b = torch.Tensor(data_list)   # 强制变为 torch.float32 ！这是bug之源

# 场景2：传入 NumPy 数组
data_np = np.array([1.0, 2.0, 3.0], dtype=np.float64)
c = torch.tensor(data_np)     # 拷贝并转换为 float32 (默认) 或保留 float64
d = torch.Tensor(data_np)     # 共享内存，张量类型为 torch.float64
data_np[0] = 99.0
print(d[0])  # 输出 99.0，张量随之改变
```

结论：除非你明确需要torch.Tensor的内存共享特性或创建未初始化张量，否则在所有场景下都应使用 torch.tensor 或 torch.as_tensor。

---

#### 3. 怎样获取一个张量的形状、维度数和元素总数？

这三个属性是理解张量形态的基础。

```
代码块
x = torch.randn(3, 4, 5)

# 形状 (Shape): 返回一个 torch.Size 对象，本质是一个元组
print(x.shape)   # torch.Size([3, 4, 5])
print(x.size())  # 等价于 x.shape

# 维度数 (Number of dimensions)，也称为阶 (rank)
print(x.ndim)    # 3

# 元素总数 (Total number of elements)
print(x.numel()) # 3 * 4 * 5 = 60
```
