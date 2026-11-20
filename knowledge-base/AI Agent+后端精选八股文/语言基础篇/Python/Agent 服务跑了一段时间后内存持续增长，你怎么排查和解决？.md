---
title: Agent 服务跑了一段时间后内存持续增长，你怎么排查和解决？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/U2mNwf4Ivi97WGkx4Sjc3qbHnqb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Agent 服务跑了一段时间后内存持续增长，你怎么排查和解决？.md
questions: 0
chars: 963
build_date: '2026-11-20'
images_removed: 1
---

# Agent 服务跑了一段时间后内存持续增长，你怎么排查和解决？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Agent 服务跑了一段时间后内存持续增长，你怎么排查和解决？.md

每个做过长稳服务的应该都踩过坑。我按“现象 → 定位 → 分析 → 根治”这条线讲，中间画一个排查路线图，最后说一套我固定的防护清单。

## 现象确认：不是内存抖动，是持续增长

上来先别急着动手，得区分两种涨法：

- 锯齿形：请求来了涨，请求结束跌回去，是正常的工作内存。
- 阶梯形：每处理一批请求就涨一个台阶，从不回落，这才是泄漏。

快速确认：用 psutil.Process().memory_info().rss 打点，每隔 10 秒记一次，画个折线图，斜率一直正的就是有问题。

## 排查路线图（我用脑图的形式画出来）

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

## 实际怎么定位（以我最近一个 Agent 为例）

Step 1：起一个 tracemalloc 对比

```
import tracemalloc
tracemalloc.start()

# ... 跑 100 轮对话 ...
snapshot = tracemalloc.take_snapshot()

# 再跑 100 轮
snapshot2 = tracemalloc.take_snapshot()
stats = snapshot2.compare_to(snapshot, 'lineno')
for s in stats[:5]:
    print(s)
```

立刻看到 Top1 是某个 messages 列表的大小在疯长。

Step 2：查谁在抓着这个列表不放

用 gc.get_referrers() 或 objgraph.show_backrefs() 画出来：

```
import objgraph
obj = problematic_messages_list
objgraph.show_backrefs([obj], max_depth=5, filename='chain.png')
```

结果发现链是：messages_list → dict → Tool 实例 → Tool.init 里 self.callback = lambda: messages_list 形成的闭包循环。这个 Tool 实例又因为放在全局 ACTIVE_TOOLS 里，永远不释放。
