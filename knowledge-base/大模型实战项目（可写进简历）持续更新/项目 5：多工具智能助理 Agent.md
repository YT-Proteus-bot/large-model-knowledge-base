---
title: 项目 5：多工具智能助理 Agent
topic: 大模型实战项目（可写进简历）持续更新
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/NFs3wJXB9iPoZkktEMEc3mSenZg
source_path: 大模型实战项目（可写进简历）持续更新/项目 5：多工具智能助理 Agent.md
questions: 3
chars: 1671
build_date: '2026-11-20'
images_removed: 0
---

# 项目 5：多工具智能助理 Agent

> 主题：大模型实战项目（可写进简历）持续更新　｜　原始文件：大模型实战项目（可写进简历）持续更新/项目 5：多工具智能助理 Agent.md

## 一、项目概述

**灵机（LingJi）** 是一个基于 LangGraph 构建 ReAct 循环的多工具智能助理 Agent。它集成搜索、计算器、数据库查询、天气查询、知识库检索 5 个工具，支持工具并行调用与自动错误恢复，最终通过 FastAPI 部署为可对外提供服务的 REST API。

> 🎯 **核心目标**：在单张 24G 显存 GPU 上，用 Qwen-7B-Chat + vLLM 实现一个能自主规划、调用多工具、从错误中恢复并给出最终答案的生产级 Agent 服务。

## 题1：1 功能清单

## 二、完整架构设计

## 题2：1 整体架构图

系统采用分层架构，自下而上分为 **基础设施层**、**工具层**、**Agent 编排层**、**API 服务层** 四层。

```
系统架构总览
┌─────────────────────────────────────────────────────────┐
│                    API 服务层 (FastAPI)                   │
│  POST /chat  │  GET /health  │  POST /tools/register    │
│  请求校验 → 调用 Graph → 流式/非流式响应 → 日志追踪      │
├─────────────────────────────────────────────────────────┤
│                  Agent 编排层 (LangGraph)                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │  Agent   │───▶│  条件边  │───▶│  ToolNode(并行)  │  │
│  │  (LLM)   │◀───│ 路由逻辑 │◀───│  5工具批量执行   │  │
│  └──────────┘    └────┬─────┘    └──────────────────┘  │
│                        │ 错误/重试/结束                   │
│                   State: messages, iterations,           │
│                   retry_count, tool_errors                │
├─────────────────────────────────────────────────────────┤
│                       工具层 (Tools)                       │
│  tavily_search │ calculator │ sqlite_query │ weather │  │
│  kb_retrieval  │  统一 Tool 接口 │  输入校验 │ 超时控制  │
├─────────────────────────────────────────────────────────┤
│                   基础设施层 (Infra)                       │
│  vLLM(Qwen-7B) │ SQLite │ ChromaDB │ Tavily │ Open-Meteo│
│  24G GPU │ 日志(loguru) │ 配置(pydantic-settings)        │
└─────────────────────────────────────────────────────────┘
```

## 题3：2 各模块职责

### 2.2.1 API 服务层
