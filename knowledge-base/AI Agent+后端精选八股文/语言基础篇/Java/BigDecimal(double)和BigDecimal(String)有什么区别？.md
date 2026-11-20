---
title: BigDecimal(double)和BigDecimal(String)有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/FGW7woRpuifOX1kqnMXcF86BnPh
source_path: AI Agent+后端精选八股文/语言基础篇/Java/BigDecimal(double)和BigDecimal(String)有什么区别？.md
questions: 0
chars: 493
build_date: '2026-11-20'
images_removed: 1
---

# BigDecimal(double)和BigDecimal(String)有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/BigDecimal(double)和BigDecimal(String)有什么区别？.md

## 面试回答

面试官： BigDecimal(double) 和 BigDecimal(String) 有什么区别？

你：

核心就四个字：精度天差地别。

new BigDecimal(0.1) 你以为存的是0.1，实际上0.1在二进制里是无限循环小数，double只能存个近似值，所以实际存进去的是 0.1000000000000000055511151231...，带一堆尾数。

new BigDecimal("0.1") 直接从十进制字符串解析，存的就是精确的 0.1。

根源在于： double类型的0.1在二进制里本身就是有损的，BigDecimal用double构造时拿到的已经是失真数据了。而字符串构造走的是精确的十进制解析路径。

实际开发三个铁律： 第一，金额字段永远用字符串构造；第二，数据库里的DECIMAL类型，Java里直接用BigDecimal接收，MyBatis/JPA会自动处理；第三，既然用了BigDecimal就别再绕回double，toString也比doubleValue靠谱。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
