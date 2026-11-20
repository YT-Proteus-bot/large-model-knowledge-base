---
title: 在高并发 AI Agent 系统中同时运行数千个 Agent 实例时，slots 的工作原理是什么？怎么正确使用它优化内存？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/AlJ9wdETTijiUgkP6Rwc2uQ8nLb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在高并发 AI Agent 系统中同时运行数千个 Agent 实例时，slots 的工作原理是什.md
questions: 1
chars: 1717
build_date: '2026-11-20'
images_removed: 1
---

# 在高并发 AI Agent 系统中同时运行数千个 Agent 实例时，slots 的工作原理是什么？怎么正确使用它优化内存？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在高并发 AI Agent 系统中同时运行数千个 Agent 实例时，slots 的工作原理是什.md

🪶 先直说原理：slots 就是剥夺了实例的 dict，换用固定槽位存储属性，从而实现瘦身和加速。 在数千 Agent 实例同时运行的情况下，这个“剥夺”带来的内存节省是几何级的。

### 工作原理：从字典查找变成“数组偏移”

普通实例每个都有个 dict，它是一个完整的哈希表，哪怕只存一个属性，也要：

- 分配至少 8 个空槽的哈希表
- 键、值指针、哈希值占满内存
- 动态扩缩容带来的额外碎片

而 slots 在类创建时就预先在实例内存布局中分配好固定槽位，每个属性对应一个槽，访问时直接通过偏移量读写，不再需要字典的哈希查找。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

每个实例直接省掉了一个字典对象（至少 56 字节+哈希表 104 字节+），当你跑 5000 个 Agent 时，能轻松省下 5000 × 160B ≈ 800KB，这还没算 GC 扫描字典的开销。

### 正确使用姿势（重点在“克制”）

在高并发 Agent 场景里，我不会给所有类都加 slots，而是针对最常实例化、属性固定的类下手，比如：

```
class ToolResult:
    __slots__ = ('tool_name', 'output', 'error', 'latency_ms')
    def __init__(self, tool_name, output, error=None, latency_ms=0):
        self.tool_name = tool_name
        self.output = output
        self.error = error
        self.latency_ms = latency_ms
```

### 优化规则（从踩坑里总结的）

1. 属性要固定

   如果你需要动态给 Agent 实例挂载临时数据（比如缓存、中间状态），slots 会让你直接报 AttributeError。此时考虑用 dict + 弱引用，或者改用专门的缓存容器（如 WeakValueDictionary）。

1. 继承要小心
- 父类有 __slots__，子类如果不定义，会自动生成 __dict__，白费力气。
- 正确做法是子类也定义 __slots__，内容可以只写新增的属性（不要重复父类已定义的），Python 会合并。

```
class Base:
    __slots__ = ('id',)
class Child(Base):
    __slots__ = ('extra',)   # 不用重复 'id'
```

别忘了 weakref

如果你需要弱引用（例如放入 WeakSet 追踪实例），必须在 slots 里显式加上 'weakref'。

```
__slots__ = ('name', '__weakref__')
```

配合 dataclasses（Python 3.10+）

现在写 Agent 的配置、消息体这些纯数据类，我直接用 @dataclass(slots=True)，省心又标准：

```
from dataclasses import dataclass
@dataclass(slots=True)
class AgentConfig:
    model: str = "gpt-4o"
    temperature: float = 0.7
    max_tokens: int = 2048
```

## 题1：什么时候不该用 slots

- 需要动态添加属性（比如混入多个 Mixin 特性不确定时）
- 类属性很少，实例总数也不多（省那点内存不值牺牲灵活性）
- 要与一些依赖 dict 的框架兼容（比如部分 ORM、序列化库）

对于海量 Agent 实例，内存优化后的直观感受：同样的机器，原来跑 2000 个 Agent 就 OOM，加 slots 后稳跑 8000 个。这就是去掉背包之后，真正能扛并发的底气。
