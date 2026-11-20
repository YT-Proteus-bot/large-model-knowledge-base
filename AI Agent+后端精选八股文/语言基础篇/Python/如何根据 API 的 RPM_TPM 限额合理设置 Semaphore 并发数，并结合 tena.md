# 如何根据 API 的 RPM/TPM 限额合理设置 Semaphore 并发数，并结合 tenacity 构建完整并发控制方案？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/LbtdwWSHhiWqTAkYb0UcvG8OnqK

这个问题很经典，调用外部 API 时几乎都会碰到。我分享一下我的思考框架和落地方式，带着具体的代码思路，既讲原理也讲坑。

---

#### 🔍 1. 先搞清楚两个概念：RPM 与 TPM

- RPM (Requests Per Minute)：每分钟最多发多少次请求。
- TPM (Tokens Per Minute)：每分钟最多消耗多少个 token（常见于 LLM API，如 OpenAI）。

你的并发方案必须同时满足两个限制，缺一不可。

---

#### 📐 2. 从限额推导合理的 Semaphore 并发数

Semaphore 控制的是“同一时刻最多有多少个请求在飞”。核心逻辑：一个槽位在 60 秒内能完成多少次请求？

假设：

- 每个请求平均耗时 D 秒（从发出到收到完整响应）
- 每个请求平均消耗 T 个 token

那么：

```
代码块
一个槽位每分钟能完成的请求数 = 60 / D
总请求数 ≤ RPM  ⇒  并发数 * (60 / D) ≤ RPM
                  并发数 ≤ RPM * D / 60
总token消耗 ≤ TPM  ⇒  并发数 * (60 / D) * T ≤ TPM
                  并发数 ≤ TPM * D / (60 * T)
```

👉 最终安全并发数：

```
代码块
semaphore_size = min( RPM * D / 60 ,  TPM * D / (60 * T) ) * 0.8
```

乘以 0.8 是安全裕度，防止网络抖动、突发流量或者请求耗时分布不均。

举一个具体的例子：

- RPM=500，TPM=200000，D=2s，T=1000
- RPM * D / 60 = 500 * 2 / 60 ≈ 16.67
- TPM * D / (60 * T) = 200000 * 2 / (60 * 1000) ≈ 6.67
- 取较小值 6.67，再乘 0.8 → 并发数设为 5 比较稳。

如果没有 TPM 限制，就只看 RPM 那条公式。

---

#### 🔒 3. 用 Semaphore 实现“在飞”并发控制（asyncio 版）

```
代码块
import asyncio
semaphore = asyncio.Semaphore(5)  # 按计算出的值动态设置
async def call_api(payload):
    async with semaphore:
        # 实际 HTTP 请求
        response = await client.post(url, json=payload)
        return response
```

这里的关键是 async with semaphore 会保证：

- 超过限制的协程会被挂起等待
- 请求完成（无论成功还是抛异常）后自动释放槽位，避免死锁

---

#### 🔁 4. tenacity 负责“被限流后的重试”

Semaphore 只控制并发数量，不能避免 429，因为你的速率可能刚好卡在临界点，或者有瞬时尖峰。这时就轮到 tenacity 出场，专门处理限流退避。

```
代码块
from tenacity import (
    retry, stop_after_attempt, wait_exponential, retry_if_result
)
def is_rate_limited(response):
    return response.status_code == 429
@retry(
    retry=retry_if_result(is_rate_limited),
    wait=wait_exponential(multiplier=1, min=2, max=60),
    stop=stop_after_attempt(5),
    reraise=True
)
async def api_call_with_retry(payload):
    async with semaphore:            # 重试期间继续占用信号量，保证这个请求
        response = await client.post(url, json=payload)
        if response.status_code == 429:
            # 可选：解析 Retry-After 头，动态指导等待时间
            retry_after = response.headers.get("Retry-After")
            # tenacity 的 wait 会自行等待，这里只需要让函数判定为需要重试
        return response
```
