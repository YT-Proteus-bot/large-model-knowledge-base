---
title: 有了equals为啥需要hashCode方法？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/GJycw4EXLifLCskW1EOcXs0Bnoc
source_path: AI Agent+后端精选八股文/语言基础篇/Java/有了equals为啥需要hashCode方法？ (2).md
questions: 0
chars: 722
build_date: '2026-11-20'
images_removed: 1
---

# 有了equals为啥需要hashCode方法？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/有了equals为啥需要hashCode方法？ (2).md

## 面试回答

面试官： 你已经有了 equals 方法，为什么还要 hashCode？

你：

这个问题问的是散列集合（HashMap、HashSet）的命根子。equals 负责判断两个对象是否逻辑相等，hashCode 负责把对象快速定位到桶里。如果只重写 equals 而不重写 hashCode，相同的对象会被塞到不同的桶，散列集合的“不可重复”就直接崩了。

合约关系必须死记：

- 两个对象 equals 相等 → hashCode 必须相等。
- hashCode 相等 → 对象不一定 equals 相等（允许哈希冲突）。
- 重写 equals 时，必须同时重写 hashCode。

为什么必须这样？

拿 HashSet 存自定义对象举例：Set 判定重复，先看 hashCode 桶里有没有东西，桶里没东西就直接放进去；桶里有东西再调 equals 精确比较。如果你只重写了 equals，用默认的 Object.hashCode()，两个逻辑相同的对象大概率算出不同的 hashCode，被扔进不同的桶，Set 根本不认为它们重复，于是你明明觉得是一样的对象，集合里却存进去好几个。

时序图说明问题：

> 〔图片：原文此处为图片或公式截图，未随文档导出〕

扩展一下： HashMap 用 key 的 hashCode 找桶，再 equals 找具体节点。如果 hashCode 没重写，你永远没法用另一个“内容相同”的 key 取出原来的值，因为桶都找不到，更别提 equals。

一句话：equals 用来比较是不是同一个人，hashCode 用来告诉你在哪个房间，房间都找错了，人当然就“丢失”了。
