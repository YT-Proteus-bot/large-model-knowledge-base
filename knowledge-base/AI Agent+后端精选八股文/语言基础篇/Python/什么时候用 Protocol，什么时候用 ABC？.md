---
title: 什么时候用 Protocol，什么时候用 ABC？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Python
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/A2MewTHEViWsX6kuALYcus2AnJ9
source_path: AI Agent+后端精选八股文/语言基础篇/Python/什么时候用 Protocol，什么时候用 ABC？.md
questions: 0
chars: 1213
build_date: '2026-11-20'
images_removed: 0
---

# 什么时候用 Protocol，什么时候用 ABC？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Python　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Python/什么时候用 Protocol，什么时候用 ABC？.md

在 Python 接口设计中，什么时候应该使用 Protocol，什么时候该用传统的 ABC（抽象基类）？

这本质上是在问：“名义子类型”（Nominal） vs “结构化子类型”（Structural），你到底要哪一种约束？

## 快速看清两者的定位

> 〔原文此处为飞书内嵌组件，内容未随导出〕

## 场景化对比：一个“消息发送器”接口

## 使用 ABC：运行时强硬约束

```
from abc import ABC, abstractmethod
class MessageSender(ABC):
    @abstractmethod
    def send(self, content: str) -> bool:
        """发送消息，必须实现"""
        ...
    def batch_send(self, contents: list[str]) -> int:
        # 共享的具体方法，子类可以直接用
        return sum(self.send(c) for c in contents)
class EmailSender(MessageSender):
    def send(self, content):
        print(f"📧 发送邮件: {content}")
        return True

# ✅ EmailSender 必须继承 MessageSender，否则实例化就报错
sender = EmailSender()
sender.batch_send(["hello", "world"])
```

适用场景：

你需要一个家族式的类体系，基类提供公共逻辑（如 batch_send），并且强制所有子类遵循同一模板。框架内部开发、插件系统常用这种方式。

## 使用 Protocol：静态鸭子类型

```
from typing import Protocol
class Sendable(Protocol):
    def send(self, content: str) -> bool: ...
def notify(sender: Sendable, msg: str):
    if sender.send(msg):
        print("✅ 通知成功")

# 第三方/旧的类，完全不继承 Sendable
class SlackNotifier:
    def send(self, content: str) -> bool:
        print(f"💬 发送Slack: {content}")
        return True

# 静态检查通过，运行也正常
notify(SlackNotifier(), "部署完成")
```
