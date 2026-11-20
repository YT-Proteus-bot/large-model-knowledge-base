---
title: 为什么Java不支持多继承？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/SMilwTsV1izC00kFaVvcH8PUnOf
source_path: AI Agent+后端精选八股文/语言基础篇/Java/为什么Java不支持多继承？.md
questions: 0
chars: 290
build_date: '2026-11-20'
images_removed: 1
---

# 为什么Java不支持多继承？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/为什么Java不支持多继承？.md

## 面试回答模板

面试官： 为什么Java不支持多继承？

你：

主要怕菱形继承的歧义，让代码复杂度失控。C++允许多继承，但容易搞出“一个类从两条路继承同一个祖先，调用同名方法时不知道用哪个”的问题，Java干脆一刀切。

Java用单继承+多接口来平衡。接口在Java 8之前只能定义行为，没有实现，不会冲突。Java 8之后接口有了默认方法，存在冲突时，编译器强制子类必须自己重写，把选择权交给开发者，避免隐式歧义。

一句话：多继承省了代码，但增加了不确定性；Java选择用更清晰的单继承+接口来保证可维护性。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
