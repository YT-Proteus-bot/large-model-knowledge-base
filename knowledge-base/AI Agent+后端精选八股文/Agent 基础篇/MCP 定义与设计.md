---
title: MCP 定义与设计
topic: AI Agent+后端精选八股文
sub_topic: Agent 基础篇
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Na85wIeNuichcjklD4LcUQ6Gn2c
source_path: AI Agent+后端精选八股文/Agent 基础篇/MCP 定义与设计.md
questions: 3
chars: 805
build_date: '2026-11-20'
images_removed: 3
---

# MCP 定义与设计

> 主题：AI Agent+后端精选八股文 › Agent 基础篇　｜　原始文件：AI Agent+后端精选八股文/Agent 基础篇/MCP 定义与设计.md

## 题1：🔌 MCP 是什么？它解决了什么问题？

MCP（Model Context Protocol，大模型上下文协议）是一套开放的标准化通信协议，专门用来让 LLM 应用与外部工具、数据源、资源进行安全、可发现的交互。

你可以把它想象成 “AI 世界的 USB-C”。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

它解决的问题：工具与数据集的碎片化、集成成本高。

在 MCP 出现之前，每个 LLM 应用想要调用外部工具（数据库、文件系统、API）都需要：

- 为每个工具写一套适配器（插件、函数调用封装）。
- 不同应用之间无法复用这些集成。
- 模型选择工具时缺乏统一描述和发现机制，导致“信息孤岛”。

MCP 做了一件事： 把工具提供者（Server）和 AI 应用（Host/Client）解耦。你写一个 MCP Server 暴露工具，所有支持 MCP 的 AI 应用都能直接发现并使用，就像你的 USB-C 充电器可以给手机、平板、笔记本充电一样。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

## 题2：⚙️ 核心工作原理、对比传统 API 的优势、适用场景

## 题3：1 核心原理：以工具为中心的发现-调用模式

MCP 基于 JSON-RPC 2.0 协议，定义了三条核心原语：

- Tools（工具）：模型可调用的函数。Server 暴露一个工具列表，每个工具包含名称、描述、参数 JSON Schema。LLM 根据描述决定是否调用。
- Resources（资源）：可供 Client 读取的数据，比如文件内容、数据库记录、实时信息。类似 REST 的 GET，但通过统一协议暴露。
- Prompts（提示模板）：预定义的对话模板，帮助用户更好地与 LLM 互动。

调用流程：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
