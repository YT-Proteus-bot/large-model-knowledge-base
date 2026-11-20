---
title: Lambda表达式是如何实现的？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/G5AqwhTOOihhmlkLe7gc9UUlnac
source_path: AI Agent+后端精选八股文/语言基础篇/Java/Lambda表达式是如何实现的？.md
questions: 0
chars: 739
build_date: '2026-11-20'
images_removed: 1
---

# Lambda表达式是如何实现的？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/Lambda表达式是如何实现的？.md

## 面试回答

面试官： Lambda 表达式底层是怎么实现的？和匿名内部类一样吗？

你：

完全不一样。表面看 Lambda 就是匿名内部类的简化写法，但底层实现是两条路。匿名内部类编译后会生成单独的 class 文件，Lambda 不会，它用的是 invokedynamic 指令，在运行时动态生成实现类，性能更好，加载更轻量。

匿名内部类是“编译期写死”。你写一个 new Runnable(){...}，javac 会生成一个 类名$1.class，里面有完整的构造方法和 run() 实现。类是固定的，加载就是这一套，没法优化。

Lambda 表达式是“运行时动态链接”。javac 只生成一条 invokedynamic 指令，告诉 JVM：此处需要一个函数式接口的实现。真正干活的是 LambdaMetafactory，它在运行时通过 MethodHandle 和 ASM 直接生成一小段字节码，动态创建一个实现了接口的新类实例。它既可以利用 JVM 的优化做内联、逃逸分析，又避免了额外 class 文件的加载和初始化开销。这种机制称为“bootstrap method”。

为什么这样做更快？

- 不用编译期生成大量 class 文件，减少编译产物体积。
- 运行时可利用 JIT 深度优化，甚至可以内联到调用者的循环里。
- 避免了匿名内部类对 this 的捕获差异，Lambda 的 this 指向外部类，不需要额外生成访问外部实例的域。

匿名内部类是编译期的死板实现，Lambda 是运行时通过 invokedynamic + LambdaMetafactory 动态生成的高效实现。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
