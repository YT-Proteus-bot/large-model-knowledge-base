# 一个 Multi-Agent 系统需要并发调用 10 个子 Agent，每个 Agent 内部又会调用 LLM，如何设计并发控制策略避免触发 OpenAI Rate Limit？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/NLRRwKFMOi3JhDk82tJcspHknlc

面对这种“10个子Agent并发，每个还自己调LLM”的场景，我最怕的就是各自为政——每个Agent自己管自己，结果瞬时几百个请求砸过去，429肯定没跑。我的思路是从上往下做三层控制：

🧱 第一层：全局闸门（Semaphore）
所有LLM调用必须走同一个 asyncio.Semaphore，不管你是主控Agent还是子Agent，抓HTTP请求前先抢令牌。比如限制最大并发 concurrency=20，就绝不会同时有超过20个请求在飞。这个Semaphore我会做成单例塞进依赖注入里，谁用LLM谁就 async with global_sem:，简单粗暴挡住一波洪峰。

⏱️ 第二层：速率整形（Token Bucket / Leaky Bucket）
Semaphore只能管并发连接数，但RPM（每分钟请求数）和TPM（每分钟token数）是另一维度限制。我一般会基于 asyncio 自己实现一个令牌桶，或者直接用 aiolimiter。

- 桶里每分钟固定补充 N 个请求令牌、M 个token令牌。
- 每次调用LLM前，除了拿并发令牌，还要用 await rate_limiter.acquire(request_tokens, token_count) 去消耗令牌。
- 如果桶空了，协程就挂起等着，自然形成平缓的请求流，不会出现尖峰。

🔗 第三层：每个Agent内部的调用收敛
为了防止子Agent无节制地 gather 一堆内部LLM调用，我会让每个子Agent也感知全局限制。做法是不给它直接调HTTP的权限，而是让它调用一个统一的 llm_service.call(prompt, max_tokens) 函数。所有速率控制逻辑都藏在这个函数背后，Agent只是传参。这样即便10个子Agent同时 gather 自己的子任务，实际出口也被统一管制了。

🛡️ 兜底策略：退避 + 重试 + 降级
网络抖动或偶发429还是有可能。我会在 llm_service 里加上：

- 指数退避重试（1s、2s、4s），并带上 jitter 防惊群。
- 如果连续重试失败，或者令牌长时间获取不到，就抛给Agent一个降级策略：用缓存回答、换小模型、或返回“当前繁忙”的占位结果，绝不把异常扩散到整个系统。

🧩 图示一下调用链（脑补流程图）

![image](../../../../assets/CBcWdnAacoF3aOx7Cy0cDcMznab/HuNrbUfBAoY7gkxCnEPcJkANn7c.png)

所有箭头最终并成一股流，两层闸门卡在前头，速率曲线就变得非常平滑。

这样设计的好处是：控制点单一、策略可配置、Agent完全不用关心限流细节。我在之前一个项目里，用这套把并发上限放到了200个Agent，OpenAI报429的次数从每天几百次降到个位数。

实际跑起来我还会加一个Prometheus指标，实时看令牌消耗速率和等待队列长度，一旦接近阈值就触发告警或者动态把部分Agent切到更经济的模型，保证整个多Agent系统既快又不越界。
