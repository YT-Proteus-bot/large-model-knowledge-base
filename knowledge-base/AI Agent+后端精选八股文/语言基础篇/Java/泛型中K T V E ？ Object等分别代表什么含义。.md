---
title: 泛型中K T V E ？ Object等分别代表什么含义。
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/LcFlwGz5Mic2oek7RZBcOYNgndP
source_path: AI Agent+后端精选八股文/语言基础篇/Java/泛型中K T V E ？ Object等分别代表什么含义。.md
questions: 0
chars: 766
build_date: '2026-11-20'
images_removed: 1
---

# 泛型中K T V E ？ Object等分别代表什么含义。

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/泛型中K T V E ？ Object等分别代表什么含义。.md

## 面试回答

面试官： 泛型里经常看到 <K, V>、<T>、<?> 这些，它们分别代表什么？

你：

这些字母本质上是类型参数的占位符，类似于方法里的形参。用哪个字母没有语法强制，但业界有一套约定俗成的含意：

- T：Type，最通用的类型。当你只关心“某个类”而不限定它是什么时用，比如 Box<T>、Comparable<T>。
- E：Element，集合里的元素，比如 List<E>、Set<E>。
- K / V：Key / Value，键值对，最典型的就是 Map<K, V>。
- ?：未知的通配符，表示“某种不确定的类型”，主要用于限制读写权限。比如 List<?> 可以接受 List<String>、List<Integer> 等，但不能往里 add 元素（除 null）。
- Object：实实在在的根类。List<Object> 是具体参数化的集合，可以往里面放任何对象，但失去了编译期的类型约束。

最容易混淆的是 ? 和 Object：

- List<Object> 是“一个能装任意对象的列表”，你可以随意 get 和 add。
- List<?> 是“一个元素类型未知的列表”，我只知道它是某个类型的列表，但不知道具体是啥。所以从里面取出来的只能当 Object，而且除了 null 不能往里放任何东西，因为编译器没法保证你放进去的类型匹配。

另外，? 还有上下界：? extends T 限定是 T 或其子类，常用于“生产者”（读取）；? super T 限定是 T 或其父类，常用于“消费者”（写入）。

时序图：泛型占位符在 Map 中的实际作用

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

记住，字母只是习惯，真正决定行为的是边界（上下界）和通配符的读写限制。
