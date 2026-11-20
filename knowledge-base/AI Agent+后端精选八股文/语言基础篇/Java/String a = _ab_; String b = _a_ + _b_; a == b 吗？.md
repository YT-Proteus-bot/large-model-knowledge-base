---
title: String a = "ab"; String b = "a" + "b"; a == b 吗？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/XIIOwC9HgiRflekQKyCcuwM7nrb
source_path: AI Agent+后端精选八股文/语言基础篇/Java/String a = _ab_; String b = _a_ + _b_; a == b 吗？.md
questions: 0
chars: 156
build_date: '2026-11-20'
images_removed: 1
---

# String a = "ab"; String b = "a" + "b"; a == b 吗？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/String a = _ab_; String b = _a_ + _b_; a == b 吗？.md

## 面试回答

面试官： String a = "ab"; String b = "a" + "b"; a == b 吗？

你：

是 true。 因为编译期会做常量折叠，"a" + "b" 直接优化成 "ab"，两个引用指向常量池里同一个字符串对象。

> 〔图片：原文此处为图片或公式截图，未随文档导出〕
