---
title: SimpleDateFormat是线程安全的吗？使用时应该注意什么？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/PO8Cw8O02idDpfkZclDckDVYntc
source_path: AI Agent+后端精选八股文/语言基础篇/Java/SimpleDateFormat是线程安全的吗？使用时应该注意什么？.md
questions: 0
chars: 952
build_date: '2026-11-20'
images_removed: 1
---

# SimpleDateFormat是线程安全的吗？使用时应该注意什么？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/SimpleDateFormat是线程安全的吗？使用时应该注意什么？.md

## 面试回答

面试官： SimpleDateFormat 是线程安全的吗？使用时候要注意什么？

你：

不是线程安全的，而且这个是经典的线上事故多发点。SimpleDateFormat 内部持有一个 Calendar 对象，在解析或格式化时，会修改这个 Calendar 字段，多个线程同时调用同一个 SimpleDateFormat 实例，Calendar 被并发修改，就会出现解析错乱、结果乱串，甚至抛出 NumberFormatException 或 ArrayIndexOutOfBoundsException。

为什么会出事？

SimpleDateFormat 的 parse 方法里会调用 calendar.clear() 清空，再设置时间，这中间没有加锁。如果线程A刚 clear 完，还没来得及设置，线程B又 clear 了一次，A 后续操作拿到的是一个空或半截的 Calendar，结果就完全不可预期。

那该怎么办？有三种安全方案：

1. 每个线程一个实例

   最简单直接，用 ThreadLocal 给每个线程配一个 SimpleDateFormat 实例，彻底隔离，互不干扰。但要注意线程池下 ThreadLocal 用完要清理，否则可能内存泄漏。

1. 每次 new 一个新实例

   如果频率不高（比如定时任务每天跑几次），完全可以 new SimpleDateFormat("yyyy-MM-dd") 用完丢弃，代码简单明了，没有共享，绝对安全。

1. 用 JDK8 的 DateTimeFormatter

   这是现在最推荐的。DateTimeFormatter 是不可变对象，天然线程安全，可以安全地定义为全局常量，多线程随便调用，毫无副作用。而且 API 设计更清晰，支持链式调用，比 SimpleDateFormat 的月/星期/小时计算靠谱得多。

补充一个坑： SimpleDateFormat 即使定义了静态 final 变量，也挡不住多个线程同时使用，因为不可变修饰只意味着引用不可变，内部状态照样可变。很多新手以为 final static 就安全了，结果线上各种诡异错误。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
