---
title: String是如何实现不可变的？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/VnFEwLcQ4ik4XAkySyvcMuJJnrc
source_path: AI Agent+后端精选八股文/语言基础篇/Java/String是如何实现不可变的？.md
questions: 1
chars: 1450
build_date: '2026-11-20'
images_removed: 0
---

# String是如何实现不可变的？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/String是如何实现不可变的？.md

String 是如何实现不可变的？底层机制是什么？

这题问的不是“不可变是什么”，而是 “Java 语言和类设计上具体用哪几道锁把 String 锁死的”。答案藏在一组精心配合的设计里。

### 四道锁：String 不可变的实现细节

### ① 类被 final 修饰 — 断子绝孙

```
public final class String { ... }
```

- 禁止任何类继承 String。
- 避免子类通过重写方法来“看似不变、实则可变”，从而破坏语义。

图标理解：🧬 无法被继承，行为不可改写。

### ② 底层数据用 private final 数组 — 内容锁死

在 JDK 8 及以前是 char[]，JDK 9 起换成了 byte[] + coder（为了紧凑存储）。

无论哪种，核心字段都是：

```
private final byte[] value;   // JDK 9+
private final byte coder;     // 编码标识
```

- private：外部无法直接访问。
- final：引用一旦初始化就不能再指向其它数组。
- 数组内容本身虽然技术上可变，但类内部从来不改它。

图标理解：🔐 数组引用固定，内容从诞生到回收永不改变。

### ③ 所有方法都不修改原数组 — 只产新对象

这是不可变最直接的体现。任何看起来“修改”字符串的方法，比如 concat、substring、replace、toLowerCase，都返回一个全新的 String 对象。

```
String s = "hello";
s.toUpperCase();          // 返回新 String "HELLO"，s 本身还是 "hello"
System.out.println(s);    // 输出 hello
```

源码中 substring 的典型实现（JDK 8）：

```
public String substring(int beginIndex, int endIndex) {
    ...
    return new String(value, beginIndex, subLen); // 新建 String
}
```

图标理解：🔁 每次“变化”都生成新实例，原值纹丝不动。

### ④ 构造时防御性复制 — 切断外部牵连

即使你通过 new String(char[]) 传入一个字符数组，String 也会拷贝一份，而不是直接持有你传入的引用。

```
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length); // 复制，而不是 this.value = value
}
```

如果没有这一步，外面的人可以修改原数组，从而影响 String 内容：

```
char[] data = {'a', 'b'};
String s = new String(data);
data[0] = 'z';
// 如果没有防御性复制，s 就变成 "zb" 了。但实际 s 仍是 "ab"
```

图标理解：🧱 连构造函数的入口都做了隔离，外部改动无法渗透。

## 题1：为什么设计成不可变？——三大收益

> 〔原文此处为飞书内嵌组件，内容未随导出〕
