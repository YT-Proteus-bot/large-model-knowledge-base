# 为什么Java中的main方法必须是public static void的？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SrkJwsIGVisBrckWAENctjcBnIg

“这个问题可以从 JVM 的设计角度倒推。main 方法是 JVM 和 Java 程序之间的唯一入口契约。JVM 在启动时，并不知道你的类有什么实例、依赖什么构造参数，它需要一种最朴素、最直接的方式来调用这个方法，不依赖任何上下文。public、static、void 这三个修饰符就是这份契约的三条核心条款。

一、public：必须对外可见

main 方法是由 JVM 从外部调用的，不在你的类内部调用。如果它被声明成 private、protected 或者默认包可见，JVM 的类加载器在加载完你的类后，就找不到这个方法，启动时会直接抛出 NoSuchMethodError: main。

JVM 不看类的内部实现，只暴露给它的入口。所以 main 方法必须是 public，这是访问控制的最低保障。

二、static：不需要实例化就能调用

JVM 启动时，只有类定义被加载，没有任何对象实例。如果 main 不是 static，它就是一个实例方法，JVM 必须先想办法创建一个对象才能调用它。但创建对象又需要调构造函数——要是你的类没有无参构造，或者构造函数里依赖了数据库连接、配置文件等外部资源，JVM 就会直接卡死在启动阶段，连错误都打印不出来。

把 main 设成 static，就等于告诉 JVM：“你不用管我是谁，直接通过类名就能调用，不需要先造一个我。”这是对调用方最友好的方式。

三、void：不需要给 JVM 返回任何值

main 方法执行完毕，程序就结束了。JVM 本身不依赖任何返回值来决定后续行为，它只关心进程的退出码。而退出码是通过 System.exit(int) 来控制的，不是通过方法返回值。所以 main 方法声明为 void 最直接。

如果 JVM 真的需要返回值，那它也完全可以约定返回 int，但 Java 的设计者选择了更简洁的方式——通过系统调用来传递退出状态，这样 main 方法的使命就很单纯：启动并运行，直到结束。

四、main：约定好的方法名

main 是 JVM 规范里写死的入口点名称。你当然可以在类里写其他叫 main 的重载方法，但 JVM 只认 public static void main(String[] args) 这个签名。如果你不小心写成了 Main（大写），或者参数用了 String...，JVM 都会找不到。

五、String[] args：接收外部参数的唯一通道

程序的启动参数，比如 java MyApp arg1 arg2，就是通过这个字符串数组传进来的。没有它，程序就失去了和启动命令交互的能力。

#### 如果去掉某个修饰符，会发生什么？

```
代码块
正常声明: public static void main(String[] args) → JVM 正常启动 ✅
去掉 public: static void main(String[] args)
  → JVM: NoSuchMethodError: main ❌
去掉 static: public void main(String[] args)
  → JVM: 类没有实例，无法调用，错误信息不一，通常启动失败 ❌
改成 int: public static int main(String[] args)
  → JVM 找不到 main 方法 ❌ (签名不匹配)
方法名写错: public static void Main(String[] args)
  → JVM 找不到 main 方法 ❌ (大小写敏感)
```

#### 核心原因速记表

> （此处为内嵌电子表格，需登录飞书查看）
