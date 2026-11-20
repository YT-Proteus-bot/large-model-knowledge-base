---
title: 项目 6：多 Agent 协作研究平台
topic: 大模型实战项目（可写进简历）持续更新
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/P0FNwplO7i3UEJk6cu0chNkWnlh
source_path: 大模型实战项目（可写进简历）持续更新/项目 6：多 Agent 协作研究平台.md
questions: 4
chars: 2642
build_date: '2026-11-20'
images_removed: 0
---

# 项目 6：多 Agent 协作研究平台

> 主题：大模型实战项目（可写进简历）持续更新　｜　原始文件：大模型实战项目（可写进简历）持续更新/项目 6：多 Agent 协作研究平台.md

## 一、项目概述

"智囊团"是一个基于多 Agent 协作的自动化研究报告生成平台。系统由规划者（Planner）、检索者（Researcher）、撰写者（Writer）、审核者（Reviewer）四个专业 Agent 组成，在监督者（Supervisor）的统一调度下，通过"规划 → 检索 → 撰写 → 审核 → 循环修改"的流水线协作，最终输出结构完整、论据充分、经过质量校验的研究报告。

> 🎯 **核心设计理念：**监督者模式（Supervisor Pattern）+ 子图隔离（Subgraph Isolation）+ 循环反馈（Iterative Refinement）。每个 Agent 封装为独立 LangGraph 子图，通过共享状态传递中间产物，监督者负责路由决策与循环终止判断。

## 二、完整架构设计

## 题1：1 系统总览

系统采用"主图 + 子图"的两层图结构：

- **主图（Supervisor Graph）**：定义全局状态、监督者路由节点、条件边与循环控制逻辑。
- **子图（Agent Subgraph）**：每个 Agent 内部是一个独立的 StateGraph，封装该角色的 Prompt 模板、工具调用、输出解析与内部状态流转。

主图通过 add_node("planner", planner_subgraph.compile()) 将子图作为节点注册，子图的输入输出通过全局 State 的字段映射完成对接。

## 题2：2 各 Agent 职责

## 题3：3 通信方式

Agent 之间不直接通信，所有信息交换通过**共享全局状态（Shared State）**完成，这是 LangGraph 的核心机制：

1. **状态传递**：每个 Agent 节点执行完毕后，将产出写入全局 State 的对应字段，下一个 Agent 从 State 中读取所需字段。
1. **字段隔离**：不同 Agent 读写不同字段，避免冲突。例如检索者只写 evidence_pack，撰写者只读 evidence_pack 并写 draft。
1. **Reduction 策略**：对于需要累积的字段（如多轮修改记录），使用 add 操作符追加；对于覆盖字段使用默认覆盖策略。
1. **子图状态映射**：子图有自己的局部 State，通过 input_schema / output_schema 与全局 State 做字段映射，子图内部状态不泄漏到全局。

## 题4：4 数据流

完整的数据流转路径如下：

```
数据流
用户输入 research_topic
    │
    ▼
┌──────────┐  写入 outline, sub_questions, search_strategy
│ 规划者    │──────────────────────────────────┐
└──────────┘                                  │
    │                                         ▼
    │                                  ┌──────────┐
    │                                  │ 检索者    │ 写入 evidence_pack
    │                                  └──────────┘
    │                                         │
    │                                         ▼
    │                                  ┌──────────┐
    │                                  │ 撰写者    │ 写入 draft, revision_round
    │                                  └──────────┘
    │                                         │
    │                                         ▼
    │                                  ┌──────────┐
    └─────────────────────────────────▶│ 审核者    │ 写入 review_result
                                       └──────────┘
                                            │
                              ┌─────────────┴─────────────┐
                              │ approved                   │ needs_revision
                              ▼                            ▼
                         输出最终报告              监督者判断循环次数
                                                   ≤ max_rounds?
                                                   ┌─── yes ──→ 回到规划者（调整大纲）
                                                   │            或撰写者（按修改建议重写）
                                                   └─── no ───→ 强制结束，输出当前草稿
```
