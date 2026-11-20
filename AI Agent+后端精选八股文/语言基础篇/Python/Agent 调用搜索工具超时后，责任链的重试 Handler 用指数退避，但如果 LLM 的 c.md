# Agent 调用搜索工具超时后，责任链的重试 Handler 用指数退避，但如果 LLM 的 context window 快满了，等待重试会浪费 token 预算，怎么设计？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/OvcawbrXDisTWYkob8Rc2kGtnhd

这个问题一眼就能看出候选人的系统设计敏感度——资源约束下的重试策略。直接给方案。

---

### 🔍 问题本质

责任链的重试 Handler 按指数退避死等，但等待期间请求级别的上下文全挂在 LLM 的 context window 里，token 预算持续燃烧。

必须把“时间开销”重新翻译成“token 开销”，让重试自己感知钱包厚度。

---

### 💡 三条硬核设计思路

#### 🧠 一、Token‑Aware 退避：给重试设“预算闸门”

重试 Handler 不再只看次数，每次执行前先算 剩余 token 预算：

```
代码块
def retry_with_budget(chain, budget_tokens):
    while chain.has_next():
        if estimate_context_tokens() > budget_tokens * 0.9:
            # 预算将尽，立即熔断并压缩上下文
            return fallback_compressed_response()
        try:
            return chain.execute()
        except Timeout:
            wait = exponential_backoff()
            # 等待期间 token 仍在消耗，因此做预算扣除预测
            if not can_afford(wait, budget_tokens):
                raise TokenBudgetExceeded()
```

✅ 把“等待”折算成 token（例如等待 10s ≈ 0.2% 窗口），达到阈值后主动降级。

---

#### 🧳 二、上下文卸载：重试不要拖着整个历史一起走

思想：重试链只保留“必要的恢复信息”，其余全部移出窗口。

1. 触发重试时，立刻将当前完整的对话压缩成摘要，只留必要工具调用参数。
1. 重试期间的临时上下文放在外部存储（如 Redis），成功后把结果插入回对话，失败则用摘要重建。

text

[原始对话历史] → 压缩 → [摘要 + 当前工具请求] ← 窗口很轻

这样重试消耗的 token 几乎为零增长，指数退避的等待不影响预算。

---

#### ⚡ 三、非阻塞注入：把重试变成“预留空位”

彻底取消等待：工具超时后，Handler 立刻返回一个 PlaceholderResult，同时在后台异步重试。

1. LLM 继续生成，看到占位符就生成类似“关于搜索结果，稍后补充”。
1. 重试成功后，通过中间件把结果注入到下一轮对话或更新消息块。
1. 只有失败时才会触发用户可见的降级。

✅ 完全解耦 LLM 生成与重试时间，token 零浪费。

---

### ⚖️ 三种策略的取舍矩阵

> （此处为内嵌电子表格，需登录飞书查看）

---

### 🧩 与责任链的优雅融合

在标准重试 Handler 前插入一个预算检查节点：

```
代码块
请求 → [Token Budget Checker] → [重试Handler] → [执行]
          ↓ 预算不足
       立即压缩上下文或降级
```
