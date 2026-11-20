# JDK 9中对字符串的拼接做了什么优化？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/L1n9wXm7KiFYSVktH6WcgDtInVg

#### 面试回答

面试官： JDK 9 中对字符串拼接做了什么优化？

你：
JDK 9 做了一个很彻底的优化，把字符串拼接从“面向对象调用”变成了“字节码指令加动态计算”。简单说，以前 "a" + "b" 编译后是 new StringBuilder().append("a").append("b").toString()，JDK 9 开始直接用 invokedynamic 指令，编译期只写一个指令，具体怎么拼由运行时动态决定。

---

JDK 8 及以前是怎么拼的？

编译器遇到 + 号，直接翻译成一串 StringBuilder 调用。看起来没什么问题，但有几个硬伤：

1. 默认容量浪费：new StringBuilder() 默认容量 16，如果拼接的字符串总长度超过 16 个字符，底层 char[] 就得扩容一次甚至多次。
1. 固定绑定：编译后就钉死了是 StringBuilder，JVM 想换一种更高效的实现方式都没门。
1. 代码臃肿：复杂拼接会产生大量 append 方法调用，字节码很长。

JDK 9 做了什么改变？

编译期不再生成 StringBuilder，而是生成一条 invokedynamic 指令。这条指令的意思是“我需要拼接这些字符串，具体怎么拼最快，你 JVM 看着办”。真正干活的是 StringConcatFactory.makeConcatWithConstants，它在运行时会根据实际参数类型和数量，动态生成一套最优策略。这套策略有几个显著优化：

- 直接分配精确长度的 byte[]：它先计算最终字符串的精确长度，一次分配到位，彻底消灭扩容。
- 直接操作底层数组：不再通过 StringBuilder.append 一层层调，而是直接往 byte[] 里塞字符，避免 StringBuilder 对象的创建和回收。
- 预计算长度再复制：整个拼接过零拷贝，没有中间 StringBuilder.toString() 时那个 Arrays.copyOf 的额外开销。

最终效果就是： 性能提升明显，GC 压力更小，而且保留了运行时动态优化的空间，未来 JVM 还可以换更高效的拼法，代码不用重编译。

---

JDK 8 vs JDK 9 字符串拼接

![image](../../../../assets/GPsSd3um4osbm8x39pGcouARnRg/GsB1bV08DoqaJ9x8FIHc1JhZnub.png)

总结： 从 StringBuilder 的固定翻译，变成了 invokedynamic 的动态优化，性能更好，扩展性更强，字节码更精简。这就是 JDK 9 给字符串拼接带来的变化。
