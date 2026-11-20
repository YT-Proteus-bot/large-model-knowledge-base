# String str=new String("hollis")创建了几个对象？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/OEO2wT7TBiJZCckQxPgc4hLUnth

#### 面试回答模板

面试官： String str = new String("hollis") 创建了几个对象？

你：

这题考的是字符串常量池。分两种情况：

情况一：常量池中还没有 "hollis"
创建两个对象。编译时，字面量 "hollis" 会先在字符串常量池里创建一个对象。运行时 new String(...) 再在堆里创建一个 String 对象，这个堆对象内部引用常量池的字符数组。所以一共是两个。

情况二：常量池中已经有 "hollis" 了（比如之前代码用过这个字符串）
只创建一个对象，就是堆里 new 出来的那个，常量池那个是复用已有的。

一句话：第一次出现时是两个，之后再写同样的就只是一个。

![image](../../../../assets/WFq9dJrbYoZGhKxf9fvcfws2nDh/LIeXbmLp3oCv8KxVseScB1mqnid.png)
