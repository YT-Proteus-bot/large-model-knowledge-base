# 项目 10：自然语言数据分析 Agent

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/TjjOwv5jfiHeuAkdoXXc34xmnfe

## 一、项目概述

"数析"是一个端到端的自然语言数据分析 Agent。用户用中文（或英文）描述分析需求，系统自动完成意图理解、数据库 Schema 映射、SQL 生成与校验、查询执行、结果可视化，并最终输出一份结构化分析报告。核心价值在于让非技术用户无需编写 SQL 即可完成数据探查与分析，同时通过 Agent 循环（SQL 自检纠错、结果异常回退）保证输出质量。

> 🎯 **核心能力闭环：**自然语言 → SQL → 数据 → 图表 → 报告，全程自动，支持多轮追问与纠错。

## 二、完整架构设计

### 2.1 系统总览

系统采用分层架构，自下而上分为五层：数据层、模型推理层、Agent 编排层、服务层、接入层。LangGraph 作为编排核心，将分析流程建模为有向图，节点间传递共享 State，支持条件分支与循环（SQL 纠错回环）。

```
代码块
subgraph ORCH[LangGraph StateGraph]
    direction TB
    N1[1. 查询理解节点] --> N2[2. Schema 链接节点]
    N2 --> N3[3. SQL 生成节点]
    N3 --> N4{4. SQL 校验节点}
    N4 -->|语法/安全不通过| N3
    N4 -->|通过| N5[5. SQL 执行节点]
    N5 --> N6{6. 结果检查}
    N6 -->|空结果/异常| N2
    N6 -->|有效| N7[7. 结果解读节点]
    N7 --> N8[8. 可视化节点]
    N8 --> N9[9. 报告生成节点]
end
N3 --> LLM[Qwen-7B-Chat 推理层]
N7 --> LLM
N9 --> LLM
N5 --> DB[(SQLite / PostgreSQL)]
N2 --> META[(Schema 元数据缓存)]
N8 --> FS[图表文件存储]
N9 --> FS
N9 --> API
API --> U</whiteboard>
```

### 2.2 各模块职责

模块

职责

关键技术

**查询理解节点**

解析用户自然语言，识别查询意图（聚合/筛选/排序/对比/趋势）、时间范围、度量维度、过滤条件；输出结构化 QueryIntent。

Qwen-7B-Chat + Pydantic 约束输出

**Schema 链接节点**

将用户提到的业务术语映射到具体表/列；基于 Schema 元数据 + 向量相似度 + 列值样例做候选排序；只把相关表的 DDL 注入后续 Prompt，控制上下文长度。

嵌入向量检索 + 规则匹配 + 列注释

**SQL 生成节点**

基于 QueryIntent + 相关 Schema DDL + Few-shot 示例，调用 Qwen-7B-Chat 生成 SQL；输出带解释的 SQL 语句。

Few-shot Prompt + 结构化输出解析

**SQL 校验节点**

三层校验：①SQL 语法解析（sqlglot）；②安全白名单（仅允许 SELECT，禁止 DROP/DELETE/UPDATE/INSERT）；③列名/表名存在性检查。不通过则把错误信息回灌给生成节点重试，最多 3 次。

sqlglot AST 解析 + 关键字黑名单

**SQL 执行节点**

通过 SQLAlchemy 连接数据库执行查询；设置超时（默认 30s）和行数上限（默认 10000 行）；返回 DataFrame。

SQLAlchemy + pandas + 超时控制
