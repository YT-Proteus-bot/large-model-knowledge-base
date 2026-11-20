---
title: 如何为 AI Agent 系统设计自定义异常层次，以及重试、降级、熔断三层容错机制？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/UF7kwNp1QiBt3Tky34HcKzVinRe
source_path: AI Agent+后端精选八股文/语言基础篇/Python/如何为 AI Agent 系统设计自定义异常层次，以及重试、降级、熔断三层容错机制？.md
questions: 0
chars: 1355
build_date: '2026-11-20'
images_removed: 1
---

# 如何为 AI Agent 系统设计自定义异常层次，以及重试、降级、熔断三层容错机制？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/如何为 AI Agent 系统设计自定义异常层次，以及重试、降级、熔断三层容错机制？.md

Agent 系统的异常不是普通 500，它是 LLM 幻觉、工具超时、上下文溢出 的复合体。异常设计不到位，容错就是猜谜。直接上架构。

## 一、自定义异常层次：让错误“会说话”

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

设计铁律

- 按可恢复性区分：RateLimit → 可重试；TokenLimit → 必须降级；WindowOverflow → 触发上下文压缩。
- 携带上下文：每个异常必须挂载 retry_after、tool_name、consumed_tokens 等字段，容错层才能决策。

## 二、三层容错机制：责任链上的闭环

```
请求 → [熔断器] → [重试器] → [降级器] → 调用
         ↑           ↑           ↑
      失败计数   指数退避    兜底策略
```

## 1️⃣ 重试 (Retry) —— 选择性重试

- 仅对瞬态错误：Timeout、RateLimit（带 After 头）、网络闪断。
- 指数退避 + 抖动，上限 3 次，不重试 TokenLimit、WindowOverflow（重试只会烧 token）。
- 实现：简单装饰器，吃进异常类型白名单。

## 2️⃣ 降级 (Fallback) —— 优雅止损

- LLM 主模型超限 → 自动切廉价模型 (gpt-4 → gpt-3.5-turbo)。
- 工具超时 → 返回缓存结果或静态兜底数据。
- 上下文溢出 → 触发摘要压缩，把历史变成“记忆块”，然后重放。
- 降级策略封装在异常类里更佳：exception.suggest_fallback()。

## 3️⃣ 熔断 (Circuit Breaker) —— 防止雪崩

- 状态机：Closed → Open (错误率 > 50%) → Half-Open。
- 熔断窗口内直接抛 CircuitBreakerOpenException，跳过重试，直接降级。
- 监控维度：按工具粒度统计，防止某个搜索 API 挂掉拖死整个 Agent。

## 架构融合：通过责任链编排

```
class AgentExecutor:
    def execute(self, task):
        try:
            # 责任链：先过熔断，再重试，最后降级
            return CircuitBreaker(
                Retry(
                    Fallback(self._execute_impl),
                    retry_on=[TimeoutError, RateLimitError]
                )
            ).execute(task)
        except AgentException as e:
            # 统一上报 + Trace 打点
            span.record_exception(e)
            raise
```
