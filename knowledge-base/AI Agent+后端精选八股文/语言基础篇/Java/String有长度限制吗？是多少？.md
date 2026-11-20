---
title: String有长度限制吗？是多少？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/LrQfwPUpJiP9WnkGu5ScRYOCnnd
source_path: AI Agent+后端精选八股文/语言基础篇/Java/String有长度限制吗？是多少？.md
questions: 0
chars: 963
build_date: '2026-11-20'
images_removed: 1
---

# String有长度限制吗？是多少？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/String有长度限制吗？是多少？.md

## 面试回答

面试官： String 有长度限制吗？是多少？

你：

有限制，但分两种场景：编译期的常量限制和运行期的动态限制，数值差别很大。

编译期限制：字符串常量不能超过 65535 个字节

当你在代码里写一个字面量 String s = "很长很长的字符串..."，这个字符串会进常量池，编译后的 class 文件里用 CONSTANT_Utf8_info 结构存储。这个结构用 2 个字节 记录字符串的字节长度，也就是最大只能记 65535（0 ~ 65535）。所以编译期常量字符串的 UTF-8 编码后的字节数 不能超过 65535 个字节。

注意是字节数不是字符数。因为 Java 用的是一种类似 UTF-8 的变长编码（Modified UTF-8），英文字符占 1 字节，常见中文占 3 字节，一些特殊字符会用 6 字节。所以一个全中文的常量，大概只能存 65535 / 3 ≈ 21000 个字符左右。

运行期限制：理论上可以到 Integer.MAX_VALUE，实际受堆内存限制

运行时动态拼接的字符串，比如 new String(char[]) 或 StringBuilder 搞出来的，长度取决于 char[] 或 JDK9 后的 byte[] 数组。数组索引是 int 类型，所以理论最大长度是 Integer.MAX_VALUE（约 21 亿）。

但事实上你基本达不到这个值：

1. 数组长度上限：Java 数组最大长度实际是 Integer.MAX_VALUE - 5 左右，因为 JVM 会给数组头预留一些空间。
1. 堆内存限制：一个长度为 21 亿的 char 数组需要占用 2 * 2.1G ≈ 4.2GB 的连续内存，远超常规 JVM 堆设置，直接 OutOfMemoryError。
1. GC 停顿：如此巨大的数组会让 GC 停顿时间严重飙升。

补充一点：String 的 hashCode 是用 int 存的，但 API 设计上并没有基于 hashCode 来限制长度，所以这个不算硬约束。

String 的长度限制来源

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

写死代码的常量别超过 65535 个字节，动态拼的也别太狂，内存扛不住。
