---
title: Agent Skill篇
topic: AI Agent+后端精选八股文
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/X3qywnFR7iDywlkos58cp6Tgnfa
source_path: AI Agent+后端精选八股文/Agent Skill篇.md
questions: 3
chars: 1918
build_date: '2026-11-20'
images_removed: 0
---

# Agent Skill篇

> 主题：AI Agent+后端精选八股文　｜　原始文件：AI Agent+后端精选八股文/Agent Skill篇.md

## AI Agent精选八股文

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 题1：Skill 和 Harness 是什么关系？生产级 Agent 如何用 Harness 管住 Skill 的调用边界？

**一句话关系：Harness 是运行时底座，Skill 是底座上被装载和调度的能力包。**

**Harness 不产生领域知识，它负责让 Skill 安全、可观测、可回滚地运行。生产级 Harness 对 Skill 的管辖体现在五个硬边界上：**

- **① 装载边界**

Skill 不是常驻的。Harness 决定“什么时候把哪个 Skill 装进运行时”，**控制发现、解析、加载和热更新**。模型只能看到当前被允许暴露的 Skill 目录，而不是整个 Skill 库。

- **② 权限边界**

Harness 拥有基础权限天花板，Skill 的 manifest 只能收窄权限，不能突破。Skill 声明的 required-tools 只是依赖说明，不会自动绑定工具，更不会把只读权限提升为写入权限。

- **③ 执行边界**

Harness 决定 Skill 是在主 Agent 循环中 Inline 执行，还是 Fork 出独立 Agent 执行。Inline 共享主上下文，适合短流程；Fork 有独立上下文和隔离，适合高副作用或长任务，但 Fork 时上下文会被截断，Harness 要负责传递必要状态。

- **④ 审批边界**

高风险 Skill 触发的工具调用在 dispatch 前经过策略引擎校验：读操作自动执行，写操作生成审批请求，受限操作直接拒绝。这个分级由 Harness 强制执行，不依赖 Skill 内部的提示词“自觉”。

- **⑤ 可观测边界**

Harness 输出可用于调试和评测的 trace，记录 Skill 被选中、装载、执行、调用了哪些工具、产生了什么结果、是否越界。没有这条链路，失败的 Skill 调用无法定位是 Skill 逻辑、权限还是下游工具的问题。

**一句话总结：Skill 定义“怎么做”，Harness 定义“能不能做、以什么权限做、做完怎么审计”。把边界放在 Harness 里，而不是写在 Skill 的提示词里，是生产级 Agent 和演示原型的分水岭。**

## 题2：Skill 到底是什么？它和 Tool、Prompt、Memory 的本质区别是什么？

定义：Skill 是把“可复用的做事方法”封装成 Agent 可调用的能力模块。

它回答的是“遇到这一类任务，应该按什么流程、用什么工具、按什么标准做、异常怎么处理”。它不是一段更长的提示词，而是一个有触发条件、执行步骤、工具依赖、输出规范和验证闭环的程序性方法包。

四者的本质区别：

> 〔原文此处为飞书内嵌组件，内容未随导出〕

关键区分：

- Prompt 解决单次表达，Skill 解决跨任务复用。
- Tool 是动词，Skill 是SOP。
- Memory 是材料库，Skill 是操作手册——Memory 提供“用户偏好 HTML 报告”，Skill 提供“生成报告时先确认数据口径、再按模板输出、最后校验关键数字”。

一个容易混淆的点： Skill 可以调用 Tool，可以读取 Memory，但它的核心价值不在于“拥有更多工具”或“知道更多事实”，而在于知道在什么条件下、按什么顺序、用什么标准去使用这些工具和事实。拥有 search_docs 这个 Tool，不等于知道“应该检索几轮、怎么判断检索够了”——后者才是 Skill。

## 题3：为什么不能把 Skill 写成更长的 System Prompt？动态加载和静态注入的代价差异在哪？

核心判断：把 Skill 写成更长的 System Prompt，本质上是把“按需激活的方法论”降级成了“每轮都要背负的固定开销”。

代价差异不是线性的，而是随 Skill 数量线性放大的。

静态注入的代价

- 20 个 Skill、每个正文约 3k tokens → 全量注入每轮 60k tokens
- 50 个 Skill、每个 500 tokens → 常驻注入 25k tokens 每轮
- 其中绝大多数内容，在绝大多数轮次里根本不会被用到
- 更关键：静态内容和 System Prompt 共享同一权威信道，模型无法区分“背景知识”和“此刻必须遵守的流程”，容易在无关任务中被无关 Skill 干扰，也容易把过时的 Skill 当成当前指令
