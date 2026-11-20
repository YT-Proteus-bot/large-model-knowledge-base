---
title: 在 Agent 工具调用链中，异常处理的最佳实践是什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Z3BhwMHKfi7whmkqQSjcBhU9nQe
source_path: AI Agent+后端精选八股文/语言基础篇/Python/在 Agent 工具调用链中，异常处理的最佳实践是什么？.md
questions: 0
chars: 3910
build_date: '2026-11-20'
images_removed: 0
---

# 在 Agent 工具调用链中，异常处理的最佳实践是什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/在 Agent 工具调用链中，异常处理的最佳实践是什么？.md

## 一句话回答

Agent 工具调用链的异常处理，不能只靠一层 try/except，而要做分层治理：参数校验层拦截非法输入，执行层做超时、重试、熔断、隔离，结果层返回结构化错误，回传层把错误变成 LLM 可决策的信息，最后还要有降级、幂等、可观测和人工兜底。

## 调用链拆解与各层职责

```
LLM 规划选工具
   ↓
生成工具参数
   ↓
参数校验
   ↓
工具执行（网络/DB/第三方 API）
   ↓
结果解析
   ↓
回传 LLM
   ↓
最终输出 / 下一步动作
```

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 最佳实践清单

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 代码示例：带重试、熔断、结构化错误的工具执行器

```
import time
import random
import logging
from dataclasses import dataclass
from typing import Any, Callable, Dict, Optional
logger = logging.getLogger(__name__)
@dataclass
class ToolError:
    code: str
    message: str
    retryable: bool = False
    details: Optional[dict] = None
class ToolExecutionError(Exception):
    def __init__(self, error: ToolError):
        self.error = error
        super().__init__(error.message)
class CircuitBreaker:
    """简易熔断器：连续失败达到阈值后，短时间内拒绝请求"""
    def __init__(self, failure_threshold=5, recovery_time=30):
        self.failure_threshold = failure_threshold
        self.recovery_time = recovery_time
        self.failures = 0
        self.open_until = 0
    def allow(self) -> bool:
        if time.time() < self.open_until:
            return False
        return True
    def record(self, success: bool):
        if success:
            self.failures = 0
        else:
            self.failures += 1
            if self.failures >= self.failure_threshold:
                self.open_until = time.time() + self.recovery_time
                logger.warning("熔断器打开，%s 秒后重试", self.recovery_time)
class ToolExecutor:
    def __init__(self, tools: Dict[str, Callable], max_retries=3, timeout=10):
        self.tools = tools
        self.max_retries = max_retries
        self.timeout = timeout
        self.breakers = {name: CircuitBreaker() for name in tools}
    def execute(self, tool_name: str, params: dict, idempotency_key: Optional[str] = None) -> dict:
        # 1. 工具是否存在
        if tool_name not in self.tools:
            return {
                "success": False,
                "error": ToolError(
                    code="TOOL_NOT_FOUND",
                    message=f"工具 {tool_name} 不存在",
                    retryable=False,
                ).__dict__,
            }
        breaker = self.breakers[tool_name]
        # 2. 熔断检查
        if not breaker.allow():
            return {
                "success": False,
                "error": ToolError(
                    code="CIRCUIT_OPEN",
                    message=f"工具 {tool_name} 暂时不可用，请稍后重试",
                    retryable=True,
                ).__dict__,
            }
        # 3. 重试循环
        last_error = None
        for attempt in range(1, self.max_retries + 1):
            try:
                # 实际执行可加超时控制，例如 asyncio.wait_for / signal.alarm / 线程池
                result = self.tools[tool_name](**params)
                breaker.record(True)
                return {"success": True, "data": result}
            except TimeoutError as e:
                last_error = ToolError("TIMEOUT", str(e), retryable=True)
            except ConnectionError as e:
                last_error = ToolError("NETWORK_ERROR", str(e), retryable=True)
            except ValueError as e:
                last_error = ToolError("INVALID_PARAMS", str(e), retryable=False)
            except Exception as e:
                last_error = ToolError("UNKNOWN_ERROR", str(e), retryable=False)
            breaker.record(False)
            # 不可重试，直接返回
            if not last_error.retryable:
                break
            # 指数退避 + 抖动
            if attempt < self.max_retries:
                sleep = min(2 ** attempt + random.uniform(0, 1), 10)
                time.sleep(sleep)
        return {"success": False, "error": last_error.__dict__}

# 使用示例
def search_api(query: str):
    # 模拟第三方工具
    return {"result": f"搜索结果：{query}"}
executor = ToolExecutor(tools={"search": search_api})
result = executor.execute("search", {"query": "大模型 Agent"})
print(result)
```
