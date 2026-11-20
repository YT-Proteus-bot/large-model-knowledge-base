---
title: 什么是AIO、BIO和NIO？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/UwEjwBQhzi1UKjko4G4cxdESnuO
source_path: AI Agent+后端精选八股文/语言基础篇/Java/什么是AIO、BIO和NIO？.md
questions: 0
chars: 943
build_date: '2026-11-20'
images_removed: 1
---

# 什么是AIO、BIO和NIO？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/什么是AIO、BIO和NIO？.md

## 面试回答

面试官： 说说你对 BIO、NIO、AIO 的理解，它们有什么区别？

你：

用餐厅服务员来比喻就很好懂。

- BIO（Blocking I/O）：一个服务员专门服务一桌客人，从点菜到结账，全程死等在这一桌上。客人没想好点什么菜，服务员就一直杵着啥也干不了。一个连接就得一个线程伺候，连接多了线程数爆炸，CPU都花在上下文切换上了。
- NIO（Non-blocking I/O）：一个服务员管好几桌，手持对讲机，不停巡视。哪桌客人招手（有数据可读），就去服务一下，服务完立刻换下一桌。不用傻等，一个线程就能处理大量连接。但服务员自己得一直巡台，有点累。
- AIO（Asynchronous I/O）：更高级，每桌配一个按铃，客人按铃（数据就绪），后台直接通知服务员去处理。服务员不用巡台，写完菜单后直接去干别的，系统回调通知结果。资源利用率最高，但实现复杂，操作系统支持也要好。

技术层面的核心差异：

- BIO：ServerSocket.accept() 阻塞等连接，InputStream.read() 阻塞等数据。每连接一线程。
- NIO：用 Selector 多路复用，一个线程轮询多个 Channel，只有就绪的才处理。非阻塞模式，配合 Buffer 使用。
- AIO：基于事件回调，AsynchronousSocketChannel 发起读/写后立刻返回，操作系统在数据就绪后主动通知应用程序处理。

适用场景：

- BIO：连接数少且固定、要求简单稳定的场景，比如传统的 ERP、小规模企业内部系统。
- NIO：高并发、连接多但多数是短连接或 IO 密集型，比如聊天服务器、网关。Netty 就是基于 NIO 的框架。
- AIO：连接数很多且数据量大的场景，比如一些大型云存储、文件传输服务。实际落地 NIO 更多，因为 AIO 相对复杂，且 Linux 下真正的异步 IO 支持不如 Windows 完善，所以 Netty 也主要基于 NIO。

时序图对比：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

一句话总结：BIO 最简单但资源浪费，NIO 是主流高并发基石，AIO 是终极理想但生态还未完全取代 NIO。
