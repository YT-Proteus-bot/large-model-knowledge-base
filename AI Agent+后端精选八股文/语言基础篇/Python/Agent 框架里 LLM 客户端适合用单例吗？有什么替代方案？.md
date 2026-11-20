# Agent 框架里 LLM 客户端适合用单例吗？有什么替代方案？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/GWHbww50ji0ezPkLdTxcYcIAn0b

关于 Agent 框架里 LLM 客户端能不能用单例，我的观点是：

可以直接用，但不推荐把它做成“传统意义上的单例”，更好的思路是“统一管理、按需使用、共享底层连接”。

下面我按这个逻辑展开，配上图标对比来看。

---

#### 🔍 LLM 客户端到底长什么样？

一个典型的 LLM 客户端封装了：

- 配置：API Key、Base URL、模型名、温度等参数
- HTTP 连接层：OkHttp / Apache HttpClient 的实例，管理连接池、超时
- 调用方法：chat(prompt)、stream() 等

在 Agent 框架中，可能同时需要调用多个模型，比如：

- 一个负责思考的工具模型（高温度）
- 一个负责精确执行的模型（低温度、JSON 模式）
- 甚至不同云厂商的模型（OpenAI + Claude）

---

#### ✅ 单例的优点（为什么有人会用）

> （此处为内嵌电子表格，需登录飞书查看）

---

#### ❌ 单例在 Agent 场景下的致命问题

但 Agent 框架的特点是多模型、多配置、动态切换，单例会暴露几个硬伤：

- ❌ 无法同时持有多种配置
  你想用一个 getInstance() 同时返回 GPT-4 和 GPT-3.5？不行，要么得在方法里传参数破坏设计，要么就得维护多个全局实例，那还叫单例吗？

- ❌ 温度、超时等参数僵化
  单例构造时定死了参数，但 Agent 的某个步骤可能需要临时调高温度以增加创造性，单例根本没法灵活应变。

- ❌ 测试和 Mock 困难
  单例是全局状态，单元测试时很难替换成 Mock 对象，会导致多个测试之间互相影响。

- ❌ 违反单一职责
  Agent 中每个工具链可能只需要一个特定的 LLM 能力，强行共用一个大而全的单例会让依赖关系混乱。

---

#### 🛠️ 替代方案（重点来了）

##### 1️⃣ 依赖注入 + 容器管理的 Bean（推荐）

不是在代码里写死单例，而是让框架容器（如 Spring）把 LLM 客户端管理成单例 Bean。

同时支持按 Qualifier 注入不同配置的客户端。

```
代码块
@Configuration
public class LLMConfig {
    @Bean
    @Qualifier("creative")
    public LLMClient creativeClient() {
        return new LLMClient(apiKey, model, 0.9); // 高温度
    }
    @Bean
    @Qualifier("precise")
    public LLMClient preciseClient() {
        return new LLMClient(apiKey, model, 0.1); // 低温度
    }
}
```

使用时通过注入拿到自己需要的实例，底层 HTTP 客户端可以统一复用。

✅ 优点：灵活、可测试、多实例并存，但管理权在容器，保持了资源的可控性。

---

##### 2️⃣ 多例客户端 + 静态共享底层连接池
