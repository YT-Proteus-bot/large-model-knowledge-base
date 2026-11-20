---
title: 为什么JDK 9中把String的char[]改成了byte[]？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/N53SwZXbeikhnEkT1ZPc1u1Mn9c
source_path: AI Agent+后端精选八股文/语言基础篇/Java/为什么JDK 9中把String的char[]改成了byte[]？.md
questions: 0
chars: 1501
build_date: '2026-11-20'
images_removed: 1
---

# 为什么JDK 9中把String的char[]改成了byte[]？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/为什么JDK 9中把String的char[]改成了byte[]？.md

“这个改动是 Java 在内存优化上的一次非常务实的手术。JDK 团队统计发现，绝大多数 Java 应用里的字符串都是拉丁字母、数字和英文标点，这些字符只需要 1 个字节就能表示。但 JDK 8 及以前，String 底层固定用 char[] 存储，每个 char 是 2 字节，这相当于给每个字符都分配了‘双倍仓位’，造成了巨大的内存浪费。JDK 9 就把底层换成了 byte[]，并加了一个编码标记，让字符串可以根据实际内容动态选择 1 字节或 2 字节存储，这就是紧凑字符串。

我拆成四层讲清楚这个优化。

🔹 第一层：原来的 char[] 问题在哪？

在 JDK 8 中，String 的内部结构大致是：

```
private final char value[];
```

char 在 Java 里固定占 2 字节，存一个 "Hello" 需要 5 个 char，也就是 10 字节。但实际上，ASCII 字符的高 8 位全是 0，完全没用上。一个典型 Java 应用里，超过 90% 的字符串都只包含 Latin-1 可表示的字符（即 \u0000 到 \u00FF），这意味着堆里大量字符串有一半的内存被浪费了。字符串是堆内存的头号占用大户，优化它就是优化整个应用。

🔹 第二层：JDK 9 的新结构 byte[] + coder

JDK 9 把 String 的内部实现改成了：

```
private final byte[] value;   // 实际存储字节
private final byte coder;     // 编码标识
```

- value 现在是一个 byte 数组，可以是 Latin-1 编码（每个字符 1 字节），也可以是 UTF-16 编码（每个字符 2 或 4 字节）。
- coder 是一个标志位，取 LATIN1（0）或 UTF16（1）。

创建字符串时，构造器会扫描所有字符：

- 如果所有字符都在 Latin-1 范围内，就用 Latin-1 编码，value 长度等于字符数。
- 如果存在超出 Latin-1 的字符（如中文、emoji），则用 UTF-16，value 长度为字符数 × 2（必要时用代理对处理增补字符）。

这样，一个 "Hello" 就只需要 5 个字节，一个 "你好" 依然需要 4 个字节（UTF-16）。该省的地方省，省不了的地方不亏。

🔹 第三层：内存和性能的收益

根据官方给出的数据，这个改动让典型应用的堆内存占用下降了 10% 到 20%，同时减少了 GC 频率。因为字符串相关的对象（包括 String 本身和底层的 byte[] 数组）体积更小，Eden 区能装更多对象，Full GC 的次数也相应减少。

性能上，有些操作会多一个分支判断（比如 charAt 需要根据 coder 决定是直接读一个字节然后转 char，还是按 UTF-16 解码），但这点开销极小，现代 CPU 的分支预测完全可以消化。整体上，内存的节省带来的 GC 收益远超过这点逻辑开销。

🔹 第四层：API 完全兼容，内部透明

这个改动完全在 String 内部实现，对开发者零感知。length() 依然返回字符数（内部会根据 coder 和数组长度换算），charAt()、substring() 等方法的语义也完全不变。这意味着你升级到 JDK 9 后，不用改一行代码，就能享受到内存节省的好处。

## 内部结构变化示意图

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
