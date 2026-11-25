---
title: Agent 数据中台：从 0 到 1 入门与实践
topic: 教学知识库
sub_topic: Agent 数据中台
doc_type: kb-doc
level: 入门
chapters: 11
chars: 58628
build_date: '2026-11-25'
---

# Agent 数据中台：从 0 到 1 入门与实践

讲解 Agent 应用背后的数据设施：数据接入、存储分层、文档解析、切分与向量化、知识库与语料管理、会话轨迹回流、数据集与标注、数据质量与安全合规，最后给出落地路线与反模式清单。

> 适合人群：零基础或刚入门的开发者、需要带新人的工程师；每章独立成节，可按需跳读，也适合整篇灌入 RAG 系统做检索。

## 目录

1. 认识 Agent 数据中台
2. 总体架构与核心术语
3. 数据接入与采集
4. 存储选型与分层建模
5. 文档解析与清洗流水线
6. 切分、向量化与检索索引
7. 知识库与语料管理
8. 会话轨迹与数据回流
9. 数据集与标注管理
10. 数据质量、元数据与安全合规
11. 成本性能、落地路线与常见坑

## 1. 认识 Agent 数据中台

### 1.1 一个售后客服 Agent 的三十秒

用户问：「上周买的降噪耳机能退吗？」一个可用的 Agent（智能体，能自主规划步骤、调用外部工具并根据结果调整行为的大模型应用）要在三十秒内完成四件事：查到退货政策原文、调订单接口确认购买时间、记住用户上一轮说过「已拆封」、把这次回答留档以便抽检。

这四件事分别依赖四类数据：政策文档、订单工具、对话轨迹、评测样本。它们通常散落在 Confluence、订单库、日志平台和运营的 Excel 里。每接一个新场景，团队就把接入、清洗、切分重做一遍，于是同一份退货政策出现三个版本，三个 Agent 给出三种答案。

Agent 数据中台（Agent Data Platform，把 Agent 所需的知识、工具描述、交互轨迹与评测数据统一接入、加工、治理，并以检索与管理接口对外提供的平台层）解决的正是这种重复与失真：接入一次，多个 Agent 复用；改动一处，全量生效。

一家零售公司的三个团队就踩过这个坑：客服团队用上周导出的 PDF，导购团队用共享盘里的旧版 Word，售后团队直接从帮助中心网页复制。三个月后，同一句「七天无理由」在三个入口分别被答成七天、十五天和「以页面为准」。问题不在模型，而在数据没有单一来源。

需要说明的是，Agent 数据中台既不是一个更大的数据库，也不是一套现成的问答界面；它的核心资产是被治理过、可被检索与评测的数据，以及围绕这些数据的元信息。

### 1.2 四类数据诉求

先厘清两个基础概念。大语言模型（Large Language Model，LLM，基于海量文本训练、能理解与生成自然语言的模型）本身不掌握企业私有事实；检索增强生成（Retrieval-Augmented Generation，RAG，先从知识库检索相关片段、再交给模型组织答案的技术路线）用外部数据补上这块短板。RAG 的效果上限，几乎由数据质量决定。

| 数据类别 | 英文名 | 一句话定义 | 典型来源 | 更新节奏 |
| --- | --- | --- | --- | --- |
| 知识数据 | Knowledge Data | 支撑回答的事实性语料 | 制度、FAQ、产品手册 | 周或月 |
| 工具数据 | Tool Data | 工具名称、用途、参数结构与返回样例 | OpenAPI、内部 RPC、MCP Server | 随接口版本 |
| 交互轨迹 | Interaction Trace | 一次会话中用户输入、模型输出、工具调用与结果的完整记录 | 网关日志、Agent 运行时 | 实时 |
| 评测与训练数据 | Evaluation & Training Data | 带标准答案或偏好判断的样本集合 | 人工标注、线上 badcase | 按迭代 |

其中的 MCP（Model Context Protocol，模型上下文协议，统一模型访问外部工具与数据源的开放协议）正在成为工具数据的事实标准。表中 badcase 指线上答错的真实案例，它是评测数据最直接的来源。

四类数据的生产与消费并不对称，这决定了接入方式。知识数据由业务方生产、模型消费，以批量同步为主，难点在解析与版本；工具数据由研发生产，靠注册与契约校验接入，关键是参数结构别漂移；交互轨迹由系统自动生产，以流式接入为主，重点是脱敏与采样；评测数据无法自动生产，必须人工标注，它最稀缺，也最该被复用。

这也解释了为什么四类数据要放进同一个中台：一次接入的成本是固定的，复用的场景越多，单场景成本越低。反过来，如果只有一两个场景，自建中台的分摊成本会高于直接采购。

四类数据不是四张孤立的表，而是围绕一次请求相互引用。一条轨迹记录的结构示例（JSON）如下：

```json
{
  "trace_id": "t-20260512-000871",
  "tenant": "shop-a",
  "turns": [
    {"role": "user", "text": "上周买的耳机能退吗"},
    {"role": "assistant", "text": "可以，7 天内未拆封支持退货",
     "tool_calls": [{"name": "query_order", "args": {"days": 7}, "status": "ok"}]}
  ],
  "retrieved_chunks": ["doc-119#p3"],
  "feedback": {"score": 4, "label": "helpful"}
}
```

trace_id 是主键：retrieved_chunks 指回知识数据，tool_calls 指回工具数据，feedback 沉淀为评测数据。数据中台的第一项设计决策，就是让这四个字段在任何一条链路上都可追溯。

### 1.3 与传统数据中台、BI 报表体系的异同

传统数据中台（Data Mid-End，把分散在各业务系统的数据加工成可复用数据资产的平台层）与商业智能（Business Intelligence，BI，用报表和指标体系呈现经营状况的分析方式）服务的是人：把数据算准、做成表、给人看。

Agent 数据中台服务的是模型：把数据切好、编好索引、在毫秒内送进模型的上下文窗口。两者共享治理方法，但交付物与验收标准不同，差别集中在七处。

还有一个容易忽略的差异：BI 的报表一旦生成，所有人看到的是同一份数字；Agent 的每次回答都是一次新的检索与生成，同一个问题问两次可能得到不同表述。因此数据中台必须保存每次检索命中的切片快照，否则线上答案无法复现，也无法定责。

| 维度 | 传统数据中台 / BI | Agent 数据中台 |
| --- | --- | --- |
| 数据主体 | 结构化表与指标 | 文档、切片、轨迹、工具描述 |
| 消费方式 | 人读报表、逐层下钻 | 模型检索、拼进上下文 |
| 正确性 | 精确、可对账 | 语义相关、可引用出处 |
| 质量验收 | 口径一致、波动阈值 | 评测集上的召回率与答案正确率 |
| 交付形态 | 宽表、指标 API | 检索接口、数据集、工具注册表 |
| 时效 | T+1 为主 | 分钟级到实时 |
| 血缘粒度 | 表与字段 | 文档、切片与版本 |

验收口径的差异最容易被低估：BI 看对账差额，Agent 看召回率（Recall，应被检索到的相关内容中实际被检索到的比例）与答案正确率。切片（Chunk，文档按语义或长度切分后作为检索单元的最小片段）则是 Agent 侧独有的管理对象，它的版本、来源页码与密级都要单独登记。

相同点同样关键：两者都要做元数据（Metadata，描述数据来源、责任人、更新时间的属性信息）、血缘（Data Lineage，数据从源头到消费的加工路径）、权限隔离与质量监控。差异只发生在最后一公里，从人的眼睛换成了模型的上下文窗口。

传统中台的现成能力也不能直接搬过来。数仓存的是聚合后的结果，Agent 需要的是原文与出处；宽表在聚合时丢掉了句子边界与页码，检索命中后无法回答「依据在哪」。权限粒度同样不同：BI 的权限通常到库表，Agent 需要到文档甚至切片，否则一份混装文档会把薪酬段落带进普通员工的上下文。

可以用一张共用的资产注册表把四类数据管起来（SQL）：

```sql
CREATE TABLE data_asset (
  asset_id   TEXT PRIMARY KEY,        -- 文档、工具、数据集统一 ID
  asset_type TEXT NOT NULL,           -- knowledge / tool / trace / eval
  source_uri TEXT NOT NULL,           -- 原始位置，用于回溯
  version    TEXT NOT NULL,           -- 检索时按版本过滤
  owner      TEXT NOT NULL,           -- 责任人，缺失即视为不合规
  updated_at TIMESTAMPTZ NOT NULL
);
```

没有 owner 和版本的资产，在 BI 里叫「野表」，在 Agent 里叫「毒知识」：它会被检索到，并被模型当作事实讲给用户。

### 1.4 在大模型应用架构中的位置

Agent 应用通常分四层，数据中台位于最底部的数据层，向上为检索层供料。

```text
应用层 Application   客服 Agent、Copilot、数据分析助手
   ↑
编排层 Orchestration   Planning · Tool Calling · Memory · 工作流
   ↑
检索层 Retrieval      Query 改写 · 混合检索 · 重排 · 上下文组装
   ↑
数据层 Data           Agent 数据中台：接入、解析、切分、索引、轨迹、评测集
```

其中向量检索（Vector Retrieval，用向量相似度从海量文本中找相关内容的检索方式）与重排（Rerank，对初步召回结果做二次精排）属于检索层，但它们依赖的向量数据库（Vector Database，专门存储向量并支持相似度查询的数据库）与语料，由数据层产出。

| 层 | 核心职责 | 典型组件 | 中台是否负责 |
| --- | --- | --- | --- |
| 数据层 Data | 把原始数据变成可检索、可评测的资产 | 采集、解析、切分、向量库、对象存储 | 是 |
| 检索层 Retrieval | 把问题变成上下文 | 查询改写、召回、过滤、重排 | 部分，索引与语料由中台提供 |
| 编排层 Orchestration | 决定下一步做什么 | 状态机、工具路由、记忆 | 否 |
| 应用层 Application | 面向用户的产品形态 | 会话界面、渠道接入 | 否 |

划边界的判据很简单：一项能力如果与具体 Agent 无关、能被多个应用复用，就下沉到数据层；否则留在编排层。把「该调哪个工具」的判断写进数据中台，是典型的越界。

参数集中在检索层配置，数据层只提供被检索的对象（YAML）：

```yaml
retrieval:
  top_k: 20          # 单路召回条数
  rerank_top_n: 5    # 重排后进入上下文的条数
  min_score: 0.35    # 低于阈值直接拒答，避免硬编
```

数据层对上层暴露的最低契约是一条检索接口：输入问题与租户，输出带出处与版本的切片。一次调用（Bash）如下：

```bash
curl -s http://127.0.0.1:8080/v1/retrieve \
  -H 'Content-Type: application/json' \
  -d '{"query":"退货政策","tenant":"shop-a","top_k":5}'
```

返回结果里，出处和版本必须随切片一起返回，否则上层无法做引用与追责（JSON）：

```json
{"query": "退货政策", "tenant": "shop-a", "top_k": 5,
 "hits": [{"chunk_id": "doc-119#p3", "score": 0.82,
           "version": "v2026-04", "source": "policy.pdf#page=3"}]}
```

接口里省略 tenant，是权限事故的常见起点；省略 version，则永远说不清答案依据的是哪一版。数据层不生成答案、不决定调用顺序，它只保证「被检索到的东西是对的、可追溯的」。

### 1.5 什么样的团队需要建，什么阶段不该建

同时满足三条再动手：复用（三个以上 Agent 共用同一批知识与工具）、规模（文档两千篇以上，或月问答五万次以上）、约束（有评测、审计或密级隔离的硬要求）。缺一条，先用轻量方案顶住。

| 阶段 | 典型特征 | 建议 |
| --- | --- | --- |
| 验证期 | 一个场景、文档不足五百篇、按周迭代 | 不建平台，托管向量库加脚本即可 |
| 复制期 | 两三个场景，知识与工具开始重复接入 | 建薄中台：统一接入、元数据、检索 API |
| 规模化 | 五个以上场景、多租户、合规审计 | 建完整中台，补齐评测与数据回流 |

多租户（Multi-tenancy，一套系统同时服务多个相互隔离的客户或部门）是分水岭：一旦出现跨部门的密级差异，靠 if-else 拼权限的脚本方案会迅速失控。

动手前先做一次六项自评，每项 1 分，低于 4 分不要立项。

| 自评项 | 判断问题 |
| --- | --- |
| 场景 | 是否已有至少一个上线并有人在用的 Agent？ |
| 复用 | 是否有两个以上场景要接同一批文档？ |
| 内容 | 是否有能拍板准确性的业务 owner？ |
| 规模 | 文档是否超过两千篇，或月问答超过五万次？ |
| 评测 | 是否能拿出三百条带标准答案的样本？ |
| 运维 | 是否有至少两人能长期维护？ |

不该建的三个信号：还没有一个跑通的场景；没有内容 owner 愿意对准确性负责；把中台当成一次向量数据库采购。薄中台大约两人六周可以跑通，完整中台通常需要五到八人投入两个季度，先想清楚谁来长期维护。

### 1.6 一个失败案例的复盘

某制造企业 2024 年立项「企业知识中台」，目标是把十二万份文档统一起来服务全公司问答。投入八人、十八个月，采购了向量数据库与数据湖。上线三个月，月活两百人，首答满意度 38%，评测集上的召回率（应被检索到的相关内容中实际被检索到的比例）只有 41%，P95 延迟 8.2 秒。

时间线也很典型：前三个月选型，中间六个月做接入与解析，接着五个月开发检索与界面，最后四个月补权限，上线时才发现没有人对内容准确性负责。十八个月里，真正花在「验证答案对不对」上的时间不足两周。

| 现象 | 根因 | 正确动作 |
| --- | --- | --- |
| 召回率仅 41% | 扫描版 PDF 未做文字识别，表格被解析成乱码 | 先抽检解析质量，再谈切分策略 |
| 答非所问 | 没有评测集，靠演示和感觉验收 | 上线前先建三百条金标准问答 |
| 无人维护 | 文档没有 owner，过期政策长期在线 | 接入时强制 owner 与有效期字段 |
| 权限越界 | 全公司共用一个索引，薪酬文档被检索到 | 检索前置租户与密级过滤 |
| 回答很慢 | 十二万份文档共用一个索引，未按场景分层 | 按场景建子索引，限制单次检索范围 |

复盘结论只有一句：把平台当成交付物，而平台的价值只能由场景证明。先建能力、后找场景，是最贵的顺序。

可执行的替代路径是用六周做一个最小闭环：五十篇文档、一百条评测、一个真实场景，指标达标后再横向复制。该企业复盘后把项目拆成三个小场景重新立项，第一个季度只做售后政策，首答满意度回到 71%。

### 1.7 四个常见坑

- 先买向量库，再想数据从哪来：采购驱动而非场景驱动，最后库很贵、内容很薄。
- 把文档数量当资产规模：十二万份里可用的往往不足两万，其余解析失败、过期、重复，是负债不是资产。
- 只存切片不存原文与版本：出问题无法回溯，也答不出「这条答案依据哪一版政策」。
- 没有评测集就上线：无法判断每次改动是变好还是变坏，调优退化为玄学。

| 症状 | 先查哪里 |
| --- | --- |
| 答案过期 | 资产版本与有效期字段是否落地 |
| 答非所问 | 解析质量与评测集，而不是先换模型 |
| 越权引用 | 检索前是否按租户与密级过滤 |
| 越建越重 | 是否有场景 owner 与明确的验收指标 |

一句话判断信号：当第二个 Agent 开始重复接入同一批文档时，就该建中台；在此之前，先把第一个场景的评测集建起来。

## 2. 总体架构与核心术语

一份 86 页的产品手册上传后，会被拆成上千个检索单元；用户提问时，只有其中几条会进入模型上下文（Context，模型一次生成时能看到的全部输入）。面向 Agent（智能体，能规划步骤、调用工具并完成任务的模型应用）的数据中台，要做的不是把文件堆进一个库，而是让每条回答都能找到依据、追溯到源、被别人复用。

### 2.1 核心术语表：11 个对象先统一语言

在 Agent 数据中台里，同一个词经常被不同角色混用，导致需求评审时各说各话。按“核心数据”和“运行与评测”两组固定 11 个词，每个词给出英文、一句话定义、主键和最容易犯的错。

**核心数据对象（8 个）**

| 术语 | 一句话定义 | 主键与粒度 | 常见误用 |
| --- | --- | --- | --- |
| 语料（Corpus） | 按业务主题、来源或租户组织，可整体治理的一批内容单元。 | `corpus_id`；一个语料含多个内容单元，可跨多个索引版本。 | 把语料等同于向量库；语料是内容边界，建不建索引是服务选择。 |
| 文档（Document） | 一次纳入中台、可独立授权和版本化的内容单元，如一份 PDF、一个网页或一条工单。 | `doc_id` + `version`；权限通常落在文档级。 | 把整个网站当一个文档，更新和授权粒度都会失控。 |
| 分块（Chunk） | 文档按标题结构或长度切开后，用于检索和引用的最小文本单元。 | `chunk_id`；由 `doc_id + version + ordinal + text_hash` 生成，保证可复现。 | 只按固定字数硬切，切断表格和标题；分块不是越小越准，过小会丢失上下文。 |
| 嵌入（Embedding） | 把文本映射为固定维度稠密向量的表示，语义相近的文本在向量空间中距离更近。 | `vector_id` + `model_version`；维度和归一化方式必须与索引一致。 | 换了嵌入模型却不重建索引，查询向量与库内向量不在同一空间。 |
| 向量索引（Vector Index） | 为大规模向量近邻搜索建立的数据结构，如 HNSW、IVF，用近似计算换取速度。 | 集合（collection）+ `index_version`；参数如 `M`、`ef_construction` 随数据规模调整。 | 把索引当唯一数据源，删除原文后无法重建；参数从别处照抄，召回率骤降。 |
| 知识库（Knowledge Base） | 面向一个业务场景，把语料、索引、权限和版本绑定在一起的服务单元。 | `kb_id`；对外以知识库为检索入口，对内可挂多个索引版本。 | 一个 Agent 建一个库，内容重复且权限难统一；知识库是服务边界，不是文件夹。 |
| 数据集（Dataset） | 为训练、评测或分析目的，按规则筛选并版本化的一组样本。 | `dataset_id` + `version`；样本要有稳定 ID、来源和标签。 | 把生产日志直接当数据集，缺去重、脱敏和标签，训练与评测都不可复现。 |
| 血缘（Lineage） | 记录数据实体从源到产物经过哪些处理、依赖哪个版本的实体关系图。 | 实体级边：`from_entity`、`to_entity`、`activity`、`run_id`。 | 只存运行日志不做解析，出事无法回答“这条答案来自哪一版解析器”。 |

**运行与评测对象（3 个）**

| 术语 | 一句话定义 | 主键与粒度 | 常见误用 |
| --- | --- | --- | --- |
| 轨迹（Trace） | 一次请求从输入、检索、组装到回答的完整结构化执行记录，可回放。 | `trace_id`；关联 `retrieval_id`、`chunk_id`、`prompt_hash`、模型版本。 | 只记最终答案；没有候选集和提示词版本，无法区分检索错还是生成错。 |
| 评测集（Evaluation Set） | 版本冻结、标注明确、可重复运行的一组评测样本及期望结果。 | `eval_id` + `version`；题目与训练语料隔离，防止泄漏。 | 拿训练集当评测集，分数虚高；题目一改就不可比。 |
| 标注任务（Annotation Task） | 把一批样本、标注指南、人员、时限和验收规则组织起来的工作单元。 | `task_id`；关联样本 ID、标注人、指南版本、一致性指标。 | 只发 Excel 不给指南和抽检，标注结果无法验收。 |

三组边界需要单独记：

- 文档是授权和更新的单位，分块是检索和引用的单位；权限通常从文档继承到分块，不能只过滤文档列表而不过滤分块。
- 语料回答“有哪些内容”，知识库回答“以什么权限、用哪个索引、对外提供哪一版服务”。
- 数据集是泛称，评测集是其中一种用途；判断标准不是名字，而是版本是否冻结、标签是否验收、题目是否与语料隔离。

### 2.2 五层架构：接入、存储、加工、服务、应用

五层依次是接入、存储、加工、服务、应用：接入层接收原始文件，存储层保存原始文件与可重建的明细，加工层把它变成可检索的分块与向量，服务层通过知识库提供检索，并把每次请求写成轨迹。五层是逻辑边界，不是五套必须独立部署的集群。小团队可以把对象存储、元数据库和索引放在同一台机器上，但谁负责什么、失败落在哪一层必须清晰。

```text
源系统 → 接入层 → 存储层 → 加工层 → 服务层 → 应用层
           │        │        │        │
           └────────┴────────┴────────┴── 元数据 / 血缘 / 权限 / 质量 / 成本
```

| 层 | 核心职责 | 输入 → 输出 | 关键设计 | 典型失败 |
| --- | --- | --- | --- | --- |
| 接入层（Ingestion Layer） | 连接源系统，完成鉴权、限流、断点续传、幂等与原始落盘。 | 数据库、对象存储、网页、API、会话 SDK → 原始文件 + manifest（接入清单，记录来源、校验和、权限、采集时间）。 | 原始区只追加不覆盖；同租户同来源的校验和去重；保留 `source_id`。 | 重复采集、覆盖原文件、字段截断。 |
| 存储层（Storage Layer） | 分原始区、明细区、服务区、元数据区存放不同形态的数据。 | 原始文件、解析文本、分块、向量、索引 → 可查询、可重建的数据。 | 原始区不可变；明细区按 `doc_id/version` 分区；索引能从明细重建。 | 把对象存储当数据库扫、索引与元数据不一致。 |
| 加工层（Processing Layer） | 解析、清洗、切分、向量化、去重、评测与标注数据构造。 | 原始/明细数据 → 分块、向量、数据集、评测集。 | 任务幂等；`pipeline_version` 固定；失败重试不产生重复分块。 | 解析失败静默、切分参数漂移、向量维度不一致。 |
| 服务层（Serving Layer） | 知识库管理、混合检索、重排、ACL（Access Control List，访问控制列表）过滤、上下文组装、轨迹回传。 | 用户查询 → 上下文、引用、执行轨迹。 | 先鉴权后检索；按 token（模型处理文本的基本计量单位）预算装箱；用别名（alias）切流。 | 越权、召回为空、上下文超限。 |
| 应用层（Application Layer） | 意图识别、工具调用、提示词编排、答案呈现。 | 上下文与工具结果 → 用户可见答案。 | 不直连向量库；只通过服务层 API；展示引用。 | 检索逻辑硬编码、无法回放、引用缺失。 |

数据自下而上流动，控制点则分布在每一层：接入层决定什么能进来，存储层决定保留多久，加工层决定如何理解，服务层决定谁能检索到什么，应用层决定如何呈现。任何一层吞掉相邻层的职责，都会让重建和追责变难。例如应用层直接连向量库，看似省了一个 API，实际绕过了 ACL、限流和轨迹，故障时无法判断是检索配置还是应用逻辑的问题。

分层的价值在变更隔离。把嵌入模型从 A 换到 B，只需重跑加工层、在新集合建索引、切换别名，应用层不用改代码；如果应用直连向量库，换模型就要改所有 Agent。故障定位也靠分层：接入延迟看采集任务，解析失败看加工任务，召回为空看服务层，答案超限看上下文预算。指标混在一起，告警就会失去指向。

元数据与血缘回答“谁产生、依赖谁”；权限回答“谁能看文档和分块”；质量回答“是否完整、一致、新鲜”；成本回答“每层存了多少、每次问答消耗多少 token”。这四项不单独成层，但每层都要埋点。一条可用的血缘至少能回答：`chunk_123` 来自 `doc_42@v3` 的第 17 页，由 `parser=pdfium-1.4`、`splitter=heading-2.1` 生成，写入 `kb_product_v7`，被 `trace_9f2` 引用；只存日志流水做不到这一点。

### 2.3 一条数据从源到模型上下文的完整链路

RAG（Retrieval-Augmented Generation，检索增强生成，先从外部知识检索内容再交给模型生成答案的方法）的完整链路，本质是把文档切成可检索的分块（Chunk，切片后的最小文本单元），再把最相关的几条送进模型上下文；这个过程可以拆成 10 个阶段。以一份 86 页、12.4 MB 的产品手册为例，表中的参数是可直接起步的基线，不是唯一答案。

| 阶段 | 关键动作与参数 | 产物与标识 | 失败信号 |
| --- | --- | --- | --- |
| 0 登记 | 校验租户、来源、ACL、文件校验和；分配 `doc_id` 与 `version`；同租户同来源的重复校验和直接复用。 | `manifest`：`doc_id`、`source_id`、`sha256`、`acl`、`version`。 | 校验和不一致；同一文档多版本无主。 |
| 1 接入 | 原文件写入 `raw/{tenant}/{source}/{doc_id}/{version}/original.pdf`，只追加。 | `raw_uri` + `manifest`。 | 路径被覆盖；采集时间缺失。 |
| 2 解析 | 提取文本、表格、页码；OCR（光学字符识别）置信度低于 0.6 的页标记人工复核。 | `parsed.jsonl`：`page`、`text`、`bbox`、`confidence`。 | 页数为 0；表格串行；乱码率高。 |
| 3 清洗 | 去页眉页脚、合并断行、去重、统一全半角；记录删除比例。 | `clean.jsonl` + `removed_ratio`。 | 删除比例超过 30%；正文被误删。 |
| 4 切分 | 按标题层级切，目标 512 token，重叠 64；保留父子块和页码。 | `chunks.jsonl`：`chunk_id`、`doc_id`、`ordinal`、`text`、`page`。 | 分块平均长度异常；表格被切断。 |
| 5 向量化 | 固定嵌入模型与维度（示例 1024 维），批大小 64，向量归一化。 | `vectors.parquet`：`chunk_id`、`vector`、`model_version`。 | 维度不一致；空文本向量。 |
| 6 索引 | 写入新集合 `kb_product_v7`，HNSW（一种近似最近邻图索引）参数 `M=16`、`ef_construction=200`；构建完成后原子切换别名。 | collection + alias。 | 索引点数少于分块数；别名未切换。 |
| 7 检索 | 查询改写；BM25 与向量各取 50 条；RRF 融合；重排后取 8 条；检索前注入 ACL 过滤。 | `retrieval_id` + 候选列表。 | 召回为空；无权限结果混入。 |
| 8 组装 | 按 token 预算装箱，预留系统提示与答案；按分数或章节排序；绑定引用编号。 | prompt + citations + `trace_id`。 | 超出窗口；引用错位。 |
| 9 回流 | 记录查询、过滤条件、候选、入选分块、`prompt_hash`、答案、时延、反馈。 | `trace`。 | 只存答案；无法回放。 |

阶段 8 产出的 prompt 就是这次请求的模型上下文；阶段 9 把上下文、答案和反馈绑定到同一条轨迹。

BM25（Best Matching 25，一种基于词频的稀疏检索算法）擅长精确术语，向量召回擅长语义相近；混合检索常用 RRF（Reciprocal Rank Fusion，倒数排名融合，按名次而不是原始分数合并多路召回），公式是 `score = Σ 1/(60 + rank_i)`；它不依赖两路分数的量纲，适合作为第一版融合策略。权限过滤必须在检索前注入，不能等召回后再删：后者会让 top-k 被无权限内容占满，也留下越权风险。

上下文组装代码接近可运行，Python 3.10+：

```python
"""context_builder.py：权限过滤 → 去重 → 按 token 预算装箱 → 生成引用"""
from dataclasses import dataclass

@dataclass
class Hit:
    chunk_id: str
    doc_id: str
    text: str
    score: float
    acl: tuple[str, ...]

def build_context(query: str, hits: list[Hit], tenant: str,
                  token_budget: int = 7000, reserve: int = 1200) -> tuple[str, dict]:
    # 1) 先鉴权再排序：避免无权限内容进入候选集，也避免事后过滤导致召回不足
    visible = [h for h in hits if tenant in h.acl]
    # 2) 按 chunk_id 去重：父子块、重叠窗口都可能重复召回同一段文字
    seen, picked, used = set(), [], 0
    for h in sorted(visible, key=lambda x: x.score, reverse=True):
        cost = len(h.text)  # 中文按字符数粗估；生产环境无论中英文都应使用模型 tokenizer
        if h.chunk_id in seen or used + cost > token_budget - reserve:
            continue
        seen.add(h.chunk_id)
        picked.append(h)
        used += cost
    # 3) 引用编号与 chunk_id 绑定，答案中的 [n] 才能回查到原文页码
    context = "\n\n".join(f"[{i}] {h.text}" for i, h in enumerate(picked, 1))
    trace = {
        "query": query,
        "tenant": tenant,
        "used_tokens": used,
        "chunk_ids": [h.chunk_id for h in picked],
        "doc_ids": sorted({h.doc_id for h in picked}),
    }
    return context, trace
```

轨迹把这条链路串起来：`trace_id` 关联查询、过滤条件、候选分块、最终进入上下文的 8 条、`prompt_hash`、模型版本和用户反馈。只有最终答案的日志不是轨迹，它无法回答“是没召回、排错了，还是模型没用对”。这份手册解析耗时 93 秒，产出 1,186 个分块；一次提问先取 50 条候选，重排后 8 条进入上下文，预算 7,000 token、预留 1,200，实际放入约 5,800 token。每个数字都应能从轨迹复算。

### 2.4 角色分工：数据工程、算法、运营、安全

角色分工解决的是“谁对结果负责”，不是“谁给谁打杂”。四个角色都必须能独立解释自己交付物的版本和验收标准。

| 角色 | 主责 | 关键交付物 | 明确不负责 |
| --- | --- | --- | --- |
| 数据工程（Data Engineering） | 接入、存储、解析清洗、调度、血缘。 | 接入 manifest、调度 DAG、数据质量报告、血缘图。 | 不决定业务答案；不调提示词。 |
| 算法（Algorithm） | 嵌入、检索、重排、评测、上下文策略。 | 索引配置、检索基线、评测报告、bad case 分析。 | 不审批源系统权限；不排标注产能。 |
| 运营（Operations） | 知识库内容运营、标注任务、反馈归集。 | 标注指南、语料更新清单、bad case 周报。 | 不改索引参数；不绕过 ACL 取数。 |
| 安全（Security） | 分级分类、ACL、脱敏、审计、合规。 | 数据分级表、访问策略、审计报告。 | 不负责日常检索调参；不默认持有全量原文。 |

RACI（Responsible、Accountable、Consulted、Informed，分别表示执行、最终负责、被咨询、被告知）能避免“大家都负责、实际上没人负责”。每行只能有一个 A：

| 活动 | 数据工程 | 算法 | 运营 | 安全 |
| --- | --- | --- | --- | --- |
| 新数据源接入 | A/R | C | I | C |
| 知识库上线 | C | A/R | R | C |
| 检索效果下降 | C | A/R | C | I |
| 标注任务验收 | I | C | A/R | I |
| 权限策略变更 | C | I | C | A/R |
| 评测集版本发布 | I | A/R | C | C |

权限配置要写成可执行策略，而不是口头约定。最小示例（省略租户维度）：`allow` 与 `deny` 冲突时 `deny` 优先，真实策略中每条权限都应带租户范围。

```yaml
roles:  # 四类角色的最小权限示例
  data_engineer:
    allow: ["source:read", "raw:read", "raw:write", "lineage:write"]
  algorithm:
    allow: ["chunk:read", "index:write", "eval:run"]
  ops:
    allow: ["kb:read", "annotation:write", "feedback:read"]
  security:
    allow: ["policy:write", "audit:read"]
    deny: ["raw:read"]  # 原文访问走临时工单，默认拒绝
```

这份策略只定义接口权限，不替代数据行级 ACL；分块能不能被看到，仍要由检索时的 ACL 字段决定。交接物也要可执行：manifest 能重放采集，DAG 能重跑解析，索引配置能重建集合，评测报告能复现分数，标注指南能计算一致性，访问策略能审计。只有群聊和会议纪要，分工就会退化成互相甩锅。

### 2.5 常见坑与排查

第一次上线最常见的四个坑：错误码是示例命名，方便在日志和告警中统一检索。

| 错误码 | 典型现象 | 根因 | 排查与修复 |
| --- | --- | --- | --- |
| `E_RAW_OVERWRITE` | 重建索引后引用全失效。 | 接入层用同一路径覆盖原文件，没有版本。 | 对 raw 前缀开启对象版本或只追加；路径带 `doc_id/version`；入库校验 `sha256`；用旧 manifest 重放验证。 |
| `E_CHUNK_DRIFT` | 同一文档重建后分块数变化、引用错位。 | 解析器或切分参数升级未记录。 | 把 `parser_version`、`splitter_version` 写入分块元数据；对比新旧 `chunk_id` 集合；先建新集合再切别名。 |
| `E_ACL_LEAK` | 低权限账号搜到受限内容。 | 只在应用层过滤，或只过滤文档不过滤分块。 | 用低权限账号跑金丝雀查询；检查索引 ACL 字段；检索前 pre-filter；定期做越权扫描。 |
| `E_CTX_OVERFLOW` | 答案被截断、模型忽略后半段。 | 只按 top-k 取，不按 token 预算；未预留输出。 | 记录 `used_tokens` 和 `finish_reason=length` 比例；按预算装箱；超长先摘要或降低 top-k；把系统提示与答案预留写进配置。 |

排查顺序建议从结果往源头倒推：先用 `trace_id` 找到进入上下文的分块，再核对分块所属文档版本，再查索引点数和别名，最后回到原始文件的校验和。顺序反了，容易在应用层反复改提示词，却修不好一个丢失的页眉。

```bash
curl -s http://localhost:6333/aliases | jq  # 查看所有别名，确认 kb_product 指向哪个物理集合
curl -s http://localhost:6333/collections/kb_product_v7 | jq '.result.points_count'  # 查看新集合点数，确认写入是否完成
```

索引点数应与明细表的分块数一致；建议把 0.5% 作为阻断阈值，先查写入失败日志，不要直接切别名。四个坑的共同点是：都破坏了可重建或可追溯。上线前用一条金丝雀文档跑完整链路，再从轨迹反向核对原始文件、分块、索引、上下文四处数量与权限，比事后救火便宜得多。

## 3. 数据接入与采集

### 3.1 接入层交付什么

数据接入（Data Ingestion，把分散在上游系统里的原始数据按统一契约搬进中台落地区的过程）的产出只有两样：数据体（原样保留的 JSON、CSV、图片）与采集元数据（这批数据从哪来、何时来、采到哪个位置）。接入层不做清洗和切分，但必须可重放、可追溯、可停止：任一时刻中断都不破坏已写入的数据。

### 3.2 六类数据源与各自的采法

| 数据源 | 接入通道 | 默认模式 | 主要风险 |
| --- | --- | --- | --- |
| 业务数据库 | JDBC 直连 / binlog CDC | CDC | 长事务、从库延迟 |
| 内部 API | HTTP + 游标分页 | 增量（游标） | 429 限流、分页漂移 |
| 文件共享 | SMB/NFS/对象存储清单 | 全量 + 清单比对 | 同名覆盖、权限失效 |
| 网页爬取 | HTTP + HTML 解析 | 全量 + URL 去重 | 反爬、页面结构变更 |
| IM 聊天记录 | 开放平台消息 API | 增量（时间窗） | 撤回、图片需另存 |
| 应用日志与埋点 | 文件 tail / Kafka | 追加式增量 | 乱序、重复上报 |

开工前先答三问：唯一键、时间字段、删除怎么表达。网页没有主键，只能用规范化 URL 当键；日志与埋点只能拿（设备号、事件名、客户端时间戳）拼键。数据库物理删除在增量里看不见，只能靠源端 is_deleted 软删标记或周期性全量比对补删。

### 3.3 全量、增量与 CDC

全量（Full Load，每次把源端数据集整体搬一遍）、增量（Incremental Load，只搬水位之后变化的数据）、CDC（Change Data Capture，订阅数据库日志逐条捕获增删改）按源端能力三选一，不是先进替代落后。水位（Watermark，标记已采到哪个位置的进度值）是三者共用的核心概念。

| 模式 | 判断依据 | 适用场景 | 代价 |
| --- | --- | --- | --- |
| 全量 | 无可靠时间字段、数据量小 | 配置表、网页、小文件集 | 重复搬运，源库压力大 |
| 增量 | 有可信 updated_at 或自增 id | 绝大多数业务表 | 依赖字段可信度 |
| CDC | 有 binlog 且要秒级 | 订单、会话等高频表 | 运维复杂，需处理 schema 变更 |

binlog 方案要求 binlog_format=ROW、唯一 server_id 与 REPLICATION SLAVE 权限，再用 Debezium 或 Flink CDC 订阅。

```sql
-- 采集前自检，任一条不满足都别开工
SHOW VARIABLES LIKE 'binlog_format';     -- 必须 ROW，STATEMENT 没有行级前后值
SHOW VARIABLES LIKE 'binlog_row_image';  -- 建议 FULL，否则拿不到更新前的旧值
SHOW MASTER STATUS;                      -- 记下 file + position 作为初始水位
```

```json
{"op":"u","after":{"id":42,"status":"done"},
 "source":{"db":"crm","table":"orders","file":"mysql-bin.000318","pos":918273}}
```

`source.file` 与 `source.pos` 就是水位，必须与数据同批落库，否则断了接不上。

时间戳增量的坑在同秒多行会漏、时钟回拨会乱，解法是复合水位加重叠窗口：水位记 (updated_at, id)，查询回退 60 秒，重复部分交给幂等吸收。

```python
# Python 3.11：复合水位时间戳增量
SQL = """SELECT id, updated_at, payload FROM orders
         WHERE updated_at >= %s AND (updated_at > %s OR id > %s)  -- 同秒靠 id 递进
         ORDER BY updated_at, id LIMIT 500"""
def next_page(conn, ts, last_id, overlap_sec=60):
    start = ts - timedelta(seconds=overlap_sec)          # 回退窗口，防边界丢行
    rows = conn.execute(SQL, (start, ts, last_id)).fetchall()
    if rows:                                              # 写库成功后才推进水位
        ts, last_id = rows[-1]["updated_at"], rows[-1]["id"]
    return rows, ts, last_id
```

游标增量（Cursor，服务端返回的下一页位置令牌）用于 API：只认服务端给的 next_token，不要自己算 offset，源端边插边翻必然重复又漏行。

### 3.4 调度：让采集按水位跑起来

调度（Scheduling，按时间或事件触发采集任务并管理其状态）要定四件事：频率、超时、失败处置、水位存哪。采集频率不应高于上游更新频率，能用水位驱动就别写死 cron。

```yaml
# collector.yaml：一个采集任务的完整配置
job: crm_orders_cdc
source: {type: mysql, host: crm-db, db: crm, table: orders}
mode: cdc
schedule: "*/5 * * * *"      # 兜底轮询，平时靠事件触发
concurrency: 2               # 最多 2 个分片并行
rate_limit: {qps: 50, burst: 100}
timeout_sec: 900
checkpoint: {store: postgres, table: collect_checkpoint}
retry: {max_attempts: 5, backoff: exponential, base_sec: 2, jitter: true}
on_call: data-platform@example.com   # 连续失败 3 次告警
```

### 3.5 断点续传、失败重试与限流

断点续传（Checkpoint，把采集进度持久化到外部存储，重启后从上次位置继续）的粒度按源定：数据库按 (file, pos) 或 (ts, id)，大文件按分片偏移，API 按 next_token。铁律是先写数据、后推水位：崩溃最多重复一批而不丢一批，重复交给幂等兜住，即至少一次（At-Least-Once，允许重复、不允许丢失）。

| 错误 | 典型场景 | 处理方式 |
| --- | --- | --- |
| 429 / 5xx / 超时 | 限流、网关抖动 | 退避重试，尊重 Retry-After |
| 401 / 403 | token 过期、无权限 | 不重试，立即告警 |
| 400 / 422 | 参数或 schema 不符 | 不重试，进死信队列 |
| 404 | 资源已被删除 | 记一条删除事件，不重试 |

限流用令牌桶（Token Bucket，按固定速率发令牌、取到令牌才放行）。QPS、并发、单批条数要一起调，只压 QPS 会让单批越滚越大、内存飙升。

### 3.6 幂等与去重

幂等（Idempotency，同一批数据重复写入多次，结果与只写一次相同）靠两道键：写入层用业务主键（Business Key，上游唯一标识一条记录的字段组合，如 order_id）做 upsert；内容层用内容哈希（Content Hash，对正文做 SHA-256 得到的指纹）判断内容是否真变，没变跳过，省下解析与向量化的成本。doc_id 取 `sha256(source_id + ":" + 业务主键)` 的前 32 位，content_hash 取正文的 sha256 全量值，两者都随批次落库。

```sql
-- PostgreSQL：内容没变就不更新，避免下游无谓重算
INSERT INTO doc_raw (doc_id, content_hash, body, watermark)
VALUES (%(doc_id)s, %(hash)s, %(body)s, %(wm)s)
ON CONFLICT (doc_id) DO UPDATE
  SET body = EXCLUDED.body, content_hash = EXCLUDED.content_hash, watermark = EXCLUDED.watermark
  WHERE doc_raw.content_hash <> EXCLUDED.content_hash;
```

### 3.7 采集元数据

采集元数据（Ingestion Metadata，描述这批数据从哪来、何时来、采到哪个位置、由谁负责的字段）。缺了它，出问题只能全链路重跑，也找不到签字人。

| 字段 | 示例 | 作用 |
| --- | --- | --- |
| upstream_system | crm-mysql-prod | 定位上游与对接人 |
| upstream_version | schema-v12 / api-v3 | 上游变更时评估影响面 |
| watermark | mysql-bin.000318:918273 | 断点续传与重放 |
| owner | 张伟（CRM 组） | 出问题找谁 |
| row_count / checksum | 5000 / sha256:ab12… | 完整性校验 |

### 3.8 常见坑

- 用 offset 翻页做增量：源端边插边翻，必然重复又漏行，只认游标或复合水位。
- 水位用 `>` 且不留重叠：同秒多行、事务提交延迟都会丢数据，用复合水位加 60 秒回退。
- 只存数据不存上游版本和责任人：上游改字段后采集静默失败，半个月后才发现答案全是旧的。

## 4. 存储选型与分层建模

Agent 数据中台同时跑三类数据：原始文件、加工后的文本与向量、服务层要秒级返回的索引和指标。没有一种数据库能同时做好，通行做法是「一类数据放一种存储，用统一主键串起来」。选错的代价不是慢，而是对不上账：向量库有、chunk（切片）表没有，检索就返回一段已不存在的文本。

### 4.1 五类存储的分工与组合链路

对象存储（Object Storage，以 Bucket + Key 组织、走 HTTP API 读写字节的存储）存原始文件与中间产物：便宜、容量近乎无限，但无事务、无索引、列举慢，不能按条件查询（S3、OSS、MinIO）。

数据湖 / 湖仓（Data Lake / Lakehouse，前者在对象存储上按目录堆放原始文件，后者再叠加表格式与元数据让文件具备表语义）承接原始落库与批量加工，解决小文件合并、Schema 演进、快照与时间旅行（Iceberg、Hudi、Paimon）。

OLAP（Online Analytical Processing，列式存储、面向大范围扫描聚合的引擎）存 chunk 元数据、采集指标与检索埋点，回答「昨天哪个知识库摄取失败最多」（ClickHouse、Doris、StarRocks）。

向量数据库（Vector Database，存高维向量并支持近似最近邻检索的系统）只做语义召回，不擅长事务、复杂 join、细粒度权限过滤（Milvus、Qdrant、pgvector）。

搜索引擎（Search Engine，基于倒排索引做关键词匹配与结构化过滤的系统）负责 BM25 关键词召回与带过滤聚合，常与向量库并用做混合检索（Hybrid Search，关键词与向量双路召回），典型产品是 Elasticsearch、OpenSearch。

| 存储 | 存什么 | 不该放什么 | 常见故障 |
| --- | --- | --- | --- |
| 对象存储 | 原始文件、中间产物 | 需条件查询的元数据 | Key 散乱，无法按前缀清理 |
| 湖仓 | ODS 原始表、DWD 明细 | 在线点查 | 小文件过多，元数据膨胀 |
| OLAP | chunk 元数据、指标、日志 | 原文件、长正文 | 单行高频更新，part 过多 |
| 向量库 | 向量 + 少量过滤字段 | 正文、权限表 | 删除不同步，悬空向量 |
| 搜索引擎 | 倒排索引、正文副本 | 唯一事实源 | 分片过多，集群不稳 |

写入链路：采集器写 `raw/{dt}/{doc_id}.pdf`，ODS 登记元数据，chunk 落明细表，向量写入向量库并回填 `chunk.vector_id`，正文同步写搜索引擎。读侧则反过来：问题向量查向量库取 top-k 的 chunk id，回 chunk 表取正文与权限，拼成上下文。

向量库与搜索引擎都是派生索引（Derived Index，可从事实源重建的加速结构），唯一事实源（Source of Truth，数据唯一可信的那份）只有对象存储加 chunk 明细表。检验标准：索引全删能否在 4 小时内重建？能才算派生索引。

### 4.2 ODS/DWD/DWS/ADS 与「原始-加工-服务」

两种视角是一回事：数仓按加工深度分层，中台按数据形态分层。ODS（Operational Data Store，贴源层）几乎不做变换，保留原始字节与采集元数据；DWD（Data Warehouse Detail，明细层）一行一个原子事实，即清洗去重后的 document（文档）与 chunk（切片，检索的最小文本单元）；DWS（Data Warehouse Summary，汇总层）按主题聚合成统计口径；ADS（Application Data Service，应用层）是供接口直读的宽表与权限视图。

| 数仓视角 | 中台视角 | 粒度 | RAG 中的表 | 能否重建 |
| --- | --- | --- | --- | --- |
| ODS | 原始 | 文件 | raw_manifest | 否，必须保留 |
| DWD | 加工-明细 | chunk | document / chunk | 能 |
| DWS | 加工-汇总 | 知识库 × 天 | kb_daily_stat | 能 |
| ADS | 服务 | 请求 | kb_chunk_view | 能 |

原始层负责「能重来」，加工层负责「算得对」，服务层负责「查得快」。解析器升级、切分参数调整、模型换代都只从 ODS 重跑 DWD；明细表因此要带 `parser_version`、`chunker_version`、`embedding_model` 三个血缘（lineage）字段，否则说不清哪批数据由哪套参数产出。

### 4.3 五张核心表的建表语句

以 PostgreSQL 15 + pgvector 为例；kb 1:N document 1:N chunk，document 1:N document_version。

```sql
-- 知识库表
CREATE TABLE kb (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  embedding_model TEXT NOT NULL,   -- 换模型须新建库
  embedding_dim INT NOT NULL,
  chunk_size INT NOT NULL DEFAULT 512,
  chunk_overlap INT NOT NULL DEFAULT 64,
  status SMALLINT NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- 文档表
CREATE TABLE document (
  id BIGSERIAL PRIMARY KEY,
  kb_id BIGINT NOT NULL REFERENCES kb(id),
  title TEXT NOT NULL,
  source_uri TEXT NOT NULL,
  mime_type TEXT NOT NULL,
  content_hash CHAR(64) NOT NULL,  -- 规范化文本 sha256
  status SMALLINT NOT NULL DEFAULT 0, -- 0待处理 1就绪 2失败
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (kb_id, content_hash)
);
```

```sql
-- chunk 表：检索最小单元
CREATE TABLE chunk (
  id BIGSERIAL PRIMARY KEY,
  doc_id BIGINT NOT NULL REFERENCES document(id),
  kb_id BIGINT NOT NULL,
  seq INT NOT NULL,
  content TEXT NOT NULL,
  token_count INT NOT NULL,
  vector_id TEXT,                  -- 向量库主键
  meta JSONB NOT NULL DEFAULT '{}'::jsonb,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (doc_id, seq)
);
```

```sql
-- 版本表
CREATE TABLE document_version (
  id BIGSERIAL PRIMARY KEY,
  doc_id BIGINT NOT NULL REFERENCES document(id),
  version INT NOT NULL,
  content_hash CHAR(64) NOT NULL,
  storage_path TEXT NOT NULL,
  parser_version TEXT NOT NULL,
  chunker_version TEXT NOT NULL,
  status SMALLINT NOT NULL DEFAULT 0, -- 0构建中 1生效 2回滚
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (doc_id, version)
);
```

```sql
-- 采集任务表
CREATE TABLE ingest_task (
  id BIGSERIAL PRIMARY KEY,
  kb_id BIGINT NOT NULL REFERENCES kb(id),
  source_type TEXT NOT NULL,       -- url / upload / s3 / api
  idempotency_key TEXT NOT NULL,
  status SMALLINT NOT NULL DEFAULT 0, -- 0排队 1运行 2成功 3失败
  retry_count INT NOT NULL DEFAULT 0,
  error_code TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (idempotency_key)
);
```

| 错误码 | 含义 | 处理动作 |
| --- | --- | --- |
| E_PARSE_001 | PDF 加密无法解析 | 标记失败，人工补密码 |
| E_CHUNK_002 | 切片后 token 数为 0 | 跳过该文档，查清洗规则 |
| E_EMBED_429 | Embedding 服务限流 | 指数退避重试，上限 5 次 |
| E_VEC_500 | 向量库写入失败 | 回滚本批 chunk 并重试 |

E_EMBED_429、E_VEC_500 可自动重试，另两个必须人工介入。三条红线：`content_hash` 用规范化文本算，用文件字节算会因换导出工具全量重嵌入；删除先置 `is_active = false`，向量库同步后由 TTL 清理；向量维度写进 kb 表，检索前校验。

### 4.4 分区、冷热分层与 TTL

分区（Partition，把大表按键拆成物理片，查询只扫命中片）解决扫描量过大、维护窗口、过期清理三件事。chunk 与采集日志按天分区保留 180 天：ClickHouse 用 `PARTITION BY toYYYYMM(created_at)`，PostgreSQL 用声明式分区，对象存储用前缀 `raw/dt=2026-11-25/{kb_id}/` 配生命周期规则。

| 层 | 介质 | 内容 | 保留 | 延迟 | 相对成本 |
| --- | --- | --- | --- | --- | --- |
| 热 | SSD + 内存索引 | 近 30 天 chunk 与向量 | 30 天 | < 50 ms | 高 |
| 温 | 对象存储标准型 | 历史文件、DWD 明细 | 180 天 | 100 ms ～ 1 s | 中 |
| 冷 | 归档 / 低频 | ODS 原始文件、旧版本 | ≥ 1 年 | 分钟级 | 低 |

冷热分层（Hot/Cold Tiering，按访问频率把数据分成热、温、冷三层）与 TTL（Time To Live，到期自动降级或删除）配合使用：ClickHouse 用表级 TTL（`created_at + INTERVAL 180 DAY TO VOLUME 'cold'`）让数据 180 天后转冷卷、720 天后删除；S3 / MinIO 用生命周期规则（30 天转 `STANDARD_IA`、365 天删）；向量库没有原生 TTL，靠 `is_active` 过滤加每日对账求差集清理。

### 4.5 常见坑

- 把向量库当主库：只删向量不删 chunk。对账用 `SELECT count(*) FROM chunk WHERE is_active` 比对向量实体数。
- 分区键选错：按 kb_id 分区，单库文档少时每片不到 10 行，更慢还堆小文件。
- 版本表不记 `parser_version`：出问题分不清是解析器还是切分参数导致，回滚只能全量重跑。

## 5. 文档解析与清洗流水线

### 5.1 四段流水线

文档解析（Document Parsing，把 PDF、Word 等文件还原成带结构与坐标的文本）是 RAG（检索增强生成，先检索资料再让模型作答）的第一道闸门：切分与检索只能看见解析结果，原文里有、结果里没有的表格单元格，后面补不回来。清洗（Cleaning，按规则删改解析文本里的噪声与敏感信息）决定知识库里几成是知识、几成是页眉广告。

流水线四段，可分段重跑：预检（MIME、页数、加密）→ 解析 → 清洗 → 质检（打分、脱敏）。段间只传统一文档模型（Canonical Document Model，与格式无关的 JSON，记录页面、块、表格与坐标）：

```json
{"doc_id":"d_001","blocks":[{"type":"heading","text":"差旅报销","page":3},
 {"type":"table","page":3,"rows":4,"cols":3,"html":"…"}]}
```

块类型共八类（heading/paragraph/list/table/formula/image/header/footer），新增须同步改切分。

### 5.2 格式矩阵

不同格式的失败模式不同，先建矩阵再谈工具。

| 格式 | 典型难点 | 判定信号 | 处理要点 |
|---|---|---|---|
| PDF（含扫描） | 双栏交错、CID 字体缺 ToUnicode、加密 | 每页字符数 | 块坐标重排；缺映射走 OCR |
| Word（doc/wps） | 样式丢失、文本框与批注游离、修订未接受 | 扩展名/zip | 先接受修订，旧格式转 docx |
| PPT | 形状顺序不等于阅读顺序、备注页、SmartArt 是图 | shape 类型 | 按 top/left 排序，备注单独成块 |
| HTML | 导航广告混入、单页应用未渲染 | content-type | 抽正文、保标题层级、存渲染快照 |
| Markdown | front matter、代码块内的表格竖线、深层嵌套列表 | 文件头 `---` | 保护代码块；front matter 转元数据 |
| 扫描件 | 倾斜、印章遮挡、手写、低分辨率 | 整册扫描页占比 | 300 DPI + 方向检测 |
| 图片 | 无文本层、EXIF 旋转、超长图 | MIME image/* | 旋转校正；长图先切分 |

旧格式（.doc/.wps/.rtf）先转换再进 Word 分支：`soffice --headless --convert-to docx --outdir /data/conv /data/a.doc`，转完校验页数与字符量。版面分析（Layout Analysis，识别标题、正文、表格区域并给出边界框）与阅读顺序（Reading Order，决定多栏文字先后）是 PDF 解析的两步；只读文本流，双栏论文会读出「左栏一行、右栏一行」的病句。

### 5.3 工具选型

| 工具 | 擅长 | 主要限制 | 适用环节 |
|---|---|---|---|
| PyMuPDF | 快、块坐标、渲染 | AGPL；表格弱 | PDF 文本层 |
| pdfplumber | 有线表格、字符坐标 | 慢；扫描件无效 | 表格页 |
| MinerU / Marker | 版面、公式、表格转 Markdown | 需 GPU、吞吐低 | 论文、研报 |
| python-docx / python-pptx | 段落样式、备注、表格 | 文本框与图表弱 | Office 主路径 |
| trafilatura | 网页正文 | 只处理 HTML | 网页 |
| PaddleOCR / Tesseract | 中文 OCR | 复杂版式需版面 | 扫描页兜底 |

默认路由：文本层 PDF 用 PyMuPDF，表格页加 pdfplumber；扫描页 PaddleOCR；要公式与版面再上 MinerU/Marker；网页 trafilatura；Office 先读结构，复杂排版转 PDF；异构兜底 Tika，格式互转 Pandoc。

### 5.4 OCR 兜底与质量闸门

OCR（Optical Character Recognition，把页面图像中的文字转成可编辑文本）。兜底按页判定、逐级升级，不是一失败就整本送 OCR。

判定：单页文本层字符数 < 20 且有整页图；整册扫描页占比 > 0.8 才整本 OCR——只看单页会误伤封面页。质量闸门（Quality Gate，用可量化阈值决定结果能否进下游）要写成纯函数才可回归测试（Python）：

```python
def gate(p, min_chars=20, max_repl=0.005, min_conf=0.75, min_lang=0.80) -> str:
    if p.text_len < min_chars and p.has_full_image:
        return "ocr"                  # 无文本层，转 OCR
    if p.repl_ratio > max_repl:
        return "ocr_retry"            # 乱码超限，重识别
    if p.ocr_conf and p.ocr_conf < min_conf:
        return "vlm"                  # 置信度不足，升级大模型
    if p.lang_conf < min_lang:
        return "review"               # 语种可疑，人工看
    return "pass"
```

OCR 固定 300 DPI、灰度、开方向检测；分辨率不足先放大。

### 5.5 表格与公式

表格结构识别（Table Structure Recognition，还原行列、合并单元格与表头关系）最容易「看着对、用起来错」。三件套一起存：HTML 保结构、Markdown 给模型读、坐标回溯原文。跨页表按「列数一致 + 表头重复率 > 0.8」合并并补 `page_range`；合错比不合更糟。

公式识别（Formula Recognition，把公式图像转成 LaTeX）：独立公式转 `$$...$$` 并保留编号，行内转 `$...$`；置信度 < 0.6 时保留原图并标 `formula_unresolved=true`，不能让模型按错公式推理。

### 5.6 清洗规则

清洗分「删噪声」与「改内容」两类，后者必须留痕。

| 规则 | 判据 | 动作 | 留痕 |
|---|---|---|---|
| 页眉页脚 | 每页首尾 3 行中某行在 ≥50% 页面重复 | 删除 | 行数与样例 |
| 重复段落 | SimHash 汉明距离 ≤3，或 5-gram Jaccard ≥0.9 | 留首次，其余标 `duplicate_of` | 段落指针 |
| 编码异常 | UTF-8 严格解码失败，或 U+FFFD 占比 >0.5% | 重解码、ftfy 修复 | 原编码猜测 |
| 乱码 | 私用区字符占比 >0.1%，或命中「锟斤拷」「â€」 | 整页重解析 | 升级复核 |
| 隐私信息 | 正则 + NER（命名实体识别）命中手机号、身份证、邮箱、卡号 | 换不可逆替身 | 类型与映射 |

编码异常（Character Encoding，字节到字符的映射）处理顺序：UTF-8 严格解码 → GB18030 → charset-normalizer 猜测，入库统一 UTF-8 + NFC。乱码（Mojibake，编码错配产生的不可读字符）要量化，不能靠肉眼抽查：

```python
import re, hashlib

def garble_ratio(t):   # 乱码率 = 替换字符 + 私用区字符
    return sum(c == "\ufffd" or 0xE000 <= ord(c) <= 0xF8FF for c in t) / max(len(t), 1)

PII = re.compile(r"(?<!\d)(1[3-9]\d{9}|\d{17}[\dXx])(?!\d)")

def mask(t, salt="kb"):
    h = lambda m: "[PII:%s]" % hashlib.sha256((salt + m.group()).encode()).hexdigest()[:8]
    return PII.sub(h, t)   # 同一实体全文同一替身，不保留原长度
```

隐私脱敏（PII Redaction，把个人身份信息换成不可逆替身）：同一实体全库映射同一替身，否则会被检索拆成多个；替身不能保留原长度，长度本身就是信息。

### 5.7 质量打分与复核队列

解析质量分（Parse Quality Score，0~1 加权分）五维：文本完整率 0.30（解析字符/页面积估算）、乱码率 0.20、结构保留率 0.20、表格还原率 0.15、语言置信度 0.15。≥0.85 入库；0.60~0.85 抽 10%；<0.60 进人工复核队列（Human Review Queue，承接待人工判定解析结果的工单表）。连续三批通过率 >98% 抽样减半，<90% 翻倍。

```sql
CREATE TABLE parse_review_queue (doc_id TEXT PRIMARY KEY, source_uri TEXT,
  score REAL NOT NULL, reasons TEXT,          -- ["ocr_conf","table_broken"]
  status TEXT DEFAULT 'pending', assignee TEXT);
```

复核界面最小要求：原文页图与解析文本并排、一键重跑、结论回写为规则或样本。

### 5.8 常见坑

1. **只取文本不取坐标**：双栏 PDF 读出交错文本，模型只能读到病句。用块坐标重排阅读顺序。
2. **单页字符数判扫描件**：封面页误判导致整本 OCR，成本翻十倍。改看整册扫描页占比。
3. **脱敏保留长度**：等长星号等于泄露「11 位手机号」。用哈希替身。
4. **一次性全量 OCR**：先用 1% 采样摸清置信度分布，再放全量。

## 6. 切分、向量化与检索索引

检索质量在文本进入模型之前就已被决定：一段两万字的操作手册，最终只有被召回的四五个片段进入上下文。切分决定片段「能不能被找到」，向量化决定「像不像」，索引决定「多快找到」。

### 6.1 四种 chunk 策略

切分（Chunking，把长文档拆成适合检索与送入模型的小片段的过程）的产物叫块（Chunk，检索的最小单元），块的长度按 token（词元，模型长度与计费的最小单位）计。判断标准只有一条：一个块只讲一件事，脱离原文也能被读懂。

| 策略 | 做法 | 优点 | 代价 | 适用 |
|---|---|---|---|---|
| 固定长度（Fixed-size） | 按 token 数硬切，尾部重叠 | 实现最简、块长均匀、吞吐稳定 | 句子被拦腰截断，标题语境丢失 | 纯文本、日志、基线验证 |
| 按标题层级（Structure-aware） | 依 Markdown `#`、HTML `h1-h6`、Word 大纲切，标题路径写入元数据 | 边界天然、可回溯章节 | 块长从几十到几千 token 不等，长节要二次切 | 手册、规范、API 文档 |
| 语义切分（Semantic Chunking） | 相邻句向量相似度跌破阈值处断开 | 贴合话题转折 | 切分阶段就要算 embedding，成本翻倍；结果随模型版本漂移 | 长文、访谈、无标题语料 |
| 父子块（Parent-Child） | 子块约 200 token 建索引负责召回，父块 1000–2000 token 送入模型 | 精度与上下文完整兼得 | 存储放大 1.5–2 倍，需拼接去重 | 合同、法规、高价值问答 |

生产环境常用组合：先按标题层级切到「节」，节内用固定长度兜底到 400–600 token，关键文档再加父子块。无论哪种策略，都要把标题路径拼到块首（`体检报告 > 血常规 > 白细胞`），否则模型看到「见下表」「如上」时无从指代。

父子块的落地结构很小：子块表存 `chunk_id, parent_id, content, embedding`，父块表只存 `parent_id, content`；召回按子块打分，取 top-k 子块后按 `parent_id` 去重，再把父块按顺序拼成一段送入模型。不做去重的话，同一个父块的五个子块会挤占全部上下文，模型反复读到同一段话。

语义切分的可操作做法：先按句末标点切句，对每句取 embedding，算相邻句的余弦相似度，在相似度低于第 5 百分位（或绝对阈值 0.72–0.80）的位置断开；文档不足 20 句时这个统计量不稳定，直接退回按标题切。

更务实的顺序是先用固定长度跑通链路、量出 Recall@5（前 5 条召回结果中命中正解的比例）基线，再换结构切分做对比；没有基线，就无法判断复杂策略带来的是收益还是只是更贵。

### 6.2 尺寸、重叠与分隔符

分隔符（Separator，切分时优先尝试的断开标记）按从粗到细排序，逐级降级；重叠（Overlap，相邻块共享的尾部文本）用来避免关键句正好落在边界上。token（词元，模型长度与计费的最小单位）决定真实上限。

参数选取的经验值：

- 尺寸取决于「回答一个问题需要多少上下文」：FAQ 与条款 300–500 token；操作步骤 500–800；叙述性文档 800–1200。低于 200 token 信息量不足、向量高度同质；高于 1500 token 主题变杂，Recall@10（前 10 条召回结果中命中正确答案的比例）明显下滑。
- 重叠取块大小的 10%–15%，或固定 1–2 句；父子块（小块建索引召回、大块送入模型）模式中父块不需要重叠。
- 中文 token 估算：BGE 一类词表的 tokenizer 约 1 token ≈ 1.3–1.5 个汉字，英文约 1 token ≈ 4 字符。直接用字符数当 token 会低估 30% 以上，入库前必须用真实 tokenizer 复核。

| 文档类型 | 尺寸(token) | 重叠 | 分隔符优先级 |
|---|---|---|---|
| 制度/合同条款 | 400 | 10% | `\n\n` → `\n` → `第X条` → `。` |
| FAQ | 整组问答 | 0 | `\n\n` → `Q:` |
| API 文档 | 按接口 | 5% | `\n## ` → `\n` |
| 表格密集页 | 整表 | 0 | 表头随每块重复 |
| 会议纪要 | 800 | 15% | `\n## ` → `\n\n` |

调参不要凭感觉：准备 30–50 条真实问题并标注标准答案所在段落，固定 embedding 模型与 top-k，每次只改一个参数，记录 Recall@5 与送入模型的 token 总量，取「召回不低于基线、token 更省」的那组。

含代码或表格的文档要在切分前做保护：先用占位符把代码块与整张表格整体替换，切完再塞回所属块，否则代码会被切成无法运行的碎片、表格会丢掉表头。

中文标点也有讲究：右引号、右括号等收尾符号应留在切点之前，切点放在收尾符号之后，避免下一块以孤立的标点开头。

```python
def chunk(text: str, size=500, overlap=50, seps=("\n\n", "\n", "。", "")) -> list[str]:
    """按分隔符优先级切分，size 为字符上限（中文近似 token）"""
    if len(text) <= size:
        return [text.strip()] if text.strip() else []
    sep = next((s for s in seps if s and s in text), "")
    if not sep:                                        # 无可用分隔符，硬切兜底
        return [text[i:i + size] for i in range(0, len(text), size - overlap)]
    out, buf = [], ""
    for part in text.split(sep):
        if len(buf) + len(part) > size and buf:
            out.append(buf)
            buf = buf[-overlap:]                       # 回退 overlap 字符形成重叠
        buf += part + sep
    return [b for b in out + [buf] if b.strip()]
```

切完必须抽样验收：随机取 50 块，逐一判断「只给这一块能否回答一个问题」，并看块长分布的 P95 是否超过目标值的 1.5 倍。

尺寸还和 top-k 联动：块越小，同一个问题需要越多块才能拼出完整答案，k 从 5 提到 10 会让上下文成本翻倍；先把块定在能装下一个完整答案的粒度，再回头调 k。

### 6.3 向量化：模型、维度、归一化

向量化（Embedding，把一段文本映射成定长稠密向量，使语义相近的文本在空间中距离更近）。模型一旦确定，整个库的向量语义空间就确定了，替换模型等于重建全库。

| 模型 | 维度 | 最大输入 | 中文 | 部署与成本 |
|---|---|---|---|---|
| bge-m3 | 1024 | 8192 | 强，可同时出稠密/稀疏向量 | 自托管，需 GPU |
| bge-large-zh-v1.5 | 1024 | 512 | 中文强、英文一般 | 自托管，成本低 |
| text-embedding-3-small | 1536 | 8191 | 中上 | API，约 $0.02/百万 token |
| text-embedding-3-large | 3072（可截断） | 8191 | 好 | API，约 $0.13/百万 token |
| gte-Qwen2-7B-instruct | 3584 | 8192 | 第一梯队 | 自托管，显存 ≥16GB |

维度不是越高越好：1024 维已能满足多数中文知识库，3072 维带来约 3 倍的存储与距离计算开销。支持套娃表示（Matryoshka Representation，训练时让前若干维即可独立使用）的模型可以截断到 512/1024 维，但截断后要重新测召回，不能想当然。

选型按三条线走：数据不能出域就选 bge-m3（中文强、支持 8192 长输入、可同时产出稀疏向量）；只需快速验证且数据可出域，用 text-embedding-3-small 先把链路跑通；追求极致中文效果且有 GPU 预算，上 gte-Qwen2-7B 一类大模型。换模型前先在 200 条查询上比 Recall@10 与人工相关性，别只看榜单。

容量估算：1 亿条 1024 维 float32 向量约 400GB 原始数据，加上 HNSW 图结构通常再乘 1.3–1.5；改 float16 或 int8 量化能砍掉一半，代价是召回率下降 1–3 个百分点。

归一化（L2 Normalization，把向量缩放到单位长度）之后，余弦相似度与内积等价，检索端可直接用更快的内积计算：

```python
import numpy as np

def l2norm(mat: np.ndarray) -> np.ndarray:
    """入库与查询都必须走这个函数，混用未归一化向量会让排序静默错乱"""
    scale = np.linalg.norm(mat, axis=1, keepdims=True)
    return mat / np.clip(scale, 1e-12, None)       # 防零向量除零
```

pgvector 用 `vector_cosine_ops` 时由库内计算余弦，FAISS 的 `IndexFlatIP` 则要求调用方先归一化。最隐蔽的事故是「旧数据归一化、新批次忘了归一化」：不报错，只是相似度分布突然变宽，召回悄悄变差。

另一个常被忽略的点是非对称检索（Asymmetric Retrieval，查询很短、文档块很长的检索场景）：查询通常只有十几个字，正文块有几百字，长度差异过大会让相似度整体偏低。部分模型要求用 `query:` 与 `passage:` 前缀区分编码方式，查询与文档必须遵守同一约定，混用会让召回率掉一截。

### 6.4 批量、限流与成本

批量（Batching）指一次请求提交多条文本。单请求取 16–64 条或 20 万 token 以内；OpenAI 兼容接口单请求最多 2048 条输入、单条不超过模型上限。并发从 4 起步，按 429 出现率上调。

成本（Cost）公式：`费用 = 总 token / 10^6 × 单价`，`总 token = 文档数 × 平均块数 × 块 token × (1 + 重叠率)`。十万篇文档、每篇 20 块、每块 400 token，约 8 亿 token，用 text-embedding-3-small 约 16 美元；自托管则换算 GPU 小时。

吞吐估算：单张 A10 跑 bge-large，512 token 以内的块约每秒 200–400 条，十万篇文档按 200 万块算需要 1.5–3 小时；走 API 则由并发与限流决定，先按 20 QPS 估时间。

断点续跑靠状态表而不是内存：每批写完把 `status` 置为 `indexed` 并记下 `last_chunk_no`，进程崩溃后从该位置继续，避免重复计费。

| 现象/错误码 | 含义 | 处置 |
|---|---|---|
| 429 | 触发速率限制 | 指数退避 + 随机抖动，并发减半 |
| 400 invalid input | 空块或超长文本 | 入库前过滤空块、按 tokenizer 截断 |
| 413 payload too large | 单请求体过大 | 减小批大小与 `max_tokens` |
| 401 | 密钥失效 | 检查环境变量，不要硬编码 |
| 5xx | 服务端临时故障 | 重试 3 次后进入死信队列（Dead Letter Queue，失败任务暂存表）待补 |
| 维度不匹配 | 向量长度与表定义不符 | 建表时写死维度，入库前断言 |

以内容哈希做缓存键，重跑流水线时同一条 chunk 不再重复付费，这也是增量更新的基础。

```python
import hashlib, time

def batch_embed(texts, embed_fn, batch=32, retries=3):
    seen = {}
    todo = [(hashlib.sha256(t.encode()).hexdigest(), t) for t in texts]
    todo = list(dict.fromkeys(todo))                   # 按内容哈希去重
    for i in range(0, len(todo), batch):
        part = todo[i:i + batch]
        for n in range(retries):
            try:
                vecs = embed_fn([t for _, t in part])
                seen.update({h: v for (h, _), v in zip(part, vecs)})
                break
            except Exception:
                time.sleep(2 ** n)                     # 指数退避；耗尽后该批留给死信队列
    return [seen[hashlib.sha256(t.encode()).hexdigest()] for t in texts]
```

### 6.5 向量库选型

向量数据库（Vector Database，专门存储向量并支持近似最近邻查询的系统）。核心索引是 HNSW（Hierarchical Navigable Small World，分层可导航小世界图，用图结构换取查询速度）与 IVF（Inverted File，用聚类倒排缩小候选集）。

| 对比项 | Milvus | pgvector | Elasticsearch | Qdrant |
|---|---|---|---|---|
| 定位 | 专用分布式向量库 | PostgreSQL 扩展 | 搜索引擎内置向量检索 | 轻量向量库 |
| 规模 | 亿级以上 | 千万级较稳 | 亿级 | 千万至亿级 |
| 混合检索 | 稠密+稀疏+标量过滤 | SQL 过滤，关键词需另拼 tsvector | 原生 BM25 + kNN，含 RRF 融合 | 稀疏向量 + payload 过滤 |
| 运维成本 | 高（需 etcd、对象存储） | 低（复用现有 PG） | 中高 | 中低 |
| 适用 | 多租户、超大规模 | 已有 PG、数据量中等 | 已有 ES、关键词诉求强 | 中小团队快速上线 |

参数经验：HNSW 的 `M=16`、`efConstruction=200`、查询期 `ef_search=64–128`，`M` 与 `ef` 越大召回越高、内存与建索引时间越久；IVF 的 `nlist≈4√N`、`nprobe` 取 1%–5%。

选型不必追最强：数据量低于 1000 万且已有 PostgreSQL，pgvector 的运维优势远大于性能差距；已经在用 Elasticsearch 做站内搜索，加一个 `dense_vector` 字段就能同时拿到 BM25 与向量；需要多租户隔离和亿级规模再上 Milvus。

落库前做一次百万向量压测：固定查询集，测 top-100 的 P99 延迟、带租户过滤后的召回率、写入十万条后的索引膨胀，这三个数字比参数表更能说明问题。

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE kb_chunk (
  id          bigserial PRIMARY KEY,
  doc_id      text NOT NULL,
  version     int  NOT NULL,                 -- 与文档版本对齐，便于回滚
  content     text NOT NULL,
  embedding   vector(1024) NOT NULL,         -- 维度必须与模型一致
  tenant_id   text NOT NULL,
  updated_at  timestamptz DEFAULT now()
);
CREATE INDEX idx_chunk_vec ON kb_chunk
  USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);
SET hnsw.ef_search = 100;                    -- 会话级参数，按召回率调
```

选型时的三个硬约束：pgvector 的 HNSW 索引对 `vector` 类型上限 2000 维（`halfvec` 为 4000 维），3072 维模型必须降维；带过滤条件的近似查询可能召回不足，pgvector 需开启迭代扫描或按租户建部分索引，Qdrant 要先给过滤字段建 payload 索引；Milvus 用 partition key 做租户隔离，比全局过滤更稳。

### 6.6 混合检索与 rerank

BM25（Best Matching 25，基于词频与逆文档频率的关键词打分算法）负责精确匹配编号、人名、条款号；向量检索负责同义改写与跨语言。两者互补，混合检索（Hybrid Retrieval，同时执行关键词与向量两路召回并合并结果）几乎总比单路好。

BM25 的两个参数：`k1` 控制词频饱和（默认 1.2），`b` 控制文档长度惩罚（默认 0.75）；块长差异大时把 `b` 降到 0.5，能减少长块霸榜。

RRF（Reciprocal Rank Fusion，按名次倒数和融合多路结果）不需要两路分数同量纲，是最省事的融合方式：

```python
def rrf(rank_lists, k=60, topn=20):
    """rank_lists 中每个列表已按各自分数降序；k 取 60 是经验值"""
    score = {}
    for lst in rank_lists:
        for rank, doc_id in enumerate(lst, start=1):
            score[doc_id] = score.get(doc_id, 0.0) + 1 / (k + rank)
    return sorted(score.items(), key=lambda x: -x[1])[:topn]
```

k 取 60 的作用是压平名次差距：k 越小，靠前的名次越占绝对优势。只有两路召回、且两路质量相近时，60 这个默认值基本够用，融合后截断到 50 条交给重排即可。

重排（Rerank，用交叉编码器 Cross-Encoder 对「查询-文档」对逐条打分再排序）精度高于向量召回，但成本随候选数线性增长，只对融合后的 30–50 条候选执行。

rerank 不是必选项：候选只有十几条、或端到端延迟要求 200ms 以内时，它带来的收益常小于延迟代价，此时优先缩小融合后的 top-k，并把标题路径拼进待排序文本。

查询改写（Query Rewrite，把口语化提问补全成可检索的检索式）在混合检索里收益明显：多轮对话中「那它的保修期呢」必须先补全为「X200 笔记本的保修期」，否则关键词与向量两路都会跑偏。

| 阶段 | 候选数 | 典型耗时 | 备注 |
|---|---|---|---|
| BM25 召回 | 100 | 20–50ms | 中文需 IK/jieba 分词，按字切会引入噪声 |
| 向量召回 | 100 | 5–30ms | 受 `ef_search` 与过滤条件影响 |
| RRF 融合 | 200 → 50 | <5ms | 只用名次，不受分数尺度影响 |
| Rerank | 50 → 8 | 100–500ms | 整条链路的主要延迟来源 |
| 生成 | 8 | 秒级 | 上下文按 8 块 × 500 token 预留 |

若改用加权分数融合（`α × 向量分 + (1-α) × BM25 分`），必须先把两路分数 min-max 归一化，且对异常分值敏感；建议先上 RRF，有评测集后再调权重。

### 6.7 增量更新、索引重建与灰度切换

增量更新（Incremental Update，只对新增与变更文档重跑切分和向量化）依赖变更检测：对清洗后的正文做 sha256，哈希未变即跳过，避免整库重刷。删除要用软删标记加定期物理清理，Milvus 与 Qdrant 的删除是逻辑标记，不 compaction 不释放空间，被删数据短期内仍可能被召回。

索引重建（Reindex，在新集合中重建全部向量）的触发条件：更换 embedding 模型、修改切分参数、向量维度变化、库大版本升级。重建绝不能原地覆盖，正确姿势是建新集合、灌数据、校验、切别名。

增量与全量要能随时互切：日常走增量，每周做一次全量重建，比对块数与抽样召回，一旦发现漂移（删除未清理、切分版本不一致）就以全量结果为准。

回滚路径必须演练：切别名前先在测试环境跑一次回滚，确认旧索引仍可写、切回不产生 5xx；没演练过的回滚路径等于没有。

| 状态 | 含义 | 下一步 |
|---|---|---|
| pending | 待处理 | 拉取原文 |
| chunked | 切分完成 | 批量向量化 |
| indexed | 已写入新集合 | 抽样校验 |
| active | 别名已指向 | 对外服务 |
| failed | 超重试次数 | 进死信队列人工介入 |

```sql
-- 同一文档串行推进：只有 active 状态才能开启新版本，避免并发写乱序
UPDATE kb_doc SET status = 'chunked', version = version + 1, retry = 0
WHERE doc_id = $1 AND status IN ('active', 'failed');
```

灰度切换（Canary Switchover，先让小比例流量走新索引，指标达标后全量）用别名（Alias，指向具体索引的逻辑名称）完成，回滚只需把别名指回旧集合。以 Elasticsearch 为例：

```json
POST /_aliases
{"actions": [
  {"remove": {"index": "kb_chunk_v3", "alias": "kb_chunk"}},
  {"add":    {"index": "kb_chunk_v4", "alias": "kb_chunk"}}
]}
```

切换前做影子双读：抽 200 条真实查询，分别打新旧索引，比较 Recall@10（前 10 条召回结果中命中正解的比例）与人工相关性标注，差异小于 2% 且 P99 延迟不劣化才切；切后保留旧索引 7 天，确认无回滚需求再删除。

### 6.8 速查表

| 症状 | 先查什么 | 处置 |
|---|---|---|
| 明明有答案却召回不到 | 块是否过大、是否丢了标题路径 | 缩到 400–600 token，块首补标题 |
| 分数普遍 0.8 以上但答非所问 | 是否混入未归一化向量、模型版本是否一致 | 统一归一化，固定模型版本 |
| 编号、型号搜不到 | 是否只走向量单路 | 加 BM25 与 RRF 融合 |
| 更新文档后仍是旧答案 | 别名指向、embedding 缓存、软删未清理 | 切别名、清缓存、执行 compaction |
| P99 延迟超 1 秒 | rerank 候选数与 `ef_search` | 候选降到 30，`ef_search` 降到 64 |

## 7. 知识库与语料管理

### 7.1 先把四个概念钉死

知识库（Knowledge Base，围绕某一业务主题组织、可被检索的文档集合）不是“一个文件夹”。它有边界（谁能读）、有版本（哪一版在线上）、有权限（谁可写）、有生命周期（何时失效）。语料（Corpus，进入知识库的原始文档及其派生分块）是它的内容物。租户（Tenant，共享一套系统但数据必须互相隔离的客户组织）是权限与计费的边界。索引（Index，为检索而生的副本，含向量与倒排表）可随时重建，它只是副本。

```json
{
  "kb_id": "kb_finance_cn", "tenant_id": "t_acme",
  "visibility": "tenant_private", "active_version": "v2024_11_05",
  "version_policy": { "keep_versions": 20, "gray_percent": 10 }
}
```

`tenant_id` 是所有隔离逻辑的强制字段，缺一即越权；`kb_id` 必须可读、可排序、不可变。

### 7.2 多知识库与多租户模型

一个租户可有多个知识库，一个知识库也可被多租户共享（如公共法规库）。隔离只有三种模型：

| 模型 | 做法 | 隔离强度 | 跨库检索 | 成本 | 适用规模 |
|---|---|---|---|---|---|
| 逻辑隔离 | 共享表与索引，每行带 `tenant_id` | 低（靠代码守规矩） | 容易 | 最低 | 百级租户 |
| 物理隔离 | 每租户独立 schema/collection | 高 | 需聚合层 | 高 | 大客户、强合规 |
| 混合 | 默认共享，白名单租户独立 | 中 | 较易 | 中 | 多数生产系统 |

无论哪种，检索入口都必须强制注入租户条件，不能信任调用方传参：

```python
def build_filter(tenant_id: str, kb_ids: list[str], f: dict) -> dict:  # 服务端拼接，客户端条件只收窄
    assert tenant_id, "tenant_id 为空即拒绝，不做兜底"
    return {"must": [
        {"key": "tenant_id", "match": {"value": tenant_id}},  # 强制行级隔离
        {"key": "kb_id", "match": {"any": kb_ids}},
        {"key": "lifecycle", "match": {"value": "active"}},   # 只召回生效语料
    ], "client": f}
```

### 7.3 版本、发布、回滚与灰度

版本（Version，语料在某一时刻的不可变快照）；发布（Publish，把某版本设为线上实际使用的版本）；别名（Alias，指向具体版本的稳定指针，如 `prod`）；灰度（Canary Release，让一小部分流量先走新版本以观察效果）。

核心设计只有一条：**线上流量永远读别名，别名指向版本，回滚就是改指针**。快照不可变，回滚是 O(1) 操作。

| 策略 | 切换方式 | 风险 | 回滚耗时 |
|---|---|---|---|
| 全量发布 | 别名直接指向新版本 | 高 | 秒级（改指针） |
| 蓝绿 | 新版本建好再切 | 中 | 秒级 |
| 灰度 | 按流量比例双读 | 低 | 秒级 |

```bash
kbctl publish --kb kb_finance_cn --version v2024_11_05 --alias canary --ratio 10
kbctl alias set --kb kb_finance_cn --alias prod --version v2024_11_05 --require-eval-pass
kbctl rollback --kb kb_finance_cn --alias prod   # 指针退回上一版本，不重建索引
```

`--require-eval-pass` 要求新版本召回率与引用准确率不低于旧版本，否则拒绝发布。

### 7.4 权限与数据隔离

权限分三层：库级（能否访问该知识库）、行级（能否看到符合条件的那部分文档）、字段级（敏感字段是否脱敏）。前两层靠 RBAC（Role-Based Access Control，基于角色的访问控制），行级靠 ABAC（Attribute-Based Access Control，基于属性的访问控制）或标签过滤。

| 角色 | 读语料 | 写语料 | 发布 | 回滚 | 看引用原文 |
|---|---|---|---|---|---|
| viewer | ✅ | ❌ | ❌ | ❌ | 仅摘要 |
| editor | ✅ | ✅ | ❌ | ❌ | ✅ |
| publisher | ✅ | ✅ | ✅ | ✅ | ✅ |
| auditor | ✅ | ❌ | ❌ | ❌ | ✅（含审计日志） |

常见漏洞是“检索后才过滤”：先查 Top-50 再剔除无权限文档，既浪费算力，也会因结果剔空而泄露“存在但无权看”的信息。应对是预过滤（Pre-filter），把权限条件放进检索请求。

### 7.5 引用溯源与答案落点

引用（Citation，答案中每个事实性陈述所对应的原文片段）；落点（Anchor，引用在原文中的精确定位：文档 ID、版本、块序号、字符区间）。没有落点的引用无法校验。

```json
{
  "doc_id": "doc_8821", "doc_version": "v2024_11_05", "chunk_id": "doc_8821#p12",
  "span": { "start": 340, "end": 402 }, "score": 0.83,
  "quote": "报销单据须在费用发生后 30 日内提交。"
}
```

`doc_version` 是必需字段：语料更新后若只记 `doc_id`，落点会指向新文本的同一区间，出现“引用与原文不符”的错位。

```python
def verify(citations, store):  # 答案落地前校验，不通过则整条降级为"未找到依据"
    for c in citations:
        text = store.get_span(c["doc_id"], c["doc_version"], c["span"])
        if text is None or c["quote"] not in text:
            return False   # 版本已清理或区间漂移
    return True
```

### 7.6 文档失效、下线与更新的传播

传播（Propagation，源文档变更后驱动下游索引、缓存与已生成答案同步失效的链路）。三层都要做，漏一层即出现“文档已删、答案还在”：

| 层 | 失效对象 | 手段 | 时效 |
|---|---|---|---|
| 索引层 | 向量与倒排条目 | 变更事件增量重建 | 分钟级 |
| 缓存层 | 检索结果、答案缓存 | 按 `doc_id` 反查删除 | 秒级 |
| 答案层 | 历史会话引用 | 展示时校验并提示已更新 | 展示时 |

```python
def offline(doc_id: str):  # 先标记再清理，避免检索读到半删状态
    db.execute("UPDATE docs SET lifecycle='offline' WHERE doc_id=%s", (doc_id,))
    cache.invalidate_tag(f"doc:{doc_id}")             # 秒级清缓存
    queue.publish("doc.offline", {"doc_id": doc_id})  # 异步删索引，幂等
```

| 错误码 | 含义 | 排查方向 |
|---|---|---|
| `KB_VERSION_NOT_FOUND` | 别名指向的版本已归档 | `keep_versions` 是否过小 |
| `CITATION_SPAN_DRIFT` | 引用区间与原文不匹配 | 是否漏记 `doc_version` |
| `TENANT_FILTER_MISSING` | 检索请求无租户条件 | 入口未强制注入 |
| `DOC_STILL_RETRIEVABLE` | 下线后仍被召回 | 删除事件未消费 |

### 7.7 语料生命周期管理

状态机固定为五态：入库（Ingest，已接收待处理）→ 生效（Active，可被检索）→ 过期（Expired，保留但默认不召回）→ 归档（Archived，冷存储，仅审计可查）→ 清除（Purged，物理删除）。状态单向前进，回退须审批留痕。

| 迁移 | 触发条件 | 是否可召回 | 保留期 |
|---|---|---|---|
| 入库 → 生效 | 解析、切分、向量化全部成功 | 是 | — |
| 生效 → 过期 | 超过 `valid_until` 或源文档变更 | 否（可配置） | 90 天 |
| 过期 → 归档 | 过期满保留期 | 否 | 1 年 |
| 归档 → 清除 | 合规到期或主动申请 | 否 | 永久 |

```sql
-- 每日扫描：把到期语料推入过期态，保留审计痕迹
UPDATE docs
SET    lifecycle = 'expired', expired_at = now()
WHERE  lifecycle = 'active'
  AND  valid_until < now()
  AND  tenant_id = :tenant_id;   -- 按租户分批，避免长事务锁表
```

批量操作要限速：百万行一次扫完会拖垮线上检索，按 5000 行分批跑。

### 7.8 常见坑速查

- **只做库级权限，没做文档级**：同一库含多部门文档，全量召回即越权。行级过滤必须写进检索条件。
- **发布原地覆盖同名索引**：新版建一半失败，线上新旧皆失。永远新建快照、改别名。
- **引用不记版本**：语料更新后区间漂移，答案与原文对不上。
- **下线只删索引不删缓存**：缓存 TTL 数小时，文档已删仍被引用。

## 8. 会话轨迹与数据回流

### 8.1 Trace 与 Span：一次请求是一棵树

用户问「上季度华东区退货率」，Agent 内部是：判定意图 → 检索指标口径 → 生成 SQL → 执行报错 → 修正重试 → 组织答案。任一环出错，用户只看到「答得不对」。

会话轨迹（Trace，一次用户请求从进入系统到返回结果所引发的全部推理步骤与工具调用的完整记录）是这条链路的黑匣子，由多个跨度（Span，轨迹内可独立计时、独立成败的最小单元，一次模型或工具调用各算一个）组成，父子关系用 parent_span_id 串成树。

一条真实 span：`{"span_id":"d4","parent_span_id":"a1","span_kind":"tool","name":"sql_exec","duration_ms":3100,"error_code":"E1005"}`，它挂在 chat.turn 根 span 下，与 answer span 一起回答三问：慢在哪（占 82% 耗时）、错在哪（E1005）、贵在哪（412 token）。

### 8.2 trace/span 字段模型

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| trace_id | string | 32 位十六进制，一次请求唯一 |
| span_id | string | 16 位十六进制，轨迹内唯一 |
| parent_span_id | string/null | 根 span 为 null |
| span_kind | 枚举 | agent / llm / tool / retriever / guard |
| name | string | 稳定英文名，不拼参数 |
| user_id_hash | string | HMAC 后的用户标识，不存明文 |
| duration_ms | uint32 | 含重试的总耗时，另记 ttft_ms |
| status | 枚举 | ok / error / timeout / cancelled |

### 8.3 埋点字段与错误码

| 分组 | 字段 | 说明 |
| --- | --- | --- |
| 输入 | input_messages | 完整 messages，含 system |
| 上下文 | retrieved_chunks | chunk_id、doc_id、score |
| 工具 | tool_name、tool_args、tool_result | 存原文，另记 result_bytes |
| 度量 | ttft_ms、duration_ms、input/output/cached_tokens | 缓存命中单独统计 |
| 错误 | prompt_version、error_code、error_msg | 版本缺失则回放不可比 |

硬规则：键必须存在，缺键会让整批写入失败；tool_result 超 8KB 只留首尾各 2KB 并置 truncated=true；存原始值不做格式化，回放与训练才能原样还原。

| 错误码 | 含义 | 可重试 | 处置 |
| --- | --- | --- | --- |
| E1001 | CONTEXT_OVERFLOW 超窗 | 否 | 截断或降 top_k |
| E1002 | LLM_RATE_LIMIT 429 | 是 | 指数退避 3 次 |
| E1003 | LLM_TIMEOUT 首 token 超时 | 是 | 降级小模型 |
| E1004 | TOOL_TIMEOUT 工具超时 | 是 | 切备用数据源 |
| E1005 | TOOL_ARGS_INVALID 参数不合 schema | 是 | 回灌报错重试一次 |

### 8.4 埋点实现（Python）

父子关系靠上下文变量传递，异步任务里也不会串线。

```python
# 语言：Python 3.11
import contextvars, uuid
from contextlib import contextmanager

_ctx = contextvars.ContextVar("span", default=None)

@contextmanager
def span(kind, name, **attrs):
    p = _ctx.get()
    rec = {"span_id": uuid.uuid4().hex[:16], "span_kind": kind, "name": name,
           "parent_span_id": p["span_id"] if p else None, "attrs": attrs}
    tok = _ctx.set(rec)            # 进入子 span
    try:
        yield rec
        rec["status"] = "ok"
    except Exception as e:
        rec.update(status="error", code=getattr(e, "code", "E2001"))
        raise                      # 埋点不能吞异常
    finally:
        _ctx.reset(tok)            # 回到父上下文
        emit(rec)                  # 脱敏后入队
```

### 8.5 脱敏与采样

脱敏做在采集侧出口，原文不落盘；规则按「先长后短」执行，避免身份证被手机号规则切碎。

| 类型 | 正则 | 替换示例 |
| --- | --- | --- |
| 手机号 | (1[3-9]\d)\d{4}(\d{4}) | 138****5678 |
| 身份证 | (\d{6})\d{8}(\d{3}[\dXx]) | 310101********123 |
| API Key | sk-[A-Za-z0-9]{16,} | sk-*** |

保格式替换让模型仍能判断字段类型却无法反查；邮箱、人名、地址靠词典与模型识别补。

采样分头采样（Head Sampling，入口按 trace_id 哈希决定是否记录整条轨迹，成本低但看不到结果）与尾采样（Tail Sampling，先缓冲整条轨迹，结束后按状态与耗时决定，能定向留下问题样本）。落地：错误、负反馈、慢于 P95（95 分位耗时）的请求 100% 保留，普通请求留 1%~5%；负反馈常在下一轮出现，热数据先落盘 24 小时再回捞补标。

### 8.6 从轨迹到评测集与微调集

| 目标 | 筛选条件 | 加工要点 | 规模 |
| --- | --- | --- | --- |
| 评测集 | 人工确认、强负反馈、边界样本 | 补 expected 与判定方式（精确匹配 / LLM 裁判 / 工具断言） | 200 ~ 2000 条 |
| SFT 集 | status=ok、工具调用完整、未点踩 | 多步 span 合并成 messages，保留 tool_calls | 500 ~ 50000 条 |
| 偏好集 | 同一 query 有 chosen 与 rejected | 配对并记录两版 prompt_version | 1000 ~ 20000 对 |

SFT（Supervised Fine-Tuning，监督微调）样本要还原成 system → user → assistant(tool_calls) → tool → assistant，失败重试的 span 必须剔除。线上轨迹不能直接当评测集：答案会过时、问题偏简单会让分数虚高，必须人工抽检并冻结版本。

### 8.7 存储与查询：ClickHouse

轨迹是写多读少、按 trace_id 取整棵树、到期即删的宽表，列存 + 高压缩 + TTL（Time To Live，数据自动过期）的 ClickHouse 正好匹配。

```sql
CREATE TABLE agent_spans (
  tenant_id String, trace_id String, span_id String, parent_span_id String,
  span_kind LowCardinality(String), name LowCardinality(String),
  start_time DateTime64(3), duration_ms UInt32, status LowCardinality(String),
  error_code LowCardinality(String),
  attributes String CODEC(ZSTD(3)), payload String CODEC(ZSTD(3))
) ENGINE = MergeTree
PARTITION BY toYYYYMMDD(start_time)
ORDER BY (tenant_id, trace_id, start_time)  -- 前缀命中即可秒回整棵树
TTL start_time + INTERVAL 90 DAY;           -- 90 天后自动清理
```

```sql
-- 还原一条轨迹：走主键前缀，毫秒级
SELECT span_id, parent_span_id, name, duration_ms, status, error_code
FROM agent_spans WHERE tenant_id = 't-01' AND trace_id = '4bf92f35...'
ORDER BY start_time;
```

别写 SELECT *，JSON 列比数值列大几十倍；跨天聚合走物化视图。

### 8.8 常见坑

- **按 span 采样**：父子节点被拆散，只剩半棵树无法回放，采样必须在 trace 入口一次性决策。
- **只记最终答案、不记中间 messages**：tool_calls 的入参出参全丢，SFT 数据直接作废。
- **先落明文再「事后脱敏」**：备份与副本里仍残留个人身份信息（PII，Personally Identifiable Information），脱敏须在 SDK 出口完成。
- **时间戳用客户端时间**：多机时钟漂移会让 span 排序错乱，因果链读出来是反的，统一服务端 UTC 毫秒。

## 9. 数据集与标注管理

### 9.1 四个概念先钉死

数据集（Dataset，为同一用途、遵循同一 schema 与同一版本策略的样本集合）不是「一个 Excel」。它同时约束四件事：字段结构、标签体系、版本策略、访问权限。样本（Record，数据集中的最小单元，一条输入加一份标注）用稳定 ID 寻址；标注（Annotation，人对样本给出的目标输出或判定结论）与样本分离存放，一条样本可挂多份标注；标注任务（Task，一次下发给标注员的批次，绑定规范版本、配额与验收标准）是把样本变成标注的运输工具。

```json
{
  "record_id": "rec_7f3a91", "dataset_id": "ds_toolcall_prod",
  "input": {"messages": [{"role": "user", "content": "查一下订单 A102 的物流"}]},
  "annotation": {"label": "tool_call", "value": {"name": "get_logistics", "arguments": {"order_id": "A102"}}},
  "meta": {"source": "trace", "trace_id": "4bf92f35", "guideline_version": "g1.3", "labeler": "u_204"}
}
```

`record_id` 必须全局唯一且永不复用；`guideline_version` 缺省时，这份标注在半年后无法解释，争议时也无从复盘。

### 9.2 数据集版本与冻结

版本（Version，数据集在某一时刻的不可变内容快照）与「每次导出加个日期」的区别只有一条：版本由内容哈希决定，内容不变则哈希不变，内容微改则必然产生新版本。冻结（Freeze，把版本置为只读并登记引用，任何写入被拒绝）是评测结果可复现的前提——同一条评测命令在 v3 上跑出 0.81、在 v4 上跑出 0.79，只有版本冻结过，这个差值才值得分析。

| 状态 | 可写 | 可被训练/评测引用 | 允许操作 |
| --- | --- | --- | --- |
| draft | 是 | 否 | 增删样本、改标签 |
| frozen | 否 | 是 | 只能派生新版本 |
| deprecated | 否 | 否（存量可读） | 标记替代版本、归档 |
| purged | 否 | 否 | 仅合规删除留墓碑 |

```python
# 语言：Python 3.11
import hashlib, json

def freeze(ds_id: str, record_ids: list[str]) -> str:
    rows = db.fetch_labels(record_ids)                # 取样本与标注的规范化 JSON
    body = "\n".join(sorted(
        f"{r['record_id']}:{hashlib.sha256(json.dumps(r['label'], sort_keys=True, ensure_ascii=False).encode()).hexdigest()}"
        for r in rows))
    digest = hashlib.sha256(body.encode()).hexdigest()[:12]   # 只对内容做哈希，与时间无关
    ver = f"{ds_id}@v{digest}"
    db.insert_version(ver, record_count=len(rows), content_hash=digest, state="frozen")
    db.bump_ref(ver)                                  # 引用计数 +1，防误删
    return ver
```

两条硬规则：冻结版本禁止原地修改，要改就派生新版本并重跑评测；删除必须查引用计数，被线上模型或评测报告引用的版本只能 `deprecated`，不能物理删。

### 9.3 六类标注任务与数据结构

| 任务类型 | 输入 | 输出 | 验收指标 | Agent 中的用途 |
| --- | --- | --- | --- | --- |
| 分类 Classification | 一段文本或一次会话 | 单个/多个标签 | 准确率、Kappa | 意图路由、风险分级 |
| 抽取 Extraction | 文本 + 字段定义 | 字段值 + 字符区间 | 字段级 F1 | 抽参数、抽实体 |
| 问答对 QA Pair | 问题 + 语料 | 参考答案 + 依据块 | 答案正确率、引用命中 | 评测检索与生成 |
| 偏好排序 Preference | 同一问题多个回答 | chosen/rejected 或 1~5 级 | 成对一致率 | 奖励模型、DPO |
| 评测集构造 Eval Set | 业务场景清单 | 期望输出 + 判定方式 | 覆盖率、难度分布 | 上线门禁 |
| 工具调用 Tool Call | 用户请求 + 工具 schema | 函数名 + 参数 + 断言 | 参数级准确率 | 工具使用是否正确 |

抽取任务必须存字符区间而非只存字符串：`{"field":"amount","text":"3,200 元","span":[128,136]}`，只存值会在文本改版后静默错位。工具调用标注要区分「参数对不对」与「该不该调用」：

```json
{"expected": {"name": "get_logistics", "arguments": {"order_id": "A102"}},
 "assertions": [{"path": "$.order_id", "op": "eq", "value": "A102"}],
 "negative_case": false}
```

`negative_case: true` 表示该请求本不应调用工具（如信息不足需反问），这类负样本比正样本更值钱，却最常被漏标。评测集则要写清判定方式：精确匹配、LLM 裁判还是工具断言，三者混用会让分数无法横向比较。

### 9.4 标注流程与工具集成

流程（Workflow，样本从入库到可用的状态流转）固定五态：待标 pending → 标注中 in_progress → 待审 review → 通过 accepted / 打回 rejected。打回必须带原因码，样本回到 pending 而不是 in_progress，避免同一人反复自审。

| 状态 | 责任人 | SLA | 不满足时 |
| --- | --- | --- | --- |
| pending | 调度 | 24h 内分发 | 触发告警 |
| in_progress | 标注员 | 单条 ≤ 3 min | 拆分任务 |
| review | 审核员 | 48h | 升级给仲裁 |
| accepted | 系统 | 即时入库 | — |
| rejected | 标注员 | 重标 ≤ 24h | 累计 3 次转专家 |

与标注工具（Label Studio、Argilla、自研平台）集成的关键是幂等：以工具侧 `annotation_id` 为唯一键写入，重复回调只更新不追加。

```python
# 语言：Python 3.11（FastAPI 回调接收）
@app.post("/hooks/label-studio")
def on_annotation(payload: dict):
    ann = payload["annotation"]
    row = {"annotation_id": ann["id"], "task_id": payload["task"]["id"],
           "labeler": ann["completed_by"], "value": ann["result"],
           "guideline_version": payload["task"]["meta"]["guideline_version"]}
    db.upsert("annotations", row, key="annotation_id")   # 幂等：重放不产生重复样本
    if payload["action"] == "updated":
        db.bump_task_revision(row["task_id"])            # 版本号递增，供抽检定位
    return {"ok": True}
```

工具只负责收集，验收标准必须留在自己系统里：工具里的「提交」不等于「合格」。

### 9.5 标注规范编写

规范（Guideline，把标签定义、判定顺序、边界处理写成可执行判据的文档）的质量直接决定 Kappa 上限。一份能用的规范至少六节：适用范围、标签定义（每标签各写 2 个正例 2 个反例）、判定优先级、边界与弃标条件、常见歧义问答、变更记录。

| 项目 | 差写法 | 好写法 |
| --- | --- | --- |
| 标签定义 | 「回答不相关」 | 「回答未包含问题所问字段，或引用块不含答案」 |
| 优先级 | 未写 | 「同时命中退款与投诉时，判投诉」 |
| 弃标 | 未写 | 「语料缺页、问题含无法解析的指代，选 skip」 |

规范必须版本化（如 `g1.3`），任务与标注都记录该字段；规范改动超出措辞范围时，旧标注不能直接复用，需按新规范重标 5% 并对比差异。把判定优先级写成代码，还能在标注前自动预判，只把冲突样本推给人。

### 9.6 一致性抽检与 Kappa

一致性（Inter-Annotator Agreement，多个标注者对同一批样本判定的一致程度）衡量的是「规范是否可执行」，不是「标注员是否努力」。做法：每个任务随机抽 5%~10% 双盲交叉标注，另埋 2% 黄金题（Golden Question，答案已知的样本，用于识别敷衍标注）。

Cohen's Kappa（科恩卡帕，扣除随机一致后的真实一致度）适用于两名标注者、同一组类别：κ = (Po − Pe) / (1 − Pe)，Po 为观测一致率，Pe 为按各类别边际分布算出的期望一致率。

```python
# 语言：Python 3.11
from sklearn.metrics import cohen_kappa_score
import numpy as np

def qc(a: list[str], b: list[str]) -> dict:
    po = np.mean([x == y for x, y in zip(a, b)])            # 观测一致率
    k = cohen_kappa_score(a, b)                             # 类别不平衡时 κ 会偏低
    n = len(a)
    boot = [cohen_kappa_score(*zip(*[(a[i], b[i]) for i in np.random.randint(0, n, n)]))
            for _ in range(1000)]
    return {"po": round(po, 3), "kappa": round(k, 3),
            "ci95": [round(np.percentile(boot, 2.5), 3), round(np.percentile(boot, 97.5), 3)]}
```

| κ 区间 | 解读 | 处置 |
| --- | --- | --- |
| ≥ 0.80 | 几乎完全一致 | 可入库 |
| 0.60 ~ 0.80 | 中等一致 | 复核分歧样本，补规范 |
| 0.40 ~ 0.60 | 一般 | 停标，重写规范后重标 |
| < 0.40 | 差 | 标签体系本身有问题 |

三个必看补充指标：原始一致率 Po（κ 悖论：某一类占 95% 时 Po 很高而 κ 很低，改用 PABAK = 2Po − 1）；各类别单独 κ（总体达标但某个小类为负，是真正的坑）；黄金题通过率（低于 90% 说明有人刷量）。抽取类用字段级 F1 取代 κ，偏好排序用成对一致率。

### 9.7 数据集血缘与可追溯

血缘（Lineage，样本从来源到最终消费方的完整链路记录）回答三个问题：线上模型用的是哪版数据、每条样本从哪来、出问题回滚到哪。链路是五级：样本 → 批次 → 版本 → 训练/评测实验 → 线上模型。

| 级别 | 主键 | 必记字段 |
| --- | --- | --- |
| 样本 | record_id | source_type、trace_id/doc_id、created_by |
| 批次 | batch_id | task_id、guideline_version、labeler、reviewer |
| 版本 | dataset_version | content_hash、record_count、frozen_at |
| 实验 | run_id | code_commit、超参、eval_report_url |
| 线上 | model_version | 灰度比例、上线时间、回滚目标 |

血缘边（Edge，两个实体之间的有向关系）单独存表，反查即可从线上模型追到原始轨迹：

```sql
-- 语言：PostgreSQL 14+，从线上模型反查数据来源
WITH RECURSIVE up AS (
  SELECT src_id, dst_id, relation, 1 AS depth
  FROM lineage WHERE dst_id = 'model:finetune-7b@v12'
  UNION ALL
  SELECT l.src_id, l.dst_id, l.relation, up.depth + 1
  FROM lineage l JOIN up ON l.dst_id = up.src_id
  WHERE up.depth < 6                                   -- 限制深度，防环
)
SELECT src_id, relation, depth FROM up ORDER BY depth;
```

用户行使删除权时，同一 record_id 可能被多个冻结版本引用，物理删除会破坏版本哈希；正确做法是写入墓碑（Tombstone，标记样本已撤回但保留占位），并在下一版本中剔除。血缘断链最常见两种情形：批次表只存标注员姓名不存 ID（重名即错配），以及导出脚本临时生成版本号却未登记 content_hash（同版本号对应不同内容）。

### 9.8 常见坑速查

- **用文件名当版本**：`train_v2_final.xlsx` 一周后无人能确认它与线上模型的对应关系，版本必须内容寻址并登记哈希。
- **只标正例不标负例**：工具调用与意图分类缺负样本，模型学会「凡问必调工具」。
- **规范不版本化**：Kappa 从 0.8 掉到 0.5 时，查不出是规范变了还是人变了。
- **抽检样本让标注员自己挑**：抽到的是容易样本，κ 虚高；抽检必须随机、双盲、含黄金题。

## 10. 数据质量、元数据与安全合规

### 10.1 五类质量规则

数据质量（Data Quality，数据满足明确业务约束的程度）若不落成可自动判定的规则，就只是主观评价。检索增强生成（Retrieval-Augmented Generation，RAG，先召回片段再交给模型组织答案的路线）对质量的要求集中在五个维度，每个维度都有确定算法：

| 维度 | 定义 | 计算式 | 阈值示例 | 失败处置 |
|---|---|---|---|---|
| 完整性（Completeness） | 必填字段与正文非空 | 非空行数 ÷ 总行数 | ≥ 99% | 阻断入库 |
| 唯一性（Uniqueness） | 同一内容不重复入库 | 1 − 重复条数 ÷ 总条数 | ≥ 99.5% | 去重后入库 |
| 时效性（Timeliness） | 语料仍在有效期内 | now() − source_updated_at | ≤ 7 天 | 告警并打过期标 |
| 解析成功率（Parse Success Rate） | 解析出可用文本的文档占比 | 成功文档数 ÷ 投递文档数 | ≥ 98% | 转人工队列 |
| 检索命中率（Retrieval Hit Rate） | 评测集中正解被召回的比例 | 命中问题数 ÷ 问题总数 | Recall@5 ≥ 0.85 | 阻断发布 |

规则要写成配置而非散落在脚本里，新增知识库只改数据不改代码：

```yaml
# quality_rules.yaml：每条规则四要素 = 作用域、维度、阈值、动作
- id: q_completeness_content
  scope: { kb_id: kb_finance_cn }      # 作用域，越小越优先
  metric: completeness
  field: content
  threshold: { op: ">=", value: 0.99 }
  action: block                        # block 阻断 / warn 告警 / tag 打标
  owner: data-quality@acme.com
```

`action: block` 只留给硬约束。把时效性设成阻断，过期日整库会集体不可用；正确做法是 `warn` 加 `lifecycle: expired` 标记，让检索自动跳过。

### 10.2 质量检查的执行与处置

检查要定时跑、结果要落表，否则无法看趋势，也无法在出事时回溯「哪天开始坏的」。

```sql
-- 每日质量快照（PostgreSQL）：一次算出完整性与失败样本，供看板与告警消费
INSERT INTO quality_metrics (stat_date, kb_id, metric, value, failed_rows)
SELECT CURRENT_DATE, kb_id, 'completeness',
       1.0 * COUNT(*) FILTER (WHERE content <> '') / COUNT(*),  -- 空正文占比
       COUNT(*) FILTER (WHERE content = '')
FROM docs
WHERE created_at::date = CURRENT_DATE
GROUP BY kb_id;
```

唯一性用内容指纹（Content Fingerprint，正文规范化后取哈希得到的定长值）实现更省：对去空白、统一全半角后的正文取 SHA-256，建唯一索引，重复文档在写入时即被数据库挡下，比事后全表扫描快一个量级。跳过规范化，同一文件改个名就会重新入库。

处置分三档：阻断（不合格数据不进索引，退回采集队列并附失败原因）、隔离（进入待处理表，人工确认后放行或丢弃）、打标（允许入库但写 `quality_flags`，检索时降权）。阈值不要一次定死，先跑两周只记录不处置，拿到真实分布再定，否则第一周就会因阈值过严阻断正常业务。

### 10.3 监控指标与告警

指标（Metric，可被持续采集的数值）盯链路健康，规则盯数据本身，两者缺一会得出「数据都对、就是没更新」的误判。

| 指标 | 采集点 | 阈值 | 含义 |
|---|---|---|---|
| 入库滞后 | 采集到可检索的耗时 | P95 ≤ 30 分钟 | 用户看到旧答案的主因 |
| 解析失败率 | 解析流水线 | > 2% | 版式变更或文件损坏 |
| 空召回率 | 检索服务 | > 8% | 语料缺失或切分不当 |
| 引用校验失败率 | 答案后处理 | > 1% | 引用区间与原文不符 |
| 检索延迟 | 检索网关 | P95 ≤ 500 ms | 影响首字延迟 |

告警（Alert，指标越过阈值后触发的通知与工单）必须分级，并带处置手册：

```yaml
# Prometheus 告警：for 抑制抖动，severity 决定通知渠道（P1 电话、P3 日报）
groups:
- name: rag-data-quality
  rules:
  - alert: IngestLagHigh
    expr: histogram_quantile(0.95, sum(rate(ingest_lag_seconds_bucket[10m])) by (le)) > 1800
    for: 15m
    labels: { severity: P1 }
    annotations: { summary: "入库 P95 滞后超过 30 分钟" }
```

排查顺序固定为：先看采集端是否还在推（对比源端文件数与入库数），再看解析队列积压，最后看索引重建任务是否卡住。同一告警 24 小时内重复五次应自动升级为工单，否则会被静音淹没。

### 10.4 元数据与血缘

元数据（Metadata，描述数据的数据）分三类：技术元数据（格式、页数、块数、向量模型）、业务元数据（所属知识库、负责人、主题域）、管理元数据（分级、保留期、是否含个人信息）。它同时是检索过滤条件、权限判定依据和影响分析的输入。

```json
{
  "asset_id": "doc_8821",
  "technical": { "format": "pdf", "pages": 42, "chunks": 118, "embed_model": "bge-m3" },
  "business": { "kb_id": "kb_finance_cn", "owner": "finance-ops", "domain": "报销" },
  "governance": { "level": "L2", "retention_days": 365, "pii": true }
}
```

血缘（Data Lineage，数据从源端到消费端的流转关系图）用有向图表示，节点是数据资产，边是一次加工动作：

```json
{ "from": "s3://raw/hr/handbook.docx", "to": "doc_8821",
  "op": "parse", "job": "ingest#4471", "at": "2026-11-20T09:12:00Z" }
```

有了边表，影响分析就是一次反向遍历：

```python
def downstream(asset_id: str, edges: dict[str, list[str]], depth: int = 5) -> set[str]:
    """反向 BFS：给定源资产，列出全部受影响的下游资产（索引、缓存、评测集）"""
    seen, frontier = set(), [asset_id]
    for _ in range(depth):                        # 限深，防止超大图卡死
        nxt = [t for n in frontier for t in edges.get(n, []) if t not in seen]
        seen.update(nxt)
        frontier = nxt
        if not frontier:
            break
    return seen
```

没有血缘，下线一份文档只能靠人回忆它被谁用过；有血缘，一次查询就能拿到要清理的索引、缓存与评测样本清单。

### 10.5 数据分级与脱敏

分级（Data Classification，按泄露后的危害程度给数据定级）决定谁能看、存多久、怎么脱敏，是权限与合规的共同前提。

| 级别 | 判定 | 例子 | 存储要求 | 默认外发 |
|---|---|---|---|---|
| L1 公开 | 已对外发布 | 官网帮助中心 | 明文 | 允许 |
| L2 内部 | 仅内部可见 | 制度、流程 | 明文加权限 | 需审批 |
| L3 敏感 | 含个人信息或商业机密 | 客户合同、工资表 | 加密加脱敏 | 禁止 |
| L4 极敏 | 法定高敏 | 身份证、银行卡 | 加密分片加审计 | 禁止 |

脱敏（Data Masking，在保留可读性的前提下替换或删除敏感值）分掩码、假名化、泛化三类。RAG 的硬要求是同一实体替换结果稳定，否则同一个人在不同块里变成不同假名，模型无法跨块推理：

```python
import hashlib, re

def mask_pii(text: str, salt: str) -> str:
    """手机号假名化：加盐哈希保证同值同结果，可关联、不可还原"""
    def _phone(m: re.Match) -> str:
        h = hashlib.sha256((salt + m.group(0)).encode()).hexdigest()[:8]
        return f"<PHONE_{h}>"                       # 不含原文，跨块一致
    text = re.sub(r"1[3-9]\d{9}", _phone, text)
    return re.sub(r"\d{17}[\dXx]", lambda m: f"<ID_{m.group(0)[-4:]}>", text)  # 保留后四位便于核对
```

脱敏必须发生在入库前。原文先落地再脱敏，泄露面一点没缩小。需要保留原文排障时，原文单独存加密区，密钥与索引库分离，取用留审计。

### 10.6 访问控制与审计

访问控制（Access Control，决定主体能否对客体执行某操作）有两种主流模型：RBAC（Role-Based Access Control，基于角色的访问控制，权限授予角色而非个人）与 ABAC（Attribute-Based Access Control，基于属性的访问控制，按主体、资源、环境属性实时判定）。检索必须在召回前过滤，先取 Top-50 再剔除，既浪费算力，也会从「结果被剔空」泄露存在性信息。

```python
def authorize(user: dict, doc: dict) -> bool:
    """ABAC 判定：级别与部门任一不满足即拒绝，条件必须进检索过滤器"""
    if doc["level"] == "L4" and user["clearance"] < 4:
        return False
    if doc["level"] in ("L3", "L4") and user["dept"] not in doc.get("allow_depts", []):
        return False
    return True
```

审计日志（Audit Log，记录谁在何时对什么数据做了什么且不可篡改的流水）至少五个字段：主体、客体、动作、结果、时间。它与业务日志分开存储、只追加、保留期更长：

```json
{ "ts": "2026-11-20T10:03:11Z", "actor": "u_1024", "role": "editor",
  "action": "doc.read", "target": "doc_8821", "result": "deny",
  "reason": "level_mismatch", "ip": "10.2.3.4", "trace_id": "t-000871" }
```

`deny` 记录比成功记录更有价值：同一账号短时间内连续 deny，通常意味着有人在试探权限边界，这正是告警该抓的信号。

### 10.7 合规要点

三条红线：个人信息、版权、日志留存。

个人信息（Personally Identifiable Information，PII，可单独或结合其他信息识别到自然人的数据）遵循最小必要与可删除。用户行使删除权时，正文、向量、缓存、备份四路都要清理，只删业务表不算删除。

版权方面，入库前确认授权范围，抓取网页遵守 robots.txt 与许可协议，对外引用标注来源，长段摘录需授权。答案中出现大段原文时，产品上要能一键回到出处。

| 数据 | 保留期 | 依据 | 到期动作 |
|---|---|---|---|
| 原始文档 | 业务存续期 | 合同约定 | 归档或删除 |
| 会话轨迹 | 6 个月 | 内部合规 | 脱敏后归档 |
| 审计日志 | 1–3 年 | 监管要求 | 转冷存储 |
| 备份 | 30 天滚动 | 恢复需要 | 覆盖 |

日志留存的关键是可查不可改：写入对象存储并开启合规保留（Object Lock），管理员也无法提前删除。留存期与删除权存在冲突，处理办法是留存期内只保留脱敏轨迹，原文按更短周期清理。

### 10.8 常见坑速查

| 现象 | 根因 | 处置 |
|---|---|---|
| 质量分很高，用户仍说答案旧 | 只查完整性，没测入库滞后 | 补时效性规则与 P95 滞后指标 |
| 同一文档反复入库 | 指纹计算前未规范化 | 统一空白与全半角后再取哈希 |
| 文档已下线仍被召回 | 缺血缘，缓存没清 | 建血缘图，按下游清单逐层清理 |
| 脱敏后无法跨块关联 | 每次随机替换 | 改用加盐哈希做稳定假名 |
| 审计日志可被运维删改 | 与业务库同权限 | 独立存储、只追加、开 Object Lock |

## 11. 成本性能、落地路线与常见坑

### 11.1 容量与成本估算：先把三笔账拆开

容量规划（Capacity Planning，上线前用可复算的公式推算存储、算力与费用的上限）不产出功能，却决定项目能否活下去。Agent 数据中台的钱花在三处，计费单位互不相同：存储按 GB·月，向量化（Embedding，把文本映射为定长浮点向量的过程）按输入 token，推理（Inference，模型根据上下文生成回答的过程）按输入加输出的 token。合成一个「每千次问答多少钱」之前，必须先分开算。

设基准场景：100 万篇文档、平均 4 页、每页 600 字，切分后每块 400 字、重叠 50 字，即有效步长 350 字。

- 原始文本：100 万 × 4 × 600 = 24 亿字符，UTF-8 中文约 3 字节，约 7.2 GB。
- 块数：24 亿 ÷ 350 ≈ 686 万块。
- 向量：1024 维、float32 每维 4 字节，每块 4 KB，共 27.4 GB；HNSW（Hierarchical Navigable Small World，一种分层可导航小世界的近似最近邻图索引）索引膨胀 1.5 倍、留 20% 副本后约 49 GB。
- 一次性 embedding：686 万 × 400 字 ≈ 27 亿字，中文 1 字约 0.7 token，约 19 亿 token，按 0.5 元/百万 token 计约 960 元。
- 推理：日活 2000 人、人均 8 次即 1.6 万次/日；每次输入 3000 token（8 个检索块加系统提示）、输出 400 token；按输入 1 元、输出 2 元每百万 token，单次 0.0038 元，约 61 元/日、1830 元/月。

```python
# Python 3.11：三类成本统一折算为元/月，改这几个常量即可复算自己的场景
DOCS, PAGES, CHARS_PER_PAGE = 1_000_000, 4, 600
STEP, DIM, IDX_INFLATE, REPLICA = 350, 1024, 1.5, 1.2
OSS = 0.12      # 元/GB·月，对象存储
VDB = 8.0       # 元/GB·月，向量库按内存计价
EMB, IN_, OUT = 0.5, 1.0, 2.0   # 元/百万 token

chars = DOCS * PAGES * CHARS_PER_PAGE
chunks = chars / STEP                              # 有效步长决定块数
raw_gb = chars * 3 / 1e9                           # 中文按 3 字节
vec_gb = chunks * DIM * 4 / 1e9 * IDX_INFLATE * REPLICA
emb_cny = chars * 0.7 / 1e6 * EMB                  # 中文字符→token 粗估
daily = 16_000
infer_cny = daily * 30 * (3000 * IN_ + 400 * OUT) / 1e6

print(f"块数={chunks/1e4:.0f}万 原始={raw_gb:.1f}GB 向量={vec_gb:.1f}GB")
print(f"存储={raw_gb*OSS+vec_gb*VDB:.0f}元/月 首建embedding={emb_cny:.0f}元 推理={infer_cny:.0f}元/月")
# 输出：块数=686万 原始=7.2GB 向量=49.4GB / 存储=396元/月 首建embedding=958元 推理=1827元/月
```

结论指向明确：推理费约为存储费的 4.6 倍，且随用量线性增长。省钱优先级是压输入 token、提缓存命中率，最后才是换便宜的存储。给非技术负责人汇报时只讲三个数：首年总成本、每千次问答单价、成本随用量增长的斜率。

### 11.2 性能优化：批处理、缓存、异步、并发控制

四类手段解决四个不同瓶颈，混用无效。批处理（Batching，把多条请求合并为一次 API 调用）解决的是「单条请求固定开销高」；缓存（Cache，按内容或查询的哈希复用已有结果）解决的是「重复计算」；异步（Async I/O）解决的是「等网络时 CPU 空转」；并发控制（Concurrency Control，限制在途请求数量）解决的是「压垮上游」。

| 手段 | 典型收益 | 主要代价 | 该用在哪 |
| --- | --- | --- | --- |
| 批处理 | 吞吐提升 3-8 倍，单价不变 | 单批 P99 上升 | 离线批量入库 |
| 向量缓存 | 增量入库成本降 60%-90% | 需内容归一化去重 | 文档重刷 |
| 检索缓存 | 重复 query 延迟降约 80% | 数据新鲜度受损 | 在线高频问答 |
| 异步并发 | 单机吞吐提升 5-10 倍 | 错误处理复杂度上升 | 全链路 |
| 信号量限流 | 消除 429 重试风暴 | 峰值出现排队 | 调外部 API |

```python
# Python 3.11：批量 embedding + 内容哈希缓存 + 信号量 + 指数退避
import asyncio, hashlib, httpx

SEM = asyncio.Semaphore(8)      # 在途请求上限，按供应商 10 QPS 留 20% 余量

def key(t: str) -> str:
    return hashlib.sha256(" ".join(t.split()).encode()).hexdigest()  # 归一化后哈希

async def embed_batch(client, texts, cache):
    keys = [key(t) for t in texts]
    todo = [t for t, k in zip(texts, keys) if k not in cache]   # 只对未命中出网
    async with SEM:                                          # 仅出网批次占坑
        for attempt in range(4):
            try:
                r = await client.post("/v1/embeddings",
                                      json={"model": "bge-m3", "input": todo})
                r.raise_for_status()
                for k, v in zip([key(t) for t in todo], r.json()["data"]):
                    cache[k] = v["embedding"]
                break
            except (httpx.HTTPError, KeyError):
                await asyncio.sleep(2 ** attempt)   # 1s,2s,4s,8s 退避
        else:
            raise RuntimeError(f"{len(todo)} 条 embedding 重试耗尽")
    return [cache[k] for k in keys]

async def embed_all(texts, batch=64):
    cache, out = {}, []
    async with httpx.AsyncClient(timeout=60) as c:
        tasks = [embed_batch(c, texts[i:i+batch], cache)
                 for i in range(0, len(texts), batch)]
        for part in await asyncio.gather(*tasks):    # 批次间并发，批次内串行
            out.extend(part)
    return out
```

三个配置项的口径：SEM=8 对应上游 10 QPS 并留余量，改大只会换来 429；batch=64 适合 400 字短块，单块超 2000 字应降到 16-32，否则触发上游按 token 截断；timeout=60 要覆盖 P99 而非平均值，否则长尾批次被误杀后整批重试，成本翻倍。缓存必须带版本：key 里混入文档版本号或 embedding 模型版本，模型换代后旧向量不失效会让召回质量静默劣化。

### 11.3 SLO 与容量规划

SLO（Service Level Objective，服务等级目标，对外承诺的可量化质量指标）由 SLI（Service Level Indicator，实际被测量的那个指标）加目标值构成；错误预算（Error Budget）等于 1 减 SLO，是允许失败的空间，一旦耗尽就该停止加功能、转做稳定性。没有 SLO 的团队只能靠投诉驱动优化，永远在救火。

| SLI | 定义 | 目标示例 | 备注 |
| --- | --- | --- | --- |
| 检索 P95 | 收到 query 到返回 top-k | ≤ 300 ms | 不含生成 |
| 首 token P95 | 请求发出到第一个流式 token | ≤ 1.5 s | 体感最关键 |
| 端到端 P99 | 完整回答返回 | ≤ 8 s | 长答案单列 |
| 可用性 | 成功请求 ÷ 总请求 | ≥ 99.5% | 5xx 与超时算失败 |
| 召回命中率 | top-10 含正确块的比率 | ≥ 85% | 离线集固定 |
| 入库新鲜度 | 源文件变更到可检索 | ≤ 15 min | 增量链路 |

容量用利特尔法则（Little's Law，并发数等于到达率乘平均停留时间）推算：峰值 20 QPS、单请求平均 1.2 秒，需 24 个并发槽；按 70% 水位运行需 35 个槽位；再乘 1.5 冗余应对突发，即 52 个。槽位不够时的表现不是报错，而是排队导致 P99 从 1.2 秒涨到 9 秒，用户以为系统挂了。

```javascript
// k6（开源负载测试工具）脚本 retrieval_load.js：验证检索 P95 是否守住 300ms
import http from 'k6/http';
import { check } from 'k6';
export const options = {
  stages: [
    { duration: '1m', target: 20 },   // 爬坡：逐步加到 20 并发
    { duration: '3m', target: 20 },   // 稳态：观察长尾是否抖动
  ],
  thresholds: { http_req_duration: ['p(95)<300'] },  // 不达标脚本直接判失败
};
export default function () {
  const res = http.post('http://localhost:8000/search',
    JSON.stringify({ q: '如何配置租户权限', top_k: 8 }),
    { headers: { 'Content-Type': 'application/json' } });
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

压测前必须预热连接池并塞入真实分布的数据量，空库压出来的 300 ms 毫无意义；同时固定 top_k 与查询长度，否则两轮结果不可比。

### 11.4 MVP 三期落地路线

MVP（Minimum Viable Product，最小可用产品）的价值在于把「能不能答对」和「能不能多团队用」拆成两件先后验证的事。三期各有明确的不做清单，不写不做清单的项目 100% 延期。

| 期 | 周期 | 目标 | 交付物 | 验收标准 | 明确不做 |
| --- | --- | --- | --- | --- | --- |
| P1 打通 | 4-6 周 | 单源单租户跑通问答 | 同步入库、固定切分、向量检索加引用回答 | 50 篇真实文档、20 题人工评测命中率 ≥ 70%、P95 ≤ 3 s | 多租户、权限、增量、评测平台 |
| P2 提质 | 6-8 周 | 答得准且便宜 | 多格式解析、混合检索、rerank、增量更新、成本看板 | 命中率 ≥ 85%、新鲜度 ≤ 15 min、单次成本可追溯 | 自动标注、多模态 |
| P3 规模化 | 8-12 周 | 多团队可治理 | 多租户与行列级权限、轨迹回流、标注评测闭环、弹性容量 | 10 倍数据量下 P95 不退化、成本可按租户归因 | 自研向量库、自研模型 |

P1 的隐藏任务是拿到一份人工标注的 100 条问答集，它决定后面所有优化的方向；没有它，P2 的 rerank 和混合检索只能靠感觉调参。P3 才引入多租户，是因为权限模型一旦提前，会拖慢所有链路改造。反过来先建平台再找场景，是最常见的死法：能力全有、没人用。

### 11.5 十条反模式与排查清单

| # | 反模式 | 典型现象 | 根因 | 排查动作 |
| --- | --- | --- | --- | --- |
| 1 | 全量重刷代替增量 | 每次更新耗时数小时、账单暴涨 | 缺内容哈希与版本水位 | 看 embedding 调用量是否与新增文档数线性相关 |
| 2 | 按固定字数硬切 | 表格、代码块被截断，答非所问 | 未按标题层级等结构切分 | 抽 20 个坏例看块的首尾边界 |
| 3 | 只做向量检索 | 编号、专名类 query 召回为 0 | 缺稀疏检索 BM25（Best Matching 25，基于词频与逆文档频率的关键词打分算法） | 用编号类 query 单独测召回 |
| 4 | top-k 拍脑袋定大 | 单次输入 2 万 token，成本高 | 未做 rerank 与上下文截断 | 统计每请求输入 token 的分布 |
| 5 | 无同源去重 | top-10 里 5 条是同一段文本 | 入库未按内容哈希去重 | 算 top-10 重复文本占比 |
| 6 | 同步阻塞入库 | 上传接口 504，用户重传产生副本 | 解析与向量化写在请求线程里 | 看接口 P99 与队列积压深度 |
| 7 | 无并发上限 | 上游 429 比例超 10%，越重试越慢 | 未限制在途请求、未退避 | 统计 429 比例与平均重试次数 |
| 8 | 缓存不带版本 | 文档更新后仍答旧内容 | key 未含版本、TTL 过长 | 改一篇文档后立刻提问验证 |
| 9 | 只看平均延迟 | 用户喊慢但均值正常 | 无 P95/P99 与慢请求采样 | 取最慢 20 条请求逐段拆耗时 |
| 10 | 没有评测集 | 每次调参靠感觉 | 未固化带标注问答集 | 能否一句话说出「命中率从多少到多少」 |

前三条属于结构问题，改了立刻见效；中间四条属于工程纪律，需要流程卡口；最后三条属于观测缺失，本质是不知道系统在干什么。排查优先级建议自上而下：先确认数据与切分正确，再查链路与限流，最后才怀疑模型——绝大多数「模型不行」的结论，根因在第 1 到第 5 条。

### 11.6 落地检查清单

| 检查项 | 通过标准 | 验证方式 |
| --- | --- | --- |
| 成本模型 | 存储、embedding、推理三项分开成表 | 用本节脚本复算一次真实规模 |
| 单位成本 | 每千次问答单价可报出 | 账单 ÷ 实际问答次数 |
| 成本归因 | 能按租户或业务线拆账 | 请求元数据带 tenant_id |
| 内容哈希 | 同源文档不重复向量化 | 抽查入库日志的跳过率 |
| 批次参数 | batch、timeout 有书面取值理由 | 配置项注释可追溯 |
| 缓存版本 | key 含文档与模型版本 | 改文档后立即提问验证 |
| 并发上限 | 外部调用全部经信号量 | 压测时 429 比例低于 1% |
| 延迟观测 | P95/P99 与慢请求采样齐全 | 看板能定位到 span 级 |
| SLO 与错误预算 | 有目标值、有耗尽后的处置预案 | 最近一次发布是否被预算卡住 |
| 评测集 | ≥ 100 条带标注问答，改动前后可比 | 每次发版跑同一套题 |
| 回滚预案 | 索引与配置可一键回退 | 演练一次旧版本切换 |
| 值班与告警 | 入库积压、429、错误率有告警 | 手动触发一次告警验证 |
