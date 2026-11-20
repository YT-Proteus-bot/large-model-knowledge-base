# 为什么Java中的main方法必须是public static void的？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SPYNw5jCWiFbtPkGbctcJG0unNR

#### 面试回答

面试官： 为什么 Java 的 main 方法必须是 public static void？

你：

这个是由 JVM 的启动机制和设计约定共同决定的。每一个修饰符都有它非这样不可的理由，缺一不可。

第一，必须是 public —— 因为 JVM 是外部调用者。
main 方法是程序的入口，但调用它的不是你自己写的类，而是外部的 Java 虚拟机。JVM 不属于该类所在的包，如果 main 不是 public，JVM 会因访问权限不足而找不到入口，直接抛 NoSuchMethodError。

第二，必须是 static —— 因为 JVM 启动时还没有任何对象。
JVM 加载完类之后，还没有来得及通过 new 创建任何实例，此时必须能通过类名直接调用方法。如果 main 是实例方法，就需要先实例化，但谁来实例化？这会产生“先有鸡还是先有蛋”的问题。所以必须声明为 static，让 JVM 以 ClassName.main(args) 方式调用。

第三，必须是 void —— 因为 JVM 不需要程序的返回值。
程序的结束是通过 main 线程的结束或 System.exit(n) 来终止 JVM 的，而不是 main 方法的返回值。如果 main 返回一个 int，JVM 并不知道该拿这个值做什么，又该交给谁。如果需要传递结束状态给操作系统，使用 System.exit 就好。

第四，方法签名必须是 main(String[] args)。
参数 String[] 用来接收命令行参数，这是规范的一部分。至于为什么不是 String... 等其他形式，JVM 规范严格约束了方法名和参数类型，只有 main(String[]) 才能被识别为入口。

能不能加其他修饰符？
比如 synchronized、final 或 strictfp 可以共存，但不会影响它作为入口的合法性。但 abstract 不行，因为 static 方法不能同时是 abstract。

所以总结：public 为了可见性，static 为了无需实例，void 为了无返回值，这三者是 JVM 冷启动时的必然要求。

---

JVM 调用 main 方法的过程

![image](../../../../assets/VNwNdDjjqopKhWxxcbScBfpWnZd/EixRbcsraobyjkx1q7zcHmhUnmb.png)
