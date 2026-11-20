---
title: OpenClaw 大模型面试题（附答案）
topic: 大模型专项面试题库
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/CtJ9w5exAijPPak2o3WcxaVvncb
source_path: 大模型专项面试题库/OpenClaw 大模型面试题（附答案）.md
questions: 5
chars: 1472
build_date: '2026-11-20'
images_removed: 0
---

# OpenClaw 大模型面试题（附答案）

> 主题：大模型专项面试题库　｜　原始文件：大模型专项面试题库/OpenClaw 大模型面试题（附答案）.md

以下 30 道题覆盖基础概念、架构设计、记忆机制、工具调用、上下文管理、安全机制等核心维度，按难度递进排列。

### 一、基础概念篇（Q1-Q8）

> 📚 #### Q1：OpenClaw 是什么？一句话说清楚它的定位。

答案： OpenClaw 是一个开源、本地优先的 AI Agent 框架，通过 WhatsApp、飞书、Telegram 等消息平台连接大模型，让 AI 不仅能“说”，还能真正“做”——执行 Shell 命令、操作文件、调用 API、控制浏览器等。

面试加分话术： “OpenClaw 和 ChatGPT 的本质区别在于——ChatGPT 是对话工具，OpenClaw 是执行框架。前者给你答案，后者帮你干活。”

> 🥇 #### Q2：OpenClaw 的前身是什么？为什么改名？

答案： 演变路径为：Warelay → Clawdbot → Moltbot → OpenClaw。改名主要是为了体现项目定位的演变——从一个简单的机器人框架成长为开放的 AI Agent 执行平台。

> 🎨 #### Q3：OpenClaw 和普通聊天机器人（如 ChatGPT）的核心区别是什么？

> 〔原文此处为飞书内嵌组件，内容未随导出〕

答案： OpenClaw 是“执行框架”，ChatGPT 是“对话工具”。

> 📍 #### Q4：OpenClaw 支持哪些消息平台？

答案： 支持 20+ 平台，包括 WhatsApp（Baileys）、Telegram（grammY）、Discord（discord.js）、Slack（Bolt）、飞书、钉钉、Signal、iMessage、IRC、Matrix、Google Chat、Microsoft Teams、LINE 等。

## 题1：OpenClaw 的开源许可证是什么？这意味着什么？

答案： MIT License。允许个人和商业使用、修改和再发布，对二次开发和商业化非常友好。

## 题2：OpenClaw 的技术栈是什么？

- 核心运行时：Node.js v22+
- 开发语言：TypeScript
- 包管理：npm / pnpm / bun
- 消息通道：各平台 SDK（Baileys、grammY、discord.js、Slack Bolt 等）
- 底层推理引擎：Pi 内核（pi-mono）

## 题3：什么是“本地优先”？OpenClaw 的数据存在哪里？

答案： 所有数据（记忆、会话、配置）默认存储在本地 ~/.openclaw/ 目录，以 Markdown 和 JSON 文件形式保存，不依赖云数据库。Workspace（~/.openclaw/workspace）存放 AGENTS.md、SOUL.md 等配置；State Dir（~/.openclaw/）存放凭证、会话历史。

## 题4：OpenClaw 支持哪些大模型？

答案： 模型无关设计，支持 GPT、Claude、Gemini、DeepSeek、Llama、Minimax 等。模型通过 provider/model 格式配置，如 openai/gpt-4o、anthropic/claude-sonnet-4-20250514，也支持 OpenRouter 聚合接口。

### 二、架构设计篇（Q9-Q15）

## 题5：OpenClaw 的五大核心组件是什么？各自职责是什么？

> 〔原文此处为飞书内嵌组件，内容未随导出〕
