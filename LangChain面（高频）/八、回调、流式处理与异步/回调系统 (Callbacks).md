# 回调系统 (Callbacks)

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/LBEfwqP1mi0i5RkjO7ZcMocLnHh

#### 1. ⚙️ LangChain 的回调系统解决了什么问题？为什么需要它？

在 LangChain 的早期版本中，想要监控一个链的运行状态，通常只能修改链的内部源码，比如在 _call 方法里硬插入 print 语句。这种做法极其脆弱：一旦框架升级，修改就会丢失；而且无法根据环境（开发/生产）灵活开启或关闭。回调系统正是为了用一种无侵入、可组合的方式解决这些横切关注点而生的。

🔍 回调系统本质上是一个事件驱动的观察者模式实现。 它在链、LLM、工具、Agent 等关键执行路径上预埋了数十个“钩子”。开发者只需实现一个或多个回调处理器，并将它们注册到链上，就能在这些事件发生时自动收到通知并执行自定义逻辑，而完全不需要触碰框架核心代码。

🎯 它具体解决的核心问题包括：

1. 全链路可观测性：在一次对话中，LLM 被调用了几次？每次 Prompt 是什么？返回了什么？消耗了多少 Token？Agent 的每一步推理（Thought/Action/Observation）是怎样的？这些信息通过回调可以被完整地记录下来，形成类似分布式追踪的调用链。每个事件都带有 run_id 和 parent_run_id，让你能串联起从用户请求到最终回答的全部过程。
1. 无侵入的日志与调试：开发复杂 Agent 时，最头疼的就是“模型为什么选了工具 A 而不是 B？”、“为什么在这一步卡死了？”。通过 on_agent_action 和 on_tool_end 回调，你可以实时打印出 Agent 的思维过程，快速定位问题，而不需要在整个 Agent 代码里到处打补丁。
1. 成本控制与资源管理：大模型调用费用高昂。你可以在 on_llm_end 回调中实时累加 Token 用量，并在达到预算阈值时抛出异常或发送告警，实现自动熔断。这在自动化任务（如 AutoGPT）中至关重要。
1. 流式输出的实现：on_llm_new_token 回调是前端实现“打字机效果”的基础。它每产生一个新 Token 就会触发一次，服务端可以将这些 Token 通过 WebSocket 或 SSE 推送给前端。
1. 安全与合规审计：在某些行业，所有 AI 交互必须存档。回调可以在不修改业务链的前提下，将所有对话原文、工具调用结果加密后存入审计系统。
1. A/B 测试与实验追踪：你可以通过回调将不同链、不同模型的运行数据发送到 MLflow 或 LangSmith，对比效果。

💡 为什么不能只用日志？

普通日志只能记录零散的字符串，无法携带结构化的上下文（如运行 ID、父子关系、标签）。回调系统提供了标准化的上下文对象，使得监控工具能够自动构建出完整的调用拓扑图，这是日志做不到的。

---

#### 2. 📋 列出至少 8 种 LangChain 支持的回调事件类型。

LangChain 的回调事件覆盖了所有关键组件。以下是我在实际项目中使用最频繁的 12 种，按组件分类：

🔗 链 (Chain) 级别

1. on_chain_start：任何链开始执行时触发。可获取链的名称、输入。
1. on_chain_end：链执行完毕时触发。可获取输出。
1. on_chain_error：链执行过程中发生异常时触发。可获取异常对象。

🧠 LLM 级别

1. on_llm_start：LLM 开始生成前触发。可获取 Prompt、模型名称。
1. on_llm_end：LLM 生成完成后触发。可获取完整的生成结果及 Token 消耗。
1. on_llm_error：LLM 调用出错时触发。
1. on_llm_new_token：流式生成时，每产生一个新 Token 就触发一次。仅在流式模式有效。

🛠️ 工具 (Tool) 级别

1. on_tool_start：工具被调用前触发。可获取工具名称和输入参数。
1. on_tool_end：工具执行完毕后触发。可获取返回值。
1. on_tool_error：工具执行出错时触发。

🤖 Agent 级别

1. on_agent_action：Agent 决定调用某个工具时触发。可获取 Action 对象（工具名、输入、日志）。
1. on_agent_finish：Agent 输出最终答案时触发。可获取 AgentFinish 对象。

📝 其他

1. on_text：某些链产生文本输出时触发，用于兼容旧版。
1. on_retriever_start/end：检索器调用前后触发。

这些事件已经足够构建一个完整的监控系统。

---

#### 3. 📝 写出一个自定义回调处理器，用于将 LLM 调用日志写入文件。

下面是一个生产可用的 LLMFileLogger 实现。它考虑了日志轮转、JSON 格式化、长文本截断、以及多线程安全。

```
代码块
import json
import time
import logging
from datetime import datetime
from logging.handlers import RotatingFileHandler
from langchain.callbacks import BaseCallbackHandler
class LLMFileLogger(BaseCallbackHandler):
    """将 LLM 调用详细信息写入文件，支持日志轮转和多线程安全"""
    def __init__(self, log_file: str = "llm_calls.log", max_bytes: int = 10*1024*1024, backup_count: int = 5):
        self._start_times = {}
        # 为每个实例创建独立的 logger，避免与 root logger 冲突
        logger_name = f"LLMCallback_{id(self)}"
        self.logger = logging.getLogger(logger_name)
        self.logger.setLevel(logging.INFO)
        # 使用 RotatingFileHandler 自动轮转
        handler = RotatingFileHandler(
            log_file, maxBytes=max_bytes, backupCount=backup_count, encoding='utf-8'
        )
        handler.setFormatter(logging.Formatter('%(message)s'))
        self.logger.addHandler(handler)
    def on_llm_start(self, serialized, prompts, **kwargs):
        run_id = str(kwargs.get("run_id"))
        self._start_times[run_id] = time.time()
        # 构建日志条目，长 prompt 截断
        log_entry = {
            "event": "llm_start",
            "model": serialized.get("name", serialized.get("id", ["unknown"])[0]),
            "prompts": [p[:500] for p in prompts],  # 截断，避免单条日志过大
            "timestamp": datetime.now().isoformat(),
            "run_id": run_id
        }
        self.logger.info(json.dumps(log_entry, ensure_ascii=False))
    def on_llm_end(self, response, **kwargs):
        run_id = str(kwargs.get("run_id"))
        start = self._start_times.pop(run_id, None)
        latency = time.time() - start if start else None
        # 兼容不同 LLM 的 token 返回格式
        token_usage = {}
        if hasattr(response, 'llm_output') and response.llm_output:
            token_usage = response.llm_output.get('token_usage', {})
        # 获取生成文本
        text = ""
        if response.generations:
            text = response.generations[0][0].text
        log_entry = {
            "event": "llm_end",
            "run_id": run_id,
            "latency_sec": round(latency, 3) if latency else None,
            "prompt_tokens": token_usage.get("prompt_tokens", 0),
            "completion_tokens": token_usage.get("completion_tokens", 0),
            "total_tokens": token_usage.get("total_tokens", 0),
            "response_preview": text[:200],
            "timestamp": datetime.now().isoformat()
        }
        self.logger.info(json.dumps(log_entry, ensure_ascii=False))
    def on_llm_error(self, error, **kwargs):
        run_id = str(kwargs.get("run_id"))
        log_entry = {
            "event": "llm_error",
            "run_id": run_id,
            "error": str(error),
            "timestamp": datetime.now().isoformat()
        }
        self.logger.error(json.dumps(log_entry, ensure_ascii=False))
```
