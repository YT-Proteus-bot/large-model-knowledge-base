---
title: 泛型中上下界限定符extends 和 super有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/R211wbeWmiPvlqkFCtRcTBxrnlf
source_path: AI Agent+后端精选八股文/语言基础篇/Java/泛型中上下界限定符extends 和 super有什么区别？.md
questions: 0
chars: 746
build_date: '2026-11-20'
images_removed: 1
---

# 泛型中上下界限定符extends 和 super有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/泛型中上下界限定符extends 和 super有什么区别？.md

## 面试回答

面试官： 泛型里的 <? extends T> 和 <? super T> 有什么区别？

你：

核心就是读写权限的差别，业内有个口诀叫 PECS（Producer Extends, Consumer Super）。

- ? extends T 是上界通配符，代表 T 或它的任意子类。这种集合只能往外读，不能往里写（除了 null）。因为编译器只知道里面装的是“某种 T 的子类”，但你往里放一个 T，它不一定是这个具体的子类型，所以写操作不安全。但读出来的东西一定可以当作 T 来用。
- ? super T 是下界通配符，代表 T 或它的任意父类。这种集合只能往里写 T 或其子类，读取只能当 Object。因为集合的实际类型可能是 T 的某个父类，你放一个 T 的子类进去一定是安全的（向上转型）。但取出来时，编译器只知道它是“某个 T 的父类”，具体是啥不确定，只能当作 Object。

实战场景：

- 你写一个方法，想遍历一个集合里的元素，只读不写，参数就用 List<? extends T>。
- 你写一个方法，想把数据放进集合，只写不读（或很少读），参数就用 List<? super T>。

一个经典例子：Collections.copy(List<? super T> dest, List<? extends T> src)，src 是生产者（extends），dest 是消费者（super）。

Mermaid 时序图：编译器对 extends 和 super 的读写校验

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

一句话：extends 适合生产者（读），super 适合消费者（写）。 记住这个，泛型方法设计不会乱。
