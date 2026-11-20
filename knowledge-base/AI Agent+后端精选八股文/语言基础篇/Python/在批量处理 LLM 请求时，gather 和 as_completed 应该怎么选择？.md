---
title: 在批量处理 LLM 请求时，gather 和 as_completed 应该怎么选择？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/YB2Mw4xtBi06bykopbpc9LbZnfb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在批量处理 LLM 请求时，gather 和 as_completed 应该怎么选择？.md
questions: 0
chars: 1682
build_date: '2026-11-20'
images_removed: 0
---

# 在批量处理 LLM 请求时，gather 和 as_completed 应该怎么选择？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在批量处理 LLM 请求时，gather 和 as_completed 应该怎么选择？.md

聊到批量调 LLM 接口，面试官其实在观察你面对高延迟、高不确定性任务时的编排思路。我习惯先从任务的“目的”出发，再反推工具。

## 先看目的，再定工具

把 LLM 请求当成一群人交作业。两种处理方式：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 批量 LLM 请求的特殊之处

LLM 调用和普通 API 不一样：

- 耗时差异巨大：哪怕同一个 prompt，时长可能从 0.5s 到 30s 不等，模型侧负载、输出长度都在波动。
- 结果可独立消费：很多时候，用户想看的是第一个靠谱的答案，而不是所有答案的排序列表。
- 资源昂贵：无谓等待慢请求既浪费 token 也占用连接，能早取消就早取消。
- 业务可能要求“尽快给点东西看”：比如多路召回的场景，先回来的先展示。

## 具体怎么选？我给三条判断线

① 你需要“保证顺序且全量聚合”吗？

比如批量翻译同一批文本，结果必须一一对应原文顺序，中间不能缺。

👉 用 gather，并设置 return_exceptions=True 来收集异常，避免一个失败全盘崩。

```
results = await asyncio.gather(*tasks, return_exceptions=True)
for i, res in enumerate(results):
    if isinstance(res, Exception):
        # 记录失败，用原文占位或标记
        ...
```

② 你想“抢第一个可用结果”或者“边出边展示”？

比如给一个问题生成多个候选回答，UI 上希望谁先出来就先显示，改善用户感知。

👉 用 as_completed，拿到一个就处理一个，甚至可以在得到满意答案后取消剩余任务。

```
for coro in asyncio.as_completed(tasks):
    try:
        answer = await coro
        # 立即推给前端或缓存
        if is_good_enough(answer):
            for t in tasks:
                t.cancel()       # 提前终止，省 token
            break
    except Exception:
        continue
```

③ 你需要在“固定总时间”内拿最大收益？

比如同时请求多个 LLM 供应商，300ms 内谁先回来用谁的，超时就不等了。

👉 这种硬实时场景，as_completed 天然适合，配合 asyncio.wait 的 timeout 也很好。

## 我的常用组合拳

实际上，大型批量任务很少只用一种，我一般会分层：

```
[第一阶段：as_completed 抢首批]
   → 得到最先完成的 30% 结果，满足用户的首屏需求
[第二阶段：gather 慢请求]
   → 等待剩余结果，用于后台补全、日志、缓存
```

也可以结合 asyncio.wait 的 FIRST_COMPLETED 特性，获得更灵活的控制权。

## 别忘记配合 Semaphore

批量打 LLM 接口，无论用哪个编排器，都必须加并发限制，不然 RPM 瞬间爆掉。我通常会把 asyncio.Semaphore 套在最外层任务里，gather 或 as_completed 只管调度，信号量管流量。

```
sem = asyncio.Semaphore(5)
async def bounded_task(task):
    async with sem:
        return await task
tasks = [bounded_task(llm_call(p)) for p in prompts]

# 再交给 gather 或 as_completed
```
