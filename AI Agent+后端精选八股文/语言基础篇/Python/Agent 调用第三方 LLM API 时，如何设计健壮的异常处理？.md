# Agent 调用第三方 LLM API 时，如何设计健壮的异常处理？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/POjGwQoKoik7HjkDqb6cp6LrnXb

### 一句话回答

把第三方 LLM API 当成不可靠依赖来设计：统一异常体系、区分可重试与不可重试、超时控制、指数退避重试、熔断、降级备用模型、限流、幂等、可观测、密钥与配额保护。

---

### 异常分类与处理策略

> （此处为内嵌电子表格，需登录飞书查看）

---

### 设计原则

> （此处为内嵌电子表格，需登录飞书查看）

---

### 代码示例：带重试、降级、超时的 LLM 客户端

```
代码块
import logging
import httpx
import openai
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    before_sleep_log,
)
logger = logging.getLogger(__name__)
class RetryableLLMError(Exception):
    """可重试异常"""
    pass
class FatalLLMError(Exception):
    """不可重试异常"""
    pass
client = openai.OpenAI(
    api_key="YOUR_API_KEY",
    timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
    max_retries=0,  # 自己控制重试
)
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=20),
    retry=retry_if_exception_type((RetryableLLMError, httpx.TimeoutException, httpx.NetworkError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
def call_llm(messages, model="gpt-4o", timeout=30):
    try:
        resp = client.chat.completions.create(
            model=model,
            messages=messages,
            timeout=timeout,
        )
        return resp.choices[0].message.content
    except openai.RateLimitError as e:
        # 429：尊重 Retry-After
        retry_after = getattr(e, "retry_after", None)
        logger.warning("触发限流，Retry-After=%s", retry_after)
        raise RetryableLLMError("限流") from e
    except openai.APIStatusError as e:
        if 500 <= e.status_code < 600:
            raise RetryableLLMError(f"服务端错误 {e.status_code}") from e
        raise FatalLLMError(f"不可重试错误 {e.status_code}: {e.message}") from e
    except openai.APITimeoutError as e:
        raise RetryableLLMError("请求超时") from e
    except openai.APIConnectionError as e:
        raise RetryableLLMError("连接失败") from e
def call_llm_with_fallback(messages):
    """主模型失败后降级到备用模型"""
    try:
        return call_llm(messages, model="gpt-4o")
    except FatalLLMError:
        raise
    except Exception as e:
        logger.error("主模型失败，降级到备用模型：%s", e)
        try:
            return call_llm(messages, model="gpt-4o-mini")
        except Exception as fallback_error:
            logger.critical("备用模型也失败：%s", fallback_error)
            # 最终兜底：返回模板回复或转人工
            return "抱歉，服务暂时不可用，请稍后重试。"

# 使用示例
if __name__ == "__main__":
    messages = [{"role": "user", "content": "你好"}]
    print(call_llm_with_fallback(messages))
```
