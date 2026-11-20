# String为什么设计成不可变的？

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/Ia6CwdBpSiQImakkQ2qcuFW7nAf

#### 面试回答

面试官： String 为什么设计成不可变的？

你：

主要是安全、高效、并发这三点。

第一，安全。 字符串常量池里的东西，比如类名、数据库连接密码，如果谁都能改，一个地方改了，其他引用全跟着变，系统就崩了。不可变能防止这种注入和篡改。

第二，线程安全。 不可变对象天生就是共享安全的，多个线程同时读同一个字符串，不用加锁。

第三，性能优化。 不可变性使得 hash 值可以缓存，做 HashMap 的 key 时取 hash 很快。另外常量池可以共享同一个字符串对象，省内存。

简单说，如果 String 可变，上面的好处全没了，还会带来莫名其妙的 bug。

![image](../../../../assets/GzUedX3MgoAEkXxs5vYc1N5unwe/PaFJbYYEkoPsL2xnU5FcI1QUnkd.png)
