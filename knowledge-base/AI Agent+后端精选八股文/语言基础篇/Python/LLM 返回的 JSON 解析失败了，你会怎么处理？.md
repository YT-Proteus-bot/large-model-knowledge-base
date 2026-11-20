---
title: LLM 返回的 JSON 解析失败了，你会怎么处理？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/XsJFwuFJ5isFDHkHQIGc5I9unRb
source_path: AI Agent+后端精选八股文/语言基础篇/Python/LLM 返回的 JSON 解析失败了，你会怎么处理？.md
questions: 1
chars: 1040
build_date: '2026-11-20'
images_removed: 0
---

# LLM 返回的 JSON 解析失败了，你会怎么处理？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/LLM 返回的 JSON 解析失败了，你会怎么处理？.md

## 题1：LLM 返回的 JSON 解析失败了，你会怎么处理？

这个问题很实在，我面高级工程师必问。它考察的不是“会不会调 API”，而是工程化思维和健壮性设计。

🧠 核心考察点

LLM 本质是概率模型，返回非严格 JSON 是常态。你需要展示纵深防御的思路，而不是赌一次解析就成功。

🛡️ 分层处理策略（从优到劣）

⚙️ 第一层：源头约束

- Prompt 工程：要求只返回 JSON，用 Markdown 代码块包裹（json ...），甚至给示例。
- 结构化输出：如果模型支持（如 GPT-4 的 response_format: { type: "json_object" } 或 Function Calling），直接从源头保证输出合法。这是最彻底的办法。

🔧 第二层：容错解析

这一层最能体现经验，常见手段：

- 正则提取：直接截取第一个 { 到最后一个 }，或从代码块中剥离。
- 修复小毛病：尾部多余逗号、属性名缺引号、使用单引号等，可以用 json5 库或 ast.literal_eval 尝试，更专业的用 json_repair 库自动修复。
- 重试一次：如果容错解析失败，把原始字符串原封不动地塞回给模型，并强调：“上一轮你返回了无效 JSON，请严格遵守格式，只输出 JSON”。通常一次纠正就够了。

🔄 第三层：重试与降级

- 有限重试：最多 2 次重试，避免无限循环。重试时可以附上解析错误信息。
- 最终降级：如果全失败，必须决定是返回默认安全值，还是抛出明确业务异常。这个决策要结合场景，例如在用户聊天中可以温和道歉，在自动化管道里可能要记录日志并告警。

💎 加分项

提到 Pydantic 模型校验：解析成 dict 后立刻用 Pydantic 做 schema 验证，缺失字段给默认值或报错，保证下游拿到的是干净对象。顺便提一句 LangChain / LlamaIndex 的输出解析器，证明你知道业界工具但不盲从。

👨🏫 面试官总结

一个过关的回答，应该包含：

✅ 意识到要从源头减少问题（prompt 设计或结构化输出）

✅ 能说出 1-2 种具体的 JSON 修复技巧（正则、json_repair）

✅ 有重试和降级的兜底意识

✅ 提及输出校验（Pydantic）会让我刮目相看

如果你只回答“用 try-except 打印错误然后重试”，那只能算初级。把这条路走透，才是我们要的人。
