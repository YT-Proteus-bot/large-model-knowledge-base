# Spring 原理与实践

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/NaU2wG7daipncPkfzJrclPdWnhh

## IoC 容器与 AOP 原理

### Bean 生命周期

#### 🧩 1. Spring IoC 容器的工作原理是什么？Bean 的生命周期有哪些阶段？

控制反转（IoC） 的核心是“把创建对象的权力交给容器”，而不是在代码里到处 new。Spring IoC 容器的内部就像一个高度自动化的工厂：它根据你提供的图纸（配置类/XML/注解），扫描原材料（类路径），生成零件蓝图（BeanDefinition），然后通过一系列流水线加工，最终生产出可以随时使用的 Bean。

![image](../../../assets/BTH1dpkNDoxSskxaSlPc3DNGnee/VLgEbQgfiou9ktxo66ucrkJWngf.png)

Bean 的完整生命阶段大致如下：

> （此处为内嵌电子表格，需登录飞书查看）

这些阶段共同保证了 Bean 在经过依赖注入、上下文感知、功能增强之后，才被暴露给应用使用。

---

#### ⚖️ 2. BeanPostProcessor、InitializingBean、@PostConstruct 的执行顺序是什么？各自的应用场景？

这三者都工作在 Bean 的初始化附近，但它们分工不同。按照执行顺序排列如下：

```
代码块
执行顺序：
1. BeanPostProcessor.postProcessBeforeInitialization()
2. @PostConstruct 方法
3. InitializingBean.afterPropertiesSet()
4. @Bean(initMethod = "...") 方法
5. BeanPostProcessor.postProcessAfterInitialization()
```

用一个具体的例子来说明。假设我们需要一个 LLM 客户端的 Bean，它需要在属性注入完成后自动预热连接池：

```
代码块
@Component
public class LLMClientBean implements InitializingBean {
    @Value("${llm.api.key}")
    private String apiKey;
    private HttpClientPool pool;
    @PostConstruct
    public void initPool() {
        // @PostConstruct: 在依赖注入完成后立即创建连接池
        this.pool = new HttpClientPool(apiKey, 10);
        System.out.println("1. @PostConstruct - 连接池已创建");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        // InitializingBean: 执行连接预连，确保前几条请求不排队
        pool.prewarm(5);
        System.out.println("2. afterPropertiesSet - 已预热5条连接");
    }
    public void customInit() {
        // 在 @Bean 的 initMethod 里指定
        System.out.println("3. customInit - 可以执行额外校验");
    }
    public String call(String prompt) {
        // 实际调用 LLM
        return pool.getConnection().invoke(prompt);
    }
}
```
