---
title: while(true)和for(;;)哪个性能好？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/LeVzwPdKfi2jUVkV45kcjkV8n5f
source_path: AI Agent+后端精选八股文/语言基础篇/Java/while(true)和for(;;)哪个性能好？.md
questions: 0
chars: 547
build_date: '2026-11-20'
images_removed: 1
---

# while(true)和for(;;)哪个性能好？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/while(true)和for(;;)哪个性能好？.md

## 面试回答

面试官： while(true) 和 for(;;) 哪个性能好？

你：

性能完全一样，没有任何区别。

因为在编译之后，两者的字节码是完全相同的，都是一个无条件跳转指令 goto。所有差异都只停留在源码层面，被 javac 一编译就抹平了。

具体解释：

while(true) 在语法上需要一个布尔表达式，而 true 是编译期常量，编译器直接把它优化成无条件跳转。

for(;;) 的循环条件为空，Java 语言规范里明确说空条件等价于 true，所以也是无条件跳转。

你反编译一下就能看到，两个写法的字节码都是一个 goto 指令往回跳，循环体完全一致。JIT 编译器更不会区别对待，它只看到相同的 goto 结构，内联、循环展开等优化也完全一样。

那为什么大家还在纠结用哪个？

主要是代码风格和工具检查。有些静态分析工具会对 while(true) 报“条件始终为真”的警告，因为它在字节码层面相当于 ifeq 判断永远不成立，然后一个 goto。而 for(;;) 本身语法上就表示无限循环，通常不会有警告。另外 for(;;) 更短，看起来更“硬核”。两者都可以选，看团队习惯，跟性能没关系。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
