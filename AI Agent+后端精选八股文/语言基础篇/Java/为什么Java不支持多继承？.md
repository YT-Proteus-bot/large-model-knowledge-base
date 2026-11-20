# 为什么Java不支持多继承？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SMilwTsV1izC00kFaVvcH8PUnOf

#### 面试回答模板

面试官： 为什么Java不支持多继承？

你：

主要怕菱形继承的歧义，让代码复杂度失控。C++允许多继承，但容易搞出“一个类从两条路继承同一个祖先，调用同名方法时不知道用哪个”的问题，Java干脆一刀切。

Java用单继承+多接口来平衡。接口在Java 8之前只能定义行为，没有实现，不会冲突。Java 8之后接口有了默认方法，存在冲突时，编译器强制子类必须自己重写，把选择权交给开发者，避免隐式歧义。

一句话：多继承省了代码，但增加了不确定性；Java选择用更清晰的单继承+接口来保证可维护性。

![image](../../../../assets/N7cmdwttAobbEMxDUnHcy8mjn1b/J2fkb1rGkoCH6rxVN63c46DKnnh.png)
