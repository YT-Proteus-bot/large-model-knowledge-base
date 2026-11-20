---
title: try/except/else/finally 四个子句分别在什么时候执行？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/TJ6MwnB8JiIms0kRyntcx2ytnTA
source_path: AI Agent+后端精选八股文/语言基础篇/Python/try_except_else_finally 四个子句分别在什么时候执行？.md
questions: 0
chars: 1349
build_date: '2026-11-20'
images_removed: 0
---

# try/except/else/finally 四个子句分别在什么时候执行？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/try_except_else_finally 四个子句分别在什么时候执行？.md

```

# 先看一个覆盖全部子句的代码结构
try:
    result = 1 / 0          # ① 尝试执行
except ZeroDivisionError:
    print("除零了")         # ② 异常匹配时
else:
    print("没异常:", result)# ③ 只有在没有异常时
finally:
    print("总会执行")       # ④ 无论如何都会执行
```

四个子句的触发条件，一张时间线图就看得清清楚楚：

```
进入 try 块
    │
    ├─ 无异常 ──→ else 块（③）
    │               │
    │               └─→ finally 块（④）
    │
    ├─ 有异常且被 except 捕获 ──→ except 块（②）
    │                               │
    │                               └─→ finally 块（④）
    │
    └─ 有异常但未被捕获 ──→ finally 块（④）
                                │
                                └─→ 异常继续向上抛出
```

## try – 监控区

里面的代码一行一行跑，Python 解释器在这里挂上异常钩子。只要不出异常，就跑完进入 else；出了异常立刻跳到 except 去匹配。

## except – 异常匹配时才进入

只有在 try 中抛出异常，且异常类型与 except 后面声明的匹配时，才会执行。如果不匹配，这个 except 就被跳过，异常继续往上抛。

- 多个 except 从上到下按顺序匹配，抓到即止。
- except Exception 不会捕获 SystemExit / KeyboardInterrupt 这类“系统级退出”。

## else – 无异常时专属

如果 try 块干干净净跑完，没有任何异常，就进 else。

📍 很多人把正常逻辑扔在 try 的末尾，这会让 try 块职责不清：你不知道哪行抛异常是你要处理的。把“只有成功才该跑的代码”放到 else 里，try 块只放可能出错的那几行，意图特别清晰。

## finally – 无论如何都要执行

不管前面是正常、异常被捕获、还是异常没被捕获直接往外抛，finally 一定在离开 try 前执行。

它甚至会在 return、break、continue 之前抢跑一步：

```
def demo():
    try:
        return 1
    finally:
        print("finally 先跑")
print(demo())

# 输出：
# finally 先跑
# 1
```

这也就引出那句经典铁律：finally 里的代码，连 return 都拦不住它。

## 各子句触发条件速查

> 〔原文此处为飞书内嵌组件，内容未随导出〕
