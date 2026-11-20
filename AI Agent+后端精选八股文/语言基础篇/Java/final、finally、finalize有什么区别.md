# final、finally、finalize有什么区别

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Y6LIwDDyrip5MAkOgqpcPQCpnnd

面试官问这三个词，大概率不是考你背概念，而是想看你有没有把语法特性和实际场景结合起来，顺便探探你对 Java 演进的理解。我按自己的认知习惯分三个独立模块拆一下。

🔹 final：限制你胡来的“封印”

它就是个修饰符，贴在类、方法、变量上，管的范围完全不一样：

> （此处为内嵌电子表格，需登录飞书查看）

⚠️ 一个容易掉的坑：final List<String> list = new ArrayList<>(); 你仍然能 list.add("A")，只是不能 list = new LinkedList<>() 而已。锁的是引用，不是对象肚子里的东西。

🔹 finally：出了事我也要打扫的“管家”
它是 try-catch-finally 里的兜底块，无论是否抛异常、是否 return，它通常都会执行（除非 System.exit() 或守护线程挂掉）。
早年写 JDBC 时，标准模板就是：

```
代码块
Connection conn = null;
try {
    conn = dataSource.getConnection();
    // 干活...
} catch (SQLException e) {
    // 处理...
} finally {
    if (conn != null) conn.close();  // 天塌了也得给我关上
}
```

现在有了 try-with-resources（实现 AutoCloseable），绝大多数 finally 释放资源的场景都可以优雅替代，但 finally 在非资源清理（比如必须要执行的一段收尾逻辑）时还是刚需。

🔹 finalize：一个几乎被官方判了“死刑”的“幽灵”
它是 Object 类里的 protected 方法，在对象被 GC 回收前由垃圾收集器调用一次。听起来像是“临终关怀”，实则是个大坑：

- ❌ 执行时机完全不确定：GC 什么时候跑，它什么时候才能被调用，你可能等到天荒地老。
- ❌ 搞不好让对象“复活”：如果在 finalize 里又把 this 赋值给一个 GC Root，对象就不死了，这会导致对象多次进出“临死状态”，严重拖垮 GC 性能。
- ❌ 性能极差：JVM 要专门为有 finalize 的对象建立队列，走一套重型的 Finalizer 线程处理。

⚠️ 注意这个演进：Java 9 开始把它标记 @Deprecated，Java 18 的 JEP 421 进一步把它标记为 deprecated for removal（正式拉黑，后续版本会移除）。现在替代方案要么是 java.lang.ref.Cleaner，要么是 try-with-resources 主动管理。说实话，我上次看到它还是在七八年前的“面试宝典”里，生产代码里如果还有 finalize，直接进重构清单。

🔹 一张图帮你永远记住区别

> （此处为内嵌电子表格，需登录飞书查看）

🗣️ 这仨单词看起来像失散多年的三兄弟，其实完全活在三个宇宙里：一个管设计约束，一个管异常收尾，一个管垃圾回收（而且已经被时代抛弃了）。下次再被问到，你就说：“final 是限制，finally 是兜底，finalize 是遗言——但现代 Java 里，你该用的只有前两个，第三个最好永远别出现在你的代码里。”
