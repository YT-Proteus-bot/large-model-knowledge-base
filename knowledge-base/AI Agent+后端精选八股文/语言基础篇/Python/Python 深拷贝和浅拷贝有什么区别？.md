---
title: Python 深拷贝和浅拷贝有什么区别？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/OhJzwwTQvi6sXsk2al6cUevknQc
source_path: AI Agent+后端精选八股文/语言基础篇/Python/Python 深拷贝和浅拷贝有什么区别？.md
questions: 0
chars: 610
build_date: '2026-11-20'
images_removed: 0
---

# Python 深拷贝和浅拷贝有什么区别？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/Python 深拷贝和浅拷贝有什么区别？.md

这道题表面上问区别，实际在考你对 Python 对象模型、引用机制 的理解。直接上干货。

## 三句话定本质

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 代码不会骗人

```
import copy
old = [[1, 2], [3, 4]]
shallow = copy.copy(old)
deep = copy.deepcopy(old)

# 修改原对象的内层元素
old[0][0] = 'X'
print("浅拷贝:", shallow)   # [['X', 2], [3, 4]]   😱 被牵连了！
print("深拷贝:", deep)      # [[1, 2], [3, 4]]     😎 不动如山
```

背后的真相：

- shallow 复制了外层列表，但里面的小列表 [1,2] 还是同一个对象。
- deep 连小列表都复制了一份全新的，自此老死不相往来。

## 进阶加分项（亮出来我会加分）

- 全是不可变对象 → 浅拷贝表现“像”深拷贝，因为修改会创建新对象，但引用逻辑没变。
- 循环引用 → deepcopy 内部用 memo 字典防无限递归，能手写一个简单版说明你懂了精髓。
- 魔术方法 → copy / deepcopy 可以定制类的拷贝行为。

一句话总结：浅拷贝共享内侧，深拷贝全家独立。搞清引用，才算真正入门 Python 对象机制。 💪
