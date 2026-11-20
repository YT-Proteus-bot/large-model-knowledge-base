#  String、StringBuilder和StringBuffer的区别？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/WgxywWi6pi1s7tkXAo7cEztVnqz

#### 面试回答模板

面试官： String、StringBuilder 和 StringBuffer 的区别是什么？

你：

核心区别就两点：可变性和线程安全。

- String 不可变，每次拼接都会在堆里创建一个新对象，产生大量临时垃圾，对 GC 压力大。适用于少量、不频繁改动的场景，比如常量定义。
- StringBuilder 可变，拼接是在原有字符数组上追加，不会反复产生新对象，性能远高于 String。但它线程不安全，适用于单线程方法内的局部变量。
- StringBuffer 也是可变，但它的方法全加了 synchronized，线程安全。代价是锁开销，吞吐量比 StringBuilder 低。适用于多线程环境下的共享变量。

一句话： 少量拼接用 String；单线程追求性能用 StringBuilder；多线程用 StringBuffer，或者用 ThreadLocal 包装 StringBuilder。
