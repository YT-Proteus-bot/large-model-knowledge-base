---
title: 高级 Agent 架构
topic: LangChain面（高频）
sub_topic: 七、代理与工具 (Agents & Tools)
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/YCqcw5pbDir9BEkHaCScKiSBnwc
source_path: LangChain面（高频）/七、代理与工具 (Agents & Tools)/高级 Agent 架构.md
questions: 1
chars: 1641
build_date: '2026-11-20'
images_removed: 0
---

# 高级 Agent 架构

> 主题：LangChain面（高频） › 七、代理与工具 (Agents & Tools)　｜　原始文件：LangChain面（高频）/七、代理与工具 (Agents & Tools)/高级 Agent 架构.md

## 题1：⚙️ AgentExecutor 是如何管理 Agent 的思考-行动循环的？内部有哪些关键逻辑？

AgentExecutor 是 LangChain 中运行 Agent 的“执行引擎”，它封装了 ReAct（Reasoning + Acting）循环，负责调度 Agent 的思考、行动和观察，直到任务完成或达到终止条件。

### 内部执行流程

AgentExecutor 的循环可以用以下流程图表示：

```
┌─────────────────────┐
│  用户输入 + 历史记忆  │
└─────────┬───────────┘
          ▼
┌──────────────────────────────┐
│ 1. 构建 Prompt                │
│ (系统指令 + 可用工具 + scratchpad) │
└─────────┬────────────────────┘
          ▼
┌──────────────────────────────┐
│ 2. 调用 LLM，获取生成文本      │
└─────────┬────────────────────┘
          ▼
┌──────────────────────────────┐
│ 3. 输出解析 (Output Parser)   │
│ 提取 Thought, Action, Action Input │
└─────────┬────────────────────┘
          ▼
      ┌───┴───┐
      │是最终答案?│
      └───┬───┘
          │ 是 → 返回最终答案
          │ 否
          ▼
┌──────────────────────────────┐
│ 4. 执行工具 (Tool Execution)   │
│ 得到 Observation               │
└─────────┬────────────────────┘
          ▼
┌──────────────────────────────┐
│ 5. 追加到 scratchpad          │
│ 回到步骤 1，继续循环           │
└──────────────────────────────┘
```

### 关键内部逻辑

- _take_next_step：核心方法，负责执行一步：调用 LLM、解析输出、执行工具（如果有 Action），返回 AgentFinish 或 AgentAction。
- Scratchpad 管理：所有中间步骤（Thought/Action/Observation）被拼接成 intermediate_steps 列表，在下一轮作为上下文注入 Prompt，让 Agent “记住”自己做了什么。
- 错误处理：如果输出解析失败，AgentExecutor 会捕获异常，将错误信息追加到 Prompt，让 LLM 修正输出（可通过 handle_parsing_errors 控制）。
- 终止条件：
- LLM 主动输出 Final Answer:。
- 达到 max_iterations（最大步数）。
- 自定义 stop 词触发。
- 工具返回 AgentFinish 信号（某些特殊工具）。
- 早停机制：达到最大步数后，若未完成，根据 early_stopping_method 决定是报错 (force) 还是让 LLM 强制总结 (generate)。

✅ 实践感悟：AgentExecutor 其实就是一个“while 循环”，但它的价值在于把错误处理、状态管理和终止逻辑都优雅地封装了。如果自己写循环，很容易遗漏边界情况。
