---
title: 三：神经网络模块 (torch.nn)
topic: PyTorch 面试八股
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/HAxGwBLA5iBs8qkyVHtcDa23nug
source_path: PyTorch 面试八股/三：神经网络模块 (torch.nn).md
questions: 5
chars: 2262
build_date: '2026-11-20'
images_removed: 0
---

# 三：神经网络模块 (torch.nn)

> 主题：PyTorch 面试八股　｜　原始文件：PyTorch 面试八股/三：神经网络模块 (torch.nn).md

## 题1：如何定义一个包含两个隐藏层的MLP？

定义一个经典的多层感知器需要继承 nn.Module，并在 init 中定义网络层，在 forward 中定义数据流。

```
import torch.nn as nn
class TwoLayerMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim1, hidden_dim2, output_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim1)
        self.fc2 = nn.Linear(hidden_dim1, hidden_dim2)
        self.fc3 = nn.Linear(hidden_dim2, output_dim)
        self.activation = nn.GELU()  # 常用的激活函数
        self.dropout = nn.Dropout(0.2)
    def forward(self, x):
        x = self.activation(self.fc1(x))
        x = self.dropout(x)
        x = self.activation(self.fc2(x))
        x = self.fc3(x)  # 最后一层不加激活（回归）
        return x
```

设计要点：

- 所有带参数的操作（nn.Linear, nn.BatchNorm）必须在 init 中实例化并赋给 self。
- 无参数的激活、Dropout 也推荐在 init 中定义，以保持代码整洁和模块可复用。
- 最后一层通常不加激活，除非是分类任务（需配合 CrossEntropyLoss 的 logits 输入）。

## 题2：nn.Module 的 init 和 forward 分别负责什么？

- init：定义模型的静态结构。在这里声明所有子模块（层）、参数和缓冲区。它负责回答“模型由哪些组件构成”。PyTorch 通过它来追踪所有需要梯度的参数和需要保存的状态。
- forward：定义模型的动态计算逻辑。它接收输入数据，按顺序调用各个子模块，执行前向传播，并返回输出。forward 的调用是隐式的，通过 model(input) 触发，绝不能直接调用 model.forward(x)，因为前者会触发框架的钩子（hooks），而后者不会。

核心原则：init 构建“骨架”，forward 注入“灵魂”。

## 题3：为什么推荐在 init 中定义所有层，而不是在 forward 中直接创建？

这是一个工程最佳实践，而非语法限制。原因如下：

1. 参数追踪与优化：只有注册在 nn.Module 上的子模块，其参数才会被自动追踪到。如果在 forward 中临时创建 nn.Linear，其参数将不会被 model.parameters() 发现，从而无法被优化器更新，也无法正确保存和加载模型。
1. 设备一致性：通过 model.to(device) 可以自动将所有注册的子模块参数和缓冲区移动到目标设备。临时创建的层不会被移动，导致计算设备错误。
1. 可复现性与调试：init 中的定义构成了模型的完整静态图，这使得 print(model) 可以展示完整结构，便于调试和版本管理。如果层是动态创建的，模型结构将不可预测。
1. 性能：动态创建层会导致不必要的内存分配和初始化开销，尤其在循环中极慢。

结论：模型的结构必须是确定的、可追踪的。init 就是这份确定性契约。

## 题4：将子模块添加到 Module 中有哪几种方式？各有什么特点？

有三种主要方式：

- self.child = nn.Linear(...)：最常见，直接赋值。子模块会自动注册，参数被追踪。
- self.add_module('child', nn.Linear(...))：功能等价于直接赋值，但模块名是字符串。常用于循环中动态添加模块，或处理模块名为变量的情况。
- nn.Sequential(*layers)：容器模块，将多个子模块按顺序封装。内部的每个模块会自动注册。它本身也是一个 nn.Module，可被赋值给 self。

特点对比：

- 直接赋值和 add_module 适用于非顺序或需要灵活控制的网络结构。
- Sequential 适合线性堆叠的网络块，代码最简洁，但缺少灵活性（无法在同一层有多个输入/输出或分支）。

## 题5：nn.Parameter 和普通张量的核心区别是什么？

核心区别：nn.Parameter 是一个 Tensor，但会被自动添加到 nn.Module 的参数列表中。

- 注册与优化：当将一个张量赋值给 nn.Module 的一个属性时，若该张量是 nn.Parameter，它会被模块自动追踪，并通过 model.parameters() 暴露给优化器。普通张量则被忽略。
- 设备与类型转换：nn.Parameter 会随 model.to(device) 或 model.float() 自动转换设备和数据类型；普通张量不会。
