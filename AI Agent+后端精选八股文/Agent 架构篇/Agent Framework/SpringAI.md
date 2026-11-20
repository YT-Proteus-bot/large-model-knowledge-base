# SpringAI

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/AFyRwmNoZiYw94kQOOhc35Junpd

### SpringAI 深度进阶

![image](../../../../assets/YmhodvCC8omorExt6mgctk2lngg/QfaLbdc5BovzXkx0v5lchJEfnZf.png)

#### [Advisor 机制、VectorStore 集成、Function Calling、多模型降级]

> （此处为嵌入式文档部件，需登录飞书查看）

#### 1. SpringAI 的 Advisor 机制是什么？如何用 Advisor 实现 RAG？

一句话：Advisor 是 Spring AI 中拦截和增强请求/响应的切面（AOP）机制，可以在调用 LLM 前后插入自定义逻辑。

它借鉴了 Spring AOP 的思想，但专门针对 AI 调用链设计。每个 Advisor 都可以在 Prompt 发送给模型之前做预处理，在模型返回之后做后处理，甚至完全替换请求。

![image](../../../../assets/YmhodvCC8omorExt6mgctk2lngg/Oq1zbIMAboHpWWxcIdKcGwISn8b.png)

Advisor 接口定义（简化）：

```
代码块
public interface RequestResponseAdvisor {
    // 前置拦截：修改 Prompt
    AdvisedRequest adviseRequest(AdvisedRequest request, Map<String, Object> context);
    // 后置拦截：修改响应
    ChatResponse adviseResponse(ChatResponse response, Map<String, Object> context);
    // 优先级
    int getOrder();
}
```

如何用 Advisor 实现 RAG？

核心思路是编写一个 RetrievalAugmentationAdvisor，在请求发送前，从向量数据库中检索相关文档，插入到 Prompt 的上下文中。

```
代码块
用户问题 → Advisor 拦截 → 向量检索 → 拼接文档 → 增强后的 Prompt → LLM → 回答
```

代码示例：实现一个简单的 RAG Advisor

```
代码块
@Component
public class RagAdvisor implements RequestResponseAdvisor {
    private final VectorStore vectorStore;
    public RagAdvisor(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }
    @Override
    public AdvisedRequest adviseRequest(AdvisedRequest request, Map<String, Object> context) {
        // 1. 获取用户原始输入
        String userQuery = request.userText();
        // 2. 从向量库检索相关文档
        List<Document> docs = vectorStore.similaritySearch(SearchRequest.query(userQuery).withTopK(3));
        // 3. 拼接检索结果
        String contextText = docs.stream()
                .map(Document::getContent)
                .collect(Collectors.joining("\n\n"));
        // 4. 构建增强后的系统提示词
        String augmentedSystem = String.format(
                "根据以下参考资料回答用户问题。如果资料中没有答案，请如实告知。\n\n参考资料：\n%s", contextText
        );
        // 5. 修改请求，注入上下文
        return AdvisedRequest.from(request)
                .withSystemText(augmentedSystem + "\n\n" + request.systemText())
                .build();
    }
    @Override
    public ChatResponse adviseResponse(ChatResponse response, Map<String, Object> context) {
        // 这里可做后处理，如提取引用、记录日志
        return response;
    }
    @Override
    public int getOrder() { return 0; }
}
```
