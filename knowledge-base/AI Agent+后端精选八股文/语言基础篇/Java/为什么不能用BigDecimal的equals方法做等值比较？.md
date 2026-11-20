---
title: 为什么不能用BigDecimal的equals方法做等值比较？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/Y1hZwUS86im61JkamCqcSo80nYf
source_path: AI Agent+后端精选八股文/语言基础篇/Java/为什么不能用BigDecimal的equals方法做等值比较？.md
questions: 0
chars: 355
build_date: '2026-11-20'
images_removed: 1
---

# 为什么不能用BigDecimal的equals方法做等值比较？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/为什么不能用BigDecimal的equals方法做等值比较？.md

## 面试回答

面试官： BigDecimal 的 equals 做等值比较有什么坑？

你：

坑在 equals 不光比数值，还比精度（标度）。比如 1.0 和 1.00，数值一样，但一个标度1，一个标度2，equals 直接判 false。金额比较，我们只关心数值，不关心小数点后有几位零，所以要用 compareTo，只比数值，标度忽略。

此外还有一个坑：new BigDecimal(0.1) 和 new BigDecimal("0.1") 的 equals 也大概率不等，因为 double 构造的 0.1 实际存的是近似值 0.10000000000000000555… 标度还巨大。所以使用 BigDecimal 时最好用字符串构造。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
