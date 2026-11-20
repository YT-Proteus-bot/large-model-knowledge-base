---
title: 什么是SPI，和API有啥区别
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Bcpdwb0Iziylzjkzbjsc9QYEnHc
source_path: AI Agent+后端精选八股文/语言基础篇/Java/什么是SPI，和API有啥区别.md
questions: 0
chars: 1049
build_date: '2026-11-20'
images_removed: 1
---

# 什么是SPI，和API有啥区别

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/什么是SPI，和API有啥区别.md

## 面试回答

面试官： 什么是 SPI？和 API 有啥区别？

你：

很多人搞混，其实用一句话就能区分：API 是你调别人的接口，SPI 是别人调你的实现。

SPI 全称 Service Provider Interface，是一种服务发现机制。

核心思想是：标准定义在核心库里，具体实现由第三方厂商提供，程序在运行时动态加载所有实现。最典型的例子就是 JDBC。你写的代码只依赖 java.sql.Driver 接口，不依赖任何具体数据库驱动。当你把 MySQL 的驱动 jar 包丢到 classpath 下，DriverManager 会自动通过 SPI 找到 com.mysql.cj.jdbc.Driver 并加载它，你的代码不用改一行。

SPI 的运作分三环：

1. 标准 API：JDK 或框架定义接口（如 java.sql.Driver）。
1. 实现方：第三方厂商写实现类，并在 META-INF/services/接口全限定名 文件里填上实现类的全限定名。
1. 调用方：核心库用 ServiceLoader.load(接口.class) 加载所有实现，循环调用。

API 和 SPI 的本质区别是调用方向：

- API：接口定义方也是实现方，比如你写的 UserService，你定义接口也实现它，别人调你的接口。调用方依赖接口和实现方。
- SPI：接口定义方是“平台方”，只定义接口规范，不实现（或只给默认实现）。具体实现由“扩展方”提供，平台方用接口调扩展方的实现。调用方向反了。

一个拔高点：双亲委派破坏的典型场景。

SPI 的核心类（如 DriverManager）在 rt.jar 由 Bootstrap ClassLoader 加载，但它要回调由应用类加载器加载的实现类（如 com.mysql.jdbc.Driver）。根据双亲委派，父加载器加载的类没法直接调用子加载器加载的类，所以 ServiceLoader 内部用线程上下文类加载器（Thread Context ClassLoader）来打破这个限制。这是面试经常连着问的点。

Mermaid 时序图：API 与 SPI 的调用方向对比

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

总结一句话：

你写 JDBC 时只管调 DriverManager，MySQL 的 Driver 怎么被找到的？就是 SPI 在干活——平台定义接口，实现方扩展，运行时动态发现。
