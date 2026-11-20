---
title: Integer a=1000，Integer b=1000，==是什么结果，如果是100呢？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/OqYOwukTvi9DCmkCuk3cQYqUnXd
source_path: AI Agent+后端精选八股文/语言基础篇/Java/Integer a=1000，Integer b=1000，==是什么结果，如果是100呢？.md
questions: 0
chars: 807
build_date: '2026-11-20'
images_removed: 1
---

# Integer a=1000，Integer b=1000，==是什么结果，如果是100呢？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/Integer a=1000，Integer b=1000，==是什么结果，如果是100呢？.md

## 面试回答

面试官： Integer a=1000，Integer b=1000，a == b 的结果是什么？如果换成 100 呢？

你：

结论很明确：1000 时是 false，100 时是 true。 核心原因在于 Java 的 整型常量池（IntegerCache），范围是 -128 ~ 127。

为什么会这样？

写 Integer a = 1000 时，编译器会调用 Integer.valueOf(1000) 来做自动装箱。这个方法并不是无脑 new 新对象，它会先检查你要的值是不是在缓存范围 -128 到 127 之间。如果在，直接返回缓存里的现成对象；如果不在，才 new 一个新的 Integer 对象。

- 1000 的情况：超出缓存范围。所以 a 和 b 是两个不同的 new Integer(1000) 对象。== 比较的是内存地址，两个不同对象自然返回 false。
- 100 的情况：在缓存范围内。a 和 b 拿到的都是常量池里同一个缓存的 Integer 对象，地址相同，返回 true。

时序图直接演示这个过程：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

补充几点：

- 缓存上限可以调：通过 JVM 参数 -XX:AutoBoxCacheMax=size 可以调高缓存上限，但一般没必要。
- 面试常问的“连环炮”：除了 Integer，Long、Short、Byte、Character 也有类似缓存，范围同样是 -128 ~ 127。Float 和 Double 则没有缓存，因为浮点数的“常用值”太难定义了。
- 用 equals 永远安全：只要进行等值比较，绝对要用 equals()，它比较的是数值而不是地址，不会掉进缓存坑。

一句话记死：Integer 缓存池 -128~127，范围内的用 == 是同一个对象，范围外的不是。
