---
title: String为什么设计成不可变的？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Ia6CwdBpSiQImakkQ2qcuFW7nAf
source_path: AI Agent+后端精选八股文/语言基础篇/Java/String为什么设计成不可变的？.md
questions: 0
chars: 322
build_date: '2026-11-20'
images_removed: 1
---

# String为什么设计成不可变的？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/String为什么设计成不可变的？.md

## 面试回答

面试官： String 为什么设计成不可变的？

你：

主要是安全、高效、并发这三点。

第一，安全。 字符串常量池里的东西，比如类名、数据库连接密码，如果谁都能改，一个地方改了，其他引用全跟着变，系统就崩了。不可变能防止这种注入和篡改。

第二，线程安全。 不可变对象天生就是共享安全的，多个线程同时读同一个字符串，不用加锁。

第三，性能优化。 不可变性使得 hash 值可以缓存，做 HashMap 的 key 时取 hash 很快。另外常量池可以共享同一个字符串对象，省内存。

简单说，如果 String 可变，上面的好处全没了，还会带来莫名其妙的 bug。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
