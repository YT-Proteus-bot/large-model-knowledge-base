# 为什么JDK 9中把String的char[]改成了byte[]？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/SSJ5wFVFLiTziEkYyNQc6CYtneb

“这个改动其实只有一个核心目的：省内存。Java 的 char 是 16 位、2 字节，但现实世界里绝大多数字符串都是拉丁字母、数字、英文标点，这些字符用 1 个字节就能表示。原来的 char[] 让每个字符都占 2 字节，相当于用双车位停每一辆轿车，浪费了一半空间。JDK 9 用 byte[] 加上一个编码标记，做到了‘该省就省’。

我把这个优化的细节拆开说。

🔹 先看原来的问题：char 的固定 2 字节

在 JDK 8 及以前，String 内部是这样的：

```
代码块
private final char value[];
```

每个字符都存为 char，2 字节。存一个“Hello”占 5 个 char，10 字节。但 ASCII 字符其实只需要低 8 位，高 8 位全是 0。堆里大量的字符串（类名、方法名、配置、JSON 字段）都是 ASCII 或 Latin-1 范围内的，内存浪费非常严重。据统计，标准的 Java 应用里，Latin-1 字符占字符串总量的 90% 以上。

🔹 JDK 9 的紧凑字符串：byte[] + coder

JDK 9 把实现改成了：

```
代码块
private final byte[] value;
private final byte coder;
```

- value 用来存编码后的字节。
- coder 是一个标记，表示这些字节是 LATIN1（0）还是 UTF16（1）。

创建字符串时，String 的构造函数会检查所有字符，如果都在 Latin-1（即 \u0000 ~ \u00FF）范围内，就用 Latin-1 编码，每个字符 1 字节；否则用 UTF-16，每个字符 2 字节或 4 字节（对于补充字符）。这种根据实际内容动态选择编码的方式，就是紧凑字符串。

例如：

- "Hello" → coder = LATIN1, byte[] 长度 5，只用 5 字节。
- "你好" → coder = UTF16, byte[] 长度 4（每个 char 2 字节），和原来差不多。

🔹 内存省了多少？

对大部分字符串，内存占用直接减半。String 对象本身还有 24 字节左右的对象头，但原来 char[] 数组本身占用的内存现在少了一半。在实际的堆 dump 里，字符串通常是最大的内存消耗者之一，这个优化能让整体堆内存下降 10%~20%，对大型应用和微服务意义重大。

🔹 性能影响：基本持平甚至略有提升

有些操作会稍微变复杂，比如 charAt，现在需要根据 coder 决定是直接取 byte 并转为 char，还是按 UTF-16 解码两个字节。但这种判断非常快，CPU 分支预测也能很好处理。而且因为内存占用减少，GC 压力变小，总体性能往往还有提升。

🔹 对 String 的其他内部方法影响

length() 返回的还是字符数，但内部要根据 coder 来计算：Latin-1 时 value.length 就是字符数，UTF-16 时 value.length / 2 才是字符数。indexOf、substring 等操作也都需要感知编码。但这些复杂性都被封装在 String 内部，我们写代码完全无感。
