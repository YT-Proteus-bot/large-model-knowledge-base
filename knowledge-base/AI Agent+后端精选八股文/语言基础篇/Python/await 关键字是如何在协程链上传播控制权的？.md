---
title: await 关键字是如何在协程链上传播控制权的？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/UwNswfeski90rgkrUPOcW7qensh
source_path: AI Agent+后端精选八股文/语言基础篇/Python/await 关键字是如何在协程链上传播控制权的？.md
questions: 0
chars: 1229
build_date: '2026-11-20'
images_removed: 0
---

# await 关键字是如何在协程链上传播控制权的？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/await 关键字是如何在协程链上传播控制权的？.md

await 在协程链上的控制权传播，本质上就是：

把执行权一层一层地“还”给事件循环，等条件就绪后再一层一层地“恢复”回来。

下面我画一条协程链，直观看一下这个过程👇

## 一条协程链的示意

```
async def task_a():
    result = await task_b()   # 点①
    return result
async def task_b():
    data = await task_c()     # 点②
    return data
async def task_c():
    resp = await fetch_data() # 点③，真正 IO 阻塞点
    return resp
```

## 控制权的传播逻辑

🔽 挂起阶段（从内向外“让路”）

1. fetch_data() 发起 IO，立即返回一个未完成的 Future。
1. task_c 执行到 await，看到这个 Future 未完成，它把自己挂起，并将这个 Future 原样返回给调用者 task_b。
1. task_b 在 await task_c() 处拿到的也是同一个未完成 Future，它同样挂起，再把这个 Future 返回给 task_a。
1. task_a 同理，最终这个未完成 Future 被一直传到事件循环手上。

💡 整个链上没有任何阻塞线程，只是把 Future 一层层抛回给事件循环。

🔼 恢复阶段（从外向内“接力”）

1. 事件循环监控这个 Future，当 IO 就绪，Future 变为已完成。
1. 事件循环根据这个 Future 找到最内层被挂起的 task_c（Python 内部通过 Future 的回调和调度器实现）。
1. task_c 从 await fetch_data() 处恢复执行，得到结果，然后它自己的 task_c() 返回，这会完成 task_b 里正在等的那个 Future。
1. task_b 接着被唤醒，再完成 task_a 等待的 Future……
1. 就这样沿着调用链逐层恢复，直到最外层拿到最终结果。

## 理解要点

- await 不是主动交出 CPU，而是把“决定谁该跑”的权力交给了事件循环。
- 控制权传播靠的是 Future 对象在协程链上传递：链上每一层都只跟自己的 await 和它产生的 Future 交互。
- 整个过程是协作式的：协程自己决定在 await 处让出，不存在抢占。

## 一句话总结

await 在协程链上传播控制权，就是沿着调用栈把未完成的 Future 一层层传回事件循环，再在 IO 就绪后一层层反向唤醒。

这就是为什么单线程也能跑成千上万的协程——每个协程只在真正需要等待时才让路，控制权通过 await 在链条上灵活流动。

这样讲，有没有把这个“接力”机制说清楚？
