# Java中创建对象有哪些种方式

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Z5iDw0KlLi5945kTQnkcSB9Cnvf

我需要一个面试的答案回答，看起来不像ai，解析要多，要有逻辑，模板好看，要有Mermaid 时序图 图内容在简单清晰：Java中创建对象有哪些种方式

#### 面试回答

面试官： Java 里创建对象有哪几种方式？

你：

常见的大概五六种，核心差别就两点：调不调构造方法，以及对创建过程的控制力度。 我把它们分成四类来说。

第一类：全靠编译器，new 关键字。
User user = new User()，这是最直接的方式，编译期就确定类型，JVM 会分配内存并调用构造方法 <init>，一切按部就班。

第二类：运行时反射，控制灵活。
可以通过 Class 或 Constructor 对象在运行时动态创建实例。Class.forName("User").getDeclaredConstructor().newInstance() 可以传参、可以访问私有构造，是框架（Spring、MyBatis）的核心。注意 Class.newInstance() 在 JDK9 已经标为过时，因为它只能调无参构造，而且异常处理不友好，现在都用 Constructor.newInstance()。

第三类：不调构造的“黑魔法”。

- clone()：前提是实现 Cloneable 接口，super.clone() 直接从内存里复制一份，不走构造，是浅拷贝。
- 反序列化：从 ObjectInputStream 里 readObject，JVM 会从字节流里重建对象，也不触发构造。这两个方式常在单例攻防里被问到，因为能绕过私有构造创建新实例。
- Unsafe.allocateInstance(Class)：直接操作内存分配对象，构造方法完全被跳过，一些序列化框架（如 Kryo）用这招提高性能，但它是 JDK 内部 API，不建议业务代码碰。

第四类：语法糖和工具。
比如 String s = "hello" 字面量、Integer i = 127 自动装箱、Lambda 表达式的内部类实例等，它们背后也会创建对象，但属于特定场景。

---

四种方式的构造调用差异

![image](../../../../assets/AeyudJG0JoItjbxFILgcyw1bnph/JVrXbRsLFoGc6cxt1kacO0y5n5f.png)

绝大多数时候用 new；框架里用反射；需要拷贝且不关心构造逻辑用 clone；远程传输或持久化用反序列化。注意哪些不调构造，这在单例安全、初始化逻辑里很容易出坑。
