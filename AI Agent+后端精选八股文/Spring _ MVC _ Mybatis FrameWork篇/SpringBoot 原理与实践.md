#  SpringBoot 原理与实践

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/NHwYwCwRDiPz1OkM3b9cXNBinUd

## 1. Spring Boot 与自动装配

#### 🧩 1. Spring Boot 自动装配的原理是什么？@SpringBootApplication 做了什么？

自动装配的本质是“根据类路径中的依赖和配置，自动推断并创建 Bean，而无需你手动编写繁琐的配置类。” 它像一位经验丰富的管家，看到你书架上有书（依赖），就会把对应的书桌、台灯（Bean）都准备好。

@SpringBootApplication 是这三者的合体：

![image](../../../assets/AxvYdyFmsooJC3x8WYhcNMQLnxe/SL9Rbymnpo7XAnxzD3DcEvEBnnf.png)

真正的魔术师是 @EnableAutoConfiguration。它通过 @Import(AutoConfigurationImportSelector.class) 导入了一个选择器，这个选择器会去读取所有依赖的 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件（Spring Boot 2.7+，更早版本是 spring.factories）。

```
代码块
加载流程：
1. 启动类 → @EnableAutoConfiguration
2. AutoConfigurationImportSelector 扫描所有 AutoConfiguration.imports 文件
3. 收集到上百个候选配置类（如 DataSourceAutoConfiguration）
4. 逐个评估 @ConditionalOnClass / @ConditionalOnMissingBean 等条件注解
5. 条件满足 → 加载该配置类 → 创建对应的 Bean 并放入容器
```

为什么加上 @Conditional 注解？
如果没有这些条件注解，所有候选配置类都会无条件加载，那你的应用就会因为缺少某些类而崩溃。例如，DataSourceAutoConfiguration 只有在类路径中存在 DataSource 类、且容器中没有你手动定义的 DataSource Bean 时才生效。这种 “按需激活” 是自动装配的精髓。

示例代码：验证自动装配
你可以在 application.yml 中通过 debug: true 查看哪些自动配置类被激活了：

```
代码块
debug: true
```

然后在启动日志中搜索 AutoConfiguration，或者用 spring-boot-actuator 的 /actuator/conditions 端点查看正匹配和负匹配的条件。

用一张图概括 @SpringBootApplication 的结构：

![image](../../../assets/AxvYdyFmsooJC3x8WYhcNMQLnxe/DOSgbtyF7oF86XxipLxc66wJnKe.png)

#### 🛠️ 2. 如何自定义一个 Spring Boot Starter？

自定义 Starter 就是把“一组可复用的自动配置 + 依赖”打包成一个独立的模块。它通常包含两个子模块：autoconfigure（自动配置逻辑）和 starter（空壳，只引入 autoconfigure 和所需依赖）。

以创建一个 agent-tool-starter 为例，它会在启动时自动创建一个 ToolService Bean，并根据配置项 agent.tool.name 来设置工具名称。

步骤一：创建 autoconfigure 模块

```
代码块
// 1. 属性类，绑定 application.yml 中的前缀
@ConfigurationProperties(prefix = "agent.tool")
public class ToolProperties {
    private String name = "default-tool";
    // getter/setter...
}
// 2. 核心服务
public class ToolService {
    private final String toolName;
    public ToolService(String toolName) { this.toolName = toolName; }
    public String call() { return "Calling " + toolName; }
}
// 3. 自动配置类，负责创建 Bean
@Configuration
@EnableConfigurationProperties(ToolProperties.class)
@ConditionalOnClass(ToolService.class)
public class ToolAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public ToolService toolService(ToolProperties properties) {
        return new ToolService(properties.getName());
    }
}
```
