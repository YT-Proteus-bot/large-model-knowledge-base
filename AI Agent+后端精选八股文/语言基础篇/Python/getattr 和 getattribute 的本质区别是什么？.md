# getattr 和 getattribute 的本质区别是什么？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/AcGyw7Npnieo5FkRxueck75MnYf

这两个名字长得像，本质差别还挺大的。
先把关系说清楚：getattribute 是属性访问的“正门”，所有访问都必须经过它；getattr 是“后门”，只有当正门找不到属性时才会打开。

下面我拆开来说👇

---

#### 🔑 本质区别一句话

- getattribute：无条件拦截。访问任何属性（不管存在与否）都会先调用它。
- getattr：有条件兜底。只有在常规查找（包括 getattribute）都失败、抛出 AttributeError 时才会被调用。

---

#### ⚡ 调用顺序：一个清晰的流程图

![image](../../../../assets/EnlHdoeMZok0tnx5CLDcORQZnPh/U42FbNb9eoZ1fGxvoMycx0VKn7c.png)

#### 🧪 示例对比

```
代码块
class Demo:
    def __init__(self):
        self.exist = "我是存在的属性"
    def __getattribute__(self, name):
        print(f"__getattribute__ 被调用：{name}")
        return super().__getattribute__(name)   # 必须走父类，否则递归
    def __getattr__(self, name):
        print(f"__getattr__ 被调用：{name}")
        return f"动态生成的 {name}"
obj = Demo()
print(obj.exist)   # 输出 __getattribute__ 被调用：exist → 返回 "我是存在的属性"
print(obj.miss)    # 输出 __getattribute__ 被调用：miss → 失败 → __getattr__ 被调用：miss → 返回 "动态生成的 miss"
```

可以看到：

- getattribute 无论属性存不存在，都会被调用。
- getattr 只在 miss 这种缺失属性时才出面。

---
