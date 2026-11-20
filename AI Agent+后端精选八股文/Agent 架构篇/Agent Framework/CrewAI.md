# CrewAI

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/URfRwgIyEi3zoQk8qc5cj7yPnfb

## CrewAI 工作流编排

![image](../../../../assets/NbaPdEKZColiMrxtZgicOftJnxh/DeXCbmKRHoSFlMxy1xQcBOKrnTd.png)

#### 1. Crew 和 Agent 是什么关系？谁决定执行顺序？

Crew 是“团队”，Agent 是“角色”。 Crew 把一组 Agent 和一组 Task 打包在一起，提供一个 kickoff() 入口来驱动一次运行。Agent 是“谁来做”——一个带有 role、goal、backstory、tools、llm 的执行实体；Task 是“做什么”——绑定到某个 Agent 的工作单元。Crew 本身不定义“怎么串”，它只是容器。

执行顺序由 Crew 的 process 属性决定，不是 Agent 决定的。

- Process.sequential（默认）：严格按照 tasks=[...] 声明顺序执行，前一个 Task 的输出作为后一个的上下文。
- Process.hierarchical：Crew 会先构建一个 manager Agent，由这个 manager LLM 在每一轮决定下一个执行哪个 Task、交给哪个 Agent，必要时还可以拒绝或重新路由。

一个容易记的划分：Agent 不知道顺序，Task 不知道顺序，Crew 的 Process 持有顺序。Sequential 是“清单顺序”，hierarchical 是“经理动态派活”。

---

#### 2. role、goal、backstory 各自影响什么？backstory 写泛有何风险？

这三者本质上是 把 prompt engineering 暴露为构造函数参数——CrewAI 把它们拼接进 Agent 的 system prompt，并在 hierarchical 模式下用于任务路由决策。

- role：决定“这个 Agent 被系统当成什么职能”。它应该读起来像一个职位名称（“Quantitative Analyst”“Population Researcher”），路由信号最强；模糊的 role（“Bob who loves numbers”）在 hierarchical 模式下会显著削弱 manager 的判断依据。
- goal：决定“这个 Agent 优化什么”。好的 goal 是可验收的，例如“Deliver actionable risk assessments with confidence levels”，而不是“Analyze data”。
- backstory：提供资质与工作风格的上下文——为什么这个 Agent 有资格做这件事、它遵循什么惯例、输出偏好是什么。它在 system prompt 中持续存在，影响每一轮的生成倾向。

backstory 写泛的核心风险是“信号稀释 + 成本膨胀”，不是“风格问题”。 一个两行、包含具体专长和输出偏好的 backstory，效果优于五段式小说；冗长的 backstory 每一轮都要付 token，却不带来成比例的准确率提升，并且在 hierarchical 模式下会稀释 routing signal，让 manager 更难判断该把任务交给谁。更隐蔽的风险是与 role/goal 不一致的泛化描述——backstory 里写了“你擅长一切数据分析”，但 role 是“合规审查员”，模型会在两个相互矛盾的指令之间漂移。工程上的经验值：backstory 控制在 200 词以内，只写“资质 + 输出偏好 + 边界”，不写生平。

---

#### 3. sequential 与 hierarchical 的核心差异？额外 LLM 调用何时必要？

核心差异不在“调度器”，而在“谁做下一个决策”。 两者复用同一个 task loop，sequential 的顺序由声明固定；hierarchical 在 loop 之上插入一个 manager Agent，每一轮由 manager LLM 决定下一个 Task/Agent。

额外 LLM 调用的量化代价：hierarchical 在每一次专家调用之前都多一次 manager 调用。一个 5-task 的 crew，sequential 是 5 次 LLM 调用，hierarchical 是 6 次，而且 manager 调用携带完整 task 列表 + 已产生的全部输出作为上下文。实际成本增量通常落在 30–50%，部分场景可到 1.5–2 倍调用量。

额外调用何时必要？ 只在 “顺序无法事先知道” 时。典型信号：

- Task 之间存在条件分支——上一个结果决定下一个该做什么，而不是固定往下走；
- 有 4 个以上 specialist，且任务分配本身需要判断；
- 需要 动态拒绝/重新路由——某个 Agent 的输出不满足要求时，manager 可以打回或改派。

反过来，顺序固定、预算敏感、要求可预测 token 成本时，应留在 sequential，把分支逻辑写在自己的 glue code 里，而不是用 manager LLM 来“猜顺序”。一句话：hierarchical 买的是“运行时路由权”，不是“更好的执行质量”。

---

#### 4. CrewAI 没有 parallel Process，如何表达并行任务？

确认：CrewAI 的 Process 枚举只有 sequential 和 hierarchical，不存在 Process.parallel。

并行通过两个层面表达，而不是第三个 Process：

层面一：Task 级 async_execution=True。 把需要并行执行的任务标记为异步，它们的 context 会在汇合点被拼接为下游任务的输入。没有相互依赖的 Task 在 sequential 流程中可以被自动并行调度——依赖关系由 Task.context 声明，而不是由 Process 枚举声明。底层实现上，async_execution 使用 daemon thread + Future 对象。

层面二：Crew / Flow 级的 asyncio.gather()。 多个独立 Crew 可以并发运行；在 Flow 中也可以用 @start() 并行启动多个任务，用 @listen 做分支汇合。

关键认知：CrewAI 的并行是依赖驱动的隐式并行，不是显式的 parallel process。你声明“谁依赖谁”，框架决定“谁能同时跑”。这跟“声明并行”是两种控制范式——前者更贴近 DAG 语义，后者更贴近线程池语义。

---

#### 5. Crews 与 Flows 如何选？审计要求高时选哪个？

选型轴不是“能力”，而是“控制权归谁”。

> （此处为内嵌电子表格，需登录飞书查看）

审计要求高时选 Flows。 但需要精确表述：Flows 本身也不内置完整审计，它提供的是可审计的结构——每个 @listen 步骤是一个明确的边界，checkpoint 可以按方法名和时间戳记录状态快照，形成执行日志。真正的审计能力仍然需要你在 Flow 的步骤边界上主动挂载 append-only 日志和 checkpoint。Crews 的根本问题是审计面太宽：你无法在运行前说清“这个 Agent 可能走哪条路”，审计只能退化为“事后看它实际走了哪条路”，这在合规场景下往往不可接受。
