---
title: 为什么对Java中的负数取绝对值结果不一定是正数？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FDbRwJTWDi6w77kxK4pcrokWnLc
source_path: AI Agent+后端精选八股文/语言基础篇/Java/为什么对Java中的负数取绝对值结果不一定是正数？.md
questions: 1
chars: 457
build_date: '2026-11-20'
images_removed: 1
---

# 为什么对Java中的负数取绝对值结果不一定是正数？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/为什么对Java中的负数取绝对值结果不一定是正数？.md

## 题1：为什么对Java中的负数取绝对值结果不一定是正数？

面试回答

面试官： 为什么Java里对负数取绝对值，结果不一定是正数？

你：

这个坑出在整数溢出上。Java里int的范围是-2147483648到2147483647，正数比负数少一个。Integer.MIN_VALUE是-2147483648，你取它的绝对值应该是2147483648，但这个数已经超出了int能表示的最大值，溢出了，转了一圈变回-2147483648，还是负数。

根因在于补码不对称。 计算机用补码存整数，补码表示范围天然不对称，负数比正数多一个。所以绝对值函数Math.abs对绝大多数负数有效，唯独Integer.MIN_VALUE不行。Long同理，Long.MIN_VALUE的绝对值也溢出。

怎么避免？ 调用Math.abs之前先判断是不是Integer.MIN_VALUE，如果是就别直接取绝对值，或者先转成long再取，long范围宽，基本够用。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
