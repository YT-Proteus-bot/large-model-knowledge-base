---
title: Agent 服务上线后内存持续增长，没有明显的代码 bug，如何排查？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/SwY6wYMAiiUnEpkR9LQc3AEln0c
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Agent 服务上线后内存持续增长，没有明显的代码 bug，如何排查？.md
questions: 0
chars: 2634
build_date: '2026-11-20'
images_removed: 0
---

# Agent 服务上线后内存持续增长，没有明显的代码 bug，如何排查？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Agent 服务上线后内存持续增长，没有明显的代码 bug，如何排查？.md

这类问题很典型——上线没 bug，但内存就是悄无声息地涨，通常是一些看似无害的持有行为被长时间放大了。下面我按自己实际排查的顺序来聊，不堆工具，更像一次真实的“案情推理”。

## 第一步：先判断“是怎么涨的”

盯监控，看内存曲线。主要有两种形态：

- 📈 慢增长，像滑梯：随着请求数增加，内存稳步上升，但偶尔 GC 后会回落一点点，整体趋势向上。这通常是对象生命周期太长，比如请求级的数据被意外挂在全局或长生命周期对象上。
- 🪜 阶梯式，突然跳一下：到达某个条件后内存猛抬一阶，然后稳住，再抬。这往往和某个大对象或缓存策略的触发有关（比如向量库加载索引、模型预热）。

用 psutil 或 Prometheus 拉一下 RSS 曲线，确定是哪种。同时看下 gc 统计：gc.get_stats()，如果分代 2 的回收频率很低但内存却涨，说明对象进了老年代出不来。

## 第二步：锁定可疑的“内存持有者”

没有明显 bug，那大概率是无意识的持有。可以这样缩圈：

## 1️⃣ 用 tracemalloc 对比两个时间点的内存快照

这是 Python 内置的，生产环境建议在预发或者单独打日志。启动时开启，隔一段时间拍两个快照，比较 top 10 差异。

```
import tracemalloc
tracemalloc.start()

# ... 处理一些请求 ...
snapshot1 = tracemalloc.take_snapshot()

# ... 再处理一些请求 ...
snapshot2 = tracemalloc.take_snapshot()
stats = snapshot2.compare_to(snapshot1, 'lineno')
for stat in stats[:10]:
    print(stat)
```

你会直接看到，比如 langchain_core.messages.HumanMessage 对象多占了几十 MB，且都来自 agent/history.py 的某一行。这就锁定了历史消息没有被清理。

## 2️⃣ 用 objgraph 看哪个类型在疯狂繁殖

```
import objgraph
objgraph.show_most_common_types(limit=20)
```

经常发现 dict 或 list 或 function（闭包）数量随请求线性增长。进一步用 objgraph.find_backref_chain 可以追踪到根引用。很多次我查到是某个工具的返回结果被放进了全局的 LRU cache 但没有设置 maxsize，或者是 functools.lru_cache 装饰的函数参数里含有了 request 对象，导致每次请求都生成不同 key，无限堆积。

## 第三步：在 Agent 场景下排查五个常见“非 bug 型泄漏”

没有明显 bug，那往往就是设计上没做“遗忘”机制。我列几个几乎必查的点：

- 💬 对话历史无限增长：Agent 经常维护一个 state["messages"] 列表，如果没做窗口裁剪，每次新交互都会追加，内存永动。

  查法：tracemalloc 里看到 HumanMessage 和 AIMessage 占比持续上升。

  修法：用 trim_messages 或滑动窗口 + 摘要压缩。

- 🗂️ 工具返回结果全量留存：比如搜索工具返回了 Wikipedia 全文，Agent 把它原样存进状态里，后续节点只用了摘要。

  查法：memory_profiler 逐行看工具调用后的增量，会发现 tool_result 赋值时内存暴增。

  修法：只保存 result[:500] 或使用生成器惰性消费。

- 🧩 全局缓存无上限：很多 Agent 会对 LLM 响应或嵌入向量做缓存，如果 key 中包含了 timestamp 或者随机种子，缓存就会无限膨胀。

  查法：检查 functools.lru_cache(maxsize=...) 或 Redis 里的 keys 数量。

  修法：设死 maxsize 或使用 TTL 淘汰。

- 📦 第三方库内部缓存：向量数据库客户端、HTTP 连接池、gRPC channel，默认可能缓存连接和元数据，长期运行不释放。

  查法：直接在代码里找到 client 对象的内部缓存属性（比如 _caches），打印大小。

  修法：定期 close() 再重建，或配置 max_idle_time。

- 🧵 异步 Task 泄漏：如果 Agent 有后台任务（比如主动拉取），asyncio.create_task 完没有 await 或 cancel()，引用的上下文（包括 ContextVar、大对象）无法回收。

  查法：asyncio.all_tasks() 看是否有大量 pending 的任务。

  修法：用 TaskGroup 或确保 finally 中 cancel。

## 第四步：针对“长尾”做定向验证

如果上面都查了还是找不到，可能是C 扩展层的内存（如 numpy、orjson）或者内存碎片。这时可以用 py-spy 或 memray 看原生堆栈。

用 memray 举例：

```
python -m memray run -o output.bin my_agent.py
python -m memray flamegraph output.bin
```

它能显示所有分配，包括 C 层面的 malloc，能暴露出到底是 Python 对象还是某个 libcurl 在囤积缓冲区。

## 总结回答（面试时可直接用的结论）

Agent 内存泄漏排查的核心思路是：先观其形，再钳其源。

🔍 用 tracemalloc 找“哪种对象在涨”；

🧲 用 objgraph 追“谁在引用它”；

🧠 在 Agent 语境下，重点查历史消息、工具结果、缓存策略、客户端连接、异步任务这五类无意识持有；

🛠️ 最后用 memray 兜底检查原生层泄漏。

真正难缠的问题，往往不是代码 bug，而是架构上少了“遗忘”的机制。加上滑动窗口、缓存上限、惰性消费和连接回收，内存就能稳住。
