---
title: Java序列化的原理是啥
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/NP2pwBBSyimAj6kPsW7c3xAknPe
source_path: AI Agent+后端精选八股文/语言基础篇/Java/Java序列化的原理是啥.md
questions: 0
chars: 1681
build_date: '2026-11-20'
images_removed: 0
---

# Java序列化的原理是啥

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/Java序列化的原理是啥.md

序列化就是把内存里的对象图（包括对象之间的引用关系）拍平成一串字节流，反序列化就是根据这串字节“复刻”出一个一模一样的对象图。注意是对象图，不是单个对象，循环引用啥的都能正确处理。

🔹 写出去的流程（ObjectOutputStream）

当你调用 writeObject(obj) 时，它干了几件事：

1. 写魔术头：0xAC ED 加上序列化协议版本号（00 05），用于快速识别这是 Java 序列化流。
1. 写类描述符：类名、serialVersionUID、字段的数量、名字、类型…… 全记下来。
1. 递归写对象树：
- 遇到基本类型，直接写值
- 遇到引用类型，就递归调用 writeObject
- 关键：维护一个“已写入对象表”（handle 表），如果同一个对象出现第二次，只写一个 handle 编号，避免重复和死循环。
1. 处理父类：如果父类也实现了 Serializable，会自动向上序列化父类字段；如果父类没实现，子类就要负责保存/恢复父类状态（通常会调用父类无参构造）。

🔹 读回来的流程（ObjectInputStream）

1. 校验魔术头。
1. 读类描述符，加载类，比对 serialVersionUID。
1. 创建对象，但不走构造方法：这是个大坑。它通过 Unsafe 或反射直接分配内存，然后直接填充字段。所以，依赖构造函数初始化的逻辑（比如初始化某些非序列化资源）会失效。
1. 递归读字段并赋值，遇到 handle 就关联之前已读出的对象，保证对象引用关系还原。

🔹 一个直观的图示（文字版）

假设有一个“人”对象，自己引自己（最佳损友是自己）：

```
Person { name="小明"; bestFriend → 指向自己 }
```

序列化过程：

1. 开始写 Person，给个 handle #1，记录类信息。
1. 写 name 字段："小明" (String，再开新 handle #2)。
1. 写 bestFriend 字段：发现是 Person 类型，一查表，已经出现过（handle #1），于是只写一个“引用 handle #1”。

反序列化时：

1. 读到 handle #1，先创建一个空的 Person 对象。
1. 读到 name，创建 String "小明" (handle #2)。
1. 读到 bestFriend 是个 handle #1，直接把它指向刚才创建的那个 Person 对象自己。

   ✅ 对象图完美还原，不会死循环。

🔹 面试容易追问的加分点

- transient：标记的字段直接跳过，反序列化后是默认值（0 / null / false）。
- writeObject / readObject：类里定义 private void writeObject(ObjectOutputStream out) 就能劫持序列化逻辑，一般用来加密敏感字段、处理兼容。
- Externalizable：完全取代默认序列化，必须自己实现 writeExternal / readExternal，序列化性能更好，但灵活性全在自己手里，挖坑也容易。
- 对单例的破坏：反序列化会创建新对象，破坏单例。解决方法是加一个 readResolve() 方法，返回单例对象来代替新创建的。

🔹 跟其他序列化框架的区别

Java 原生序列化是重量级、平台绑定、不可读，而且安全漏洞多（Fastjson 那个问题本质上就是反序列化设计太灵活）。现在跨系统通信基本都用 Protobuf、JSON，但 Java 原生序列化还活在 RMI、某些缓存、深拷贝场景。

这机制就像给对象拍一套 CT 片子，每个细胞的连接都记下来了，片子拿回来就能直接打印个一模一样的。但正因为记了太多东西，才又慢又危险。懂它的原理，主要就是为了在它踩到你线上环境的时候，你能从报错日志里一眼揪出来是 handle 断了还是字段类型偷偷变了，而不是只会回滚版本。
