---
title: String中intern的原理是什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/WLJGwYWzhivO1ukqeshc20Jknrb
source_path: AI Agent+后端精选八股文/语言基础篇/Java/String中intern的原理是什么？.md
questions: 1
chars: 1659
build_date: '2026-11-20'
images_removed: 0
---

# String中intern的原理是什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/String中intern的原理是什么？.md

### 一句话说清楚它是干嘛的

intern() 就是一个“去字符串常量池里报到”的方法。

调了它之后，JVM 会保证：拿到的引用一定指向常量池里的那个唯一的字符串对象。

### 先备点基础：字符串常量池

Java 里用双引号写的字符串（字面量）会自动扔进一个叫 字符串常量池 的特殊区域。池子的作用就是复用，省内存、方便 == 比较。

从 JDK 7 开始，这个常量池从永久代搬到了堆里，和普通对象混居，但是有自己专用的数据结构——StringTable（本质就是个哈希表）。

### intern() 的工作流程（核心原理）

你可以把 intern() 理解成两步：

1. 去 StringTable 里查 —— 等价于拿当前字符串去常量池 equals 检查是否已存在一个一样的值。
1. 分情况处理：
- ✅ 如果池子里已经有这个串 → 直接返回池里的引用（不管外面新建了多少个）。
- ❌ 如果池子里还没有 → 那就把当前这个字符串对象“登记”到池子里，并返回这个引用。

⚠️ 关键就在第二步的“登记方式”上，JDK 版本不同，行为完全不一样。

### 版本差异 — 面试必杀题

面试官一般会追着问：“JDK 6 和 JDK 7 以后有什么不同？” 这块答出来，高分稳了。

> 〔原文此处为飞书内嵌组件，内容未随导出〕

🔥 一句话记忆：JDK 6 是真“搬家复制”，JDK 7 是“登记户口，人还住原地”。

### 用代码把原理打出来

下面这个例子面试可以直接写出来讲，说服力拉满：

```
String s1 = new String("hello");   // "hello" 字面量已经在常量池了
String s2 = s1.intern();           // 返回常量池里已有的引用
String s3 = "hello";               // 直接指向常量池
System.out.println(s1 == s2);      // false  (s1 是堆里新对象)
System.out.println(s2 == s3);      // true   (都是常量池那个)
```

再看看 JDK 7+ 的经典坑：

```
String s4 = new String("jo") + new String("vy");   // "jovy" 只在堆里，常量池没有
String s5 = s4.intern();   // 把 s4 的引用“登记”进常量池
String s6 = "jovy";        // 字面量去常量池取，拿到的就是刚才登记的 s4 引用
System.out.println(s4 == s5);   // JDK7+ true，JDK6 false
System.out.println(s5 == s6);   // true
```

这里能看出 JDK 7 的 “登记引用” 策略：省了一次复制对象的开销。

## 题1：底层长什么样？

intern() 是 native 方法，底层维护的就是前面提到的 StringTable。

你可以把它想象成一个固定大小的 HashSet<String>，通过 -XX:StringTableSize 调大小（JDK 6 默认 1009，JDK 8 默认 60013）。

当表太小时，链表会变长，intern 性能下降，所以高并发下有时会调大它。

### 常见的坑 & 注意事项

- 别滥用 intern：虽然能省内存，但往 StringTable 里塞太多不同字符串会导致表膨胀，查询变慢，还可能因数据太多引发 GC 停顿。
- “java” 这个串天生就在池子里：JVM 启动时会自动把一些关键字放进去，你测试时要注意别被误导。
- 永远别用 == 代替 equals 写业务逻辑，intern 只是优化手段，不是可靠性保证。
