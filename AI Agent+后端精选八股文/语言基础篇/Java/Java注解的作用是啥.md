# Java注解的作用是啥

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/JfwLwtb7GiphnAkqhZvcX8wunHb

🔹 注解到底是什么，先给它卸个妆

注解本质上就是个不会动的标签。你贴在类上、方法上、字段上，它自己啥也不干，就是个标记。真正干活的，是后面读这个标签的注解处理器（编译期的插件，或者运行时的反射代码）。

🔹 按作用分，我自己习惯归为三档

> （此处为内嵌电子表格，需登录飞书查看）

🔹 面试可以这样展开说

✅ 编译期做检查
@Override 帮你检查是不是真的重写了父类方法，写错一个字母编译器就直接甩错误——这比代码评审可靠一百倍。

✅ 编译期生成代码
我最喜欢的例子是 Lombok。你写一个 @Data，反编译 class 文件之后 getter/setter/equals/hashCode/toString 全给你生成了。注解本身不做这些，是 Lombok 的注解处理器在编译期间操作了抽象语法树（AST），把代码塞进去了。

✅ 运行时框架整合
Spring 扫描到 @Service，就把类注册成 Bean；看到 @Transactional，就生成一个 AOP 代理帮你管理事务。底层都是 class.getAnnotation() 或 method.isAnnotationPresent()，对着反射一顿操作。注解在这里就是框架和程序员之间的“暗号”。

🔹 自定义注解怎么写，可以背个极简模板

```
代码块
@Retention(RetentionPolicy.RUNTIME)  // 活到运行时，反射能读到
@Target(ElementType.METHOD)          // 只能贴在方法上
public @interface Loggable {
    String value() default "";
}
```

然后写个切面或拦截器，一看到有这个注解的方法就自动打日志。这一套下来，注解的“标记+处理”模式就闭环了。

🔹 面试容易漏掉的元注解

光说怎么用不行，还得知道元注解（注解的注解）：

- @Retention 控制生命周期（SOURCE/CLASS/RUNTIME）
- @Target 控制贴哪儿
- @Inherited 子类能自动继承
- @Documented 进 javadoc
- @Repeatable 能反复贴（Java 8 以后）

追问如果问你“注解里的值为什么只能放常量”，那是因为注解的属性在编译时就得确定，写进 class 文件的结构里，没法存动态对象。
